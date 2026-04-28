# Session 12: IPC router (slice 3.6)

**Date:** 2026-04-21
**Phase:** 3 — Component framework
**Branch:** master

---

## Goals

- Land the IPC router described in DESIGN.md §IPC Framework: typed
  component ops, a message router with async-default / sync-shortcut
  dispatch, per-slot inbox queues, typed capabilities in messages,
  send- and recv-side cap scanning, and `slot_ref_retain` /
  `slot_ref_release` for cap-held slot refs.
- Plumb the typed `struct nx_component_ops` into the component
  descriptor (promoted from `const void *` since slice 3.4) and into
  `nx_component` itself (so the router can find handlers).

## What Was Done

### Registry + descriptor type plumbing

- `framework/registry.h`: forward-declares `struct
  nx_component_descriptor` and adds two caller-owned fields to
  `nx_component`:
  - `void *impl` — the `self` pointer passed to every
    `ops->handle_msg(self, msg)` invocation.
  - `const struct nx_component_descriptor *descriptor` — back-link to
    the descriptor so the router can reach `descriptor->ops`.
  Like `manifest_id` / `state`, these fields are caller-owned; the
  registry never touches them. Slice 3.9's descriptor walker sets
  both during boot; today's tests set them by hand.
- `framework/component.h`: introduces `struct nx_component_ops`
  (init / enable / pause / resume / disable / destroy / handle_msg —
  all nullable) and types `nx_component_descriptor.ops` as `const
  struct nx_component_ops *`. The lifecycle callbacks are declared
  now so slice 3.9 can plumb them into `nx_component_init` / etc.
  without changing the descriptor shape; only `handle_msg` is
  exercised by the router in this slice.
- Existing 3.4 tests (`component_deps_test.c`) updated to pass a
  real all-NULL `struct nx_component_ops` instead of an `int`
  sentinel.

### `framework/ipc.h` — types and API

- `enum nx_cap_kind` (currently just `NX_CAP_SLOT_REF`;
  `NX_CAP_HANDLE` / `NX_CAP_VMO_CHUNK` arrive with Phases 5 / 7).
- `enum nx_cap_ownership` = `NX_CAP_BORROW` (default) /
  `NX_CAP_TRANSFER`.
- `struct nx_ipc_cap` — `{ kind, ownership, union u{slot_ref},
  cap_id, claimed }`. `claimed` is the handshake the router uses to
  tell retained caps from dropped caps on handler return.
- `struct nx_ipc_message` — `{ src_slot, dst_slot, msg_type, flags,
  n_caps, caps, payload_len, payload, _next }`. `_next` is
  framework-owned (queue linkage) and callers must not touch it.
- Public API:
  - `nx_ipc_send(msg)` — router entry. Looks up the connection edge
    from `src_slot` to `dst_slot` (`NX_ENOENT` if none), runs
    `nx_ipc_scan_send_caps`, then dispatches SYNC (direct call) or
    ASYNC (enqueue).
  - `nx_ipc_dispatch(slot, max)` — drain helper for tests. Pops up
    to `max` messages from `slot`'s inbox, invokes the active
    component's `ops->handle_msg(impl, msg)`, runs the recv-cap
    sweep, returns count dispatched. Stops on first handler error.
    Slice 3.9 replaces explicit calls to this with a pinned per-CPU
    dispatcher thread.
  - `nx_ipc_inbox_depth(slot)` / `nx_ipc_reset()` — introspection
    and test-only teardown.
  - `nx_ipc_scan_send_caps(sender, msg)` — returns `NX_EINVAL` on
    the first slot-ref cap whose target has no registered incoming
    edge from `sender`.
  - `nx_ipc_scan_recv_caps(recv, msg)` — returns the count of
    unclaimed `TRANSFER` caps (borrows are silently dropped per
    DESIGN).
  - `nx_slot_ref_retain(self, cap, purpose, out_conn)` — promotes a
    borrowed/transferred slot ref into a registered connection edge
    owned by `self`; marks `cap->claimed = true`; returns the new
    connection via `out_conn` for the caller to release later.
  - `nx_slot_ref_release(self, target)` — walks `self`'s outgoing
    edges, finds the one to `target`, unregisters it. No-op if the
    edge is already gone.

### `framework/ipc.c` — the router

- Per-slot inbox stored in a side table (`g_queues` — a singly-linked
  list of `struct ipc_slot_queue`) rather than extending
  `struct nx_slot`. Keeps the registry API clean of IPC-specific
  fields; slice 3.9 can swap the lookup for a hash without touching
  the slot type.
- Connection-edge lookup via `nx_slot_foreach_dependency(from, cb,
  ctx)` + a first-match callback. Registry's per-slot outgoing list
  makes this O(deg(from)) without a global scan.
- `invoke_handler` is the single code path for SYNC and ASYNC post-
  dequeue; it checks for `active->descriptor->ops->handle_msg`, calls
  it, then unconditionally runs `nx_ipc_scan_recv_caps` — even when
  the handler errors, borrowed caps must drop and transfers must be
  accounted for.
- Send-time: forged caps rejected before the connection edge is
  even consulted; the dispatch path and the queue stay clean.
- `nx_ipc_reset()` free s every queue head but leaves caller-owned
  `nx_ipc_message` storage alone — it clears `_next` links so
  dangling pointers don't linger after a reset.
- `nx_slot_ref_retain()` calls `nx_connection_register(self, target,
  NX_CONN_ASYNC, false, NX_PAUSE_QUEUE, &err)` — retain doesn't
  carry per-connection mode / stateful / policy metadata yet. When
  that matters (slice 3.8 with hooks on retained edges), the cap
  payload can grow a mode hint.

### Tests — 16 new in `test/host/component_ipc_test.c`

- **Shared fixture** — `ipc_fixture_init(&f, mode)` builds a sender
  + receiver pair, registers both slots + components, binds each to
  its slot, wires the descriptor / `impl` fields on the receiver so
  the router can dispatch, and registers a sender→receiver edge in
  the chosen mode.
- **Sync:** `sync_send_invokes_handler_directly`,
  `sync_send_without_edge_returns_enoent`,
  `sync_send_to_slot_without_active_handler_returns_enoent`,
  `send_rejects_null_or_incomplete_message`.
- **Async:** `async_send_enqueues_and_dispatch_drains` (FIFO order +
  granular dispatch count), `dispatch_stops_on_handler_error`,
  `dispatch_on_empty_slot_is_zero`.
- **Cap scanning:**
  `scan_send_caps_accepts_registered_slot_ref`,
  `scan_send_caps_rejects_forged_slot_ref`,
  `send_rejects_forged_caps` (the real `nx_ipc_send` path — handler
  must not fire),
  `scan_recv_caps_drops_borrowed_silently`,
  `scan_recv_caps_flags_unclaimed_transfer`.
- **retain/release:**
  `slot_ref_retain_registers_edge_and_claims_cap` (edge appears +
  recv-scan no longer flags the claimed transfer),
  `slot_ref_release_unregisters_edge` (double-release is safe),
  `slot_ref_retain_rejects_bad_caps` (NULL, wrong kind, null target,
  already-claimed).
- **Reset:** `ipc_reset_clears_all_inboxes`.

## Results

- `make test` → **137/137 pass (31 python + 100 host + 6 kernel), 0
  leaks, 0 errors, exit 0**.
- Production `kernel.bin` unchanged — `framework/ipc.c` is built
  into host tests only. Slice 3.9 brings it into the kernel build.

## Key Findings

- **Side table for inboxes, not a field on `nx_slot`.** Extending
  `struct nx_slot` with `void *fw_private` would have been easier to
  write but harder to maintain: every future framework subsystem
  would want its own slot-attached storage, turning the slot struct
  into a junk drawer. Lookup-by-pointer in `g_queues` is O(slots)
  but the registry API stays clean; slice 3.9 can swap to a hash or
  a generational side index without any header churn.
- **`invoke_handler` runs recv-scan even on handler error.** First
  draft only ran the sweep on success; that left unclaimed transfer
  caps invisible when a handler returned non-zero. Running the
  sweep unconditionally matches DESIGN.md's "drop borrowed refs on
  handler return, flag unclaimed transfers" without qualification.
- **`_next` in the public message type is an odd public field.**
  Callers must not set or inspect it, but it has to live on the
  message struct because the inbox uses the message itself as its
  list node (no per-message allocation). Documented it as
  framework-owned and left a comment at the declaration. An
  alternative — allocating a queue-node wrapper per message — adds
  an alloc per send and a potential failure mode on enqueue. For
  the kernel side (slice 3.9), the MPSC queue will also use an
  intrusive next pointer; keeping the field here means the kernel
  doesn't need a different message type.

## Decisions

- `struct nx_component_ops` ships with the full lifecycle callback
  set even though only `handle_msg` is exercised in 3.6. Adding
  them now keeps the descriptor type stable for slice 3.9's boot
  path; delaying would force a second update of every registered
  component.
- `nx_ipc_send` refuses to send without a registered connection
  edge. An "open" send that implicitly creates an edge would defeat
  the R1–R3 audit invariants the registry was built to support.
- Retain registers an ASYNC + non-stateful + queue edge by default.
  The cap type doesn't carry richer metadata yet; when slice 3.8
  (hooks) needs mode-per-retain, the cap grows a small metadata
  block rather than reshaping the router API.
- Dispatch stops on first handler error. An alternative — log and
  continue — would hide mis-behaving components. Stopping surfaces
  the problem and lets the caller decide.
- `nx_ipc_reset()` is test-only. Production code never resets the
  router; the kernel uses lifecycle teardown instead. The
  `nx_graph_reset` / `nx_ipc_reset` pair at the top of every test
  is part of the host-side convention, not the production API.

## Open / Deferred

- **MPSC lock-free queue.** Slice 3.9 swaps the singly-linked FIFO
  for a Vyukov-style per-CPU queue so ISRs can enqueue from any
  context. The dequeue contract (single consumer per queue) already
  matches; only the enqueue path needs atomicity.
- **Pause-protocol interaction.** Slice 3.8 teaches the router to
  check each connection's pause flag on dispatch and route per the
  pause policy (queue / reject / redirect). Today every message
  dispatches regardless of component state.
- **Lifecycle → ops plumbing.** `nx_component_init` etc. still only
  update the state field; they don't call `ops->init` yet. Slice
  3.9 threads the ops table through the lifecycle verbs so a real
  component's init actually runs.
- **Hooks on send/recv.** DESIGN.md places `HOOK_IPC_SEND` /
  `HOOK_IPC_RECV` on the router path. Slice 3.8 wires them up.

## Next

- **Slice 3.7 — `tools/verify-registry.py`.** Static checker for
  rules R1–R8 from DESIGN.md §AI Verification. R7 is the
  regenerate-and-diff gate the deterministic generator from slice
  3.5 was designed to support; R1–R6 are per-component code checks;
  R8 is the ISR-to-slot-deref call-graph check. Wire as `make
  verify-registry` and make it a prerequisite of `make` and `make
  test`.
