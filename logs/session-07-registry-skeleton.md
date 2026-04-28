# Session 7: Component Graph Registry skeleton (slice 3.1)

**Date:** 2026-04-20
**Phase:** 3 — Component framework
**Branch:** master

---

## Goals

- Open Phase 3 with a small, vertical slice: build the registry's bookkeeping skeleton (slot / component / connection register / lookup / unregister / traversal / generation counter) on the host side, fully test-covered, with no kernel integration yet.
- Establish the API shape that lifecycle, IPC router, and `verify-registry` will plug into in later slices.

## What Was Done

### `framework/registry.h` — public API

Defined the framework-side surface:

- Caller-owned structs: `nx_slot`, `nx_component`, `nx_connection`. Caller owns storage; registry only bookkeeps pointers.
- Six framework enums: `nx_lifecycle_state`, `nx_conn_mode`, `nx_pause_policy`, `nx_slot_mutability`, `nx_slot_concurrency`, plus `NX_E*` error codes (`NX_OK`, `NX_EINVAL`, `NX_ENOMEM`, `NX_EEXIST`, `NX_ENOENT`, `NX_EBUSY`, `NX_ESTATE`).
- Mutation API: `nx_slot_register / swap / unregister`, `nx_component_register / unregister / state_set`, `nx_connection_register / retune / unregister`. Connection register returns the new connection on success or NULL with an `*err` out-param on failure (avoids overloading return).
- Lookup: `nx_slot_lookup(name)`, `nx_component_lookup(manifest_id, instance_id)`. Component identity is the (manifest, instance) pair so the same manifest can have multiple instances (e.g., `uart_pl011`/`uart0`, `uart_pl011`/`uart1`).
- Traversal: global `foreach_slot/component/connection`, plus per-slot `foreach_dependent` (incoming edges = "who depends on me") and `foreach_dependency` (outgoing edges = "what I depend on"). Naming matches DESIGN.md §Traversal API.
- `nx_graph_reset()` for test setup — releases all internal nodes, zeroes counts and generation, does **not** touch caller-owned struct fields.
- `nx_graph_generation()` — monotonically incremented on every mutation. Lays the ground for snapshot consistency (slice 3.2) and the change log (slice 3.3).

### `framework/registry.c` — implementation

Three internal node types (`slot_node`, `component_node`, `conn_node`), kept as singly-linked lists indexed off three globals. Each `slot_node` carries its own `incoming` / `outgoing` connection lists so `foreach_dependent`/`foreach_dependency` are O(deg) without rescanning the global list.

- `slot_register` rejects NULL inputs (`EINVAL`) and duplicate names (`EEXIST`).
- `slot_unregister` rejects slots with live connection edges (`EBUSY`) — caller must tear wiring down first.
- `slot_swap` requires the new component (if non-NULL) to be registered; passing NULL unbinds. Both paths bump generation.
- `component_register` keys uniqueness on the (manifest_id, instance_id) pair.
- `component_unregister` rejects components still active in any slot (`EBUSY`); enforced by walking slot list.
- `connection_register` accepts `from_slot == NULL` for boot/external edges; `to_slot` is required.
- `connection_unregister` cleans up both global and per-slot lists.
- Internal allocations use plain `malloc`/`free`. Decoupled from the host test's `mt_track` pool because `mt_reset()` would dangle our pointers if we shared. When the registry lands in-kernel, these become `kmalloc`/`kfree` — single point to swap.
- No concurrency: host-side only, single-threaded. SMP serialisation belongs to the recomposer in a later slice.

~280 lines total.

### `test/host/registry_test.c` — 25 tests

Coverage shape:

- Init / empty: zero counts, zero generation.
- Slot lifecycle: register, lookup, duplicate-name `EEXIST`, NULL inputs `EINVAL`, missed lookup, unregister, `EBUSY` while connections live.
- Component lifecycle: register, lookup, duplicate (manifest, instance) `EEXIST`, distinct instances of the same manifest, `state_set` updates state and bumps generation, `unregister` blocked while active in a slot then succeeds after `slot_swap(s, NULL)`.
- Slot swap: sets `active`, bumps generation, rejects unknown components.
- Connection lifecycle: register basic, register with `from=NULL` (boot edge), register with NULL `to` → `EINVAL`, register with unknown slot → `ENOENT`, retune, unregister cleans both lists.
- Traversal: `foreach_slot/component/connection` visit-all, neighbourhood walk for dependent/dependency.
- Generation: monotonic across register / retune / unregister sequence.
- Cycling: 100 iterations of register-all → unregister-all → assert zero residue.

`SLOT(name, iface)` and `COMP(name, manifest)` macros declare auto-storage structs, kept tight enough that each test reads as a single thought. Each test starts with `nx_graph_reset()` so it begins on a clean global state regardless of what the previous test left behind.

### `test/host/Makefile`

Added `registry_test.c` and `../../framework/registry.c` to `SRCS`, and `../../framework` to the `vpath`. No changes to the top-level Makefile (registry is host-only this slice).

## Key Findings

- **Caller-owned storage outlives test scope.** First run crashed in `generation_monotonically_increases` — actually in the *next* test, when `nx_graph_reset()` walked stale `slot_node` entries left over from the prior test's stack-local slot vars and tried to write to `slot->active`. Two fixes: (1) `nx_graph_reset()` releases registry nodes only, never dereferences cached `slot`/`component` pointers; (2) `nx_slot_unregister()` no longer touches `s->active` either. The rule, made explicit in registry.c: the registry never writes to caller storage outside the deliberate APIs that name it (e.g., `slot_swap`).
- **Plain `malloc`/`free` for internals is the right call for the slice.** `mt_track` is for catching test-code leaks, not framework leaks; if we tracked registry nodes through `mt_alloc`, `mt_reset()` between tests would free them and dangle our globals. When the registry hits kernel, the call sites swap to `kmalloc`/`kfree` — a one-line indirection isn't necessary yet.

## Decisions Made

- **Connection register returns pointer + out-param error**, not int + out-param pointer. Reads naturally at call sites and lets `if (!c) goto fail;` work; the `int *err` lets callers distinguish `EEXIST` / `ENOENT` / `EINVAL` when they care.
- **`from_slot == NULL` is a legal "boot edge"**, not an error. The boot sequence needs to declare initial wiring without inventing a fake source slot.
- **Slot identity is `name`; component identity is `(manifest_id, instance_id)`.** Matches DESIGN.md §Registered Entities. A future `gen-config.py` will own uniqueness of these strings; the registry just enforces no-duplicates at registration.
- **No hash tables yet.** Linked-list lookup is O(N). v1 has tens of slots/components — a hash is premature. Add when we cross 100s.
- **No concurrency primitives in this slice.** SPEC's preemption-atomics rule applies to mutating shared counters in production code; the registry is test-only here. Snapshots, seqlock, change log are slice 3.2/3.3 work.

## Status at End of Session

- `make test` → **42/42 pass** (36 host + 6 kernel), 0 leaks, 0 errors. Of the 36 host tests: 9 PMM, 2 mem_track, **25 new registry**.
- Production `kernel.bin` is unchanged — registry is built and exercised host-side only this slice.
- `framework/` directory now non-empty: registry.{h,c}.
- `tools/` directory still empty beyond `run-qemu.sh`. `gen-config.py`, `validate-config.py`, `verify-registry.py` are next-slice work.

## Next Steps

Phase 3 continues with these slices, ordered by what later slices depend on:

- **3.2 Snapshot + JSON serialization + change log + event subscribe.** The pieces hooks, the recomposer, and the `kstat` dump all consume.
- **3.3 Lifecycle state machine (`framework/component.c`).** Wire transitions to `nx_component_state_set` + emit GRAPH_EV_COMPONENT_STATE.
- **3.4 Dependency injection mechanism.** `COMPONENT_REGISTER` macro + `.components` linker section + descriptor-driven `resolve_deps()`. Needs a real allocator on the kernel side, so this slice may force the `framework_alloc` indirection.
- **3.5 Config tooling (`gen-config.py`, `validate-config.py`).**
- **3.6 IPC router with cap-array scanning + `slot_ref_retain/release`.**
- **3.7 `verify-registry.py` static checker (R1–R8).** Wire as `make verify-registry` build gate.
- **3.8 Hooks attached to connection edges + slot mutation events.**
- **3.9 Wire registry + framework into kernel boot path.**

---

**Files Changed:**
- `sources/nonux/framework/registry.h` — new, public registry API (~150 lines).
- `sources/nonux/framework/registry.c` — new, host-side implementation (~280 lines).
- `sources/nonux/test/host/registry_test.c` — new, 25 tests covering register/lookup/swap/retune/traversal/cycling.
- `sources/nonux/test/host/Makefile` — added registry_test.c and ../../framework/registry.c to SRCS; added ../../framework to vpath.
