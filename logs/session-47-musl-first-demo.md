# Session 47: slice 7.6c.3b — first EL0 demo against musl

**Date:** 2026-04-25
**Phase:** 7 slice 7.6c.3b (musl integration, sub-slice 2 of 3)
**Branch:** master

---

## Goals

Get a freshly-linked EL0 C program — built against musl's `libc.a +
crt1.o + crti.o + crtn.o`, NOT libnxlibc — to run end-to-end on
nonux: `_start` -> `__libc_start_main` -> `__init_libc` ->
`__init_tls` -> `__init_ssp` -> stage2 -> `main` -> `write(1, ...)`
-> `_exit(rv)`, with the exit code observable in the process
table.  Push AUXV onto the user stack in `sys_exec` so musl's libc
init has the entropy + page-size lookup it expects.

## Scope choices

- **Drop_to_el0 path for the v1 demo, not sys_exec.**  Slice 7.6c.4
  builds the System V argv layout in `sys_exec` but not in
  `drop_to_el0` — the latter just sets sp to top-of-window with the
  backing zero-initialized.  musl's init walks "argc/argv-NULL/
  envp-NULL/auxv-NULL" dereferencing each pointer, and zero-init
  gives it natural terminators (each pointer reads as 0 = end-of-
  list).  drop_to_el0 means we don't need a parent program to
  exec the musl child; the test scaffold stays identical to
  every other slice-7 ktest.  Sys_exec → musl validation is a
  7.6c.3c follow-up (closes the AUXV-consumption gap when malloc
  enters and AT_PAGESZ becomes load-bearing).
- **AUXV is pushed but not consumed yet.**  `sys_exec` now writes
  three pairs (AT_PAGESZ, AT_RANDOM, AT_NULL) between envp's NULL
  terminator and the argv strings, plus a 16-byte pad referenced
  by AT_RANDOM.  But our musl demo runs via `drop_to_el0` (no
  sys_exec), so musl walks an empty auxv at runtime — AT_RANDOM is
  NULL, falls back to musl's hash-based canary; AT_PAGESZ is 0,
  doesn't matter for a non-malloc demo.  The AUXV emit-side
  correctness gets indirectly verified by existing sys_exec ktests
  (slice 7.4c, 7.4d, 7.6c.4) still passing — the layout extension
  doesn't shift any of the slots they observe.
- **A new demo, not a re-link of `posix_libc_prog.c`.**  The
  existing `posix_libc_prog.c` calls `nxlibc_write` /
  `nxlibc_strlen` / `nxlibc_exit` — names defined only in
  `libnxlibc.a`.  Re-linking against musl would force a source
  change to use POSIX-standard names (`write` / `strlen` / `_exit`).
  Cleaner: a new `posix_musl_prog.c` that uses POSIX names from
  the start, with a distinct exit code (57) so the ktest harness
  can tell the two programs apart.  Both sources stay in the tree,
  side-by-side coverage.
- **Enable EL0 access to FP/SIMD via `CPACR_EL1.FPEN = 0b11`.**
  musl's NEON-using `memset` (the first "real" libc call from
  `__init_libc`) traps the moment the kernel boots a musl-linked
  EL0 program — `dup v0.16b, w1` is a SIMD instruction and our
  pre-7.6c.3b boot left FPEN at 0 (trap from any EL).  No FP-
  context save/restore on schedule yet — single-EL0-task-at-a-time
  pattern in our ktests means the FP register file stays coherent
  for the running task.  Cross-task FP persistence (long-running
  busybox shell parented over forked children that also use FP) is
  a future "FP context save on schedule" slice.

## What Was Done

### `framework/syscall.c` — AUXV push in `sys_exec`

- `sys_exec`'s post-flip layout build extends past the existing
  envp NULL with three AUXV pairs:

  ```
  u[3 + argc] = 6;            /* AT_PAGESZ */
  u[4 + argc] = 4096;
  u[5 + argc] = 25;           /* AT_RANDOM */
  u[6 + argc] = window_base + at_random_offset;
  u[7 + argc] = 0;            /* AT_NULL */
  u[8 + argc] = 0;
  ```

- 16-byte AT_RANDOM pad placed below the strings, 8-aligned via
  `(str_offset - 16) & ~7`.  Pad bytes filled with
  `(seed >> (i * 4)) ^ (i * 0x37)` where `seed = nx_monotonic_raw()`
  (CNTPCT_EL0).  Pseudo-random — not cryptographic; a real entropy
  source is a Phase-9 follow-up.

- `fixed_size = 8 * (3 + argc)` becomes `8 * (9 + argc)`: the +6
  accounts for 3 AUXV pairs × 2 longs per pair.  `sp_offset` is
  computed from the AT_RANDOM pad's offset, not from `str_offset`,
  so sp lands below the random pad which lands below the strings.

- New include: `core/cpu/monotonic.h` for `nx_monotonic_raw`.

- Layout doc-comment updated: the stack diagram now shows the
  AUXV pairs sitting between `envp[0] = NULL` and the strings.

### `core/mmu/mmu.c` — enable EL0 FP/SIMD

- Right after the SCTLR_EL1 write that turns on MMU + caches,
  CPACR_EL1.FPEN is set to 0b11 (no trap on FP/SIMD from any EL):

  ```
  uint64_t cpacr;
  asm volatile ("mrs %0, cpacr_el1" : "=r"(cpacr));
  cpacr |= (3UL << 20);
  asm volatile ("msr cpacr_el1, %0" :: "r"(cpacr) : "memory");
  asm volatile ("isb");
  ```

  Comment notes: the kernel itself is `-mgeneral-regs-only` so it
  never touches FP/SIMD; cross-task FP persistence is deferred.

### `Makefile` — extend musl build to crt objects + new demo rules

- `MUSL_CRT1 / MUSL_CRTI / MUSL_CRTN` vars (alongside the existing
  `MUSL_LIBC`).  All four targets share a single rule that calls
  `make -C $(MUSL_DIR) lib/libc.a lib/crt1.o lib/crti.o lib/crtn.o`
  (musl produces them in one make invocation).
- `musl-libc` phony alias now depends on all four artefacts.
- New build rules for `test/kernel/posix_musl_prog.elf`:
  ```
  $(LD) -n -T test/kernel/init_prog.ld -o $@ \
      $(MUSL_CRT1) $(MUSL_CRTI) \
      test/kernel/posix_musl_prog.o \
      --start-group $(MUSL_LIBC) --end-group \
      $(MUSL_CRTN)
  ```
  `--start-group/--end-group` handles the libc-internal undefined
  references that come up across `_init_libc` / `__init_tls` /
  `__init_tp` (each pulled in separately, but with circular
  dependencies between them — single-pass linking misses some).
- `posix_musl_prog_blob.S` (`.incbin` wrapper) added to `KTEST_S`;
  `ktest_posix_musl.c` added to `KTEST_C`.
- Clean rule extended to remove `posix_musl_prog.elf`.

### `test/kernel/posix_musl_prog.c`

Minimal demo, 8 lines of body:

```c
typedef long          ssize_t;
typedef unsigned long size_t;
extern ssize_t write(int fd, const void *buf, size_t n);
extern void    _exit(int status) __attribute__((noreturn));

int main(int argc, char **argv, char **envp)
{
    (void)argc; (void)argv; (void)envp;
    write(1, "[musl-ok]", 9);
    _exit(57);
}
```

No `<unistd.h>` — pulling that in would drag the rest of musl's
header tree which is more than this demo needs.  Inline forward
declarations match musl's signatures.

### `test/kernel/posix_musl_prog_blob.S`

Standard `.incbin` wrapper — same shape as every other slice-7
blob (`init_prog_blob.S`, `posix_libc_prog_blob.S`, etc.).
Symbols `__posix_musl_prog_blob_{start,end}` for the ktest to
slice the bytes out of `kernel-test.bin`'s `.rodata`.

### `test/kernel/ktest_posix_musl.c`

Same scaffold as `ktest_posix_libc.c`:

1. Creates a host process via `nx_process_create("musl-host")`.
2. Loads the embedded blob via `nx_elf_load_into_process`.
3. Spawns a kthread that calls `drop_to_el0(entry, sp)`.
4. Polls the process table until state == EXITED + exit_code == 57.

Two important divergences from the libnxlibc test:

- **`sp_el0 = top - 64`, not `top - 16`.**  musl's `__init_libc`
  walks envp/auxv past sp; `top - 16` lets envp[0] (read at sp+16)
  fall on the boundary -> permission fault.  64 bytes of slack
  give crt1 enough zero-padded headroom to find natural
  terminators.  Sys_exec-launched programs get a real layout
  built by the kernel and don't need this slack.
- **No marker check.**  musl's `write(1, ...)` translates to
  `__NR_write (64)` -> `NX_SYS_WRITE (8)`, but fd 1 in our v1
  isn't a valid handle (no stdio plumbing yet) — sys_write
  rejects with NX_EBADF, the `[musl-ok]` marker doesn't reach
  UART.  The program ignores write's return and exits cleanly,
  so exit_code == 57 is the load-bearing signal.  Slice 7.6c.3c
  will wire stdio so the marker shows up in the live ktest log.

### Commentary fields updated

- `posix_musl_prog.c` — file header documents what flow it
  validates (8 stages from `_start` to `_exit`).
- `ktest_posix_musl.c` — long header explains why drop_to_el0
  + why no marker check + the sys_exec-musl deferral.

## Key Findings

- **Two link-line surprises after my first attempt.**  First, the
  link tried (and succeeded!) without `--start-group` even though
  musl's libc.a has internal cycles (`__init_libc` calls into
  `__init_tls` calls into `__init_tp` calls into `__set_thread_area`
  + back-references).  GNU ld's first pass picks them up because
  they're all needed by the unconditional path from `_start` →
  `_start_c` → `__libc_start_main`.  But to be safe against
  future demos with more selective coverage, I wrapped libc.a in
  `--start-group/--end-group` — costs nothing on a clean link,
  protects against partial-coverage cycles.
- **NEON memset traps EL0 by default.**  `dup v0.16b, w1` at
  `__init_libc`'s call to `memset(local_aux, 0, 304)` — first
  thing that fires in libc init.  The `mov w0, #-38; svc 0`-shaped
  failure mode I expected from missing syscalls didn't apply
  here: the hardware trap fires before the syscall layer is even
  reached.  ESR=0x1FE00000, EC=7 ("trap from FP/SIMD without
  permission"), the giveaway.  Single-line fix: bump
  CPACR_EL1.FPEN to 0b11 in `mmu_init` post-SCTLR.
- **drop_to_el0's sp_el0 must leave headroom for envp/auxv
  terminator reads.**  Stock `top - 16` works fine for hand-coded
  asm EL0 programs (they don't read past sp), but musl's
  `__init_libc` walks "argc/argv-NULL/envp-NULL/auxv-NULL"
  dereferencing each — envp[0] at sp+16 lands exactly at the
  user-window boundary.  Caught by ESR=0x9200000E
  (data-abort-from-lower-EL, permission fault, level 2) +
  FAR=0x48200000 (= window_top, exactly).  Fix is local to the
  ktest: 64 bytes of slack instead of 16.  A future "drop_to_el0
  pre-populates argc=0 + terminators" extension would let any
  crt1 land cleanly without per-test sp tuning, but slice
  7.6c.3b's scope doesn't justify it.
- **musl's `--start-group` link-line dance happens at runtime,
  not link time.**  When `--start-group/--end-group` wraps
  libc.a, ld iteratively pulls in objects until no new
  unresolved symbols remain.  My demo + crt1 reach
  `__libc_start_main`; that pulls in `__init_libc` /
  `__init_tls` / `__init_tp` / `_Exit` / `write`; those pull
  in `memset` / `memcpy` / `__set_thread_area` /
  `__syscall_cp_asm` / `__syscall_cp_c` / `__syscall_ret` /
  `__stack_chk_fail` (canary check, never fires); each of those
  pulls in their dependencies; cycles converge in two passes.
  Final ELF: 7.3 KB (vs libnxlibc-linked posix_libc_prog at
  2.3 KB) — musl pulls in much more even for a minimal demo
  (TLS bootstrap, locale state, signal mask init, etc.).
- **musl's libc.a doesn't need the four svc-direct asm files
  for our v1 demo.**  Predicted in slice 7.6c.3a's session log
  that we might have to patch `clone.s / vfork.s / __unmapself.s
  / restore.s` if the linker pulled them in.  It didn't — the
  demo doesn't call fork/vfork/pthread_create/sigaction, so the
  linker drops those objects.  The four asm files stay stock
  for now; slice 7.6c.3c's malloc demo might pull
  `__unmapself.s` (only on pthread teardown, which static demos
  don't do).  More likely to surface when we hit busybox.

## Decisions Made

- **`--start-group` around libc.a in the link line.**  Defensive
  against future cyclic references; current build doesn't need
  it but it's the standard musl link incantation.
- **Sub-slice scope: drop_to_el0 demo only, no sys_exec → musl
  test.**  Adding the latter requires either a libnxlibc parent
  program that execs the musl child (parent still uses
  libnxlibc since it's hand-written-against-libnxlibc) plus
  ramfs-seeding the musl child.  That's a non-trivial addition
  for a "first demo" slice.  Slice 7.6c.3c will add the test
  naturally because malloc requires AT_PAGESZ != 0, which only
  happens via the sys_exec AUXV-push path.
- **Distinct exit code 57.**  Programs sequence: `posix_main_prog`
  47, `posix_libc_prog` 53, `posix_musl_prog` 57.  The values
  differ enough that a regression in any one is immediately
  pinpointable from the ktest summary.
- **No FP context save/restore.**  Slice scope is "first demo",
  not "FP-correct multitasking".  The single-EL0-task-at-a-time
  test convention doesn't exercise FP-state cross-corruption.
  A future slice (likely after pthread bring-up) adds proper FP
  save/restore on context switch.
- **Rolling Session 42 to archive.**  HANDOFF.md's session list
  caps at 5; after inserting Session 47, Session 42 (slice
  7.6c.0 EL0 C-runtime bootstrap) moves to the archive.

## Status at End of Session

- `make test` → **404/404 pass (51 python + 274 host + 79
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test
  (ktest_posix_musl).
- `make musl-libc` builds `libc.a + crt1.o + crti.o + crtn.o`
  (~2.75 MB libc + 4 KB total crt set).
- `posix_musl_prog.elf` is 7.3 KB linked, containing the full
  __libc_start_main + __init_libc + __init_tls + memset/memcpy/
  exit code path.  ELF entry 0x48000000 (user-window base) per
  init_prog.ld.
- Live ktest log gains `posix_musl_prog_runs_main_through_init_libc_and_exits_57`
  (no embedded marker — see ktest header comment for why).

## Next Steps

**Slice 7.6c.3c — malloc + remaining demos, close 7.6c.3.**

1. **Add `brk` syscall** (or anonymous `mmap`).  musl's `mallocng`
   extends the heap via `brk` first, falling back to `mmap` if
   brk returns -1.  brk is the simpler shape: one new syscall +
   a per-process `brk_addr` field tracked alongside `ttbr0_root`,
   sized within the existing 2 MiB user window (musl's malloc
   is incremental; v1 demos won't exhaust).  Updates
   `__nx_translate` to map `__NR_brk = 214` to a new
   `NX_SYS_BRK = 17`.
2. **Add a sys_exec → musl-linked-child test** to validate the
   AUXV-push end-to-end.  A small parent program (libnxlibc-
   linked, like `argv_parent_prog`) execs the musl child via
   `nx_posix_execve("/musl_prog", argv, NULL)`; child reads
   argv via crt1's stack walk; AUXV gets consumed by
   `__libc_start_main`.  Exit code propagates through wait().
3. **Wire stdio** so `write(1, ...)` reaches UART (or a /dev/tty1
   handle).  Today's musl demo's `[musl-ok]` marker silently
   gets EBADF'd.  Either: (a) pre-allocate fd 1/2 in every
   process to a debug-write handle, or (b) make
   `NX_SYS_WRITE` magic-fd-handle fd=1/2 → `NX_SYS_DEBUG_WRITE`
   like libnxlibc does at the libc layer.  (b) is closer to
   how Linux works (kernel knows about stdio via the open file
   table); (a) is closer to what we have now.
4. **Re-link `posix_printf_prog`** against musl — pulls in
   malloc + locale init.  Live ktest log gains musl-formatted
   printf markers (different from libnxlibc-formatted ones,
   useful for spotting format-string regressions).
5. **Optionally re-link `argv_parent_prog`.**  Currently uses
   libnxlibc only for `_exit + write`; might "just work" against
   musl as long as the kernel-side ENOSYS-y syscalls aren't
   hit during init.

**Deferred (unchanged):**

- Proper cross-test task reap on `wait()`.
- HANDLE_FILE fork inheritance.
- FP context save/restore on schedule (lands when first
  multi-EL0-FP-using-task ktest forces it).

---

**Files Changed:**
- `sources/nonux/framework/syscall.c` — sys_exec pushes AUXV (3 pairs + 16-byte pad); +include core/cpu/monotonic.h
- `sources/nonux/core/mmu/mmu.c` — enable CPACR_EL1.FPEN at boot so EL0 can use FP/SIMD
- `sources/nonux/Makefile` — `MUSL_CRT{1,I,N}` vars; build rule for `posix_musl_prog.elf`; KTEST_S/KTEST_C list extensions; clean
- `sources/nonux/test/kernel/posix_musl_prog.c` — new (minimal POSIX-named demo, exit code 57)
- `sources/nonux/test/kernel/posix_musl_prog_blob.S` — new (`.incbin` wrapper)
- `sources/nonux/test/kernel/ktest_posix_musl.c` — new (drop_to_el0 + check exit_code == 57)
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.6c.3b updated to "complete" with full body; 7.6c.3c plan refined
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 42 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
