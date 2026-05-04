# Session 103: Slice 8.6 — Runtime Async↔Sync Mode Switching

**Date:** 2026-05-04
**Phase:** 8 — Runtime Recomposition
**Branch:** master

---

## Goals

Implement slice 8.6 — runtime per-connection IPC mode switching: flip an edge between
`NX_CONN_ASYNC` and `NX_CONN_SYNC` at runtime without restart, drain in-flight handlers
before the retune, and expose it through both a kernel-internal API and an EL0 syscall.

---

## What Was Done

### `framework/recompose.c` — extend `build_pause_order` for REWIRE

Added a loop after the slot-changes pass: for each `NX_CONN_REWIRE` entry, add the
receiver (`to_slot`) to the pause set (dedup). This ensures the receiver is paused and
all in-flight handlers drain before the edge mode is changed.  After the retune, the
hold queue is flushed through the router, which now uses the new mode.

### `framework/config.c` + `framework/config.h` — `nx_config_set_conn_mode()`

Convenience wrapper that looks up two slots by name, builds a conn-only `recomp_plan`
with one `NX_CONN_REWIRE` entry, and calls `nx_recompose()`.  The full pause/drain/
retune/resume cycle runs automatically.

### `framework/syscall.h` + `framework/syscall.c` — `NX_SYS_CONFIG_REWIRE` (45)

New EL0 syscall: `(nx_handle_t h, const char *from_slot, const char *to_slot, int mode)`.
Validates the config handle, copies slot names from user space, validates mode
(NX_CONN_ASYNC=0 or NX_CONN_SYNC=1), calls `nx_config_set_conn_mode`.

### `test/host/slice_8_6_test.c` — 6 host tests

1. `rewire_changes_mode_and_drains_tgt` — edge mode changes, `to_slot` paused+resumed.
2. `rewire_async_to_sync_dispatches_inline` — SYNC send calls handler immediately.
3. `rewire_sync_to_async_queues_message` — ASYNC send goes to inbox, requires pump.
4. `rewire_hold_queue_flushed_via_new_sync_mode` — pre-paused tgt, REWIRE(SYNC),
   hold queue flushed inline.
5. `config_set_conn_mode_rewires_edge` — convenience API end-to-end.
6. `config_sys_rewire_syscall` — `NX_SYS_CONFIG_REWIRE` via trap frame dispatch.

### `test/kernel/ktest_conn_mode.c` — 2 kernel tests

Direct `nx_connection_retune()` (no recompose overhead, safe on single-CPU):

1. `conn_mode_async_to_sync_dispatches_inline` — async send → MPSC; yield to deliver;
   retune to SYNC; sync send → handler runs inline, count increments immediately.
2. `conn_mode_sync_to_async_dispatches_via_queue` — retune to ASYNC; async send →
   MPSC; yield until dispatcher delivers.

### Pre-existing bug fixes discovered during testing

**Dangling slot-nodes** (`ktest_pause.c`, `ktest_recompose.c`):
- `nx_slot_unregister` returns `NX_EBUSY` if any connection edge is still present.
  Both fixtures registered an edge between stack-allocated slots but called
  `nx_slot_unregister` without first calling `nx_connection_unregister`.  The
  slot-node stayed in `g_slots` pointing at dead stack memory; subsequent
  `slot_node_find` calls would dereference the stale pointer and fault.
- Fix: unregister the connection before the slots in teardown.  Documented the pattern
  in `TESTING-GUIDE.md` §"Kernel test fixture pattern".

**Dangling component-nodes** (`ktest_config.c`, `ktest_recompose.c`):
- `nx_component_destroy` transitions DESTROYED but does NOT remove from `g_components`.
  Test fixtures left DESTROYED components in the registry; a subsequent
  `component_node_find` could read a garbage `manifest_id` pointer from the reused
  stack frame and crash in `strcmp`.
- Fix in `registry.c`: skip DESTROYED components in `component_node_find`.
- Fix in test teardown: explicitly call `nx_component_unregister` on DESTROYED
  components while their structs are still alive.

**Static slot pattern for kernel test fixtures**:
- Re-designed ktest_pause and ktest_recompose to use `static struct nx_slot` storage
  so slot-nodes in `g_slots` always point at valid (never-freed) memory.
- Documented in `TESTING-GUIDE.md` §"Kernel test fixture pattern".

**`sched_rr_enable` missing idle task enqueue**:
- After a live swap from sched_priority → sched_rr, `g_idle_task` was NOT in
  sched_rr's runqueue (it had been enqueued into sched_priority by `sched_start()`).
  The ktest (idle) task called `nx_task_yield()` and never got CPU back — sched_rr's
  `pick_next` cycled through the counting tasks forever.
- The comment in sched_rr said "idle is always READY, so we always have a fallback at
  the tail of the runqueue" but the implementation did not enforce this on enable.
- Fix: `sched_rr_enable` now calls `sched_rr_enqueue(s, &g_idle_task)` if idle is not
  already in the runqueue.

**QEMU timeout** bumped 450 → 900 s to accommodate the extra conn-mode tests.

---

## Key Findings

1. **REWIRE with conn-only plan**: pausing `to_slot` before the retune is essential —
   it drains in-flight handlers and routes the hold-queue flush through the new mode.
   On single-CPU the direct `nx_connection_retune()` is also safe without pause, but
   the full recompose path is the correct production API.

2. **Slot-node teardown ordering**: `nx_slot_unregister` must be called AFTER
   `nx_connection_unregister`.  The test-infrastructure pattern is now:
   ```
   nx_connection_unregister(edge);
   nx_slot_swap(slot, NULL);
   nx_component_unregister(comp);
   // nx_slot_unregister NOT needed if slot is static
   ```

3. **Static vs stack slots**: static slot storage is the correct default for kernel
   test fixtures.  The slot-node lives for the binary's lifetime; only components
   (per-test) need explicit unregistration while their structs are in scope.

4. **`sched_rr_enable` idle fix is a genuine latent bug**: the original 134-test suite
   was showing the same hang but the Makefile `#`-comment syntax had silently dropped
   `ktest_live_swap.c` from `KTEST_C`, so live_swap tests were never actually running.

---

## Test Results

```
make test-host   → 459/459 pass
make test-kernel → 136/136 pass
make verify-iface-fresh: 0 drift
make verify-registry: 0 findings (R2,R4,R9)
```
