# Session 100: Slice 8.3 — Runtime Config Manager

**Date:** 2026-05-03
**Phase:** Phase 8 — Runtime Recomposition
**Branch:** master

---

## Goals

- Implement `framework/config.c` runtime config manager + handle API + EL0 syscall surface (`NX_SYS_CONFIG_*`).
- EL0 program opens a config handle, queries the live composition via snapshot, fires `nx_recompose` through the syscall gate.

## What Was Done

### framework/config.h + config.c

New runtime config manager module.  Three public entry points:

- `nx_config_open(t, &h)` — allocates a `NX_HANDLE_CONFIG` handle in the caller's handle table.  A static sentinel provides the non-NULL object pointer.  The handle type is the revocation point.
- `nx_config_query_snapshot(snap)` — walks `nx_graph_foreach_slot` and fills a fixed-size snapshot struct (up to 32 entries) with per-slot `(name, impl_id, lifecycle_state)` tuples plus the current generation counter.
- `nx_config_swap_component(slot_name, new_impl)` — finds the named slot via `nx_slot_lookup`, finds the first `NX_LC_READY` component with the given `manifest_id` via `nx_graph_foreach_component`, builds a single-slot `recomp_plan`, and calls `nx_recompose()`.

String comparisons in the kernel build avoid `memcmp` (not in the kernel string library) by using a local `cfg_streq` that walks byte-by-byte.  `safe_copy` uses only `strlen` + `memcpy` (both available in both builds).

### NX_HANDLE_CONFIG added to handle.h

`NX_HANDLE_CONFIG = 9` inserted before `NX_HANDLE_TYPE_COUNT`.  The `nx_handle_alloc` type-range guard checks `type >= NX_HANDLE_TYPE_COUNT`, which now = 10, so the new type passes.

### Three new syscalls (syscall.h + syscall.c)

| Syscall | Number | Signature |
|---------|--------|-----------|
| `NX_SYS_CONFIG_OPEN` | 42 | `() → nx_handle_t / NX_E*` |
| `NX_SYS_CONFIG_QUERY` | 43 | `(h, struct nx_config_snapshot *buf) → NX_OK / NX_E*` |
| `NX_SYS_CONFIG_SWAP` | 44 | `(h, slot_name, impl_name) → NX_OK / NX_E*` |

All three validate the handle type before acting.  QUERY + SWAP use `copy_path_from_user` / `copy_to_user` for the pointer arguments.

### Host tests (test/host/slice_8_3_test.c) — 10 tests

1. `nx_config_open` allocates `NX_HANDLE_CONFIG`
2. Closing invalidates the handle
3. Snapshot on empty graph → 0 slots
4. Snapshot reports registered slots (name + impl_name match)
5. Swap returns `NX_ENOENT` for unknown slot
6. Swap returns `NX_ENOENT` when no READY component
7. Direct API swap replaces the active component
8. `NX_SYS_CONFIG_OPEN` syscall returns positive handle
9. `NX_SYS_CONFIG_QUERY` syscall populates snapshot
10. `NX_SYS_CONFIG_SWAP` syscall fires recompose end-to-end

### Kernel tests (test/kernel/ktest_config.c) — 4 tests

1. `config_snapshot_after_bootstrap_has_expected_slots` — verifies bootstrap slots exist via `nx_slot_lookup` and snapshot API returns valid data.  Note: the snapshot is truncated at 32 entries because the registry holds many leaked per-task `caller_slot` entries (no reap-on-wait yet); direct `nx_slot_lookup` is the authoritative check.
2. `config_sys_open_and_close_via_svc` — SVC `NX_SYS_CONFIG_OPEN` returns valid handle; `NX_SYS_HANDLE_CLOSE` invalidates it.
3. `config_sys_query_via_svc_populates_snapshot` — SVC query with snapshot buffer at `mmu_user_window_base()` returns correct generation + non-empty slot count.
4. `config_swap_api_replaces_active_component` — direct API swap test: registers test slot + two components, swaps via `nx_config_swap_component`, verifies comp_b becomes ACTIVE, cleans up.

## Key Findings

- **Stale kernel `.o` files.** The top-level Makefile has no header dependency tracking (no `.d` files).  Modifying `handle.h` doesn't automatically trigger recompilation of `handle.c` unless a `make clean` is run first.  The stale `handle.o` used the old `NX_HANDLE_TYPE_COUNT = 9`, rejecting the new `NX_HANDLE_CONFIG = 9` with `NX_EINVAL`.  Fix: always run `make clean` before `make test-kernel` when header enums change, or add dependency tracking.
- **Registry pollution from leaked caller_slots.** Every task spawned by fork/exec registers a `caller_slot` in the component registry.  Since reap-on-wait is not yet implemented (`sys_wait` doesn't call `nx_process_destroy`), all child task caller_slots persist indefinitely.  By the time the config ktest runs (last in the suite), hundreds of slots are registered.  The snapshot is truncated at 32, with recently-registered caller_slots filling the window before bootstrap slots.  The fix for the ktest was to use `nx_slot_lookup("scheduler")` directly for the name-presence check, and use the snapshot API only for `num_slots > 0` + `generation > 0`.
- **`free()` is a no-op for slab allocations.** The kernel's kheap slab allocator (for allocations ≤ 256 bytes) never reclaims memory — `free()` is a no-op.  Registry slot_nodes (~40 bytes) never return to the pool.  This is a known limitation documented in `kheap.c`.

## Decisions Made

- **Config handle has no per-instance state.** A static `g_config_sentinel` integer provides the non-NULL object pointer.  The handle type provides revocation; the handle grants full composition access in v1 (no privilege separation needed for a single-user research kernel).
- **Snapshot format is a fixed-size flat struct.** `struct nx_config_snapshot` with up to 32 `struct nx_config_snapshot_entry` entries.  Fits on the kernel stack (~2192 bytes) and is safe to pass to `copy_to_user`.  The 32-entry cap is conservative given known registry pollution in the test environment; production usage (7 slots) is well within.
- **`memcmp` not in kernel string lib — use `cfg_streq`.** Rather than adding `memcmp` to the kernel string library, a local `cfg_streq` byte-walk is used in `config.c`.  This is the minimum change that compiles for both builds.

## Status at End of Session

- All tests green: `make test-tools` **102/102**, `make test-host` **437/437** (10 new host tests), `make test-kernel` **132/132** (4 new kernel tests).
- `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings (R2, R4, R9).
- **Next: slice 8.4** — second scheduler `sched_priority` implementation.

## Next Steps

- **Slice 8.4** — `components/sched_priority/`.  Conformance suite reused from Phase 4.  Kernel boots with `sched_priority` from `kernel.json` start-to-finish.  Standalone validation (no live swap yet).

---

**Files Changed:**
- `framework/handle.h` — added `NX_HANDLE_CONFIG` before `NX_HANDLE_TYPE_COUNT`
- `framework/config.h` — new: config manager API declarations
- `framework/config.c` — new: `nx_config_open`, `nx_config_query_snapshot`, `nx_config_swap_component`
- `framework/syscall.h` — added `NX_SYS_CONFIG_OPEN=42`, `NX_SYS_CONFIG_QUERY=43`, `NX_SYS_CONFIG_SWAP=44`
- `framework/syscall.c` — added `#include "framework/config.h"`; added `sys_config_open`, `sys_config_query`, `sys_config_swap` handlers; wired into dispatch table
- `Makefile` — added `framework/config.c` to `FW_C`; added `test/kernel/ktest_config.c` to `KTEST_C`
- `test/host/Makefile` — added `slice_8_3_test.c` to `SRCS`; added `../../framework/config.c` to framework sources
- `test/host/slice_8_3_test.c` — new: 10 host tests for slice 8.3
- `test/kernel/ktest_config.c` — new: 4 kernel tests for slice 8.3
