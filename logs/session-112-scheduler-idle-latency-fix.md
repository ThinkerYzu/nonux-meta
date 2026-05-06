# Session 112: Scheduler idle-latency fix — dispatcher waitq + zombie dequeue

**Date:** 2026-05-06
**Phase:** Post-Phase-9b (between Phase 9b and Phase 9)
**Branch:** master

---

## Goals

- Diagnose and fix the `make run-busybox` slowness: every command and keypress felt sluggish.

## What Was Done

### Root-cause analysis

Traced five independent scheduling latency sources, each blocking the CPU on idle's WFI loop unnecessarily:

1. **Idle not preempted on wakeup** — `sched_rr_enqueue` / `sched_priority_enqueue` never set `need_resched` on the idle task when a task became runnable.  Idle could sit in WFI for up to one full quantum (200 ms at 10 Hz / 2-tick quantum) before the scheduler preempted it.

2. **IPC enqueue doesn't signal the dispatcher** — `nx_dispatcher_enqueue()` pushes to the MPSC queue but the dispatcher kthread is a permanent member of the round-robin runqueue.  When busybox blocked after a syscall, `yield()` might rotate the dispatcher to the tail and leave idle at the head, causing a ≤200 ms stall before the dispatcher got CPU.

3. **Dispatcher busy-polls** — When the MPSC queue was empty, the dispatcher called `nx_task_yield()` and stayed in the runqueue, racing with idle and ktest_main for CPU time.

4. **Zombie tasks burn quanta** — `nx_process_exit()` parked the task in a `for (;;) wfe` loop but never dequeued it from the scheduler runqueue.  Each completed busybox command left a zombie that consumed a full 200 ms quantum per round-robin cycle.

5. **`nx_task_yield()` is a no-op when only idle is runnable** — With the dispatcher BLOCKED and no other tasks, `pick_next()` returns idle (`next == curr`) and `sched_check_resched()` returns immediately with no context switch.  This made WAIT_FOR polling loops in ktests burn their iteration budget in microseconds before any timer tick fired.

### Fixes

**`sched_rr_enqueue()` / `sched_priority_enqueue()`** — when a non-idle task is enqueued and the current task is `g_idle_task`, set `curr->need_resched = 1`.  The next IRQ return calls `sched_check_resched()` and immediately preempts idle.  Covers both the case where idle is running WFI and the case where `nx_waitq_wake_one()` fires from an IRQ handler.

**Dispatcher waitq** — replaced the dispatcher kthread's `nx_task_yield()` tail with `nx_waitq_wait_unless(&g_disp_wq, 0, disp_has_pending, NULL)`.  The dispatcher transitions to `NX_TASK_BLOCKED` (dequeued from the runqueue) when its MPSC queue is empty.  `nx_dispatcher_enqueue()` now: (a) increments `g_disp_pending` before the MPSC push (so the `wait_unless` predicate sees a non-zero count and avoids lost wakeups across the Vyukov MPSC transient window), (b) calls `nx_waitq_wake_one(&g_disp_wq)` after the push, which calls `sched_rr_enqueue(dispatcher)` and triggers the idle-preemption fix above.  Result: when busybox makes a syscall, the dispatcher is woken and freshly added to the runqueue tail; busybox then blocks and the `yield()` rotation in `sched_check_resched()` deterministically picks the dispatcher.

**Zombie dequeue in `nx_process_exit()`** — before the `for (;;) wfe` loop, dequeue the current task from the scheduler runqueue via `sched_ops_for_test()->dequeue()`.  Zombie tasks no longer accumulate in the round-robin cycle.  Process struct and task struct are not freed here — `sys_wait` still needs `exit_code` and `state`.  Full reap-on-wait (`nx_process_destroy`) remains a deferred follow-up.

**WFI in `sched_check_resched()` when `next == curr`** — when the scheduler has no other runnable task to switch to, issue `wfi` before returning.  This makes `nx_task_yield()` wait for a real interrupt rather than returning instantly.  Needed so that ktest WAIT_FOR polling loops (which call `nx_task_yield()` up to N times) cover real wall-clock time — in particular so the `waitq_wait_returns_on_deadline` test's 50 ms deadline can fire before the 1024-iteration budget expires.

### Failure cascade discovered and fixed

The dispatcher waitq change initially caused two regressions in the kernel tests:

- `waitq_wait_returns_on_deadline` FAIL — WAIT_FOR burned 1024 iterations in microseconds (instant because dispatcher BLOCKED, only idle in runqueue).  The 50 ms deadline hadn't fired yet when KASSERT ran.
- Crash at ELR=1 — `reap_waiter()` was called on a still-blocked kthread (wq3 was still in `g_deadline_list`), freeing its memory.  A later `nx_waitq_tick_deadlines()` call dereferenced the freed deadline node → instruction abort at address 0x1.

Both resolved by the WFI fix above.

Also attempted an "idle-last `pick_next()`" approach (always prefer non-idle tasks in `pick_next()`), but reverted it: ktest_main runs on the idle task's execution context, so if `pick_next()` never returns idle while the dispatcher is also runnable, ktest_main is permanently starved and the test binary produces zero output.

## Key Findings

- The idle task running WFI is only preempted by timer quantum expiry (2 ticks = 200 ms) unless something explicitly sets `need_resched = 1`.  No path in the previous code did this on wakeup.
- `nx_dispatcher_enqueue()` pushes to an MPSC queue — it never calls the scheduler's `enqueue()`, so the old enqueue fix would not have helped the IPC path.  The dispatcher waitq makes `enqueue(dispatcher)` happen at message-arrival time.
- The Vyukov MPSC `pop()` can return NULL even with items in-flight (transient inconsistency).  Solved by counting pending messages in `g_disp_pending`: the `wait_unless` predicate sees the count, not the queue state, so a push+wake that races the sleep path is never lost.
- An "idle-last `pick_next()`" that skips `g_idle_task` in the scan breaks ktest_main because ktest_main runs on the idle task's execution context and needs to be scheduled normally after being woken.

## Decisions Made

- **Dequeue zombie in `nx_process_exit()`, not in `sys_wait()`** — dequeuing at exit is the minimal correct fix: `sys_wait` only needs `exit_code`/`state`, which remain valid.  Full `nx_process_destroy()` in `sys_wait` is still the correct long-term fix but is deferred.
- **WFI in `sched_check_resched()` rather than changing WAIT_FOR budgets** — the WAIT_FOR macro is used across many tests with carefully chosen budgets.  Changing the scheduler to make `yield()` meaningful when nothing else is runnable is the right abstraction boundary.
- **Did not implement "idle-last `pick_next()`"** — the two correct fixes (enqueue signal + dispatcher waitq) make idle preemption deterministic without changing round-robin semantics.

## Status at End of Session

- `make test-tools` → **102/102**
- `make test-host` → **476/476**
- `make test-kernel` → **151/151**
- `make verify-iface-fresh` → 0 drift
- `make verify-registry` → 0 findings

## Next Steps

- Start Phase 9 — per-process memory management rework (L3 4 KiB pages, VMAs, demand paging, COW fork).  See `IMPLEMENTATION-GUIDE.md §Phase 9`.

---

**Files Changed:**
- `components/sched_rr/sched_rr.c` — `sched_rr_enqueue`: set `idle->need_resched = 1` when non-idle task enqueued while idle is current
- `components/sched_priority/sched_priority.c` — `sched_priority_enqueue`: same idle-preemption signal
- `framework/dispatcher.c` — dispatcher waitq (`g_disp_wq`, `g_disp_pending`, `disp_has_pending`); `nx_dispatcher_enqueue` increments count + wakes waitq; kthread blocks on `wait_unless` instead of yielding; `nx_dispatcher_init` initializes waitq + count; `nx_dispatcher_reset` resets count
- `framework/process.c` — `nx_process_exit`: dequeue task from scheduler runqueue before `for (;;) wfe`; added includes `core/sched/sched.h` and `interfaces/scheduler.h`
- `core/sched/sched.c` — `sched_check_resched`: issue `wfi` when `next == curr` (no other runnable task)
