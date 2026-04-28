# Session 48: slice 7.6c.3c — closes 7.6c.3 musl integration

**Date:** 2026-04-25
**Phase:** 7 slice 7.6c.3c (musl integration, sub-slice 3 of 3 — closes 7.6c.3)
**Branch:** master

---

## Goals

Close out slice 7.6c.3 by giving the musl-linked demos enough kernel-
side scaffolding that they actually run useful programs end-to-end.
Concretely: stdout has to reach UART, sys_exec → musl-linked-child
has to validate AUXV consumption in the wild, and `printf` has to
produce visible output.  Three new kernel syscalls (`brk`,
`writev`), one ABI tweak (magic-fd handles for stdio), and three
new EL0 demos plus their ktests.

## Scope choices

- **Magic-fd inside `sys_write` / `sys_read`, not per-process file
  table entries.**  The latter is "do stdio properly" (open a real
  handle to `/dev/console-equivalent` at process create); the
  former is "if fd 1 or 2 has no allocated handle, route to
  `sys_debug_write`".  Magic-fd is closer to what libnxlibc's libc
  does already, costs ~10 lines, and is forward-compatible (when
  proper stdio lands, the magic branches become dead code and
  go).  Critically: the magic only fires when the lookup returns
  `NX_ENOENT` — programs that legitimately allocate fd 1 or 2
  (e.g. `NX_SYS_PIPE` in a fresh handle table) get the regular
  dispatch, so no existing test breaks.
- **Add `NX_SYS_BRK` even though mallocng uses mmap.**  musl's
  default malloc is mallocng which uses `mmap(MAP_ANON)` for heap
  extension, but musl ships a `lite_malloc` / `oldmalloc` /
  `__simple_malloc` family that uses `SYS_brk`.  Programs that
  pull these in (e.g. via `__simple_calloc` in some early-init
  paths) need brk.  brk is also dirt-simple — track a per-process
  `brk_addr`, bump within a fixed heap region inside the existing
  user-window backing.  No extra kernel allocator needed; the 2 MiB
  window already has room.  mmap (the bigger-deal allocator) is a
  follow-up slice.
- **Add `NX_SYS_WRITEV` because musl's `__stdio_write` uses it.**
  Discovered the hard way: my musl-linked printf demo successfully
  reached `exit()` and propagated the right exit_code, but no
  output reached UART.  The trace: musl's `printf` formats into a
  FILE buffer, then `exit()`'s `__stdio_exit` walks open streams
  and calls `f->write(f, 0, 0)` to flush.  That's `__stdio_write`,
  which uses `SYS_writev`.  Without `NX_SYS_WRITEV` the writev
  returns `-ENOSYS`, musl marks the FILE `F_ERR`, the flush
  silently fails.  Implementation: kernel-side iovec walker that
  copy_from_user's the iovec array, then dispatches each entry
  through `sys_write` so the magic-fd / handle-table / CHANNEL /
  FILE branches all apply uniformly.
- **A new musl-linked printf demo, not a re-link of
  `posix_printf_prog.c`.**  Same rationale as slice 7.6c.3b: the
  existing libnxlibc demo uses `nxlibc_*` names that don't link
  against musl.  New `posix_musl_printf_prog.c` uses standard
  POSIX names + a distinct exit code (67) so the ktest summary
  pinpoints regressions cleanly.
- **Link `libgcc.a` for the printf demo.**  musl's `vfprintf`
  pulls in `__netf2 / __fixtfsi / __addtf3 / __multf3 / __subtf3 /
  __floatsitf` (long-double softfloat helpers used by the `%f`
  code path).  Even without `%f` in the format string, vfprintf's
  dispatcher reaches the float code at link time so the helpers
  are required.  `LIBGCC := $(shell $(CC) -print-libgcc-file-name)`
  resolves the cross-toolchain's `libgcc.a` and adds it to the
  --start-group bundle.  Demos that don't pull vfprintf (the
  posix_musl_prog write+exit demo from slice 7.6c.3b) still link
  fine without `LIBGCC`.
- **Use `exit()` not `_exit()` in the printf demo.**  Stock
  ANSI-C convention: `exit()` runs `__stdio_exit` (flushes open
  FILEs); `_exit()` skips it.  Without the flush musl's stdout
  buffer gets discarded — the program's exit_code propagates fine
  but no output reaches UART.  Caught when `[musl-ok]` showed up
  but `[musl-printf ...]` didn't.

## What Was Done

### `framework/syscall.{h,c}` — three new bits

- **`sys_write` / `sys_read` magic-fd routing** (slice 7.6c.3c
  patch to existing handlers).  After `nx_handle_lookup`, if the
  lookup returned `NX_ENOENT` AND the handle index is 0/1/2, route
  to the appropriate stdio fallback: read from fd 0 returns 0 (no
  console input source in v1); write to fd 1/2 routes to
  `sys_debug_write` (UART).  Programs with allocated fd 1/2 (e.g.
  `NX_SYS_PIPE` allocator) get the regular dispatch.

- **`NX_SYS_BRK = 17`** + `sys_brk(uint64_t requested)`.  Linux
  `brk(2)` semantics: requested == 0 returns current break;
  otherwise tries to set break, returns new break (== requested
  on success, == old break if out-of-range).  Heap region is
  `[base + NX_PROCESS_HEAP_OFFSET .. base + NX_PROCESS_HEAP_LIMIT)`
  = `[base + 1 MiB .. base + 1.5 MiB)` = 512 KiB inside the
  existing 2 MiB user-window backing.  No extra allocation; we
  track the high-water mark per process.

- **`NX_SYS_WRITEV = 18`** + `sys_writev(h, iov, count)`.  Walks
  the iovec array (cap 16 entries), copy_from_user's the
  `{iov_base, iov_len}` pairs into a kernel-side buffer, then
  dispatches each non-empty entry through `sys_write`.  Stops on
  the first short write per Linux's writev convention.  Returns
  the running total or the first error.

### `framework/process.{h,c}` — `brk_addr` field

- New `uint64_t brk_addr` in `struct nx_process`, initialized in
  `nx_process_create` to `mmu_user_window_base() +
  NX_PROCESS_HEAP_OFFSET`.  Reset in `sys_exec` after the address-
  space switch (fresh image gets a fresh heap).  Inherited
  byte-for-byte across `fork` (parent and child see the same
  brk-as-of-fork; the user-window backing was already byte-copied
  by `mmu_copy_user_backing`).
- Two macros: `NX_PROCESS_HEAP_OFFSET = 1 MiB` and
  `NX_PROCESS_HEAP_LIMIT = 1.5 MiB` (both relative to user-window
  base) define the heap region.

### `third_party/musl/arch/aarch64/syscall_arch.h`

`__nx_translate` switch gains two cases: `__NR_brk = 214 → 17`
and `__NR_writev = 66 → 18`.  Module-comment translation table
updated to match.

### `third_party/musl/src/thread/aarch64/syscall_cp.s`

Same two cases added inline: `cmp x1, #66; b.eq .Lnx_writev`
+ `cmp x1, #214; b.eq .Lnx_brk` + the corresponding label
blocks (`mov x8, #18; b .Lnx_run` and `mov x8, #17; b .Lnx_run`).

### Top-level `Makefile`

- `MUSL_LIBC` rule now does `rm -rf $(MUSL_DIR)/obj
  $(MUSL_DIR)/lib` before invoking `make -C` to flush the stale
  `.lo` cache when our patch deps trigger.  Documented gotcha
  from slice 7.6c.3a — musl's incremental rebuild only tracks
  `.c → .lo` deps, not arch-header inclusion.
- `LIBGCC := $(shell $(CC) -print-libgcc-file-name)` resolves
  the cross-toolchain's libgcc archive.
- New rules for `posix_musl_printf_prog.elf`,
  `musl_exec_parent_prog.elf` (libnxlibc parent that execs the
  musl child via the initramfs-seeded `/musl_prog` path), and
  their blob wrappers.  KTEST_C/KTEST_S list extensions; clean.
- Initramfs gains `posix_musl_prog.elf:/musl_prog` so sys_exec
  can load it via path.

### Three new EL0 demos

- **`musl_exec_parent_prog.c`** — libnxlibc-linked.  Forks +
  `nxlibc_execve("/musl_prog", argv, NULL)`.  Parent waits +
  emits `[musl-exec-ok]` if status == 57.  `argv = { "musl_prog",
  NULL }` so sys_exec's AUXV-push path runs with non-trivial
  argument shape.

- **`posix_musl_printf_prog.c`** — musl-linked.  Single `printf`
  with `%d %u %x %s %c \n`, then `exit(67)`.  Final ELF is 37 KB
  (vs 7.3 KB for the bare write+exit demo) — vfprintf pulls in
  the locale state + softfloat helpers + stdio buffer machinery.
  Live ktest log gets the formatted output:
  `[musl-printf d=-1 u=4294967295 x=deadbeef s=ok c=Q]`.

- **`posix_musl_prog.c` ktest update** — assertion now requires
  `nx_syscall_debug_write_calls() >= 1` because the magic-fd
  path makes `[musl-ok]` actually reach UART.

### Three new ktests

- `ktest_musl_exec.c` — parent forks + execs the musl child;
  asserts `[musl-exec-parent][musl-ok][musl-exec-ok]` triple +
  parent exit_code 0 + ≥3 debug_writes.
- `ktest_posix_musl_printf.c` — drops to EL0 with `sp_el0 =
  top - 64` (same envp/auxv-walk slack as the non-printf demo);
  asserts exit_code 67 + ≥1 debug_write.
- `ktest_posix_musl.c` (updated) — adds the missing
  debug_write count assertion.

### One host test update

- `file_syscall_test.c` `sys_write_on_stale_handle_returns_enoent`:
  changed handle from 1 to 3.  The slice-7.6c.3c magic-fd path
  intercepts ENOENT-on-fd-1/2 and routes to debug_write — still
  ENOENT on any other unallocated handle.  Added new test
  `sys_write_to_unopened_stdout_routes_to_debug_write` that
  documents the new contract.

## Key Findings

- **musl's stdout buffering swallows printf without `exit()`.**
  My first cut of the printf demo used `_exit(67)` — exit_code
  propagated fine but no output reached UART.  Tracing: `printf`
  buffers into `FILE.buf`, no flush happens (line buffering is
  off because `__stdout_write`'s ioctl-isatty probe failed and
  set `f->lbf = -1`); flush only fires from `__stdio_exit` which
  is called by `exit()` not `_exit()`.  Real C programs almost
  always use `exit()` or just `return` from main (which calls
  exit), so this is a v1-test footgun, not a long-term issue.
- **`writev` is not optional.**  Even after the magic-fd routing
  worked for `write()`-shaped demos (posix_musl_prog), the printf
  demo silently failed because musl's `__stdio_write` is built
  around writev exclusively (no `write` fallback).  Pretty much
  any musl program that uses stdio at all needs writev support.
  Lesson noted for future syscall-coverage gaps: trace the
  reachable syscall surface end-to-end, not just per-feature.
- **`vfprintf` pulls in libgcc softfloat even with no `%f`.**
  `format-string-controlled` dispatch means the `%f` code path
  is reachable from the linker's perspective; libgcc's TF-arith
  helpers are required even when no actual float-formatting will
  fire at runtime.  `LIBGCC := $(shell ... -print-libgcc-file-name)`
  in our Makefile is the standard way; --start-group/--end-group
  resolves the libc.a ↔ libgcc.a cycle (musl's vfprintf calls
  libgcc helpers; libgcc's `_Unwind_*` calls musl's symbols if
  exception unwinding were enabled — it's not, but the linker
  resolves the cycle iteratively anyway).
- **`make test` rebuild gap on header changes.**  My `.gitignore`
  / makefile pattern is `%.o: %.c` with no header dep tracking,
  so changes to `framework/syscall.h` (added enum values) didn't
  trigger ktest_syscall.o recompile.  ktest_syscall stale-checked
  `tf.x[8] = NX_SYSCALL_COUNT` which now had a different value
  → routed to `sys_brk(0)` which returned 0 instead of NX_EINVAL.
  Workaround: nuke `*.o` files (excluding third_party) and
  rebuild.  Real fix is gcc's `-MMD` flag generating `.d` files
  that make includes — a future infrastructure-polish slice.
- **The musl-clean cascade in $(MUSL_LIBC) rule paid off.**  Slice
  7.6c.3a documented the gotcha that musl's `.lo` cache doesn't
  track arch-header inclusion.  Adding `rm -rf $(MUSL_DIR)/obj
  $(MUSL_DIR)/lib` to the rule means a touch to syscall_arch.h
  triggers a full musl rebuild automatically.  Cost: ~30 s on
  patch-touch, zero on everything else.  Three patches landed
  in this session (the brk + writev cases) and the rule fired
  correctly each time.

## Decisions Made

- **`NX_SYS_BRK = 17`, `NX_SYS_WRITEV = 18`.**  Sequential slots,
  same naming pattern as the rest of `enum nx_syscall_number`.
  No reserved gaps for future syscalls; we'll add them in order
  as they're needed.
- **Heap is 512 KiB.**  Plenty for v1 demos (musl mallocng
  chunked allocations rarely exceed a few KiB; busybox sh /
  printf handful of bytes per run).  When the limit becomes
  an issue (likely slice 7.6d busybox), bump it or shift to a
  per-process growable mmap-backed heap.
- **`NX_IOV_MAX_LOCAL = 16`.**  Linux's IOV_MAX is 1024.  v1's
  stdio buffer-flush emits 1 or 2 iovecs per writev call; 16
  is generous.  Bumping is one constant + a compile away.
- **Magic-fd is `lookup-then-fallback`, not `intercept-first`.**
  The first cut intercepted h==1/2 unconditionally → broke the
  pipe-roundtrip test (`posix_pipe_prog` allocates a pipe and
  uses fds 1+2 as channels).  Lookup-then-fallback preserves
  the regular dispatch for legitimate handles.
- **No `mmap` syscall this session.**  Mallocng wants
  `mmap(MAP_ANON)`, not brk; programs that depend on dynamic
  malloc beyond the printf demo's stack-buffered output will
  hit -ENOSYS.  Adding mmap was originally part of 7.6c.3c's
  scope but the printf demo turned out to need writev (not
  mmap) for its actual code path.  Mmap lands when the first
  malloc-using demo forces it — likely slice 7.6d's busybox.
- **Three new kernel tests, total now 81.**  Each tests a
  distinct concern: `posix_musl_prog` (drop_to_el0 musl,
  no-AUXV path), `musl_exec_parent_forks_and_execs_musl_child`
  (sys_exec → AUXV consumption), `posix_musl_printf` (full
  vfprintf + writev + exit-flush stack).
- **Rolling Session 43 to archive.**  HANDOFF.md's session list
  caps at 5; after inserting Session 48, Session 43 (slice
  7.6c.1 libnxlibc.a) moves to the archive.

## Status at End of Session

- `make test` → **407/407 pass (51 python + 275 host + 81
  kernel), 0 leaks, 0 errors, exit 0**.  +3 kernel tests
  (musl_exec, posix_musl_printf, sys_write_to_unopened_stdout
  on the host side; the existing posix_musl ktest got the
  marker assertion added).
- Slice 7.6c.3 closed.  musl is fully usable for v1 demos:
  stdio works, fork/exec works, printf with all the
  non-floating-point conversions works.
- Initramfs gains `/musl_prog` (37 KB? no — `posix_musl_prog`
  is the small 7.3 KB one; the printf demo is loaded via
  drop_to_el0 from kernel-test.bin, not initramfs).  Live
  ktest log gains the markers `[musl-exec-parent]`,
  `[musl-exec-ok]`, `[musl-printf d=-1 ...]`.

## Next Steps

**Slice 7.6d — busybox cross-compile + boot to shell.**  Now that
musl works, build busybox against it and boot to a shell prompt.
Concretely:

1. `tools/build-busybox.sh` — fetches busybox source, applies a
   minimal config (no networking, no modutils, no login — just
   core CLI: sh, ls, echo, cat, ps, mkdir), cross-compiles a
   static aarch64 binary against our patched musl.
2. Pack busybox into the initramfs at `/bin/busybox` plus
   symlinks for each applet.  Bump `RAMFS_FILE_CAP` if needed
   (busybox is ~700 KB-1 MB).
3. `/init` becomes busybox `sh`.  Exit: `make run` lands on a
   shell prompt over the serial console.

**Likely follow-ups during 7.6d:**

- **Add `mmap`.**  busybox's argument parsing or shell expansion
  will trigger mallocng allocations beyond the printf-stack-buffer
  case.  Slice 7.6c.3c's brk + writev get us this far; mmap is
  what unblocks "real programs".
- **Patch `clone.s / vfork.s / __unmapself.s / restore.s`.**
  Predicted in slice 7.6c.3a; busybox's shell forks subshells
  for pipes (`echo foo | cat`) — that hits `__clone` and
  potentially `vfork` depending on musl version.  Slice 7.6c.3
  left them stock because no demo pulled them in; busybox will.
- **Bump `NX_PROCESS_TABLE_CAPACITY` again or land proper
  cross-test reap on `wait()`.**  Busybox under heavy
  smoke-testing will create dozens of short-lived processes;
  the current 32-cap will run out.

**Slice 7.7 — interactive smoke tests.**  After busybox boots,
scripted harness drives `ls /`, `echo hello | cat`, `ps`,
`mkdir /tmp` against the shell over the serial console; captured
output is compared against expected.  Closes Phase 7.

**Deferred (unchanged):**

- HANDLE_FILE fork inheritance — needed when busybox does
  `cat file | grep pattern` (output redirection across pipe-
  forked-child).
- FP context save/restore on schedule — the busybox shell
  forking pipe-children that all use NEON-`memset` will be
  the first ktest with cross-task FP-state coherence.
- Real /dev/console as a real handle (vs the magic-fd fallback).

---

**Files Changed:**
- `sources/nonux/framework/syscall.h` — `NX_SYS_BRK = 17`, `NX_SYS_WRITEV = 18` enum entries + doc comments
- `sources/nonux/framework/syscall.c` — `sys_brk` + `sys_writev` implementations; `sys_write` / `sys_read` magic-fd routing; dispatch table extensions
- `sources/nonux/framework/process.h` — `brk_addr` field + `NX_PROCESS_HEAP_{OFFSET,LIMIT}` macros
- `sources/nonux/framework/process.c` — initialize brk_addr in create + propagate across fork
- `sources/nonux/Makefile` — `LIBGCC` resolution; new build rules for `musl_exec_parent_prog.elf` + `posix_musl_printf_prog.elf`; KTEST_S/C list extensions; initramfs gains `/musl_prog`; `MUSL_LIBC` rule wipes `obj/+lib/` on dep change
- `sources/nonux/third_party/musl/arch/aarch64/syscall_arch.h` — translation table gains `__NR_brk → 17` + `__NR_writev → 18`
- `sources/nonux/third_party/musl/src/thread/aarch64/syscall_cp.s` — asm cancellation-point translation gains the same two cases
- `sources/nonux/test/kernel/posix_musl_prog.c` — (no change, but the matching ktest now asserts marker)
- `sources/nonux/test/kernel/ktest_posix_musl.c` — assertion `nx_syscall_debug_write_calls() >= 1` added
- `sources/nonux/test/kernel/musl_exec_parent_prog.c` — new (libnxlibc parent that execs `/musl_prog`)
- `sources/nonux/test/kernel/musl_exec_parent_prog_blob.S` — new
- `sources/nonux/test/kernel/ktest_musl_exec.c` — new
- `sources/nonux/test/kernel/posix_musl_printf_prog.c` — new (musl-linked printf demo)
- `sources/nonux/test/kernel/posix_musl_printf_prog_blob.S` — new
- `sources/nonux/test/kernel/ktest_posix_musl_printf.c` — new
- `sources/nonux/test/host/file_syscall_test.c` — stale-handle test uses h=3 (h=1/2 are now magic); new test for the magic-fd routing contract
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.6c.3c marked complete with full body; closes 7.6c.3
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 43 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
