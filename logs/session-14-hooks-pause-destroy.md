# Session 14: hooks + pause protocol + destroy guard (slice 3.8)

**Date:** 2026-04-21
**Phase:** 3 — Component framework
**Branch:** master

---

## Goals

- Land the hook framework the rest of the framework already assumes
  exists — a per-hook-point chain with priority-sorted insert, safe
  unregister-during-dispatch, and a typed context union.
- Turn `nx_component_pause` / `_resume` into the real Session-3 pause
  protocol: cutoff → drain → `pause_hook` → `ops->pause`, with slot
  pause-state visible to the IPC router.
- Teach the IPC router to honour per-edge `NX_PAUSE_QUEUE` /
  `REJECT` / `REDIRECT` policy when the destination slot is paused,
  including a loop guard on REDIRECT fallbacks.
- Close the last lifecycle gap: `nx_component_destroy` must refuse
  while the component is still bound to a slot, matching
  `nx_component_unregister`'s existing check.
- Fold the `spawns_threads ⇒ pause_hook` rule into the manifest
  schema and validator so the requirement is enforced at build time.

## What Was Done

### Hook framework — `framework/hook.{h,c}` (~250 LOC)

Per-hook-point chains indexed by `enum nx_hook_point`, caller-owned
`struct nx_hook` storage. Seven hook points land in 3.8:
`IPC_SEND`, `IPC_RECV`, and the four lifecycle verbs around
`component_{enable,disable,pause,resume}`, plus `SLOT_SWAPPED` (which
piggy-backs on the registry's existing swap event — runtime
dispatch of `SLOT_SWAPPED` is slice 3.9's job once the registry
has a post-commit notifier).

**Chain shape.** Plain `struct nx_hook *_next` singly-linked, int
priority (lower runs first), stable insertion within a priority
bucket, O(chain-length) registration. Scale is observability /
tracing, not data-path fan-out.

**Typed context union.** `struct nx_hook_context` carries three
arms — `.ipc` for IPC_SEND / IPC_RECV, `.comp` for lifecycle, `.swap`
for swap events. One `void *data` pointer would have made every hook
blind-cast; the union gives authors a typed view with zero runtime
cost.

**Unregister-during-dispatch: mark-then-sweep.** A global
`g_dispatching` counter bumps on entry to `nx_hook_dispatch` and
decrements on return. Unregister on an in-flight chain flips
`_dead = true` on the node and defers the unlink; the outermost
dispatch sweeps every dead node from every chain before returning.
Dispatch walks with `nxt = h->_next` snapshotted before invoking the
callback, so newly-registered hooks aren't observed mid-walk.

### Slot pause state + fallback — `framework/registry.{h,c}`

New `struct nx_slot` fields, both initialised explicitly in
`nx_slot_register` (see below):

- `_Atomic(enum nx_slot_pause_state) pause_state` — `NONE →
  CUTTING → DRAINING → DONE`. Lives on the slot, not the
  bound component: the router holds slot pointers and the component
  can swap mid-pause, so the flag must be slot-side. Accessed via
  `nx_slot_set_pause_state` (release-store) and
  `nx_slot_pause_state` (acquire-load) — barriers are in from day
  one so the SMP upgrade is flip-the-barrier, not restructure.
- `struct nx_slot *fallback` — REDIRECT target. Written via
  `nx_slot_set_fallback` (rejects self-loops and unregistered
  slots). `NULL` + `REDIRECT` policy is fail-closed (`NX_ENOENT`).

**Atomic init.** Without `atomic_init`, a caller's `struct nx_slot`
zeroed by compound-literal assignment is undefined per C11 even
though every host compiler we test on happened to do the right
thing. `nx_slot_register` now calls `atomic_init(&s->pause_state,
NX_SLOT_PAUSE_NONE)` and clears `fallback` explicitly, so the
registry-visible state is well-defined regardless of how the caller
zero-initialised the struct.

**Helper rename.** The previously-private
`component_is_active_anywhere` helper is now public as
`nx_component_is_bound`, with a companion
`nx_component_foreach_bound_slot` that walks every slot whose
`active` pointer equals the given component. The pause protocol
uses both.

### Pause protocol — `framework/component.c`

`nx_component_pause` now runs:

1. `dispatch_lifecycle_hook(NX_HOOK_COMPONENT_PAUSE, ...)`. Hook
   ABORT returns `NX_EABORT` with no state change.
2. Walk every bound slot and `nx_slot_set_pause_state(... CUTTING)`.
   New `nx_ipc_send` calls into that slot immediately apply the
   edge's pause policy.
3. Walk again and set `DRAINING`, then drain the inbox synchronously
   via `nx_ipc_dispatch(slot, (size_t)-1)`. Drain reentry — a
   handler calling `nx_ipc_send` to a *different* slot — is fine;
   only the receiving slot's pause flag matters.
4. If `ops->pause_hook` exists, call it. A non-zero return aborts
   the pause: the verb returns that error and the lifecycle state
   stays `ACTIVE`. (Slot may be left in `DRAINING`; slice 3.9 will
   harden the rollback once the kernel boot path owns the dispatcher.)
5. If `ops->pause` exists, call it. Same abort-on-error semantics.
6. Walk every bound slot and set `DONE`.
7. Drive the registry state transition `ACTIVE → PAUSED`.

`nx_component_resume` is symmetric: hook dispatch → `ops->resume` →
per-slot `pause_state = NONE` + `nx_ipc_flush_hold_queue(slot)` →
registry transition `PAUSED → ACTIVE`.

**Unbound component is still a legal pause target.** A registered
component that was never bound to a slot still transitions — the
bound-slot walks are no-ops, `pause_hook` / `pause` still fire, and
recomposition can progress.

### Destroy guard — `framework/component.c`

`nx_component_destroy` now refuses with `NX_EBUSY` when
`nx_component_is_bound(c)` is true, complementing
`nx_component_unregister`'s existing check. Guards against the
sequence "disable → destroy while still bound → later unregister"
which would otherwise leave `slot->active` dangling at a destroyed
component.

### IPC router — `framework/ipc.{h,c}` (~150 LOC added)

Everything is unchanged on the fast path (pause_state == NONE).
Slow path:

- **IPC_SEND hook.** Fires before edge lookup, `ctx.edge == NULL`.
  ABORT → `NX_EABORT` (new error code).
- **Pause-policy branch.** When the dst slot is not in `NONE`:
  - `NX_PAUSE_QUEUE`: append to a per-`(src, dst)` hold queue
    (side-table keyed by slot pointer pair; keeps `struct nx_slot`
    type-stable and lets a slot hold messages from many callers).
  - `NX_PAUSE_REJECT`: return `NX_EBUSY`.
  - `NX_PAUSE_REDIRECT`: recurse on `do_send(msg, depth + 1)` after
    setting `msg->dst_slot = dst->fallback`. `dst->fallback == NULL`
    → `NX_ENOENT`; `depth >= 4` → `NX_ELOOP` (new error code).
    The caller's `msg->dst_slot` reflects the final hop on return —
    documented behaviour; the async path relies on it as the queue
    key for the dispatcher.
- **IPC_RECV hook.** Fires inside `nx_ipc_dispatch` per message
  before the cap scan / handler. ABORT drops the message (counts
  as dispatched) and continues the drain — observability hooks can
  filter noise without stalling the queue.
- **`nx_ipc_flush_hold_queue(dst)`.** Called by the resume path.
  Detaches every hold entry with that `dst`, replays each message
  via `nx_ipc_send` so hooks / pause policy / cap-scan run normally,
  then frees the entry.
- **`nx_ipc_hold_queue_depth(src, dst)`.** Test introspection.
- **`nx_ipc_reset`.** Now frees both inbox and hold-queue side
  tables.

### Manifest validation — `tools/schemas/manifest.schema.json`, `tools/validate-config.py`

Two new optional booleans on the manifest: `spawns_threads` and
`pause_hook`. New rule in `validate-config.py`:
`spawns_threads == true` without `pause_hook == true` fails the
validation with an explicit "pause protocol needs a hook to quiesce
component-spawned threads" error. Two tests cover both directions
(`test_spawns_threads_without_pause_hook_errors` /
`test_spawns_threads_with_pause_hook_ok`).

### Tests — `test/host/component_{hook,pause,destroy}_test.c`

- **`component_hook_test.c` (10 tests)** — the hook framework in
  isolation. Register / unregister / dispatch, priority ordering
  (ascending, stable within bucket), abort short-circuit,
  self-unregister during dispatch, peer-unregister during dispatch
  (deferred removal), empty-chain and out-of-range dispatch, reset.
- **`component_pause_test.c` (14 tests)** — the pause protocol and
  router policy. Pause drives slot through `CUTTING → DRAINING →
  DONE`; resume clears it and calls `ops->resume`. `QUEUE` / `REJECT`
  / `REDIRECT` each have a happy-path test plus REJECT returns
  `NX_EBUSY`, REDIRECT with `fallback == NULL` returns `NX_ENOENT`,
  and the `A↔B` ping-pong loop returns `NX_ELOOP`. IPC_SEND /
  IPC_RECV hooks fire per send / dispatch; IPC_SEND abort prevents
  routing. Lifecycle hooks fire on each verb. `pause_hook` returning
  an error leaves the component in `ACTIVE`. Dispatcher-only
  components (no `pause_hook`) still quiesce correctly. A registered
  but never-bound component still transitions.
- **`component_destroy_test.c` (4 tests)** — `NX_EBUSY` while still
  bound; success after unbind; success on a never-bound component;
  guard blocks even when bound to multiple slots simultaneously.

### Test counts

- Python: **49** (47 → +2 for the `spawns_threads` rule).
- Host:   **128** (100 → +10 hook + 14 pause + 4 destroy).
- Kernel: **6** (unchanged).
- Total:  **183 / 183 pass**, 0 leaks, 0 errors, exit 0.
- `make verify-registry` still reports "ran R2,R4; ai-verified R1,
  R3,R5,R6,R7,R8" (no new rule — see below).

## Decisions

**Hook chain data structure is plain `struct nx_hook *_next`.** The
repo has no list primitive yet; a singly-linked chain with
priority-ordered inserts costs O(chain) on register and O(chain) on
dispatch, which is fine for observability scale. Promoting to a
skiplist / intrusive list primitive can wait until a real workload
justifies it.

**Hold queue is a side table, not a field on `struct nx_slot`.**
A slot can simultaneously hold messages from many distinct
senders, so the natural key is `(src, dst)`, not just `dst`. Keeping
the storage external also keeps `struct nx_slot` stable — which is
the same reason the inbox lives in `ipc.c`'s `g_queues` table.

**REDIRECT retargets `msg->dst_slot` in place.** The alternative —
making a stack-allocated copy of the message for each recursive
hop — can't be enqueued if the recursive edge is async, because the
enqueue keeps a pointer to the message. Mutating in place means the
enqueued message carries the final dst, which is what the dispatcher
needs, at the cost of the caller's message reflecting the redirect.
Documented in `ipc.c`.

**Pause-hook failure leaves the component in `ACTIVE`, not an
intermediate state.** The verb returns the hook's error code and
does not drive the registry transition; slot `pause_state` is left
mid-protocol (likely `DRAINING`). Slice 3.9 will add a rollback path
once the kernel-side dispatcher owns drain lifetimes. In the host
build the test is `ASSERT_EQ_U(c.state, NX_LC_ACTIVE)`; it
explicitly does not assert the slot state, to avoid baking the
current intermediate value into the spec.

**Atomic init.** The original design assumed caller-owned
`struct nx_slot` zero-init (via compound literal or memset) was
good enough for `_Atomic(enum) pause_state`. In practice, C11's
guarantees on plain-assignment of a struct containing `_Atomic`
members are weaker than that; the cleanest fix was to call
`atomic_init` inside `nx_slot_register` so the registry's published
view of the field is always well-defined, regardless of caller
hygiene. (Discovered via test regressions when the IPC fixtures'
compound literals occasionally published a non-zero `pause_state`.)

**No new AI-verification rule.** R1–R8 stay. Hook handlers run on
the dispatcher — same slot-resolve-locality domain as message
handlers — so R8 already covers them. [AI-RULES.md](../AI-RULES.md)
gets a one-line note under R8 pointing at hooks as an in-scope call
site; the machine check for R8 is still deferred to the Layer-2
rubric.

**`pause_hook` deadline.** DESIGN calls for a 1 ms wall-clock
deadline. Host-side has no timer; slice 3.8 calls the hook
synchronously and trusts the component. The deadline lands with
slice 3.9 on the kernel boot path, where the dispatcher thread owns
a monotonic clock.

## What's Next

**Slice 3.9 — wire registry + framework into the kernel boot path.**
Walk the `nx_components` linker section, register each descriptor's
slot + component, call `nx_resolve_deps` → `nx_component_init` →
`nx_component_enable` in dependency order. Swap the host FIFO inbox
for an MPSC lock-free queue plus a per-CPU dispatcher thread. Boot
log dumps the initial composition via `nx_graph_snapshot_to_json`.
This also brings `pause_hook`'s 1 ms deadline online, a rollback
path for mid-protocol pause failures, and the real runtime
`SLOT_SWAPPED` hook dispatch.

**Phase 4 — context switch + scheduler.** `sched_rr` becomes the
first component under the new framework. `verify-registry.py` R8's
machine extension becomes feasible once ISR / kthread entry points
have a tagging convention.

## Files Touched

- `framework/hook.h`, `framework/hook.c` — new.
- `framework/registry.h`, `framework/registry.c` — slot pause state
  + fallback + atomic_init + bound-slot introspection + NX_ELOOP /
  NX_EABORT.
- `framework/component.h`, `framework/component.c` — `pause_hook`
  op + pause / resume protocol + destroy guard + lifecycle hook
  dispatch.
- `framework/ipc.h`, `framework/ipc.c` — hold queue + pause-policy
  router + IPC_SEND / IPC_RECV hooks + flush + REDIRECT loop guard.
- `test/host/component_{hook,pause,destroy}_test.c` — new (28 tests).
- `test/host/Makefile` — three new test files + `hook.c` under test.
- `tools/schemas/manifest.schema.json` — optional `spawns_threads`
  / `pause_hook`.
- `tools/validate-config.py` — `check_pause_hook_required`.
- `tools/tests/test_validate_config.py` — two tests.
- `proj_docs/nonux/{DESIGN,AI-RULES,HANDOFF,IMPLEMENTATION-GUIDE,
  README}.md`, this session log.
