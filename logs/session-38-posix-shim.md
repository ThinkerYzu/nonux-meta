# Session 38: slice 7.4d — `components/posix_shim/` + C-compiled EL0 demo

**Date:** 2026-04-25
**Phase:** 7 slice 7.4d (closes slice 7.4)
**Branch:** master

---

## Goals

Close slice 7.4 by wrapping the `NX_SYS_*` syscall set in a
POSIX-flavoured C API that EL0 programs can link against directly
— so future shell/busybox code (slice 7.6) doesn't have to
hand-roll `svc #0` assembly.  The end product is a header-only
component + a C-compiled EL0 demo proving the wrappers produce
the right SVCs end-to-end.

## Scope choices

- **Header-only, not a .c library.**  Every wrapper is a
  `static inline` function in `posix.h`.  Reasons:
  1. The code belongs to EL0, not EL1.  A `.c` file compiled
     into `kernel.bin` would add text that never runs at EL1 —
     just trap-back-to-EL1 paths the compiler can't help with.
  2. EL0 binaries link at VA `0x48000000` (the user window);
     kernel code at `0x40080000`.  Mixing them in one linker
     pass requires PIC/relocation gymnastics that buy nothing.
  A `static inline` header gets compiled into whichever EL0
  binary `#include`s it, at whatever flags that binary uses —
  exactly what we want.
- **No errno, no macros, no libc glue.**  v1 wrappers return raw
  `nx_status_t` — positive on success (pid / bytes / handle),
  negative on `NX_E*`.  Consumers test for negative values
  directly.  A future slice (or external libc integration in 7.6)
  can layer on `errno`, `WIFEXITED`, etc. — those are
  ergonomics, not ABI.
- **argv / envp accepted but ignored.**  The POSIX `execve`
  signature takes `(path, argv, envp)` for source-compat, but
  v1's `NX_SYS_EXEC` is a single path string.  The wrapper
  discards argv/envp so existing POSIX code compiles; a future
  slice can plumb argv/envp through `sys_exec` into the new
  address space's stack, like Linux does.
- **C-compiled EL0 demo, not just a compile-test.**  The
  wrappers need an end-to-end regression so ABI drift (wrong
  syscall number, mangled register assignment) fails loudly.
  `test/kernel/posix_prog.c` exercises the full fork/waitpid/
  exit path.  The ktest compares state afterwards (exit_code
  in the process table + three markers in the ktest log).  If
  the wrappers ever emit the wrong SVC, the markers won't fire
  or the exit code won't match.
- **`_start` as the ELF entry, no crt0.**  The demo doesn't use
  argc/argv, so wiring a crt0 that sets them up would be
  overkill.  `drop_to_el0` sets `sp_el0` to top-of-user-window
  before `eret`, so `_start` has a valid stack from its first
  instruction.  A real crt0 lands when slice 7.6's busybox
  needs POSIX-style `main(argc, argv)`.
- **`-fno-pic`, freestanding, `-mgeneral-regs-only`.**  Same
  flag set as the kernel (no outline atomics, no FP/SIMD, no
  PIC) so the EL0 binary is pure integer code that the loader
  can drop into the user window unchanged.  `fno-pic` in
  particular avoids GOT/PLT indirection that would need
  fix-up; with the linker pinning everything at
  0x48000000 absolutely, every reference is already correct.

## What Was Done

### `components/posix_shim/`

New directory with three files:

- **`manifest.json`** — minimal: `{"name": "posix_shim",
  "version": "0.1.0"}`.  No `iface` (doesn't bind to a kernel
  slot), no `requires`, no `spawns_threads`, no `pause_hook`.
  Present mostly for metadata parity with other components and
  so `tools/validate-config.py` can walk it without flagging a
  missing manifest.
- **`posix.h`** — header-only.  Defines:
  - Type aliases `nx_posix_pid_t`, `nx_posix_fd_t`,
    `nx_posix_ssize_t` (plain int / int64_t).
  - Syscall-number constants `NX_POSIX_SYS_*` mirroring
    `framework/syscall.h`'s enum (header-local so EL0 code
    doesn't pull the kernel header).
  - Flag / whence constants matching `interfaces/fs.h` values.
  - Low-level `nx_posix_svcN(nr, a0, …)` dispatchers using
    register-asm pinning of `x8` / `x0..x5`, a `"memory"`
    clobber, and `asm volatile` to prevent reordering.
  - POSIX-style wrappers: `nx_posix_debug_write / exit / fork /
    execve / waitpid / open / close / read / write / lseek`.
- **`README.md`** — documents the component's header-only
  nature, the POSIX mapping gaps, and why it's not in
  `kernel.json`.

### `test/kernel/posix_prog.c`

Tiny EL0 C program using the wrappers.  Flow:

```c
void __attribute__((noreturn)) _start(void) {
    nx_posix_pid_t pid = nx_posix_fork();
    if (pid == 0) {
        nx_posix_debug_write("[posix-child]", 13);
        nx_posix_exit(23);
    }
    nx_posix_debug_write("[posix-parent]", 14);
    int status = 0;
    (void)nx_posix_waitpid(pid, &status, 0);
    if (status == 23) nx_posix_debug_write("[posix-ok]", 10);
    nx_posix_exit(0);
}
```

### `test/kernel/posix_prog_blob.S`

Embeds `posix_prog.elf` into `kernel-test.bin`'s `.rodata`
between `__posix_prog_blob_start` / `_end` labels.  Same
pattern as `init_prog_blob.S`.

### `Makefile`

- New `POSIX_PROG_CFLAGS` with the EL0 flag set.
- New rules:
  - `test/kernel/posix_prog.o: test/kernel/posix_prog.c
    components/posix_shim/posix.h`
  - `test/kernel/posix_prog.elf: test/kernel/posix_prog.o
    test/kernel/init_prog.ld` (reuses the slice-7.3 linker
    script — same user-window VA).
  - `test/kernel/posix_prog_blob.o:
    test/kernel/posix_prog_blob.S test/kernel/posix_prog.elf`
    (explicit `.incbin` dependency so the blob rebuilds when
    the ELF changes).
- `KTEST_C += test/kernel/ktest_posix.c` +
  `KTEST_S += test/kernel/posix_prog_blob.S`.
- `clean` target drops `test/kernel/posix_prog.elf`.

### `test/kernel/ktest_posix.c`

New ktest `posix_shim_fork_child_exit_23_parent_waits_and_
emits_ok`:

- Drains stranded user-process tasks via
  `sched_rr_purge_user_tasks(sself, NULL)` — same workaround
  `ktest_exec` uses, needed because prior slice-7.4 ktests
  leave forked-child tasks idling in `wfe` and the 1-second
  rotation time would blow the test past the 15 s QEMU
  timeout.
- Creates `posix-host` process, loads the embedded
  `posix_prog.elf` via `nx_elf_load_into_process`, spawns a
  kthread pinned to it, drops to EL0 at the ELF entry.
- Yields up to 2048 times waiting for debug_write counter ≥ 3.
- Verifies a process with exit_code == 23 exists (searches
  pid > host_pid to avoid matching stranded prior-test
  processes).
- Dequeues the host task; leaves the child task stranded (same
  convention as every other slice-7.4 test — proper reap
  lands with 7.5 or 7.6).

## Key Findings

- **Register-pinning inline asm needs `"memory"` + `volatile`.**
  Without the `"memory"` clobber, the compiler could
  reorder memory traffic around the SVC (fatal for syscalls
  like `write` that read user memory the kernel will consume).
  Without `asm volatile`, it could even eliminate the SVC if
  the return value looked unused.  Both are standard for
  syscall wrappers, but worth stating explicitly for a later
  reader.
- **`-fno-stack-protector` isn't strictly needed but is
  defensive.**  Our toolchain defaults vary; some distros
  enable `-fstack-protector-strong` by default which would
  pull in `__stack_chk_fail`, a libc symbol our freestanding
  link can't resolve.  Explicitly disabling avoids the
  one-off link failure on a contributor's machine.
- **`static inline` functions in EL0 C code work without
  `-fno-plt` tricks.**  The compiler inlines them at every call
  site, so there's no external symbol to resolve at all.  If we
  later move to `extern inline` for faster compile / smaller
  code, we'd need the `.c` file (and then either link it into
  every EL0 binary or ship as a static library).  For v1, the
  inline-everywhere approach is the simplest working shape.
- **Reusing `init_prog.ld` is fine because every EL0 test binary
  lives at the same VA.**  There's exactly one user window
  (`0x48000000`, 2 MiB), and every ELF we load targets it.
  A per-process ELF model with variable entry points would
  change this, but we're not there yet.

## Decisions Made

- **No link-time component registration for posix_shim.**  Other
  components (uart_pl011, sched_rr, mm_buddy, ramfs, vfs_simple)
  each have a `COMPONENT_REGISTER` macro emitting a
  `struct nx_component_descriptor` into the `nx_components`
  linker section + a `kernel.json` binding.  posix_shim has
  neither — it's userspace-only.  The manifest exists for
  metadata parity but gen-config doesn't emit any bindings
  because posix_shim isn't in `kernel.json`'s components map.
- **Demo uses `_start`, not `main`.**  Introducing a `main`
  entry means crt0 to set up argc/argv and call `main` — real
  code, more surface area, more reasons to fail.  `_start`
  sidesteps the whole C runtime story until busybox forces us
  to write the crt0 anyway.
- **Test data lives in `test/kernel/`, not in the component
  dir.**  `posix_prog.c` *uses* posix_shim but is conceptually
  a ktest artefact, not a component artefact.  Lives with the
  other `user_prog_*.S` and `ktest_*.c` files for discoverability.
- **Reuse `ktest_exec`'s scheduler-purge helper.**  Declared
  the helper `extern` at the top of `ktest_posix.c` rather
  than promoting it to a public API in sched_rr's header.
  Still a test-only escape hatch; promoting it would codify a
  workaround into the public surface.
- **Session 33 rolls to HANDOFF-ARCHIVE.**  HANDOFF.md's
  session list is capped at 5 entries; after adding Session 38
  the old Session 33 (slice 7.2, per-process address spaces)
  moves to the archive.

## Status at End of Session

- `make test` → **394/394 pass (51 python + 274 host + 69
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test.
- `make run` boots cleanly.
- Live EL0 marker catalog:
  `[el0] hello` · `[el0-chan-ok]` · `[el0-file-ok]` ·
  `[el0-rdr-ok]` · `[el0-elf-ok]` · `[fork-parent]` ·
  `[fork-child]` · `[wait-parent]` · `[wait-child]` ·
  `[wait-ok]` · `[exec-parent]` · `[exec-ok]` ·
  `[posix-parent]` · `[posix-child]` · `[posix-ok]`.
  Fifteen complete EL0 round-trips — each a distinct
  syscall or process-mechanism milestone.
- Slice 7.4 is closed: fork / exec / wait / posix are all
  live.  A shell written on top of posix_shim could now
  spawn an ELF child, wait for it, and read its exit code
  without a single line of assembly.

## Next Steps

**Slice 7.5 — pipes + basic signals.**

- `NX_SYS_PIPE(fds[2])` wraps `framework/channel.{h,c}`.  Two
  endpoints with asymmetric rights (one `NX_RIGHT_READ`,
  one `NX_RIGHT_WRITE`); `posix_read` / `posix_write` on
  them dispatch through channel send/recv via the existing
  handle-type polymorphism in `sys_read` / `sys_write`.
- `NX_SYS_SIGNAL(pid, signo)` delivers SIGTERM / SIGKILL.
  v1 polls pending signals at syscall entry/exit rather
  than async-interrupting the target — keeps the plumbing
  minimal (no signal stack, no signal mask, no siginfo).
  Real signal delivery (async trap frame replay) lands in
  a later slice.
- Exit criterion: `echo hello | cat` round-trip under
  posix_shim (pipe + two forked processes + exec on both).

**Deferred / opportunistic:**

- crt0 for `main(argc, argv)`.  Needed when slice 7.6's
  busybox expects POSIX main.
- Real reap on `wait()` — destroy the child process + task
  when status is collected.  Removes the
  `sched_rr_purge_user_tasks` workaround across ktests.
- Per-test quantum override via `kernel.json` config knob.

---

**Files Changed:**
- `sources/nonux/components/posix_shim/manifest.json` — new
- `sources/nonux/components/posix_shim/posix.h` — new
- `sources/nonux/components/posix_shim/README.md` — new
- `sources/nonux/test/kernel/posix_prog.c` — new (C-compiled EL0 demo)
- `sources/nonux/test/kernel/posix_prog_blob.S` — new (embeds posix_prog.elf)
- `sources/nonux/test/kernel/ktest_posix.c` — new (kernel test)
- `sources/nonux/Makefile` — `POSIX_PROG_CFLAGS`, posix_prog.{o,elf}, posix_prog_blob.o rules; `KTEST_C += ktest_posix.c`; `KTEST_S += posix_prog_blob.S`; clean
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.4d filled in with as-built details
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 33 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
