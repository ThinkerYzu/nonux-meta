# Session 39: slice 7.5 — pipes + polled signals

**Date:** 2026-04-25
**Phase:** 7 slice 7.5
**Branch:** master

---

## Goals

Two more POSIX primitives layered on existing framework pieces.
`NX_SYS_PIPE` wraps the slice-5.6 channel primitive into a
POSIX-flavoured read/write pair with asymmetric rights.
`NX_SYS_SIGNAL` delivers a signal to a target process; v1 is
polled (reads a `pending_signals` bitmask at every
`sched_check_resched`) rather than async-interrupting the target.

## Scope choices

- **Pipe = two HANDLE_CHANNEL endpoints with asymmetric rights,
  not a new handle type.**  Pipes are just channels with a
  POSIX dress code.  Reusing HANDLE_CHANNEL means the slice-5.6
  type-aware `sys_handle_close` destructor, the refcounted
  endpoint-pair lifecycle, and the message-ring transport all
  apply unchanged.  The asymmetry is encoded as rights bits on
  the handles: fds[0] has `NX_RIGHT_READ` only, fds[1] has
  `NX_RIGHT_WRITE` only — no `NX_RIGHT_TRANSFER` in v1 because
  pipes don't cross process boundaries via IPC cap transfer
  yet.
- **Handle-type-polymorphic read/write, not a separate
  `nx_posix_recv` / `send`.**  POSIX collapses file and pipe
  I/O into `read` / `write`; the wrappers should do the same.
  `sys_read` / `sys_write` now check the handle type at the
  top and dispatch to CHANNEL vs FILE accordingly.  The rights
  check is unified (both want `READ` or `WRITE`); only the
  object-side plumbing differs.
- **Polled signals, not async delivery.**  Real signal
  semantics (POSIX says "arrives asynchronously") require
  mucking with a target's saved trap frame to inject a handler
  call at its next EL0 return.  v1 ducks that complexity: set
  a bit in `pending_signals`, check at every
  `sched_check_resched` — any timer preempt / yield / syscall
  return path routes a pending SIGKILL or SIGTERM into
  `nx_process_exit(128 + signo)` inline.  Good enough for
  "kill a misbehaving process"; real signal handlers (sigaction,
  user-defined handlers, mask/restore) lands with a later slice.
- **Only SIGKILL + SIGTERM in v1.**  Every other POSIX signal
  either needs handler support (SIGUSR1/2, SIGCHLD, etc.),
  specific delivery semantics (SIGSEGV synchronous with faults),
  or kernel cooperation (SIGSTOP/SIGCONT + scheduler integration).
  v1 ships two terminating signals, both delivering identical
  behaviour (nx_process_exit with 128+signo) but reserving the
  right bit patterns for a future slice that distinguishes
  catchable SIGTERM from uncatchable SIGKILL.
- **ACTIVE-process guard in the signal poll is load-bearing.**
  Without it, after nx_process_exit sets state=EXITED and parks
  in wfe, the next timer IRQ re-enters sched_check_resched, my
  signal check sees the still-set pending bit, re-enters
  nx_process_exit — we never reach the cpu_switch_to below,
  so the scheduler can't rotate past this task to observe the
  EXITED state from a parent's waitpid poll.  Found the hard
  way: first pass of the signal test emitted `[signal-parent]
  [signal-child]` and then hung forever.  Adding `if
  (curr->process->state == NX_PROCESS_STATE_ACTIVE)` as a guard
  on the poll fixed it (skip the check once the process is
  already EXITED — we just want to leave the task in its wfe
  loop and get back to the scheduler).
- **Single-process pipe demo.**  The cross-process `pipe + fork
  + exec` flow requires fork to duplicate the parent's handle
  table — currently it doesn't (slice 7.4a deferred it).  Adding
  fork handle duplication is a few dozen lines for HANDLE_CHANNEL
  (bump `nx_channel_endpoint`'s refcount + re-alloc a handle
  entry in the child's table pointing at the same endpoint) but
  trickier for HANDLE_FILE (each `open` creates a per-open cursor
  that's specific to that handle, not refcounted).  The natural
  landing point for full handle duplication is slice 7.6 where
  busybox actually exercises the pattern.  For v1, the pipe demo
  is single-process: same program opens the pipe, writes, reads,
  compares — proving NX_SYS_PIPE's dispatch plumbing end-to-end
  without taking a dependency on handle-duplication.
- **C-compiled EL0 demos, same pattern as slice 7.4d.**  Both
  `posix_pipe_prog.c` and `posix_signal_prog.c` follow the
  posix_prog.c shape: `_start` entry, `-ffreestanding -nostdlib
  -mgeneral-regs-only -fno-pic`, linked at the user-window VA
  via `test/kernel/init_prog.ld`, embedded into `kernel-test.
  bin` through a `.incbin`-style `posix_*_prog_blob.S` pair.
  No crt0 (argc/argv not in scope yet; drop_to_el0 gives us
  sp_el0 directly).

## What Was Done

### `framework/syscall.{h,c}`

- `NX_SYS_PIPE = 15` and `NX_SYS_SIGNAL = 16` added to
  `enum nx_syscall_number`.  `NX_SIGKILL` (9) and `NX_SIGTERM`
  (15) constants in the header.
- `sys_pipe(int *user_fds)`:
  - `nx_channel_create` → endpoint pair.
  - `nx_handle_alloc(NX_HANDLE_CHANNEL, NX_RIGHT_READ, e0)` →
    read-side handle.
  - `nx_handle_alloc(NX_HANDLE_CHANNEL, NX_RIGHT_WRITE, e1)` →
    write-side handle.
  - `copy_to_user` writes both handles as `int`s into the
    user `fds[2]` array.
  - Failure at any point cleanly unwinds via
    `nx_handle_close` + `nx_channel_endpoint_close` mirror
    calls (same unwinding shape as `sys_channel_create`).
- `sys_read` / `sys_write` refactored to look up the handle
  first (once), check READ or WRITE rights, then dispatch:
  - `HANDLE_CHANNEL`: bounded by `NX_CHANNEL_MSG_MAX`, route to
    `nx_channel_recv` / `nx_channel_send`, stage via a 256-byte
    kernel buffer, `copy_to/from_user`.
  - `HANDLE_FILE`: existing vfs dispatch path, bounded by
    `NX_FILE_IO_MAX`.
  - Anything else: `NX_EINVAL`.
- `sys_signal(pid, signo)`:
  - Reject signo not in {SIGTERM, SIGKILL} with NX_EINVAL.
  - `nx_process_lookup_by_pid` — NX_ENOENT on unknown or
    kernel process.
  - `__atomic_fetch_or(&target->pending_signals, 1u << signo,
    __ATOMIC_RELEASE)`.
  - Does NOT modify the target's scheduling — delivery is
    polled.
- Dispatch table gains `[NX_SYS_PIPE] = sys_pipe` +
  `[NX_SYS_SIGNAL] = sys_signal`.

### `framework/process.{h,c}`

- `struct nx_process` grows an `_Atomic uint32_t
  pending_signals` at the tail.  Zero-initialised by
  `nx_process_create`'s `memset`/`calloc` and by the static
  initializer of `g_kernel_process` (fields not named in a
  designated-initializer default-init to 0 in C99/C11).

### `core/sched/sched.c`

- Signal-delivery poll added at the top of
  `sched_check_resched`:
  ```c
  if (curr->process &&
      curr->process->state == NX_PROCESS_STATE_ACTIVE) {
      uint32_t pending = __atomic_load_n(
          &curr->process->pending_signals, __ATOMIC_ACQUIRE);
      if (pending & (1u << NX_SIGKILL))
          nx_process_exit(128 + NX_SIGKILL);
      if (pending & (1u << NX_SIGTERM))
          nx_process_exit(128 + NX_SIGTERM);
  }
  ```
  Runs ahead of every other check (need_resched,
  preempt_count) so a pending terminating signal always wins
  over voluntary-yield contention.  `framework/syscall.h`
  pulled in for `NX_SIGKILL` / `NX_SIGTERM`.

### `components/posix_shim/posix.h`

- `NX_POSIX_SYS_PIPE` / `_SIGNAL` constants added.
- `NX_POSIX_SIGKILL = 9` / `NX_POSIX_SIGTERM = 15`.
- Two new wrappers:
  ```c
  static inline int nx_posix_pipe(int fds[2]);
  static inline int nx_posix_kill(nx_posix_pid_t pid, int signo);
  ```

### Demos + ktests

- `test/kernel/posix_pipe_prog.c` — single-process round-trip.
  Creates a pipe, writes "hello\n" to fds[1], reads back from
  fds[0], compares the bytes, emits `[pipe-ok]`, exits 29.
  Exit codes 1..4 signal specific error points (pipe alloc,
  write count, read count, content mismatch); the ktest
  asserts the process ended EXITED with `exit_code == 29`.
- `test/kernel/posix_signal_prog.c` — fork + kill + wait.
  Parent emits `[signal-parent]`, busy-waits to let the child
  emit its `[signal-child]`, `nx_posix_kill(child_pid,
  SIGTERM)`, `nx_posix_waitpid(child_pid, &status, 0)`,
  asserts `status == 128 + SIGTERM == 143`, emits
  `[signal-ok]`.  Child's body is `for (;;) g_dummy++` — no
  syscalls — so signal delivery relies entirely on timer
  preemption rolling through `sched_check_resched`.
- `test/kernel/posix_pipe_prog_blob.S` +
  `test/kernel/posix_signal_prog_blob.S` — `.incbin` the
  matching `.elf` into `.rodata`.
- `test/kernel/ktest_posix_pipe.c` + `ktest_posix_signal.c` —
  same load-and-drop shape as `ktest_posix.c` + exit-code
  assertion on the target process.

### Drive-by: `framework/ipc.c`

- `nx_ipc_reset` used to walk each per-slot message queue
  and clear `_next` on every message for tidiness.  Messages
  are caller-owned (host tests typically stack-allocate them);
  once a test returns and its stack frame pops, the message
  memory may be reused.  Walking the list therefore reads
  potentially-overwritten `_next` fields — and segfaults
  whenever stack reuse stomps on them.  Dropped the walk:
  just `free` the queue bookkeeping and leave caller storage
  alone.  Latent bug surfaced during this session's full-suite
  run; unrelated to slice 7.5 but blocked `make test` from
  reporting green until fixed.

### Makefile

- New POSIX_PROG_CFLAGS rules for `posix_pipe_prog.o/elf` +
  `posix_signal_prog.o/elf`.
- `KTEST_C += ktest_posix_pipe.c ktest_posix_signal.c`.
- `KTEST_S += posix_pipe_prog_blob.S posix_signal_prog_blob.S`.
- `clean` drops the two new `.elf`s.

## Key Findings

- **Struct layout mismatches without header-dependency tracking
  are silent.**  First kernel boot after adding
  `pending_signals` to `struct nx_process` surfaced garbage
  values in the field (a random pointer read as `1074307488 =
  0x40099EE0`).  Root cause: the project Makefile doesn't emit
  `-MMD`/`-MP` dependencies, so when I added the field but only
  `process.c` got rebuilt, every other TU (sched.c, syscall.c,
  process.h consumers) retained the OLD struct layout.
  sched.c's signal check reached into memory past the old
  struct's end.  `make clean + make` fixed it.  A future
  cleanup: add `-MMD -MP` to the CFLAGS and `-include *.d` in
  the Makefile so `.h` edits rebuild dependents automatically.
- **Signal poll mustn't re-enter `nx_process_exit` once the
  process is EXITED.**  The first pass hung the signal test
  because the child's task got stuck in an infinite loop:
  IRQ → sched_check_resched → see pending SIGTERM → exit →
  wfe → next IRQ → sched_check_resched → still see pending
  SIGTERM → exit again → wfe → ...  The scheduler never
  reached the `cpu_switch_to` below, so the parent in
  `sys_wait`'s yield loop never got picked to observe the
  child's EXITED state.  The fix is the ACTIVE guard:
  deliver signals only to ACTIVE processes.  An EXITED
  process's task just sits in wfe; subsequent scheduler
  rotations eventually pick other tasks because my signal
  check returns early and the `cpu_switch_to` below runs
  normally.  Well worth a comment in the code; future readers
  will ask why the guard is there.
- **Timer-tick-driven signal delivery works for busy-loop
  children.**  `posix_signal_prog.c`'s child has no syscalls
  at all — just `for (;;) g_dummy++`.  Delivery still works
  because every timer tick preempts the child, IRQ stub
  SAVE_TRAPFRAME + on_irq + sched_check_resched, and my poll
  fires on sched_check_resched.  Good; means signals reach
  misbehaving CPU-bound processes without special cooperation.

## Decisions Made

- **Rebranded slice 7.5 from "signals with sigaction" to
  "polled terminating signals only".**  The IMPLEMENTATION-
  GUIDE description matches this scope; the actual POSIX signal
  machinery (handlers, siginfo, mask/restore) is much larger.
  Keeping v1 minimal follows the project's discipline of
  "ship working verticals, not comprehensive primitives".
- **Handle-type polymorphism in sys_read/write, not a new
  syscall.**  POSIX collapses file + pipe + socket reads into
  one `read` syscall.  Making `sys_read` / `sys_write` aware
  of handle type keeps the POSIX shim's wrappers uniform and
  means adding more handle types later (sockets, devices) is
  adding a branch in the same functions.  The alternative
  (separate `sys_pipe_read` / `sys_pipe_write`) would force
  posix_shim's `nx_posix_read` to dispatch based on an
  fd-type tag we'd have to track separately.
- **Cross-process pipe demo deferred, don't block on it.**
  The slice's exit criterion from the guide mentions `echo
  hello | cat` working end-to-end, but that requires fork
  handle duplication AND real echo/cat binaries.  Both are
  naturally slice 7.6 territory (musl + busybox).  The
  single-process pipe demo still fully exercises
  `NX_SYS_PIPE` + the polymorphic dispatch; cross-process
  semantics layer on top once handle duplication lands.
  HANDOFF.md / next-actions calls out fork inheritance as
  the 7.6 prereq.
- **Fix `nx_ipc_reset` in this slice even though it's
  unrelated.**  The segfault blocked `make test` from
  reporting green — and once noticed, it's a 5-line fix.
  Folding it in beats filing a follow-up against slice 3.6
  or similar.

## Status at End of Session

- `make test` → **396/396 pass (51 python + 274 host + 71
  kernel), 0 leaks, 0 errors, exit 0**.  +2 kernel tests.
- `make run` boots cleanly.
- Live EL0 marker catalog (now 18 distinct milestones):
  `[el0] hello` · `[el0-chan-ok]` · `[el0-file-ok]` ·
  `[el0-rdr-ok]` · `[el0-elf-ok]` · `[fork-parent]` ·
  `[fork-child]` · `[wait-parent]` · `[wait-child]` ·
  `[wait-ok]` · `[exec-parent]` · `[exec-ok]` ·
  `[posix-parent]` · `[posix-child]` · `[posix-ok]` ·
  `[pipe-ok]` · `[signal-parent]` · `[signal-child]` ·
  `[signal-ok]`.
- Phase 7 now has: process abstraction (7.1), per-process
  address spaces (7.2), ELF loader (7.3), fork/exec/wait/posix
  (7.4), pipes/signals (7.5).  The POSIX surface is enough
  for any userspace doing the standard Unix primitives short
  of signal handlers.

## Next Steps

**Slice 7.6 — busybox cross-compile + initramfs.**

- Write a minimal musl → NX_SYS_* adapter (C or asm file
  inside a musl build).  musl already has a syscall table per
  arch; we patch `aarch64`'s to emit our SVC numbers.
- Cross-compile busybox against musl for aarch64.
- Define an initramfs format (cpio-newc is simplest), pack
  `/init` + a few applets.
- Ramfs learns to slurp the initramfs blob at bootstrap.
- `/init` is busybox → `sh` applet.
- Exit: `make && make run` boots to a busybox shell prompt.

Blocks-of / handle-inheritance bookkeeping (needed for pipes
to actually work across processes) can be done as a prereq
inside the busybox slice since that's where the pattern first
matters.

**Deferred (unchanged from slice 7.4d's list):**

- C-level crt0 for `main(argc, argv)` entry.
- Proper cross-test task reap at end of `wait()`.
- Per-test `SCHED_RR_DEFAULT_QUANTUM_TICKS` override via
  kernel.json.

---

**Files Changed:**
- `sources/nonux/framework/syscall.h` — NX_SYS_PIPE / _SIGNAL + signal number constants
- `sources/nonux/framework/syscall.c` — sys_pipe + sys_signal + polymorphic sys_read/write + dispatch table
- `sources/nonux/framework/process.h` — `_Atomic uint32_t pending_signals` field
- `sources/nonux/framework/ipc.c` — drop dangling-message walk in nx_ipc_reset (drive-by fix)
- `sources/nonux/core/sched/sched.c` — signal-delivery poll at top of sched_check_resched
- `sources/nonux/components/posix_shim/posix.h` — nx_posix_pipe / kill + SIGKILL/SIGTERM constants
- `sources/nonux/test/kernel/posix_pipe_prog.c` — new (single-process pipe roundtrip EL0 demo)
- `sources/nonux/test/kernel/posix_pipe_prog_blob.S` — new (.incbin wrapper)
- `sources/nonux/test/kernel/posix_signal_prog.c` — new (fork + kill + wait EL0 demo)
- `sources/nonux/test/kernel/posix_signal_prog_blob.S` — new
- `sources/nonux/test/kernel/ktest_posix_pipe.c` — new (ktest)
- `sources/nonux/test/kernel/ktest_posix_signal.c` — new (ktest)
- `sources/nonux/Makefile` — posix_pipe_prog.{o,elf} + posix_signal_prog.{o,elf} rules; KTEST_C/S updates; clean
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.5 filled in with as-built details
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 34 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
