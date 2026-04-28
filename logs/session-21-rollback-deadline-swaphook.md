# Session 21: slice 3.9b.2 — pause-failure rollback, pause_hook deadline, SLOT_SWAPPED dispatch

**Date:** 2026-04-22
**Phase:** 3 slice 3.9b.2 (closes Phase 3)
**Branch:** master

---

## Goals

Close Phase 3 by hardening the three correctness gaps left at the end
of slice 3.9b.1:

1. **Pause-failure rollback.** When `pause_hook` or `ops->pause`
   returned non-zero, the bound slots stayed stuck in DRAINING and any
   NX_PAUSE_QUEUE-held messages stayed held forever. Slice 3.8's test
   explicitly documented this: "Slot may be in DRAINING; slice 3.9 will
   harden the rollback."
2. **`pause_hook` 1 ms wall-clock deadline.** DESIGN.md's
   §Recomposition Protocol promises pause_hook is a quick "finish any
   pending work" signal, not an arbitrary-duration callback. Enforce.
3. **Runtime `NX_HOOK_SLOT_SWAPPED` dispatch.** `hook.h` already has
   the enum and union arm, but `nx_slot_swap` only emitted the
   change-log event. Fan out to the hook chain so observers that live
   on hooks (not subscribers) see binding changes.

## What Was Done

### `core/cpu/monotonic.h` — new, inline-only

A deadline primitive that works freestanding. Three pieces:

- `nx_monotonic_raw(bool *ok)` — reads `cntpct_el0` on kernel,
  `clock_gettime(CLOCK_MONOTONIC)` on host. `ok` flags clock read
  failure (host errno path) so callers can treat "broken clock" as
  "budget exceeded".
- `nx_monotonic_freq()` — `cntfrq_el0` on kernel, `1e9` on host.
- `struct nx_deadline` + `nx_deadline_start(d, budget_ns)` +
  `nx_deadline_exceeded(d)`. Budget is converted once at start into
  units of the counter, so the per-check path is one mrs + one uint64
  compare. No division or floating point in the hot path —
  `-mgeneral-regs-only` safe.

The kernel build keeps using the ARM generic timer (cntpct_el0) rather
than the existing `core/timer/` API because the timer module is about
scheduling tick delivery, not arbitrary wall-clock sampling. Slice 8's
recomposer will use the same primitive for its overall transaction
budget.

### `framework/component.c` — pause rollback + deadline

- New static `rollback_pause(c)` — walks every bound slot through
  `slot_resume_cb`, i.e. `pause_state → NONE + nx_ipc_flush_hold_queue`.
  That's exactly the symmetric inverse of the partial pause we did up
  to the failure point, and it reuses the existing resume code path so
  future changes to resume semantics automatically apply to rollback.
- `nx_component_pause` calls `nx_deadline_start(1 ms)` before
  `ops->pause_hook`, checks on return, and converts an otherwise-OK
  hook into `NX_EDEADLINE` if the budget is blown. On any failure
  (hook err, hook deadline, ops->pause err), rollback runs before the
  non-zero status is returned.

### `framework/registry.{c,h}`

- New `NX_EDEADLINE = -9` error code in registry.h.
- `nx_slot_swap` now dispatches `NX_HOOK_SLOT_SWAPPED` after
  `emit_event`. The hook is observational — it fires *after* `s->active`
  has already been updated — so ABORT returns are ignored here. Pre-swap
  veto would need a different emit point and is out of scope for 3.9b.2.

### Tests

Host (+4 cases in `component_pause_test.c`):

- `pause_hook_failure_rolls_back_and_replays_held_messages` — stages a
  NX_PAUSE_QUEUE-held message, runs a pause with a failing hook, and
  checks that the held message replays to the live inbox rather than
  staying stuck.
- `pause_op_failure_after_hook_success_rolls_back` — covers the second
  failure path: `pause_hook` returns OK, `ops->pause` returns
  NX_EBUSY. Verifies rollback still fires.
- `pause_hook_exceeding_deadline_returns_edeadline_and_rolls_back` —
  uses `clock_gettime` + nanosleep to burn 2 ms inside the hook.
  Verifies `nx_component_pause` returns `NX_EDEADLINE`, `ops->pause`
  never ran, and slot is back to NONE.
- `slot_swap_fans_out_to_slot_swapped_hook_chain` — registers a hook
  on `NX_HOOK_SLOT_SWAPPED`, does three swaps (bind, re-swap, unbind),
  and checks the captured `(slot, old_impl, new_impl)` tuple is
  correct for each.

Also updated `pause_hook_returning_error_aborts_transition` to assert
the new contract (`pause_state == NONE`) and removed its stale "slice
3.9 will harden" comment.

Kernel (+1 case in `ktest_bootstrap.c`):

- `slot_swap_fans_out_to_hook_chain_on_kernel` — same shape as the host
  test, but uses a freshly-registered test-only slot so it doesn't
  perturb the live bootstrap composition. Confirms the hook fires
  end-to-end in the real kernel image, not just on host.

Net: `make test` → **244/244** (51 python + 167 host + 26 kernel), up
from 239/239.

## Key Decisions

- **`NX_HOOK_SLOT_SWAPPED` fires AFTER the swap, not before.** This
  matches the existing IPC_RECV / COMPONENT_STATE events which are also
  observational post-commits, and matches the hook's union-arm name
  (`u.swap`, with `old_impl` and `new_impl`). A pre-swap veto would
  need a separate point; the current set of callers (change-log
  readers, recomposer invariant checks) all want post-commit.
- **Rollback reuses `slot_resume_cb` rather than a bespoke path.**
  A partial pause leaves slots in CUTTING or DRAINING and possibly
  with hold-queue entries — exactly what `slot_resume_cb` already
  handles (set NONE + flush). Writing a separate rollback would
  duplicate that logic and drift.
- **Deadline check wraps `ops->pause_hook` only, not `ops->pause`.**
  `pause_hook` is declared "quick"; `ops->pause` is the driver /
  subsystem's chance to flush its own state and may legitimately take
  longer (disk flush, debuffer). If an operator wants a deadline on
  `pause`, they can enforce it inside the op.
- **`NX_EDEADLINE = -9`.** New code rather than overloading `NX_EBUSY`
  or `NX_ESTATE` — diagnostics want to distinguish "hook refused" from
  "hook took too long".

## Known Issues / Follow-ups

None specific to this slice. Slice 3.9b.2 closes Phase 3. Follow-ups
deferred to later phases:

- The deadline primitive could land on the recomposer's per-transaction
  budget check in Phase 8; it's designed to be reused there.
- `verify-registry.py` R6 could grow a rule that every `pause_hook` in
  a manifest is statically bounded (no loops over external inputs, no
  blocking calls). Opportunistic; not tracked as work.

## Next Actions

Phase 4 was already closed in Session 19; Phase 3 is now fully closed.
**Next milestone:** Phase 5 — virtual memory and handle system.
`components/mm_buddy/`, `framework/handle.{c,h}`, `framework/syscall.c`,
and a first userspace (EL0) process. See
[IMPLEMENTATION-GUIDE.md §Phase 5](../IMPLEMENTATION-GUIDE.md#phase-5-virtual-memory-and-handle-system).

## Files Touched

- `core/cpu/monotonic.h` — **new**
- `framework/component.c` — pause rollback + deadline
- `framework/registry.h` — `NX_EDEADLINE` code
- `framework/registry.c` — `NX_HOOK_SLOT_SWAPPED` dispatch in `nx_slot_swap`
- `test/host/component_pause_test.c` — updated one test + 4 new cases
- `test/kernel/ktest_bootstrap.c` — 1 new case
