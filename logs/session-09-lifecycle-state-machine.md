# Session 9: Lifecycle state machine (slice 3.3)

**Date:** 2026-04-21
**Phase:** 3 — Component framework
**Branch:** master

---

## Goals

- Formalise the component lifecycle on top of the registry. Slice 3.2 left
  `nx_component_state_set` as a raw state-setter that accepts any transition;
  slice 3.3 is the layer that enforces the legal edge set (UNINIT → READY →
  ACTIVE → PAUSED → READY → DESTROYED, etc.) and exposes it as six
  self-explanatory verbs: `init`, `enable`, `pause`, `resume`, `disable`,
  `destroy`. Every transition still flows through `state_set`, so
  subscribers and the change log observe the lifecycle automatically —
  3.2's plumbing didn't need to change.

## What Was Done

### `framework/component.h`

- Six public verbs returning `NX_OK` / `NX_EINVAL` / `NX_ENOENT` /
  `NX_ESTATE`.
- `nx_lifecycle_transition_legal(from, to)` predicate — exposed so tests
  can introspect the matrix and so future callers (the recomposer, the
  invariant harness) can decide-without-mutating.
- `nx_lifecycle_state_name()` — lowercase enum-to-string; reuses the same
  names the JSON serialiser already produces (`"uninit"`, `"ready"`, ...).
- Header-level ASCII diagram of the legal edges.

### `framework/component.c`

- `transition_from(c, expected_from, to)` — the single helper every verb
  (except `disable`) calls. Returns `NX_EINVAL` on NULL, `NX_ESTATE` if
  the current state doesn't match `expected_from` OR if the matrix rejects
  the edge (belt-and-suspenders), otherwise delegates to
  `nx_component_state_set`.
- `nx_component_disable` is the only verb with two legal source states
  (ACTIVE or PAUSED), so it's inlined rather than forced through the
  single-source helper.
- `nx_component_init` does UNINIT → READY in one step. The explicit
  `NX_LC_INIT` value is matrix-legal (UNINIT → INIT → READY) but unused
  by v1 verbs; it's reserved for async bring-up so the framework can
  later split `init` into start/complete halves without breaking the
  matrix.

### Legal edges (matrix)

```
UNINIT → INIT               (reserved, not driven by v1 verbs)
UNINIT → READY              (init)
INIT   → READY              (reserved)
READY  → ACTIVE             (enable)
READY  → DESTROYED          (destroy — terminal)
ACTIVE → PAUSED             (pause)
ACTIVE → READY              (disable)
PAUSED → ACTIVE             (resume)
PAUSED → READY              (disable)
DESTROYED → *               (nothing — no resurrection)
```

Explicitly rejected:

- UNINIT → ACTIVE / PAUSED / DESTROYED (skipping the machine)
- ACTIVE → DESTROYED, PAUSED → DESTROYED (must disable first)
- Any → UNINIT (no going backwards)
- READY → READY, ACTIVE → ACTIVE (no self-loops)

### Tests added (13 new, total now 75 host + 6 kernel = 81)

- **Matrix predicate:** `matrix_allows_every_expected_legal_edge`,
  `matrix_rejects_known_illegal_edges`, `state_name_covers_every_enum`.
- **Per-verb happy + rejection:** `init_drives_uninit_to_ready`,
  `enable_requires_ready`, `pause_and_resume_toggle_active_and_paused`,
  `disable_accepts_active_and_paused`, `destroy_requires_ready`.
  Each verb exercises both the success path and at least one illegal
  prior-state call that must return `NX_ESTATE`. `destroy_requires_ready`
  also asserts every verb returns `NX_ESTATE` once the component is
  DESTROYED (no resurrection).
- **Argument validation:** `lifecycle_api_rejects_null_component`,
  `lifecycle_api_rejects_unregistered_component` (tests the NX_ENOENT
  path that bubbles up from `state_set` via the transition helper).
- **Event plumbing:** `lifecycle_transitions_emit_state_events_and_bump_gen`
  — subscribes an `NX_EV_COMPONENT_STATE`-only callback, walks a full
  init→enable→pause→resume→disable→destroy cycle, asserts 6 events with
  the expected (from, to) pair on the last one, and confirms rejected
  transitions DO NOT fire events or bump the generation.
- **Full happy path:** `full_happy_path_runs_clean`.
- **100-cycle residue check:** `lifecycle_cycle_100_iterations_clean` —
  register/init/enable/disable/destroy/unregister × 100; asserts
  `nx_graph_component_count()` returns to 0 each round. Requires
  resetting `.state = NX_LC_UNINIT` on the caller-owned struct between
  iterations because the registry treats `nx_component` as caller-owned
  storage and doesn't touch it on unregister. (Fine — this is the same
  rule the 3.1 cycle test already documented; just calling it out
  explicitly for the lifecycle version.)

### Wiring

- Added `component_test.c` to `test/host/Makefile` SRCS.
- Added `../../framework/component.c` to the SUT list in the same file.
- Existing `vpath %.c` rule already covered `../../framework`, so no
  further wiring needed.

## Results

- `make test` → **81/81 pass (75 host + 6 kernel), 0 leaks, 0 errors,
  exit 0**.
- `make` (production kernel) still builds clean — `kernel.bin` is
  unchanged this slice (framework is host-side only until slice 3.9
  wires it into boot).

## Key Findings

- **`state_set` stays a raw setter.** I briefly considered adding the
  matrix check inside `nx_component_state_set` itself, but that would
  have broken the two existing 3.2 tests that exercise it with a
  UNINIT → ACTIVE jump (`component_state_set_updates_state_and_bumps_gen`
  and `subscribe_observes_full_event_taxonomy` — both use the raw setter
  as a convenient way to drive an `NX_EV_COMPONENT_STATE` event without
  routing through the lifecycle API). Keeping `state_set` raw lets
  component.c own the state-machine enforcement without reshaping the
  change-event taxonomy test. The matrix check is centralised in the
  transition helper instead, which is the only "sanctioned" mutator that
  callers outside component.c should use.
- **INIT state kept but unused.** The enum already had `NX_LC_INIT`
  defined in 3.1. I made UNINIT → INIT matrix-legal (and INIT → READY)
  so async bring-up can split `init()` later without a matrix change.
  `nx_component_init` itself does the atomic UNINIT → READY in v1.
- **No `cycle100` residue check on the registry counters beyond
  component_count.** Slots aren't touched by the lifecycle test, so the
  existing 3.1 cycle test (`register_unregister_cycle_leaves_no_residue`)
  already covers slot/connection residue. Keeping the lifecycle
  cycle test focused on the component side avoids overlap.
- **Registry + framework now ~800 LOC combined** (registry ~750,
  component ~90). component.c is deliberately small; the matrix is the
  interesting part, not the verbs.

## Decisions

- Matrix as an open-coded switch rather than a 2D bool array.
  Nine edges, six states — one screen, reads more obviously than sparse
  integer indexing. Easy to extend when PAUSING lands with the pause
  protocol.
- `disable` accepts both ACTIVE and PAUSED. DESIGN §Component Lifecycle
  permits both; forcing a PAUSED component to resume before disabling
  would be gratuitous.
- Verbs are thin wrappers; all interesting logic is in the predicate and
  the single `transition_from` helper. This keeps the per-verb failure
  modes identical and keeps the surface area small for the future static
  checker (R1/R8).
- Lifecycle verbs return `NX_ENOENT` (via `state_set`) when the component
  isn't in the registry, not `NX_ESTATE`. Rationale: the error tells
  the caller "you're holding a stale pointer", which is different from
  "you're in the wrong state". The transition helper's early matrix
  check still runs first, but since `state_set` performs the registry
  lookup and has the authoritative answer, the helper defers to its
  return code.

## Open / Deferred

- **PAUSING intermediate state.** DESIGN §Component Lifecycle shows
  `ACTIVE → PAUSING → PAUSED` (cutoff + drain). For v1 single-core,
  `nx_component_pause` does the direct ACTIVE → PAUSED transition. The
  per-slot `pause_flag` machinery (DESIGN §Pause Protocol) lands with
  slice 3.8 (hooks on connection edges + slot mutation events) and
  Phase 8 (runtime recomposition). When it does, the matrix grows two
  edges (ACTIVE → PAUSING, PAUSING → PAUSED) and the `pause` verb
  becomes the caller's synchronous face over an async drain — same
  matrix, same API shape.
- **Slot-binding cross-check on destroy.** A component whose lifecycle
  state reaches DESTROYED should not still be bound as any slot's
  `active` impl. The registry enforces this on `nx_component_unregister`
  (NX_EBUSY) but not on `nx_component_destroy`. For v1 the rule is "the
  caller unbinds first"; when the framework drives the lifecycle
  end-to-end (slice 3.9), destroy will walk the slot list and refuse
  cleanly. Skipping for this slice keeps the scope tight and the
  existing unregister check already catches the mistake before the
  component is freed.

## Next

- **Slice 3.4 — Dependency injection.** The `COMPONENT_REGISTER` macro
  expands to a static `struct component_descriptor` in the
  `.components` linker section; `gen-config.py` emits
  `gen/<component>_deps.h`; descriptor-driven `resolve_deps()` writes
  slot pointers and registers connection edges via
  `nx_connection_register`. Likely the slice that forces the
  `framework_alloc` indirection (test build → `mt_alloc`, kernel →
  `kmalloc`), since descriptors will want to allocate backing storage.
