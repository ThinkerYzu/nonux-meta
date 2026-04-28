# Session 36: slice 7.4b — `NX_SYS_WAIT` + DAIF-mask fix in `nx_process_exit`

**Date:** 2026-04-24
**Phase:** 7 slice 7.4b
**Branch:** master

---

## Goals

Close the fork/wait pair started by slice 7.4a.  A parent that
forks needs to be able to reap its child's exit code — that's
`wait()`.  Mechanically simpler than fork (no trap-frame replay,
just a poll loop) but exposed a subtle interrupt-masking bug in
`nx_process_exit` that had been latent since slice 7.1.

## Scope choices

- **Blocking-via-poll, not wake-on-exit.**  Real kernels
  maintain a per-process "children-waiting" wait queue so
  `wait()` blocks efficiently.  v1 polls with `nx_task_yield()`
  in a loop — simpler, works correctly, costs some CPU while the
  parent is waiting.  For the ktest harness (one parent + one
  child + bounded run time) this is fine; a real workload with
  many concurrent waiters is when the wait queue becomes worth
  the complexity.
- **No zombie reaping.**  `wait()` returns the child's exit_code
  but doesn't free the child's process struct or address space.
  A pid-leak exists.  Acceptable in v1 because the
  process-table capacity (16 slots) is far above what any
  current test exercises.
- **Self-wait rejected with `NX_EINVAL`.**  Matches POSIX's
  treatment of `wait()` on the caller's own pid.  Unknown pid
  and kernel-pid (0) both return `NX_ENOENT`.
- **Skip `nx_process_reset_for_test` in `ktest_wait`.**  Earlier
  ktests (specifically `ktest_fork`) strand EL0 tasks in `wfe`
  because there's no way to dequeue a task whose process we
  don't hold a handle to.  Resetting the process table at the
  top of `ktest_wait` would leave those stranded tasks with
  dangling `task->process` pointers; the next
  `sched_check_resched` would dereference the garbage and flip
  TTBR0 to a bad value, crashing EL0 on the next instruction
  fetch.  The fix is to NOT reset — user processes from prior
  tests stay alive (their tasks keep cycling harmlessly through
  `wfe`), and the wait test's child-finding logic filters by
  pid > its own parent's pid so it can't mistake a stranded
  process for its own child.  A proper cross-test cleanup lands
  when `nx_process_destroy` grows a "dequeue all tasks pointing
  at this process" pass.

## What Was Done

### `framework/syscall.{h,c}` — `NX_SYS_WAIT`

- `NX_SYS_WAIT = 13` appended to the enum.
- `sys_wait(pid, user_status)`:
  - `nx_process_lookup_by_pid(pid)` → `NX_ENOENT` if NULL or
    the kernel process.
  - `nx_process_current()` → reject self-wait with `NX_EINVAL`.
  - `while (target->state != NX_PROCESS_STATE_EXITED)
    nx_task_yield();` — unbounded loop; kernel parks the parent
    until the child exits.
  - `copy_to_user(user_status, &target->exit_code, 4)` if
    user_status is non-NULL.  Failure non-fatal — still returns
    the pid so the caller knows the target exited even if the
    status copy failed.
  - Returns pid.
- Host stub returns `NX_EINVAL` (no scheduler / no user pointer
  semantics).

### `framework/process.c` — IRQ-enable in `nx_process_exit`

- Added `msr daifclr, #2` before the `for (;;) wfe` park loop.
- Why: hardware masks DAIF.I on every exception entry (SVC,
  IRQ, etc.).  Until slice 7.4b, `nx_process_exit` inherited
  that masked state and entered `wfe` with interrupts masked.
  WFE does resume the CPU when a pending interrupt exists, but
  the interrupt isn't *delivered* until the mask is cleared —
  so the CPU exits `wfe`, falls through to `b 1b`, re-enters
  `wfe`, and the IRQ queues indefinitely.  The scheduler never
  preempts.  Unmasking before the park lets timer ticks deliver
  normally.

### EL0 + kernel test

- `test/kernel/user_prog_wait.S`:
  - fork.
  - Child branch: debug_write("[wait-child]"); exit(42).
  - Parent branch: save child pid in x19; debug_write(
    "[wait-parent]"); wait(x19, &status_buf); load w2 from
    status_buf; compare to 42; if equal, debug_write(
    "[wait-ok]"); park.
- `test/kernel/ktest_wait.c`:
  - Creates `wait-parent` process, spawns kthread pinned to it,
    drops to EL0.
  - Yields up to 1024 times waiting for debug_write counter ≥
    3 (parent-marker + child-marker + wait-ok).
  - Independently verifies the child's process struct reached
    EXITED with exit_code == 42 — only searches pids >
    wait_parent_pid to avoid matching stranded processes from
    earlier ktests.
  - Dequeues the wait parent task; child task stays stranded
    (same convention as earlier EL0 ktests).

## Key Findings

- **Hardware IRQ-mask + WFE is a silent trap.**  The bug only
  manifested when a child task tried to block on `wfe` *and* a
  parent was simultaneously polling for its exit: the parent
  could never get scheduled back because the child's wfe loop
  ate all CPU.  Other slices (7.4a fork, 7.3 ELF) had EL0 tasks
  that also parked in wfe but got dequeued externally by the
  ktest — which worked because the main body continued to run
  regardless of whether the EL0 task was stuck.  Only 7.4b
  required the parent to make forward progress *through* the
  scheduler, which needed IRQs to fire for timer ticks to do
  their preemption job.  Lesson: any code path that expects
  `wfe` to sleep until preemption must unmask IRQs first.
- **Cross-test cleanup is a real hole.**  Stranded tasks with
  dangling `task->process` pointers will silently corrupt
  TTBR0 on the next context-switch touching them.  Workaround
  in v1: tests that reset the process table must first dequeue
  or eliminate all user tasks.  Properly: `nx_process_destroy`
  should walk the scheduler's runqueue and dequeue any task
  whose process matches.  Noted for a future follow-up.

## Decisions Made

- **Unmask IRQs in `nx_process_exit`, not in `nx_syscall_
  dispatch`.**  The broader fix would be "unmask IRQs at the
  top of every syscall".  Correct semantics but touches every
  syscall path — scope creep.  Unmasking in
  `nx_process_exit` is surgical: the only syscall that
  parks-forever currently.  If a future syscall also needs to
  sleep (e.g., blocking channel recv), it can unmask itself.
  If several grow that need, we revisit the blanket approach.
- **Poll-with-yield, not wake-queue.**  Simpler v1 mechanism;
  matches the "we don't have blocking semantics yet" state of
  the rest of the kernel.  Real wait-queues land when there's
  a second consumer that would also benefit (channels is the
  likely candidate, currently non-blocking).
- **Skip process-reset in ktest_wait.**  The "right" fix is
  universal cross-test cleanup, but that's a broader change.
  Skipping the reset is a local workaround that keeps this
  slice focused.  Documented the reason in the ktest body
  so a future reader understands why.

## Status at End of Session

- `make test` → **392/392 pass (51 python + 274 host + 67
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test.
- `make run` boots cleanly.
- EL0 markers now visible across the run:
  `[el0] hello` · `[el0-chan-ok]` · `[el0-file-ok]` ·
  `[el0-rdr-ok]` · `[el0-elf-ok]` · `[fork-parent]` ·
  `[fork-child]` · `[wait-parent]` · `[wait-child]` ·
  `[wait-ok]`.  Ten complete EL0 round-trips, each a distinct
  syscall or process-mechanism milestone.
- Fork + wait + exit form a closed loop: a parent can now
  spawn a child, wait for it to finish, and read its exit
  code — the spine of any Unix-style shell workload.

## Next Steps

Slice 7.4c — `NX_SYS_EXEC(path)`.

- Blocked on the deferred 7.3.5 ramfs bump.  `RAMFS_FILE_CAP`
  is currently 256 bytes; a minimal static ELF is ≥ 1 KiB (our
  build emits ~66 KiB due to phdr alignment).  Need to raise
  the cap before exec can load from a ramfs path.  Possibly
  bump to 64 KiB to accommodate typical test programs.
- Then `sys_exec(path)`:
  1. open + read the ELF via vfs_simple into a kernel buffer.
  2. `nx_elf_parse` + validate.
  3. Allocate a fresh address space via
     `mmu_create_address_space`.
  4. `nx_elf_load_into_process(current, blob, len)` into the
     fresh space.
  5. Swap: `current->ttbr0_root = new_root`;
     `mmu_switch_address_space(new_root)`;
     `mmu_destroy_address_space(old_root)`.
  6. Modify the current trap frame: `ELR_EL1 = entry`,
     `SP_EL0 = new_user_window_top`, `x0..x30 = 0`.
  7. Return.  Dispatcher finishes normally; eret delivers
     control to the new ELF's entry.
  Handle table carries over by default (exec doesn't close
  inherited fds in v1).

Slice 7.4d then wraps the NX_SYS_* set in a POSIX API
component, and 7.5–7.7 follow for pipes, signals, busybox.

---

**Files Changed:**
- `sources/nonux/framework/syscall.h` — `NX_SYS_WAIT = 13`
- `sources/nonux/framework/syscall.c` — `sys_wait` + dispatch entry
- `sources/nonux/framework/process.c` — IRQ-enable in `nx_process_exit`
- `sources/nonux/test/kernel/user_prog_wait.S` — new (EL0 fork+exit+wait demo)
- `sources/nonux/test/kernel/ktest_wait.c` — new (kernel test)
- `sources/nonux/Makefile` — `KTEST_S += user_prog_wait.S`; `KTEST_C += ktest_wait.c`
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.4b filled in with as-built details
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 31 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
