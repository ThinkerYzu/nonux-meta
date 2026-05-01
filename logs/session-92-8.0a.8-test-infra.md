# Session 92: Slice 8.0a.8 — Cross-Cutting Test Infrastructure

**Date:** 2026-05-01
**Phase:** Phase 8 — Runtime Recomposition, Group B (IPC migration)
**Branch:** master

---

## Goals

- Land slice 8.0a.8: six reusable test infrastructure pieces that every later
  slice in Group B / Group C depends on.
- Reach the ~70 ktest budget for Group B's 8.0a sub-slices (combined across 8.0a.1–8.0a.8).
- Close slice 8.0a.

## What Was Done

### 1. Mock component (test/host/mock_component.h)

Header-only helper (`static` functions, safe for single inclusion per TU).
Provides `struct mock_handle` with embedded `mock_state` holding `handler_rc`,
`call_count`, `last_msg`, and an optional `custom_fn` override.  Setting
`comp.impl = &state` before `nx_component_register` gives each instance
independent state without globals — two mocks coexist in one test without
interference.

### 2. Hook-chain inspector (test/host/hook_inspector.h)

Header-only observe-only hook.  `struct hook_inspector` registers at a
caller-supplied priority, records up to 32 firings (captures `ipc_src`,
`ipc_dst`, `ipc_msg`), always returns `NX_HOOK_CONTINUE`.  `hook_inspector_reset`
clears events without re-registering; `hook_inspector_teardown` unregisters.

### 3. Recompose event logger (inline in slice_8_0a8_test.c)

Uses `nx_graph_subscribe(recompose_logger_cb, l)` for real-time capture and
`nx_change_log_read(since_gen, …)` for post-hoc queries.  `recompose_logger_init`
snapshots `nx_change_log_total()` as a watermark so that earlier events (before
the test started) are filtered out.

### 4. Pause-injector tests

Eight host tests exercise `nx_slot_set_pause_state`, `nx_slot_set_fallback`,
and `nx_slot_call_blocking`'s pause-state dispatch:
- REJECT policy → `NX_EBUSY`
- REDIRECT (no fallback) → `NX_ENOENT` (fail-closed)
- REDIRECT (with fallback) → still `NX_ENOENT` in v1 (blocking-call path does
  not follow the fallback chain; noted in `slot_call.c` comment)
- State transitions (NONE/CUTTING/DRAINING → NONE) verified via `nx_slot_pause_state`
- `in_flight_calls` zero before and after dispatch
- `resume_waitq.waiters` empty on newly-registered slot

QUEUE policy + actual park/wake is a kernel-only test (on host,
`nx_waitq_wait_with_deadline` calls the `cpu_switch_to` abort stub).

### 5. Cap-forgery harness

Six host tests for `nx_ipc_scan_send_caps` / `nx_ipc_scan_recv_caps`:
- Null caps array → `NX_OK`
- Forged `NX_CAP_SLOT_REF` (no outgoing edge from sender) → `NX_EINVAL`
- Connected slot_ref → `NX_OK`
- Multi-cap: stops at first forged → `NX_EINVAL`
- `NX_CAP_BORROW` unclaimed → silent drop (0 error count)
- `NX_CAP_TRANSFER` unclaimed → protocol error (count == 1)

### 6. Equivalence-runner macro + identity tests

`ASSERT_CALL_EQUIVALENT(direct_rc, blocking_rc)` defined as
`ASSERT_EQ_U((unsigned)(a), (unsigned)(b))` — intended for use in slice 8.0c's
per-callsite migration validation.  Five host tests demonstrate the pattern:
direct component-op vs dispatcher-mediated path produce identical call count
and rc with no hooks, with an observe-only hook, and across N sequential calls.

### Kernel ktests (2 new)

Both use the spawn-and-poll pattern established in slice 8.0a.6:

1. `hook_inspector_observe_only_does_not_alter_blocking_call_result` —
   registers an `NX_HOOK_IPC_SEND` observer before spawning the blocking-call
   kthread; asserts the observer fired ≥1 time and the result is still 0.

2. `cap_scan_rejects_forged_slot_ref_cap_during_blocking_call` — `filesystem.root`
   slot is not in the spawned kthread's `caller_slot` outgoing edges; attaching
   it as a `NX_CAP_SLOT_REF` in the message → `NX_EINVAL` from the cap-scan step
   inside `nx_slot_call_blocking`; `in_flight_reply_buf` cleared on the error path.

### Snapshot buffer bump

`bootstrap_snapshot_json_contains_bound_impl` uses a static buffer to render
the graph snapshot to JSON.  Three ktest kthreads spawned in slices 8.0a.6,
8.0a.7 (none), and 8.0a.8 each add ~250 bytes of JSON (1 slot + 4 edges per
kthread); by the time the snapshot test runs the JSON exceeds 4 KiB.  Bumped
4096 → 8192 with ~4 KiB headroom for future growth.

## Key Findings

- **Ktest linker ordering differs from source order.** Tests from the END of
  `ktest_bootstrap.c` ran BEFORE earlier tests (snapshot test), because the
  ktest linker section is ordered by object placement, not by source position.
  This caused the snapshot buffer overflow to manifest immediately.

- **REDIRECT in blocking-call is fail-closed even with fallback wired.** The
  comment in `slot_call.c` explains the gap: the blocking-call path cannot
  easily recurse without trampling `in_flight_reply_buf` state.  Test explicitly
  validates this `NX_ENOENT` contract so 8.0c migration can verify it.

- **mock_component.h's impl-pointer pattern.** Setting `comp.impl = &state`
  directly (before `nx_component_register`) works because the registry never
  touches `impl` — it's purely caller-owned.  This avoids globals and lets two
  independent mocks share a test function safely.

## Decisions Made

- **Header-only for mock_component.h and hook_inspector.h** — both are
  currently included by a single TU (`slice_8_0a8_test.c`); static functions
  avoid link-order issues and compile faster.  If slice 8.0c needs them in
  multiple TUs, promote to a `.c` file then.

- **Recompose logger uses subscribe + change_log_read together** — subscribe
  gives real-time capture (useful if the test needs to assert on events *during*
  an operation), while `nx_change_log_read(since_gen, …)` gives filtered
  post-hoc access.  Both are exercised so future tests can pick the right tool.

- **Equivalence-runner is a simple macro, not a fixture** — the heavy lifting
  (mock setup, dispatch, assertion) belongs in the test body.  A macro suffices
  for the comparison step; 8.0c tests will compose it with their own fixtures.

## Status at End of Session

- `make test`: **566/566 pass** (93 python + 350 host + 123 kernel; +38 vs 528 baseline)
- `make verify-iface-fresh`: 0 drift
- `make verify-registry`: 0 findings
- Slice 8.0a **closed**.  Source commit `d983e7a` in `sources/nonux/`.

## Next Steps

- Slice **8.0b** — activate generated `handle_msg` shims on every component:
  replace the NULL `handle_msg` in sched_rr, mm_buddy, ramfs, vfs_simple,
  procfs, uart_pl011 with the generated `<iface>_dispatch_msg` shim that routes
  `msg->msg_type` to the per-op handler function.  This is a pure mechanical
  substitution with no behavioral change — the equivalence-runner macro from
  8.0a.8 validates each substitution.

---

**Files Changed (source repo `sources/nonux/`):**
- `test/host/mock_component.h` — new: programmable mock component helper
- `test/host/hook_inspector.h` — new: observe-only recording hook helper
- `test/host/slice_8_0a8_test.c` — new: 36 host tests exercising all 6 infra pieces
- `test/host/Makefile` — added `slice_8_0a8_test.c` to SRCS
- `test/kernel/ktest_bootstrap.c` — 2 new ktests + snapshot buffer bump 4096 → 8192
