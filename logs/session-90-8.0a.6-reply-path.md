# Session 90: slice 8.0a.6 — reply path end-to-end (per-CPU pool + dispatcher integration + `nx_slot_call_blocking` body + posix_shim reply routing)

**Date:** 2026-05-01
**Phase:** Phase 8 — Group B (slice 8.0a, sub-slice 6 of 8)
**Branch:** master (source-side commit `34dd1cb` in `sources/nonux/`; this session's checkpoint is the proj_docs paperwork that records it)

---

## Goals

- Land slice **8.0a.6** — close the blocking-call round-trip end to end:
  per-CPU reply-message pool, dispatcher-side reply synthesis under
  `slot->in_flight_calls`, full `nx_slot_call_blocking` body sequence
  (per [SLOT-CALL-API.md §"Body sequence"](../SLOT-CALL-API.md#body-sequence)),
  posix_shim's reply-routing handler that copies the payload into the
  caller's kstack `in_flight_reply_buf` and wakes its `reply_waitq`.
- Synthesize an `NX_EABORT` reply when the IPC_RECV hook chain aborts
  on a `NX_MSG_FLAG_REPLY_REQUESTED` request, so the caller's
  `reply_waitq` always wakes (DESIGN.md §"ABORT on a blocking-call edge
  synthesizes a reply").  IPC_SEND-hook ABORT on the caller side stays
  on the kstack — no synthetic reply needed since the caller never
  reaches the dispatcher.
- Add the first end-to-end blocking-call ktest against the live kernel
  composition; pin the dispatcher / pool / posix_shim contract with
  ~13 host TESTs that exercise the validation paths and the
  request → reply → free flow.

## What Was Done

### Slice 8.0a.6 (source-side commit `34dd1cb`, `sources/nonux/`)

Pure additive — no production callers consume `nx_slot_call_blocking`
yet (slice 8.0c migrates `framework/syscall.c`'s ~30 vfs sites; slice
8.0b lights up generated `handle_msg` shims on the five production
components).  Existing direct `vops->read(self, …)` paths are
untouched.  The new round-trip is exercised through tests only.

**`framework/dispatcher.c` (+245 lines).**  Three additions on top of
the slice-3.9b.1 dispatcher:

1. **Reply-message pool.**  `struct nx_reply_pool_entry` lays out a
   complete reply: `struct nx_ipc_message msg` (first member, so the
   message pointer the dispatcher hands to `pump_once` casts back to
   the entry directly) followed by `char payload[NX_REPLY_PAYLOAD_MAX]`.
   `g_reply_pool[NX_REPLY_POOL_SIZE]` (256 entries, matching
   SLOT-CALL-API.md §"Per-CPU reply pool" — 2× safety margin against
   `NX_PROCESS_TABLE_CAPACITY = 128`) is backed by an atomic-bitmap
   in-use mask (`g_reply_pool_inuse[NX_REPLY_POOL_BITMAP_WORDS]`).
   `reply_pool_alloc` walks word-by-word, finds the lowest zero bit
   via `~cur & (cur + 1)`, and CASs it.  Pool exhaustion fires
   `kpanic` on kernel build (log + WFE spin) / `abort()` on host
   build — a dropped reply would silently hang a blocked caller, which
   is worse than failing loudly.  `reply_pool_owns(m)` pointer-range-
   tests against `&g_reply_pool[0]..&g_reply_pool[NX_REPLY_POOL_SIZE]`;
   `reply_pool_free(e)` clears the bit corresponding to `(e -
   g_reply_pool)`.  Test-only helpers (`capacity_for_test`,
   `in_use_for_test`, `reset_for_test`, `owns_for_test`) compiled in
   on every build for both the host suite and `ktest_bootstrap.c`.

2. **`build_reply(req, rc)`** — allocates a pool entry, zeros both
   the message header and the payload (slice 8.0b's wrapper-emitted
   reply structs append op-specific output fields after the header;
   clearing now keeps every reply bit-stable), writes
   `((struct nx_reply_header *)e->payload)->rc = (int32_t)rc`, and
   inverts src/dst (`reply.src = req.dst`, `reply.dst = req.src`) so
   the reply lands at the caller's `caller_slot`.  `flags =
   NX_MSG_FLAG_REPLY` (replies never carry `REPLY_REQUESTED`,
   preventing infinite reply loops at the dispatcher level).

3. **`nx_dispatcher_pump_once` reshape.**  Pre-handler IPC_RECV hook
   stays as before — but ABORT on a `REPLY_REQUESTED` request now
   `build_reply(msg, NX_EABORT)` + `nx_dispatcher_enqueue` (the
   synthetic reply lands on the caller's `caller_slot` in the next
   pump iteration).  Handler invocation wraps `slot->in_flight_calls`
   atomic increment/decrement (acq_rel) — the pause protocol's drain
   step waits on this counter (DESIGN.md §"Slot-side pause + blocking-
   call metadata").  After `invoke_handler` returns, if
   `REPLY_REQUESTED`, `build_reply(msg, rc)` + enqueue.  Pool-owned
   replies (delivery-side branch) free the entry after their handler
   runs.  `nx_dispatcher_reset` also resets the pool so host fixtures
   don't leak across boundaries.

**`framework/dispatcher.h` (+22 lines).**  Test-surface declarations
for the four reply-pool helpers.  Public surface (`nx_dispatcher_init`,
`nx_dispatcher_enqueue`, `nx_dispatcher_pump_once`,
`nx_dispatcher_reset`) unchanged.

**`framework/slot_call.h` (+34 lines).**  Three additions:
`struct nx_reply_header { int32_t rc; }` (slice 8.0b's wrapper-emitted
reply structs embed it as their first field; slice 8.0a.6 round-trips
just the header), `NX_REPLY_POOL_SIZE = 256u`, `NX_REPLY_PAYLOAD_MAX
= 512u` (header is 4 bytes; 512 leaves headroom for slice 8.0b
without per-op pool tuning yet).

**`framework/slot_call.c` (+174 lines).**  `nx_slot_call_blocking`
body lands in full per [SLOT-CALL-API.md §"Body sequence"](../SLOT-CALL-API.md#body-sequence):

1. **Validate** — NULL args / dst_slot mismatch / missing src_slot
   → `NX_EINVAL`.  Caller must be a task with `caller_slot_active`
   true; `msg->src_slot == &task->caller_slot`; ISR / kthread / boot
   code paths must use the existing async `nx_ipc_send` instead.
2. **No recursive blocking calls** — `task->in_flight_reply_buf
   != NULL` on entry → `NX_EINVAL`.  v1 has a single in-flight slot
   per task; nested blocking calls are out of scope.
3. **Force `NX_MSG_FLAG_REPLY_REQUESTED`** so a misbehaving wrapper
   can't turn a blocking call into a fire-and-forget that hangs the
   reply waitq.
4. **Pause-state walk** — `nx_slot_pause_state(slot)` atomic load;
   `NX_SLOT_PAUSE_NONE` → break.  Otherwise look up the per-task
   outgoing edge via `nx_slot_foreach_dependency(src, …)` (registered
   by slice 8.0a.5's `wire_caller_slot`):
   - `NX_PAUSE_REJECT` → return `NX_EBUSY` (no enqueue).
   - `NX_PAUSE_REDIRECT` → v1 fails closed with `NX_ENOENT` (the
     redirect chain walker would need to re-enter the body without
     trampling the in-flight reply buf state; deferred until a real
     REDIRECT consumer lands).
   - `NX_PAUSE_QUEUE` → `nx_waitq_wait_with_deadline(&slot->resume_waitq,
     0)` (indefinite); on wake re-check pause_state.  The slot's
     resume path's `nx_waitq_wake_all` releases all blocked callers.
5. **Stash** `task->in_flight_reply_buf{,_len}` + zero
   `in_flight_reply_rc` *before* enqueue so the reply leg's
   `posix_shim_handle_msg` can route into them as soon as it runs.
6. **Fire IPC_SEND hook chain.**  ABORT → clear in-flight buf state,
   return `NX_EABORT` directly (no synthetic reply needed; we never
   reached the dispatcher).
7. **Cap-scan** via `nx_ipc_scan_send_caps` (DESIGN.md R3 — every
   `NX_CAP_SLOT_REF` in `msg->caps` must correspond to a registered
   outgoing edge from the per-task slot).  Forged caps → `NX_EINVAL`,
   clear in-flight buf.
8. **`nx_dispatcher_enqueue(msg)`**.  Failure → clear in-flight buf
   + return.
9. **`nx_waitq_wait_with_deadline(&task->reply_waitq, 0)`** —
   indefinite; the reply path's `nx_waitq_wake_one` (from
   `posix_shim_handle_msg`) is the only legitimate wakeup source for
   this waitq in v1.
10. **Return** `task->in_flight_reply_rc`; clear the in-flight buf
    state on the way out.

**Edge lookup helper** (`find_outgoing_edge`).  `framework/ipc.c`'s
`find_edge` is `static`; rather than widen visibility we walk the
slot's outgoing deps via `nx_slot_foreach_dependency` + a tiny
`edge_search { target, hit }` capture struct.  The cost is
O(out-deps), only paid when the dest slot is paused.

**`components/posix_shim/posix_shim.c` (+108 lines).**  `handle_msg`
switches from the slice-8.0a.4 NX_EINVAL stub to the real reply-
routing body:

- `task_from_caller_slot(msg->dst_slot)` does `container_of` back to
  `nx_task`; guarded by `slot->iface == "task"` (the tag
  `wire_caller_slot` writes — only per-task slots carry it, so the
  back-conversion is exact).
- Reject non-reply messages (`!(msg->flags & NX_MSG_FLAG_REPLY)`):
  tasks are senders, not receivers, except for replies — refuse with
  `NX_EINVAL`.  The dispatcher won't synthesize another reply because
  reply messages don't carry `REPLY_REQUESTED`.
- Validate `task && caller_slot_active && in_flight_reply_buf != NULL`
  — if any fails, increment `reply_unbound_caller` counter and return
  `NX_EINVAL` (this case shouldn't happen post-Path-A but guards
  against fixture mistakes in tests).
- Truncation guard: `payload_len < sizeof(nx_reply_header)`,
  `payload_len > in_flight_reply_buf_len`, or `payload == NULL` →
  set `in_flight_reply_rc = NX_EINVAL`, increment `reply_truncations`,
  but **still wake the caller** so it doesn't hang forever on the
  reply_waitq.  Slice 8.0b's wrapper-emitted reply structs MUST size
  the caller's reply_buf at least `sizeof(nx_reply_header)`.
- Happy path: `memcpy(in_flight_reply_buf, msg->payload,
  msg->payload_len)`; `in_flight_reply_rc = ((const struct
  nx_reply_header *)msg->payload)->rc`; increment `replies_routed`.
- `nx_waitq_wake_one(&task->reply_waitq)`.

Adds counters (`messages_handled / replies_routed / reply_truncations
/ reply_unbound_caller`) plus `nx_posix_shim_replies_routed_for_test
/ _reply_truncations_for_test` accessors so tests can read them
without poking the struct directly.

**`test/host/slot_call_test.c` (new file, 530 lines).**  13 new TESTs:

| Group | TEST | Coverage |
|---|---|---|
| Reply-pool basics | `reply_pool_capacity_is_NX_REPLY_POOL_SIZE` | Capacity == 256, in-use == 0 after reset. |
| Reply-pool basics | `reply_pool_reset_clears_inuse_count` | Reset zeroes the bitmap. |
| Validation | `slot_call_blocking_rejects_null_args` | NULL slot or NULL msg → NX_EINVAL. |
| Validation | `slot_call_blocking_rejects_dst_slot_mismatch` | `msg->dst_slot != slot` → NX_EINVAL. |
| Validation | `slot_call_blocking_rejects_missing_src_slot` | `msg->src_slot == NULL` → NX_EINVAL. |
| Validation | `slot_call_blocking_rejects_src_not_caller_slot` | `msg->src_slot != &task->caller_slot` → NX_EINVAL. |
| Validation | `slot_call_blocking_rejects_recursive_call` | `in_flight_reply_buf != NULL` on entry → NX_EINVAL; pre-existing buf state preserved. |
| Validation | `slot_call_blocking_paused_with_reject_policy_returns_ebusy` | Mutate cloned edge's policy to REJECT, set slot pause_state to CUTTING → NX_EBUSY; no in-flight buf stashed. |
| Validation | `slot_call_blocking_ipc_send_abort_returns_eabort` | IPC_SEND hook ABORT → NX_EABORT; in-flight buf cleared; no enqueue (`pump_once == 0`). |
| Dispatcher | `dispatcher_posts_reply_when_request_has_reply_requested` | `pump_once` invokes svc handler, allocates pool entry, leaves `in_flight_calls == 0`. |
| Dispatcher | `dispatcher_does_not_post_reply_for_non_reply_requested_request` | Plain request: handler runs, no pool allocation. |
| Dispatcher | `dispatcher_in_flight_counter_returns_to_zero_after_handler` | Counter wraps the handler invocation cleanly. |
| Dispatcher | `dispatcher_synthesized_reply_carries_handler_rc_in_header` | Handler returns -42; second pump (delivery) frees the pool entry. |
| Dispatcher | `dispatcher_pool_entry_freed_after_reply_delivered` | Two pumps drive request → reply → delivery; pool returns to 0. |
| ABORT path | `dispatcher_synthesizes_eabort_reply_when_recv_aborts_reply_requested` | IPC_RECV ABORT skips handler (`g_svc_handler_count == 0`) but allocates synthetic reply; second pump delivers + frees. |
| ABORT path | `dispatcher_no_reply_synthesized_when_recv_aborts_non_reply_requested` | RECV ABORT on plain request: no synthesis. |

The fixture (`slot_call_fixture_setup`) builds a synthesized graph
mirroring the live composition's posix_shim wiring — a slot named
`"posix_shim"` bound to a placeholder component, a `"svc"` slot bound
to a `svc_test` component with a real `handle_msg` (counts
invocations + returns a configurable rc), and a `posix_shim → svc`
async-queue connection.  `nx_task_create("caller", …)` then runs
`wire_caller_slot` against this fixture (since posix_shim is now
bound), which clones the parent edge into a `caller_slot → svc`
edge — that's the edge `nx_slot_call_blocking` walks for pause
policy.

Why no full end-to-end blocking-call test here: host has no real
scheduler, so `nx_waitq_wait_with_deadline(&task->reply_waitq, 0)`
yields into the abort()-stub `cpu_switch_to`.  The full round-trip
(caller blocks → dispatcher runs → reply → caller resumes) is
exercised in the kernel ktest below.

**`test/host/Makefile` (+1 line).**  `slot_call_test.c` added to
`SRCS` after `dispatcher_test.c`.

**`test/kernel/ktest_bootstrap.c` (+106 lines).**
`slot_call_blocking_round_trip_via_uart_returns_handler_rc` — the
first end-to-end blocking call against the live kernel composition.
A fresh kthread (real `nx_task_current()` with `caller_slot_active
== true` from slice 8.0a.5) issues `nx_slot_call_blocking` to
`char_device.serial` (bound to `uart_pl011`, which has a real
`handle_msg` returning 0).  The dispatcher kthread runs the request,
posts the reply, runs `posix_shim_handle_msg` to copy the payload
into `reply_buf` and wake the caller's `reply_waitq`; the caller
resumes and `nx_slot_call_blocking` returns rc=0.  The ktest yields
the CPU (bounded `max_yields = 4096`) until the kthread sets
`g_blocking_call_finished`, then asserts rc==0 and that the pool
returned to its pre-call level (handler-rc plumbed correctly +
leak-free).  Quiesces the kthread by removing it from the runqueue
in its tail loop.

`uart_pl011`'s `handle_msg` was authored as a smoke-test stub
(returns 0 for any input).  Slice 8.0b activates the generated
typed shim; for slice 8.0a.6 the round-trip exercises only the
framework infrastructure, not op-payload semantics, so the smoke
stub suffices.

### Key decisions

- **Atomic-bitmap pool over a free-list.**  Free-list-with-CAS is
  also viable but the bitmap is simpler (no node header carved out
  of the entry itself, no ABA corner cases).  Single-CPU v1 has just
  one producer (the dispatcher kthread / pump_once caller) so the
  CAS contention is zero in practice; SMP will replace this with
  per-CPU pools (each CPU drains its own queue, allocates from its
  own pool — no cross-CPU contention at all).
- **`build_reply` zeroes the entire 512-byte payload, not just the
  header.**  Slice 8.0b's wrapper-emitted reply structs append op-
  specific output fields; clearing now keeps every reply bit-stable
  across builds and avoids exposing arbitrary stale memory through
  the wrapper's `memcpy` into the caller's buffer.
- **REDIRECT v1 fails closed with NX_ENOENT.**  The redirect chain
  walker would need to re-enter the body without trampling
  `in_flight_reply_buf` state.  Doable but adds a second copy of the
  validate / stash / hook / cap-scan / enqueue sequence inside a
  bounded `redirect_depth++` loop, and v1 has zero REDIRECT
  consumers.  Lifted to a shared helper when one lands.
- **Force `NX_MSG_FLAG_REPLY_REQUESTED` unconditionally in the body.**
  The wrapper sets it (per the SLOT-CALL-API.md template) but a
  hand-rolled caller could omit it and silently hang the reply waitq.
  Belt-and-suspenders.
- **Pool exhaustion is a panic, not a return.**  A blocked caller
  can't recover from a dropped reply — `nx_waitq_wait_with_deadline`
  is indefinite (budget_ns == 0).  Failing loud at allocation time
  surfaces the bug immediately rather than a hung process.
- **`task_from_caller_slot` guards via `iface == "task"`.**  Pure
  `container_of` would mis-decode any slot anywhere on the stack
  that happened to be at the right offset relative to its embedding
  struct.  The iface tag is set exclusively by `wire_caller_slot`,
  so checking it gates the back-conversion to known-good entries.
- **Truncation/oversized/null-payload reply still wakes the caller.**
  Setting `in_flight_reply_rc = NX_EINVAL` and waking is preferable
  to silently leaving the caller blocked — the wrapper sees the
  error and returns it instead of hanging.  The truncation counter
  observed via `nx_posix_shim_reply_truncations_for_test` lets ktests
  assert the malformed-reply path without depending on
  reply_waitq-induced state.
- **Host-side coverage is dispatcher + validation; ktest is round-trip.**
  No real `cpu_switch_to` on host — `nx_waitq_wait_with_deadline`
  with budget 0 would abort the test runner.  Instead host TESTs
  exercise `nx_dispatcher_pump_once` directly and use the four
  validation paths that bail before any waitq park.
- **Fixture mirrors live composition rather than the production
  `g_posix_shim`.**  Host tests don't run `nx_framework_bootstrap`,
  so production posix_shim is unbound; `wire_caller_slot` would
  soft-skip and the per-task slot wouldn't get an outgoing edge.
  Building a synthesized posix_shim binding in the fixture (slot
  + component + svc dep) means `wire_caller_slot` actually clones
  the edge, which is what each TEST's pause-policy / dispatcher
  assertions need.

### Tests at end of Session 90

- `make test` → **519/519 pass** (93 python + **305** host (was 289;
  +16) + **121** kernel (was 120; +1)).
- `make test-interactive` → **7/7 pass**.
- `make verify-iface-fresh` → 0 drift.
- `make verify-registry` → 0 findings.

## Next Steps

Slice **8.0a.7** — `posix_shim_on_dep_swapped` STATE_LOST handler +
`nx_handle_table_invalidate_for_slot()`.  When a dep swap forces
handle-table invalidation, the slice-3.9 mechanism walks every handle
whose backing slot just lost state and clears it; userspace sees
`NX_EBADF` on next op.  Required before live-swap is safe across
non-trivial state.

After 8.0a.7, slice 8.0a.8 lands the cross-cutting test infrastructure
(mock component, hook-chain inspector, recompose event logger,
pause-injector fixture, cap-forgery harness, equivalence-runner
macro) — closes 8.0a.

---

**Files Changed (`sources/nonux/`, commit `34dd1cb`):**
- `framework/dispatcher.c` — reply-message pool + dispatcher reply
  synthesis under `slot->in_flight_calls` + ABORT-path EABORT reply +
  pool reset on `nx_dispatcher_reset`.
- `framework/dispatcher.h` — test-surface declarations for the
  reply-pool helpers.
- `framework/slot_call.c` — full body for `nx_slot_call_blocking`
  (validate / pause-state walk / stash / hook / cap-scan / enqueue /
  block / clear / return) + edge lookup helper.
- `framework/slot_call.h` — `struct nx_reply_header`,
  `NX_REPLY_POOL_SIZE`, `NX_REPLY_PAYLOAD_MAX`.
- `components/posix_shim/posix_shim.c` — `handle_msg` switched from
  NX_EINVAL stub to real reply-routing body + counters + test-only
  accessors.
- `test/host/Makefile` — `slot_call_test.c` added to `SRCS`.
- `test/host/slot_call_test.c` — 13 new TESTs covering reply-pool
  basics, validation paths, dispatcher reply-posting, and ABORT-path
  synthesis.
- `test/kernel/ktest_bootstrap.c` — first end-to-end blocking-call
  ktest against the live kernel composition (uart_pl011 round-trip).

**Files Changed (`proj_docs/nonux/`, this commit):**
- `logs/session-90-8.0a.6-reply-path.md` — this log.
- `HANDOFF.md` — Current Status / Phase checklist / Next Actions / Session Logs advanced; Session 85 archived.
- `IMPLEMENTATION-GUIDE.md` — slice 8.0a.6 row marked ✓; Phase 8 status + Last Updated footer refreshed.
- `HANDOFF-ARCHIVE.md` — Session 85 entry pulled forward from HANDOFF per the keep-last-5 convention.
