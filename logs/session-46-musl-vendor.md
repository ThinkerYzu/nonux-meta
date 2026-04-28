# Session 46: slice 7.6c.3a — vendor musl + patch syscall layer

**Date:** 2026-04-25
**Phase:** 7 slice 7.6c.3a (musl integration, sub-slice 1 of 3)
**Branch:** master

---

## Goals

Slice 7.6c.3 ("vendor + cross-build musl, swap libnxlibc for
libc.a on the demo link lines") is a genuinely multi-day project:
musl assumes a Linux-shaped kernel — Linux syscall numbers, AUXV
on the stack at entry, brk/mmap for malloc, exit_group, wait4,
rt_sigaction, set_tid_address, openat — and we have ~13 syscalls.
Splitting the slice into sub-slices keeps each session
single-shaped:

- **7.6c.3a (this session):** vendor the musl source tree, stand
  up the build (`make musl-libc` produces `libc.a`), patch the
  one chokepoint where Linux syscall numbers turn into `svc 0`
  so musl emits *our* `NX_SYS_*` numbers instead.  No EL0
  program links against the patched libc.a yet — that lands in
  7.6c.3b.
- **7.6c.3b:** push AUXV onto the user stack in `sys_exec`
  (musl's `__libc_start_main` reads `AT_RANDOM` for the stack
  canary + `AT_PAGESZ` for malloc), build musl's `crt1.o` (or
  use libnxlibc's existing `crt0.S` if the layout matches),
  re-link `posix_libc_prog` against `libc.a + crt1.o`.  First
  demo working.
- **7.6c.3c:** add `brk` (or anonymous `mmap`) so musl's
  malloc has a heap.  Re-link `posix_printf_prog` (printf
  with width/precision/floats from musl) + `argv_parent_prog`
  + close the slice.

This session's exit criterion: `make musl-libc` → `libc.a`
produced clean, `make test` → 403/403 still passing, no demos
linked against musl yet.

## Scope choices

- **Vendor musl as a tree, not a submodule.**  `third_party/musl/`
  is a verbatim extraction of `musl-1.2.5.tar.gz` (sha256
  `a9a118bb...`, the official 2024-03-01 release).  No
  submodule + no fetch step keeps `make test` offline-clean
  (the project's been air-gapped-friendly since slice 7.6b).
  Cost: 14 MB / 2697 files added to the repo.  Future musl
  upgrades are a re-extract + re-apply-the-two-patches; the
  patch surface is small enough that this is fine.
- **Patch musl's syscall site, don't rewrite musl's libc.**
  musl funnels every syscall through one of two paths:
  C-level `__syscallN(n, ...)` in `arch/aarch64/syscall_arch.h`,
  or asm-level `__syscall_cp_asm` in
  `src/thread/aarch64/syscall_cp.s` (the cancellation-point
  variant used for I/O calls so pthread_cancel can interrupt
  them).  Both files end with `svc 0` after putting `n` in `x8`.
  Patch both to translate `n` (Linux number) → our `NX_SYS_*`
  number before `svc`, fail closed with `-ENOSYS` for unmapped
  numbers.
- **`__nx_translate(long n)` returns `-1` on miss; caller
  emits `-ENOSYS = -38`.**  Two-step pattern keeps the
  translation table compact (one switch, one tested return
  value) while letting the asm-level patch share the same
  numbers without textual duplication.  When `n` is a
  compile-time constant (every `__syscall(__NR_foo, ...)`
  site is), `-O2` const-folds the switch to a single `mov` —
  no runtime overhead vs the stock `svc` emit.
- **Translate only "args already match" syscalls in
  this sub-slice.**  Nine Linux syscalls map to ours
  with no argument shuffle:

  | Linux                       | nonux               |
  |-----------------------------|---------------------|
  | `__NR_close       (57)`     | `NX_SYS_HANDLE_CLOSE (2)` |
  | `__NR_lseek       (62)`     | `NX_SYS_SEEK         (9)` |
  | `__NR_read        (63)`     | `NX_SYS_READ         (7)` |
  | `__NR_write       (64)`     | `NX_SYS_WRITE        (8)` |
  | `__NR_exit        (93)`     | `NX_SYS_EXIT        (11)` |
  | `__NR_exit_group  (94)`     | `NX_SYS_EXIT        (11)` (alias) |
  | `__NR_kill       (129)`     | `NX_SYS_SIGNAL      (16)` |
  | `__NR_execve     (221)`     | `NX_SYS_EXEC        (14)` |
  | `__NR_wait4      (260)`     | `NX_SYS_WAIT        (13)` (drops options + rusage) |

  Linux's `wait4(pid, status, options, rusage)` puts pid in
  x0 + status in x1; our `NX_SYS_WAIT(pid, status)` reads
  the same two registers.  Extra args (options, rusage) sit
  in x2/x3 and are ignored.  Same trick for `execve` —
  `(path, argv, envp)` → `(x0, x1, x2)` matches NX_SYS_EXEC
  exactly after slice 7.6c.4.
- **Don't try to translate openat / pipe2 / clone yet.**  These
  need argument shuffling (openat drops `dirfd`; pipe2 drops
  `flags`; clone has the SIGCHLD-only fork shape vs the full
  thread-clone surface).  -ENOSYS is fine for now — the
  programs we'll link in 7.6c.3b/c don't exercise those paths.
- **Don't touch `clone.s / vfork.s / __unmapself.s /
  restore.s` yet.**  These four asm files also emit `svc`
  with Linux numbers, but they're only pulled in by linker
  reachability — pthread (clone, __unmapself), vfork (rare),
  signal-handler return (restore_rt).  None of our v1
  demos hit these, so the linker drops them.  Slice 7.6c.3b
  will re-evaluate when actual demos pull objects.

## What Was Done

### `third_party/musl/` — vendored 1.2.5 source tree

- Extracted `musl-1.2.5.tar.gz` (sha256
  `a9a118bbe84d8764da0ea0d28b3ab3fae8477fc7e4085d90102b8596fc7c75e4`)
  to `third_party/musl/`.  Stock upstream sans the two patches below.
- Upstream's own `.gitignore` covers the build outputs (`obj/`,
  `lib/`, `*.lo`, `*.a`, `config.mak`) so they stay out of git
  automatically.

### `third_party/musl/arch/aarch64/syscall_arch.h` — patched

The C-level syscall dispatcher.  Stock musl puts `n` in `x8`
and emits `svc 0`.  Our patch:

1. Adds `static inline long __nx_translate(long n)` — a switch
   over the nine mapped Linux numbers, default `-1`.
2. Each `__syscallN(...)` calls `__nx_translate`, returns
   `-38` (-ENOSYS) on miss, otherwise programs `x8` with the
   nonux number and falls through to the same `__asm_syscall`
   macro.

Spot-checked codegen confirms const-folding works:
- `_Exit(ec)` → `mov x8, #11; svc #0` (loop).
- `execve` → `mov x8, #14; svc #0`.
- `getpid` → `mov w0, #-38; ret` (no syscall — Linux 172 not
  in our table, the unreachable svc is dead-code-eliminated).
- `open_by_handle_at` → `mov x0, #-38; bl __syscall_ret`.

### `third_party/musl/src/thread/aarch64/syscall_cp.s` — patched

The asm-level cancellation-point dispatcher.  Stock version
takes the syscall nr in `x1`, moves it into `x8` directly, and
emits `svc`.  Our patch interposes the same translation table
inline in asm:

```
cmp x1, #57;   b.eq .Lnx_close
cmp x1, #62;   b.eq .Lnx_lseek
... (9 cases) ...
movn x0, #37        // -ENOSYS (-38) for unmapped
ret
.Lnx_close: mov x8, #2;  b .Lnx_run
... (8 mov's, the last falls through) ...
.Lnx_run: mov x0, x2; ... ; svc 0
```

`__cp_begin / __cp_end / __cp_cancel` symbols are still emitted
— `pthread_cancel.c` references them as a PC range (so the
cancel handler can detect "the syscall instruction itself was
interrupted by a cancel signal") + as a branch target.  We're
not running pthread in v1, but the static linker still resolves
the references.

### `Makefile` — `musl-libc` + `musl-clean` targets

- `MUSL_DIR := third_party/musl`, `MUSL_LIBC := $(MUSL_DIR)/lib/libc.a`,
  `MUSL_STAMP := $(MUSL_DIR)/.configured` — a stamp file gates
  the configure step.
- `$(MUSL_STAMP):` runs `./configure --target=aarch64-linux-gnu
  --prefix=/usr/local/musl --disable-shared CC=$(CC)
  CROSS_COMPILE=$(CROSS)` and touches the stamp.
- `$(MUSL_LIBC):` depends on the stamp + the two patched
  files; rule body is `$(MAKE) -C $(MUSL_DIR) lib/libc.a`.
- `musl-libc` phony alias for the archive target.
- `musl-clean` runs musl's own `make clean` + removes the
  stamp + `config.mak`.
- `test` target gains `musl-libc` as a dep so any future
  patch breakage is caught by `make test`.  No-op rebuild
  is fast (just `make[1]: 'lib/libc.a' is up to date`).

## Key Findings

- **Two svc paths, not one.**  First pass patched only
  `syscall_arch.h` and rebuilt — `make test` was green and the
  C-path codegen looked right (`_Exit` had `mov x8, #11`).
  But `objdump -d obj/src/unistd/write.lo` showed `write` calls
  `__syscall_cp` (not `__syscall`); `__syscall_cp` resolves to
  `__syscall_cp_asm` in `src/thread/aarch64/syscall_cp.s` which
  bypasses the C-level dispatcher entirely.  Caught by spot-
  checking the disassembly of three syscalls (`write`, `close`,
  `_Exit`) instead of just the const-folded ones.  The fix is
  the inline translation table in the asm file — see
  "What Was Done" above.
- **musl's incremental rebuild doesn't track the arch headers.**
  After the first patch to `syscall_arch.h`, an incremental
  `make lib/libc.a` did *not* regenerate `obj/src/exit/_Exit.lo`
  — its `.lo` predated the header edit but make's rule is
  `*.lo: *.c`, not `*.lo: *.c arch/aarch64/syscall_arch.h`.
  Stale `.lo` files were re-archived into the new `libc.a`,
  leaving the patched translation invisible.  Fix: `rm -rf obj/`
  for any patch to `arch/aarch64/syscall_arch.h`.  This is a
  musl-internal mtime issue, not something to fix at our
  Makefile level — our `$(MUSL_LIBC)` rule does depend on the
  header, but the dep only triggers `make -C $(MUSL_DIR)
  lib/libc.a`, which musl's own dep graph then short-circuits.
  `musl-clean` handles it; alternative would be to make our
  rule do `cd $(MUSL_DIR) && rm -rf obj/ && make`, but that's
  costly (40 s full rebuild) for the common case.  Documenting
  the gotcha here is enough.
- **`mov x0, #-38` works without movn explicit form.**  GCC's
  assembler accepts the immediate as an alias for `movn x0, #37`
  — verified in the disassembly:
  `0xffffffffffffffda` (= -38 unsigned, = -38 signed) lands in
  `x0`.  Used `movn x0, #37` explicitly in the asm file for
  clarity (the C path leaves it to the compiler).
- **Configure script can't probe Linux/glibc on a freestanding
  toolchain, but doesn't need to.**  Our `aarch64-linux-gnu-gcc`
  is glibc-targeted but musl's configure only does compile-only
  feature probes (`-c -o /dev/null`), no link-tests against
  glibc.  Configure runs clean with our toolchain.
- **The slice-7.6c.4 argv push is exactly what musl needs.**
  musl's `__libc_start_main` reads argc/argv/envp from `[sp]`
  per System V — same shape sys_exec already builds.  No
  additional kernel-ABI work needed for slice 7.6c.3a; the
  AUXV gap (musl wants `AT_RANDOM` for the stack canary +
  `AT_PAGESZ` for malloc) is a 7.6c.3b problem that comes up
  the moment we actually link a demo against `libc.a + crt1.o`.

## Decisions Made

- **Sub-slice the slice.**  `7.6c.3a / 7.6c.3b / 7.6c.3c` —
  one session each.  The HANDOFF / IMPLEMENTATION-GUIDE doc
  bumps reflect the new shape; the original 7.6c.3 description
  is preserved as the union of the three sub-slices.
- **`musl-libc` is a `make test` dep, not a `make` (default)
  dep.**  Default `make` builds the kernel without touching
  musl — keeps `kernel.bin` cycle-fast.  `make test` validates
  the patched `libc.a` builds (catches future patch breakage)
  but adds <1 s after the first build (no-op stat check).
- **`-ENOSYS` not `-EINVAL` for unmapped syscalls.**  POSIX
  reserves errno 38 (`ENOSYS`) for "function not implemented"
  — exactly what an unmapped syscall is.  Programs that probe
  for capability via `errno == ENOSYS` will see the right
  signal.  Stub still kills any path that depends on the
  syscall's side effects (which is correct — silent success
  would mask real bugs).
- **Don't pre-patch the four other svc-direct asm files.**
  `clone.s / vfork.s / __unmapself.s / restore.s` will need
  attention in 7.6c.3b/c when the linker actually pulls them
  in.  Leaving them stock now keeps the diff against upstream
  minimal + makes 7.6c.3b's job easier to scope (`grep -l
  "svc" src/...` will surface exactly the next set of files
  that need translation).
- **Rolling Session 41 to archive.**  HANDOFF.md's session list
  caps at 5; after inserting Session 46, Session 41 (slice
  7.6b initramfs) moves to the archive.

## Status at End of Session

- `make test` → **403/403 pass (51 python + 274 host + 78
  kernel), 0 leaks, 0 errors, exit 0**.  No regressions; no
  new tests (sub-slice 7.6c.3a is build-side only).
- `make musl-libc` → 2.75 MB `third_party/musl/lib/libc.a`
  with our `NX_SYS_*` translation built in.  Spot-checked
  codegen confirms the patches land where expected.
- `make musl-clean` cleans the build artifacts + the configure
  stamp.
- The kernel tree is unchanged — no `kernel.bin` re-link, no
  ABI changes.

## Next Steps

**Slice 7.6c.3b — first demo against musl.**  The plan:

1. Push AUXV onto the user stack in `sys_exec`.  At minimum:
   `AT_PAGESZ` (so malloc rounds correctly), `AT_RANDOM` (so
   musl's stack canary has 16 bytes of entropy — even
   pseudo-random is fine for v1; `crc32(monotonic_ticks)` will
   do), `AT_NULL` terminator.  Layout sits between envp's NULL
   and the argv strings (above) per System V.
2. Decide on crt1.  Two options: (a) rebuild musl with our
   own crt1.o that reads argc/argv/envp from `[sp]` exactly
   like our existing `components/posix_shim/crt0.S` does, or
   (b) use musl's stock crt1.o which already does it
   correctly.  Option (b) is the path of least resistance —
   the stock crt1.s is ~30 lines and the layout match is
   trivial to verify by inspection.
3. Re-link `posix_libc_prog.elf` against
   `third_party/musl/lib/libc.a + third_party/musl/lib/crt1.o`
   instead of `components/posix_shim/libnxlibc.a`.  Same
   ktest, same exit code (53).  Diff that succeeds is
   sub-slice 7.6c.3b's exit criterion.
4. Expect to hit one or two more svc-direct asm files
   (`clone.s` likely, since `_init` may pull in tls bootstrap
   even for static).  Patch them with the same translation
   table approach.

**Slice 7.6c.3c — malloc + remaining demos.**  posix_printf_prog
will pull in malloc the moment it `printf`s a `%d` (musl's
formatter uses `nl_langinfo` which needs allocated locale state).
We'll need:

- A `brk` syscall (or anonymous `mmap`) so musl's
  `mallocng` can extend the heap.  Brk is simpler — one new
  syscall + a per-process `brk_addr` field tracked alongside
  `ttbr0_root`.
- Possibly stub `clock_gettime` (formatter doesn't, but
  internal locale init might).
- Re-link `posix_printf_prog` and `argv_parent_prog` (the
  latter currently uses libnxlibc only for `_exit`, so it
  should "just work" against musl as long as ENOSYS-y syscalls
  aren't hit).

**Deferred (unchanged):**

- Proper cross-test task reap on `wait()`.  Will continue to
  hit caps; lands before slice 7.6d.
- HANDLE_FILE fork inheritance.

---

**Files Changed:**
- `sources/nonux/third_party/musl/` — new (musl 1.2.5 source tree, ~14 MB)
- `sources/nonux/third_party/musl/arch/aarch64/syscall_arch.h` — patched (translation table + per-N early-return)
- `sources/nonux/third_party/musl/src/thread/aarch64/syscall_cp.s` — patched (inline translation in asm)
- `sources/nonux/Makefile` — `MUSL_*` vars + `$(MUSL_STAMP)` + `$(MUSL_LIBC)` + `musl-libc` + `musl-clean`; `test` deps on `musl-libc`
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.6c.3 sub-sliced into 7.6c.3a / b / c
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 41 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
