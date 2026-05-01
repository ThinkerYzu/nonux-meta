# Session 91: slice 8.0a.7 — handle-table invalidation on dep swap

**Date:** 2026-05-01
**Phase:** Phase 8 — Group B (slice 8.0a, sub-slice 7 of 8)
**Branch:** master (source-side commit `68b4718` in `sources/nonux/`; this session's checkpoint is the proj_docs paperwork that records it)

---

## Goals

- Land slice **8.0a.7** — `posix_shim_on_dep_swapped` STATE_LOST handler +
  `nx_handle_table_invalidate_for_slot()`.  Required before live-swap is safe
  across non-trivial state: when the vfs dep is replaced and the new component
  starts with fresh state, all open FILE/DIR handles backed by the old
  component must be invalidated so userspace sees `NX_EBADF` on the next op.
- Design decision confirmed at session start: **Option B** — add
  `struct nx_slot *slot` (NULL = immune) to `struct nx_handle_entry`; new
  `nx_handle_alloc_with_slot()` + `nx_handle_set_slot()` helpers; only
  `sys_open` / `sys_opendir` wire the slot (~2 call sites in `syscall.c`).

## What Was Done

### Slice 8.0a.7 (source-side commit `68b4718`, `sources/nonux/`)

Pure additive on top of slice 8.0a.6.  No existing behavior changed.

**`framework/component.h` (+18 lines).** Two additions:

1. Forward declarations `struct nx_slot;` and `struct nx_component;` (needed by
   the new callback signature; full definitions remain in `framework/registry.h`).
2. `NX_SWAP_STATE_LOST (1u << 0)` — flag passed to `on_dep_swapped` when the
   incoming component starts with fresh, incompatible state.
3. `on_dep_swapped` callback added to `struct nx_component_ops` (at the end;
   all existing component ops structs use designated initializers so the new
   field defaults to NULL cleanly):
   ```c
   int (*on_dep_swapped)(void *self,
                         struct nx_slot      *dep_slot,
                         struct nx_component *old_comp,
                         struct nx_component *new_comp,
                         uint32_t             flags);
   ```
   The framework invocation path (Phase 8.1+ Group C) wires this into the
   recomposition orchestrator; for v1 it is called directly from tests.

**`framework/handle.h` (+42 lines).** Three changes:

1. `struct nx_handle_entry` gains `struct nx_slot *slot` (NULL = immune to
   slot-based invalidation).  The field is last so existing aggregate
   initializers that don't set it default to NULL.
2. `nx_handle_alloc_with_slot()` — alloc + wire slot in one call (used by the
   two sys_open paths).
3. `nx_handle_set_slot()` — post-alloc setter; no-op if the entry already has
   a non-NULL slot (prevents accidental double-wiring).
4. `nx_handle_table_invalidate_for_slot()` — the invalidation walk.

**`framework/handle.c` (+70 lines).** Five changes:

1. `#include "framework/process.h"` — pulls in `nx_process_for_each`.
2. `nx_handle_table_init`: adds `t->entries[i].slot = NULL;` to the zeroing
   loop so a re-init'd table never inherits a stale slot pointer.
3. `nx_handle_alloc`: adds `e->slot = NULL;` alongside the other field
   initializations so freshly allocated entries start immune.
4. `nx_handle_close`: adds `e->slot = NULL;` in the closing zeroing block so
   a reused slot doesn't inherit the previous entry's slot.
5. `nx_handle_duplicate`: captures `t->entries[src_idx].slot` after lookup
   succeeds, then propagates it to the new entry via `nx_handle_set_slot`.
   This ensures dup'd FILE/DIR handles are also invalidated on dep swap.
6. New functions `nx_handle_alloc_with_slot`, `nx_handle_set_slot`,
   `invalidate_slot_cb` (static), `nx_handle_table_invalidate_for_slot`.

`invalidate_slot_cb` walks one process's handle table: for each entry whose
`slot == dep_slot` and `type != NX_HANDLE_INVALID`, it zeroes `type/rights/
object/slot`, increments `generation` (same stale-handle mechanism as
`nx_handle_close`), and decrements `count`.  Always returns 0 so
`nx_process_for_each` walks all processes.

**`framework/syscall.c` (+6 lines net).** `sys_open` captures the vfs slot
immediately after `resolve_vfs` succeeds:
```c
struct nx_slot *vfs_slot = nx_slot_lookup("vfs");
```
Both alloc sites (HANDLE_DIR inside `#if !__STDC_HOSTED__`, HANDLE_FILE in the
common path) switch from `nx_handle_alloc` to `nx_handle_alloc_with_slot(...,
vfs_slot, ...)`.  In host builds where `nx_slot_lookup("vfs")` returns NULL,
the helper degrades to plain `nx_handle_alloc` — existing host test behavior is
unchanged.

**`components/posix_shim/posix_shim.c` (+24 lines).** Two changes:

1. `#include "framework/handle.h"` added (for
   `nx_handle_table_invalidate_for_slot`).
2. `posix_shim_on_dep_swapped` implemented:
   ```c
   static int posix_shim_on_dep_swapped(void *self,
                                        struct nx_slot      *dep_slot,
                                        struct nx_component *old_comp,
                                        struct nx_component *new_comp,
                                        uint32_t             flags)
   {
       (void)self; (void)old_comp; (void)new_comp;
       if (flags & NX_SWAP_STATE_LOST)
           nx_handle_table_invalidate_for_slot(dep_slot);
       return NX_OK;
   }
   ```
3. `.on_dep_swapped = posix_shim_on_dep_swapped` added to
   `posix_shim_component_ops`.

**`test/host/handle_test.c` (+179 lines).** Nine new TESTs covering slice
8.0a.7's surface:

| TEST | Coverage |
|---|---|
| `handle_alloc_with_slot_wires_slot_field` | `alloc_with_slot` stores the slot pointer in the raw entry. |
| `handle_alloc_with_null_slot_leaves_entry_immune` | NULL slot → entry's slot stays NULL. |
| `handle_set_slot_wires_on_valid_handle` | `set_slot` on a live entry sets the pointer. |
| `handle_set_slot_noop_if_already_wired` | Second `set_slot` call is a no-op; first value preserved. |
| `handle_close_clears_slot_field` | `nx_handle_close` zeroes the slot pointer. |
| `handle_duplicate_propagates_slot` | `nx_handle_duplicate` carries the source entry's slot onto the new entry. |
| `handle_duplicate_null_slot_stays_null` | Dup of a non-slot entry leaves the dup's slot NULL. |
| `handle_invalidate_for_slot_clears_matching_entries` | `invalidate_for_slot` zeroes matching entries, preserves others; count decrements correctly. |
| `handle_invalidate_for_slot_ignores_null_slot_entries` | Entries with NULL slot are unaffected by invalidation. |

Both invalidation tests use `nx_process_create` to get a well-isolated handle
table and capture the initial count (which includes 3 pre-installed CONSOLE
handles) as a baseline so assertions are independent of process-create
side-effects.

### Key decisions

- **Option B confirmed at session start** (slot field on `nx_handle_entry`,
  ~2 wiring call sites in syscall.c).  Option A (walk the handle table
  comparing object-pointer patterns) would have required type-specific dispatch
  and knowledge of the file-object layout inside invalidation — worse coupling.
- **`nx_handle_duplicate` propagates the slot.**  A dup'd FILE handle is still
  backed by the same vfs dep; it must also be invalidated if that dep loses
  state.  The propagation is free (one pointer copy inside the existing lookup
  path).
- **`NX_SWAP_STATE_LOST` lives in `component.h`, not handle.h.**  It's a
  framework lifecycle flag, not a handle concept.  Any future component that
  needs dep-swap notification can use it without depending on handle machinery.
- **Clean build revealed stale-object ABI mismatch.**  `struct nx_handle_entry`
  grew by 8 bytes (the new `slot` pointer); `struct nx_process` grew
  proportionally.  Incremental builds with stale `process.o` triggered heap
  corruption in tests.  Fix: `make clean && make test`.

### Tests at end of Session 91

- `make test` → **528/528 pass** (93 python + **314** host (was 305; +9) + **121** kernel).
- `make test-interactive` → **7/7 pass**.
- `make verify-iface-fresh` → 0 drift.
- `make verify-registry` → 0 findings.

## Next Steps

Slice **8.0a.8** — cross-cutting test infrastructure that every later slice in
Group B / Group C depends on:

- Configurable mock component (programmable `handle_msg` behavior + call counter).
- Hook-chain inspector (records hook firings + ABORT decisions per point).
- Recompose event logger (snapshots registry mutations during swap).
- Pause-injector ktest fixture (drives pause states from outside the slot).
- Cap-forgery harness (sender claims unauthorized caps → NX_EINVAL assert).
- Per-callsite equivalence-runner macro (paired direct-vs-blocking-call
  invocation with assertion).

After 8.0a.8, Group B continues with 8.0b (activate generated `handle_msg`
shims on all 5 production components).

---

**Files Changed (`sources/nonux/`, commit `68b4718`):**
- `framework/component.h` — `NX_SWAP_STATE_LOST` + `on_dep_swapped` in `nx_component_ops` + forward decls.
- `framework/handle.h` — `slot` field on `nx_handle_entry` + 3 new function decls.
- `framework/handle.c` — `#include process.h`; slot zeroing in init/alloc/close; slot propagation in dup; `alloc_with_slot`, `set_slot`, `invalidate_for_slot` implementations.
- `framework/syscall.c` — `sys_open` FILE + DIR allocs switched to `nx_handle_alloc_with_slot` with `vfs_slot`.
- `components/posix_shim/posix_shim.c` — `posix_shim_on_dep_swapped` + `.on_dep_swapped` in ops.
- `test/host/handle_test.c` — 9 new TESTs covering the slice 8.0a.7 surface.

**Files Changed (`proj_docs/nonux/`, this commit):**
- `logs/session-91-8.0a.7-handle-invalidation.md` — this log.
- `HANDOFF.md` — Current Status / Phase checklist / Next Actions / Session Logs advanced; Session 86 archived.
- `IMPLEMENTATION-GUIDE.md` — slice 8.0a.7 row marked ✓; Phase 8 status + Last Updated footer refreshed.
- `HANDOFF-ARCHIVE.md` — Session 86 entry pulled forward from HANDOFF per the keep-last-5 convention.
