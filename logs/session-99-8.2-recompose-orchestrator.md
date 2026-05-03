# Session 99 — Slice 8.2: `framework/recompose.c` Orchestrator

**Date:** 2026-05-03
**Slice:** 8.2 — `framework/recompose.c` orchestrator
**Status:** CLOSED

---

## Goals

Implement `nx_recompose()` — the runtime recomposition orchestrator for slice 8.2.  Deliver:

- `framework/recompose.h` — public API types + `nx_recompose()` declaration
- `framework/recompose.c` — five-phase orchestrator implementation
- `test/host/recompose_test.c` — 6 host tests
- `test/kernel/ktest_recompose.c` — 2 kernel tests
- Makefile + test/host/Makefile wired up

---

## What Was Built

### `framework/recompose.h`

Public API:
- `enum nx_slot_change_action` — `NX_SLOT_REPLACE`
- `enum nx_conn_change_action` — `NX_CONN_ADD`, `NX_CONN_REMOVE`, `NX_CONN_REWIRE`
- `struct nx_slot_change` — slot + action + new_comp + state_lost flag
- `struct nx_conn_change` — from/to + action + mode/stateful/policy
- `struct recomp_plan` — changes[] + connections[] + timeout_ms
- `int nx_recompose(const struct recomp_plan *)` — five-phase atomic orchestrator

### `framework/recompose.c`

Five-phase implementation:

1. **VALIDATE** — NULL checks, FROZEN slot guard, new_comp must be in `NX_LC_READY`
2. **PAUSE** — Kahn's topological sort on the induced subgraph (dependents before
   dependencies); cutoff→drain→pause_hook→ops->pause per component.  Rollback
   (resume in reverse) on any pause failure.
3. **REWIRE** (4a–4c):
   - 4a: Apply `NX_CONN_REMOVE` connection changes
   - 4b: Disable old comp (PAUSED→READY), swap slot (`nx_slot_swap`), destroy old
   - 4c: Apply `NX_CONN_ADD` / `NX_CONN_REWIRE` connection changes
4. **RESUME** (reverse pause order):
   - If `slot->active->state == NX_LC_READY`: new comp → enable + `slot_clear_pause`
   - If `slot->active->state == NX_LC_PAUSED`: existing comp → `nx_component_resume`
5. **NOTIFY** — walk `nx_slot_foreach_dependent` for each swapped slot;
   fire `on_dep_swapped` on all upstream callers

`timer_pause()` / `timer_resume()` bracket steps 2–4 on the kernel build.

`slot_clear_pause()` — sets pause_state=NONE + flushes hold queue (so messages
queued during the swap window reach the new component).

### Tests

**Host (6 tests):**
1. `recompose_leaf_slot_replaces_component` — basic swap, old DESTROYED, new ACTIVE
2. `recompose_frozen_slot_returns_eperm` — NX_EPERM guard
3. `recompose_new_comp_wrong_state_returns_estate` — NX_ESTATE guard
4. `recompose_hold_queue_delivers_to_new_component` — pre-queued msgs reach new comp
5. `recompose_conn_add_registers_edge` — ADD connection change
6. `recompose_rollback_on_pause_hook_fail` — rollback to original state on error

**Kernel (2 tests):**
1. `recompose_kernel_leaf_slot_swap` — new comp handles post-swap MPSC messages
2. `recompose_kernel_hold_queue_reaches_new_component` — hold-queue delivery via MPSC

---

## Key Findings

### `nx_waitq_wake_all` deferred

`slot_call.c` documents that `slot's resume path's nx_waitq_wake_all will release us`
for QUEUE-policy blocking callers parked on `resume_waitq`.  Adding
`nx_waitq_wake_all(&s->resume_waitq)` to `slot_resume_cb` in `component.c`
caused a kernel crash (ESR=DataAbort, FAR=0xffffffffffffffe4=NULL-28, ELR inside
`nx_task_yield`) in the pre-existing `pause_kernel_drains_inflight_mpsc_messages` test.

Root cause: the interaction of `nx_waitq_wake_all` (which calls
`nx_preempt_disable` + `waitq_irq_restore`) with a pending timer interrupt
(that was held while IRQs were masked during the critical section) and the
scheduler's runqueue state after the drain-step yields is unsafe at this call
site.  The `nx_waitq_wake_all` call will be added to both `slot_resume_cb` and
`slot_clear_pause` in a dedicated slice once blocking callers across pause/recompose
are exercised by a test.

### Topo sort: Kahn's on induced subgraph

Only slots explicitly in `plan->changes` participate in the sort.  Dependents
outside the plan rely on their pause-policy (QUEUE by default) to buffer
messages during the swap window — they are NOT paused.  This is correct for v1
where plans are small (leaf swap); future multi-slot plans that need coordinated
pause of callers can extend the sort to include transitive dependents.

### Resume check: state-based, not flag-based

The resume step distinguishes replaced vs. paused-but-retained components by
`slot->active->state`: `NX_LC_READY` = new component (enable it), `NX_LC_PAUSED`
= old component kept (resume it).  Clean and future-proof — no separate
"was_replaced" tracking array needed.

---

## Test Results

| Suite              | Before | After  |
|--------------------|--------|--------|
| `make test-tools`  | 102/102 | 102/102 |
| `make test-host`   | 421/421 | 427/427 |
| `make test-interactive` | 7/7 | 7/7 |
| `make test-kernel` | 126/126 | 128/128 |
| `verify-registry`  | 0 findings | 0 findings |
| `verify-iface-fresh` | 0 drift | 0 drift |

---

## Files Changed

| File | Change |
|------|--------|
| `framework/recompose.h` | **new** — public API |
| `framework/recompose.c` | **new** — orchestrator implementation |
| `test/host/recompose_test.c` | **new** — 6 host tests |
| `test/kernel/ktest_recompose.c` | **new** — 2 kernel tests |
| `Makefile` | `framework/recompose.c` added to `FW_C`; `ktest_recompose.c` added to `KTEST_C` |
| `test/host/Makefile` | `recompose_test.c` + `../../framework/recompose.c` added |

---

## Next

**Slice 8.3** — `framework/config.c` runtime config manager + handle API +
EL0 syscall surface (`NX_SYS_CONFIG_*`).  EL0 program opens a config handle,
queries the live composition via snapshot, fires `nx_recompose` through the
syscall gate.
