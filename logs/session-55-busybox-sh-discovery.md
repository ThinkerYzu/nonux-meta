# Session 55: slice 7.6d.N.0 — first attempt at `busybox sh`, captured OOM

**Date:** 2026-04-25
**Phase:** 7 slice 7.6d.N.0 (busybox-as-shell, discovery sub-slice)
**Branch:** master

---

## Goals

Move from "busybox `--help` runs end-to-end" (slice 7.6d.3c,
session 54) toward the slice-7.6d.N goal of `/init = busybox
sh` booting to a shell prompt.  This session is the
discovery-driven first step: write a kernel test that drives
`busybox sh -c "exit 42"` and observe what fails.  No kernel
changes — just a new userspace ktest that exercises the
already-working `execve("/bin/busybox", ...)` path with a
different argv.

## Scope choices

- **Use `argv[0] = "sh"`, not `"/bin/busybox"`.**  busybox
  dispatches to its applet table via `basename(argv[0])`.
  `argv[0] = "sh"` routes into the `sh` applet (= ash, per
  `CONFIG_SH_IS_ASH=y` in `nonux_defconfig`) without needing
  any `/bin/sh -> busybox` symlink in the ramfs.  The ramfs
  still resolves the path `"/bin/busybox"` for the actual
  ELF load; only the in-process applet dispatch reads `argv[0]`.

- **Use `-c "exit 42"`, not bare `sh`.**  One-shot,
  non-interactive shell run avoids ash's interactive setup
  (TTY mode set, line-editing init, prompt expansion, history
  file open).  No stdin reads, no fork-into-applet dispatch.
  `exit` is a shell builtin so the script doesn't need
  execvp() either.  The exit code 42 is an arbitrary
  "shell parsed + ran the -c argument" sentinel.

- **No kernel changes this sub-slice.**  Discovery only.
  All gaps surfaced will land in 7.6d.N.1+.

- **New ktest, not a modification of `posix_busybox_help`.**
  Both tests have value going forward — `--help` covers the
  no-shell NOFORK applet path, `sh -c` covers the shell
  startup path.  Keep both passing.

## Implementation

Three new files, one Makefile edit.

### `test/kernel/posix_busybox_sh_prog.c`

libnxlibc-linked program.  Forks, child does:
```c
static char a0[] = "sh";
static char a1[] = "-c";
static char a2[] = "exit 42";
char *cargv[] = { a0, a1, a2, 0 };
nxlibc_execve("/bin/busybox", cargv, 0);
```
Parent waits, prints `[bbsh-status=NN]` (hex) + one of
`[bbsh-ok]` (status == 42) or `[bbsh-failed]` (anything else).
Same overall shape as `posix_busybox_help_prog.c`.

### `test/kernel/posix_busybox_sh_prog_blob.S`

`.incbin` of the linked ELF, with `__posix_busybox_sh_prog_blob_{start,end}`
symbols matching the existing convention.

### `test/kernel/ktest_posix_busybox_sh.c`

Drives the program through the existing kernel-test
infrastructure (process_create, elf_load_into_process,
sched_spawn_kthread + drop_to_el0).  Same yield-cap as the
`--help` test (32768 ticks) because exec'ing 1.22 MB takes
plenty of memcpys plus ash startup syscalls.  Asserts only
that the parent reaches its post-wait status print + emits
≥3 debug_writes; the actual child status is captured in the
live log marker.

### Makefile

Added the three sources to `KTEST_C` / `KTEST_S` / build rules
+ the cleanup list.  No changes outside the test-side wiring.

## Discovery

Live ktest log fragment:

```
posix_busybox_sh_parent_forks_and_execs_busybox_sh [bbsh-parent]sh: out of memory
[bbsh-status=01][bbsh-failed]PASS
```

What this tells us:

- **busybox loaded successfully.**  `exec` got past ELF
  parsing + PT_LOAD copy + AUXV push, and dropped into EL0
  at busybox's entry.
- **musl init ran.**  `__libc_start_main` reached `main`
  without faulting (slice 7.6d.3c's TPIDR_EL0 + AT_PHDR
  fixes are still working).
- **busybox dispatched to ash.**  `basename(argv[0])`
  matched the `sh` applet entry; ash got control.
- **ash startup ran far enough to print a diagnostic.**
  The string `sh: out of memory` is from busybox's
  `bb_msg_memory_exhausted` (libbb/messages.c:17), called
  from `xmalloc()` when the underlying allocator returns
  NULL.
- **Process exited cleanly** with status 1 (ash's standard
  "fatal error" exit code).  No SIGSEGV (would be 139), no
  SIGILL (132), no kernel halt.

## Root cause

musl 1.2.5 `config.mak`: `MALLOC_DIR = mallocng`.  mallocng
is the only malloc compiled into our libc.a — there's no
`oldmalloc` (brk-only) fallback at runtime.

mallocng allocates slabs via `mmap()`:
```
src/malloc/mallocng/malloc.c:249:
    p = mmap(0, needed, PROT_READ|PROT_WRITE,
             MAP_PRIVATE|MAP_ANON, -1, 0);
```
Every first malloc() call needs to fault in a slab → mmap.
Our `arch/aarch64/syscall_arch.h` translation table
(slice 7.6c.3a) doesn't map `__NR_mmap` (222), so the call
returns -ENOSYS, mmap returns MAP_FAILED, slab allocation
returns NULL, malloc returns NULL, ash's xmalloc trips its
OOM exit.

Note: brk IS implemented (slice 7.6c.3c), but musl's mallocng
doesn't use brk at all — the brk slot in our 8 MiB user
window is sitting there unused by the actual allocator.

## Next sub-slice plan

**7.6d.N.1 — Minimal anonymous mmap.**  Add `NX_SYS_MMAP`
that handles the mallocng shape:
- `MAP_PRIVATE | MAP_ANON | (-1 fd)` only.
- Page-sized, page-aligned address returned (round-up `len`).
- No `MAP_FIXED`; kernel chooses the address.
- Source the address space from a per-process bump arena
  carved out of the user window.  Likely layout:
  ```
  0     ─ +1.91 MiB : busybox text+data+bss
  +5 MiB           : TLS area (4 KiB)
  +6 MiB ─ +7.5 MiB : brk heap
  +7.5 MiB ─ +8 MiB : EL0 stack (grows down)
  ```
  The unused `+2 MiB ─ +5 MiB` region (3 MiB) is the
  natural mmap arena.  Bump pointer; OOM when bump exceeds
  arena.
- `munmap` can be a no-op (mallocng tolerates it; PMM
  reclaim happens at process exit anyway).
- Update musl's translation table: `__NR_mmap (222) →
  NX_SYS_MMAP`.  No need to map `__NR_munmap (215)` if
  we make it a no-op via the `default: -ENOSYS` fall-through
  + a libc-side workaround… actually mallocng treats
  `munmap` failure as fatal, so we DO need to map it as a
  successful no-op.  `__NR_munmap (215) → NX_SYS_MUNMAP`.

Then re-run the `sh -c "exit 42"` test, observe the next
failure mode.  Most likely candidates if mmap unblocks
malloc:
1. `sigaction()` (`__NR_rt_sigaction = 134`) for ash's
   signal-handling init.  Could stub as "always succeeds,
   handlers ignored" if we don't have signal delivery yet.
2. `getuid()` / `geteuid()` (`__NR_getuid = 174`,
   `__NR_geteuid = 175`).  Stub returning 0 (root).
3. `ioctl(0, TIOCGWINSZ)` for terminal size — could just
   return -ENOTTY and let ash fall back to default 80x24.
4. Open of `/etc/passwd` or similar — file doesn't exist,
   open returns -ENOENT, ash should tolerate.

## Files changed

- New: `test/kernel/posix_busybox_sh_prog.c` (113 lines)
- New: `test/kernel/posix_busybox_sh_prog_blob.S` (15 lines)
- New: `test/kernel/ktest_posix_busybox_sh.c` (107 lines)
- Modified: `Makefile` (3 list additions + 1 build rule
  block; cleanup-list entry).  No production-kernel changes.

## Test results

`make test` → 85/85 PASSED, 0 leaks, 0 errors, exit 0.

The new test `posix_busybox_sh_parent_forks_and_execs_busybox_sh`
is included in that count.  It asserts only parent-side
liveness (parent reached the marker print, emitted ≥3
debug_writes), which is what we want for a discovery sub-slice.

---

**Last Updated:** 2026-04-25
