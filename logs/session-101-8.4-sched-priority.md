# Session 101: Slice 8.4 — sched_priority second scheduler

**Date:** 2026-05-03
**Phase:** 8 — Runtime Recomposition
**Branch:** master

---

## Goals

- Implement `components/sched_priority/` — a fixed-priority scheduler that
  passes the Phase 4 conformance suite and provides correct `set_priority`
  semantics (unlike `sched_rr` which returns `NX_EINVAL` uniformly).
- Wire `sched_priority` into `kernel.json` and boot end-to-end in QEMU.
- All `make test` targets continue to pass.

## What Was Done

### sched_priority component

Created `components/sched_priority/sched_priority.c` with:
- `SCHED_PRIORITY_LEVELS 8` buckets (priority 0 = default/lowest, 7 = highest),
  each an intrusive `nx_list_head`.
- `enqueue`: appends to `queues[0]`; `dequeue` scans all buckets (O(n)).
- `pick_next`: scans highest → lowest, skips `NX_TASK_BLOCKED` tasks.
- `yield`: rotates the current task (`nx_task_current()`) within its own bucket;
  falls back to rotating the head of the highest non-empty bucket when
  `nx_task_current()` is NULL (host tests).
- `set_priority`: moves task to new bucket; returns `NX_OK` for 0–7, `NX_EINVAL`
  otherwise, `NX_ENOENT` if task is not queued.
- `tick`: identical to `sched_rr` — quantum countdown, `need_resched` on expiry.
- Lifecycle counters (`init_called`, `enable_called`, …) for ktest introspection.
- `sched_priority_purge_user_tasks`: drains stranded EL0 user-process tasks from
  all buckets; same role as `sched_rr_purge_user_tasks`.
- `sched_rr_purge_user_tasks` exported as a `__weak` compatibility shim so the
  20+ ktest teardown helpers that `extern`-declare the `sched_rr` name continue
  compiling without modification.  Marked `__weak` so the strong definition in
  `sched_rr.c` wins in the host build (where both components are compiled in).

### Host tests

Created `test/host/component_sched_priority_test.c` (16 tests):
- 7 conformance wrappers (`nx_conformance_scheduler_*`).
- 9 priority-specific cases: valid/invalid range for `set_priority`, `ENOENT`
  when not queued, `pick_next` honours priority order, FIFO order within the
  same priority, `set_priority` moves to correct bucket, BLOCKED task skipped.

### Build system

- `test/host/Makefile`: added `component_sched_priority_test.c`,
  `../../components/sched_priority/sched_priority.c`, and the component
  directory to `vpath`.
- `kernel.json`: changed `scheduler.impl` from `sched_rr` to `sched_priority`.
  `gen/sources.mk` updated automatically via Make dependency tracking.

### Kernel test updates

- `ktest_sched_bootstrap.c`: replaced `sched_rr_state_mirror` with
  `sched_priority_state_mirror` (queues[8] × 16 bytes = 128-byte prefix, then
  counters); updated KTEST names and `manifest_id` check.
- `ktest_sched.c`: updated extern declaration and ops-pointer assertion to
  reference `sched_priority_scheduler_ops`.

## Key Findings

- **`__weak` is the right tool for the compatibility shim.**  Both `sched_rr.c`
  and `sched_priority.c` are compiled in the host test build; `__weak` lets
  `sched_rr.c`'s strong definition win there while `sched_priority.c`'s
  definition is used in the kernel build (where only one scheduler is compiled
  in per `gen/sources.mk`).

- **State-mirror offset arithmetic.**  `sched_priority_state` starts with
  `queues[8]` (8 × 16 bytes = 128 bytes of list-head storage) before the
  `unsigned` counters.  The ktest mirror struct must match exactly; two-pointer
  `struct { void *next, *prev; }` per entry is correct on AArch64.

- **yield via `nx_task_current()` is correct for the kernel.**  The fallback
  (rotate head of highest non-empty bucket) is only exercised in host tests
  that call `yield` without setting a current task; kernel runtime always has
  a current task.

## Decisions Made

- **No new op added to `nx_scheduler_ops` for `purge_user_tasks`.**  It is
  test-only infrastructure; adding it to the IDL-generated interface would be
  spec pollution.  The `__weak` shim approach is the minimal, safe path.

- **Priority range 0–7 (8 levels).**  Matches the `SCHED_PRIORITY_LEVELS 8`
  constant, gives a realistic range without bloating state, and leaves room to
  increase without interface changes.

- **`enqueue` always places tasks at priority 0.**  Callers use `set_priority`
  to promote.  This matches the scheduler.h spec ("zero is the default").

## Status at End of Session

- `make test-tools` **102/102 pass**
- `make test-host` **453/453 pass** (16 new)
- `make test-kernel` **132/132 pass**
- `make verify-registry` 0 findings (R2,R4,R9)
- `make verify-iface-fresh` 0 drift
- Kernel boots with `sched_priority` from `kernel.json`; all POSIX-layer,
  busybox, and recomposition ktests pass.

## Next Steps

- **Slice 8.5** — headline live swap: EL0 program running N background tasks
  swaps `sched_priority → sched_rr` (or vice versa) via the config handle
  mid-run; tasks survive, leak-free, behaviour change observable.

---

**Files Changed:**
- `components/sched_priority/sched_priority.c` — new: priority scheduler impl
- `components/sched_priority/manifest.json` — new: component manifest
- `components/sched_priority/README.md` — new: component documentation
- `test/host/component_sched_priority_test.c` — new: 16 host tests
- `test/host/Makefile` — add sched_priority to build
- `kernel.json` — `scheduler.impl`: `sched_rr` → `sched_priority`
- `test/kernel/ktest_sched_bootstrap.c` — update state mirror + assertions
- `test/kernel/ktest_sched.c` — update extern + ops assertion
