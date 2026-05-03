# Session 98: Slice 8.1 — Kernel-side pause/drain/resume

**Date:** 2026-05-03
**Phase:** Phase 8 — Runtime Recomposition, Group C
**Branch:** master
**Commit:** 7918fac

---

## Goals

- Implement slice 8.1: lift pause/drain/resume from host-only infrastructure
  into the kernel build.
- Roll back the slice-3.9 inline workaround in `slot_drain_cb` (the no-op
  `nx_ipc_dispatch` call on the kernel path).
- Add end-to-end ktest: pause a slot, drain hold queue, resume, verify no
  message loss.

---

## What Was Done

### 1. `nx_dispatcher_enqueue` — enqueue-side `in_flight_calls` tracking

The core fix: `in_flight_calls` on a slot now spans the full message lifetime
on the kernel build — from the moment the message is pushed into the MPSC queue
until the handler has returned.

**Old semantics (kernel):** `in_flight_calls` was incremented just before
`invoke_handler` and decremented just after.  There was a window between enqueue
and handler invocation where `in_flight_calls == 0` but messages were still in
the queue.

**New semantics (kernel):** `in_flight_calls` is incremented in
`nx_dispatcher_enqueue` (before the `nx_mpsc_push`).  The decrement in
`nx_dispatcher_pump_once` is unchanged (after handler returns).

The ABORT path in `nx_dispatcher_pump_once` now also decrements
`in_flight_calls` on the kernel build to balance the enqueue-side add.

**Host build unchanged** — host tests retain the old handler-bracket semantics
because the per-slot inbox (not the dispatcher MPSC) is the host async path,
and existing host tests assert `in_flight_calls == 0` after `pump_once` in
ways compatible with both orderings (they only check the post-pump value).

### 2. `slot_drain_cb` — kernel drain loop replaces no-op

`framework/component.c::slot_drain_cb`:

```c
/* Kernel */
while (atomic_load_explicit(&s->in_flight_calls, memory_order_acquire) > 0)
    nx_task_yield();
```

After CUTTING is set (which causes new sends with QUEUE policy to go to the
hold queue), any messages already in the MPSC have been tracked via
`in_flight_calls`.  The yield loop gives the dispatcher kthread CPU time to
drain them; once the counter reaches zero the slot is quiescent and the pause
can proceed to DONE.

Added `#if !__STDC_HOSTED__` include of `core/sched/sched.h` for
`nx_task_yield`.

### 3. `test/kernel/ktest_pause.c` — 3 new ktests

| Test | What it verifies |
|------|-----------------|
| `pause_kernel_slot_transitions_to_done` | Basic pause → DONE, resume → ACTIVE lifecycle on the kernel build. `pause_hook`, `pause`, `resume` ops all called correctly. |
| `pause_kernel_drains_inflight_mpsc_messages` | Enqueue one async message without yielding, then immediately call `nx_component_pause`.  Drain step must yield until the dispatcher delivers the message — proves enqueue-side `in_flight_calls` tracking works. |
| `pause_kernel_hold_queue_drains_on_resume` | Pause first, then send two messages (both land in hold queue).  Resume; hold queue flushes back through IPC router into the dispatcher MPSC.  Yield until both are delivered.  Verifies zero message loss across the pause cycle. |

### 4. Makefile — ktest_pause.c added, QEMU timeout bumped

`ktest_pause.c` added to `KTEST_C`.  QEMU `test-kernel` timeout bumped from
300 s to 360 s; the `waitq_wait_returns_on_deadline` test involves a real 50 ms
deadline sleep and was borderline with the 300 s total budget.

---

## Test Results

```
make verify-registry   →  0 findings (R2, R4, R9)
make verify-iface-fresh→  0 drift
make test-tools        →  102/102
make test-host         →  421/421
make test-kernel       →  126/126 (123 prior + 3 new pause tests)
```

---

## Key Decisions

- **Enqueue-side `in_flight_calls`** over "drain-complete waitq": a separate
  drain waitq per slot would require new fields on `struct nx_slot` and a
  dispatcher-side wake path.  Moving the existing counter to enqueue time
  achieves the same guarantee with zero new data structures and no SMP risk
  (the counter was already `_Atomic`).
- **Yield loop, not busy-wait**: `nx_task_yield()` in the drain loop gives the
  dispatcher kthread CPU time to process the in-flight message.  Busy-waiting
  would deadlock on a cooperative scheduler (the dispatcher needs to be
  scheduled to decrement the counter).
- **Host-only guard on the enqueue add**: host tests in `slot_call_test.c` and
  `slice_8_0a8_test.c` check `in_flight_calls == 0` after `pump_once` and pass
  regardless of when the add happens (they don't check the value between enqueue
  and pump), so the host path could also have been changed.  Keeping it
  host-unchanged reduces diff surface and preserves the existing host-test
  documentation comments.

---

## Next Step

**Slice 8.2** — `framework/recompose.c` orchestrator: `struct recomp_plan`,
`nx_recompose()`, topological pause order from registry edges, rollback on
pause-hook failure, connection rewiring (`conn_change`).  `timer_pause()` /
`timer_resume()` bookends.  Demo: recompose a leaf slot.
