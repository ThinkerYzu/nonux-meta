# Session 106: 9b.2 — Handle Entry Redesign + Open/Close Routing

**Date:** 2026-05-04
**Phase:** 9b (Handles as Capabilities)
**Branch:** master

---

## Goals

- Implement slice 9b.2: replace `{ type, rights, void *object }` in `nx_handle_entry` with a union `{ void *object; uint32_t id }` and rename `slot` → `target`; add `NX_HANDLE_RESOURCE` type; switch console pre-install to RESOURCE; update `sys_open`, `sys_handle_close`, `sys_read`, `sys_write`, and related syscalls.
- Add ~8 host tests.
- Keep all 469 existing host tests passing.

## What Was Done

### handle.h / handle.c — struct and API changes

- Added `NX_HANDLE_RESOURCE = 9` to `enum nx_handle_type` (between CONSOLE and CONFIG; CONFIG shifted to 10, TYPE_COUNT to 11).
- Replaced `void *object` with `union { void *object; uint32_t id; }` — CHANNEL/VMO/DIR/CONFIG handles continue using `object`; RESOURCE handles use `id`.
- Renamed `slot` → `target` in `struct nx_handle_entry`.  All internal usages (`nx_handle_table_init`, `nx_handle_close`, `nx_handle_set_slot`, `nx_handle_alloc`, `nx_handle_duplicate`, `invalidate_slot_cb`) updated accordingly.
- Added `nx_handle_alloc_resource(t, rights, id, target, out)` — allocates a RESOURCE entry; accepts `id == 0` (valid for console singleton).
- Added `nx_handle_entry_get(t, h)` — validates handle and returns a const pointer to the live entry, or NULL on stale/invalid.  Allows callers to access `entry->id` and `entry->target` without a separate API.
- `nx_handle_alloc_with_slot` parameter renamed `slot` → `target` (same semantics).

### process.c — console pre-install switched to RESOURCE

Replaced three `nx_handle_alloc(CONSOLE, ..., &g_nx_console, ...)` calls with `nx_handle_alloc_resource(RESOURCE, id=0, char_slot, ...)`.  `nx_slot_lookup("char_device")` returns NULL on host builds where the slot isn't registered; NULL target is accepted and means "immune to slot invalidation" — same behaviour as before.  Removed `#include "framework/console.h"` (no longer needed); added `#include "framework/registry.h"` for `nx_slot_lookup`.

### syscall.c — multiple sites updated

- **`sys_open`**: replaced `void *file_obj = (void*)(uintptr_t)vfs_id; nx_handle_alloc_with_slot(FILE, ...)` bridge with `nx_handle_alloc_resource(RESOURCE, vfs_id, vfs_slot, &h)`.
- **`sys_handle_close`**: removed `NX_HANDLE_FILE` arm; added `NX_HANDLE_RESOURCE` arm that calls `nx_vfs_close(res->target, res->id)` unless `target == char_device_slot` (console singleton — no-op close).  Changed outer guard from `if (rc == NX_OK && object)` to `if (rc == NX_OK)` so RESOURCE entries with `id=0` (object=NULL via union) still reach the RESOURCE arm.
- **`lookup_file_object`** (used by sys_seek): added `NX_HANDLE_RESOURCE` to accepted types.  The `(uint32_t)(uintptr_t)object` bridge still extracts the id correctly via union aliasing on little-endian AArch64.
- **`sys_read`**: refactored `h==0` and general path to use `const struct nx_handle_entry *cur_entry` (eliminates `nx_handle_lookup` + separate `h==0` table access).  Added `NX_HANDLE_RESOURCE` arm: if `target == char_slot` → console read; else → `nx_vfs_read(target, id, ...)` with `nx_slot_lookup("vfs")` fallback for handles whose target was not copied (e.g. legacy FILE handles).
- **`sys_write`**: same RESOURCE arm added (using `nx_handle_entry_get` to get target/id after the existing lookup).
- **`sys_seek`**: uses `nx_handle_entry_get` to prefer `entry->target` over `nx_slot_lookup("vfs")` fallback.
- **`sys_ioctl`**: added RESOURCE acceptance for the console case (`ie->target == char_device_slot`).
- **`sys_fork` handle inheritance loop**: changed `if (!src->object) continue` guard to `if (src->type == NX_HANDLE_INVALID) continue`; added `NX_HANDLE_RESOURCE` retain arm (skips console id=0 or char_slot entries — re-installed by `nx_process_create` in the child); copies `dst->target = src->target`.
- **`sys_dup3`**: added RESOURCE close arm; added RESOURCE retain arm; copies `e->target = src_entry->target` at install.
- **`sys_fcntl` F_DUPFD**: added RESOURCE retain arm; copies `e->target = fcntl_src->target` at install.
- **`ppoll`**: added `NX_HANDLE_RESOURCE` switch case — dispatches to PPOLL_KIND_CONSOLE (char_device target) or PPOLL_KIND_FILE (vfs target).

### Test suite updates

- **`test/host/handle_test.c`**: renamed `t.entries[idx].slot` → `t.entries[idx].target` throughout (7 sites).
- **`test/host/slice_8_0b_test.c`** (pre-existing 9b.1 omission): updated all pre-9b.1 fake ops for both `vfs_fake_*` and `fs_fake_*` sections to use the 9b.1 `uint32_t id` API.  Fixed field references `req.file` → `req.id`, `r->out_file` → `r->rc`; revised two "direct vs dispatch equivalence" tests to compare returned ids rather than status codes.
- **`test/host/slice_9b_2_test.c`** (new, 8 tests): handle_alloc_resource type; id=0 accepted; id+target accessible via entry_get; entry_get on valid/closed/invalid; close clears target; nx_process_create pre-installs RESOURCE console handles.

## Key Findings

- **NX_HANDLE_CONFIG enum shift requires make clean.** Adding `NX_HANDLE_RESOURCE = 9` shifted `NX_HANDLE_CONFIG` from 9 to 10.  The host test Makefile has no header dependency tracking, so `config.o` retained the old enum value 9 until `make clean` was run.  Same pattern as Session 100 (`NX_HOOK_POINT_COUNT` shift).

- **`slice_8_0b_test.c` missed in 9b.1.** The session-105 test updates skipped `slice_8_0b_test.c`, leaving pre-9b.1 fake op signatures (`void *file` parameter) and old message field names (`req.file`, `r->out_file`).  Masked by stale `.o` files until `make clean` was forced.  Fixed in this session.

- **`(void*)(uintptr_t)id` union aliasing still works for legacy FILE handles.** `sys_seek` and the `!src->object` fork guard rely on the union's little-endian behaviour (id stored in low 4 bytes of the 8-byte union = same as old pointer value for small ids).  Documented.

- **Console target is NULL on host.** `nx_slot_lookup("char_device")` returns NULL in host test builds where the slot registry is empty.  The comparison `entry->target == char_slot` is `NULL == NULL` → TRUE, routing console RESOURCE handles to the console path.  Works correctly by accident of symmetry; intentional by design.

## Status at End of Session

- `make test-tools` → **102/102 pass**
- `make test-host` → **477/477 pass** (8 new)
- `make verify-iface-fresh` → 0 drift
- `make verify-registry` → 0 findings (R2, R4, R9)
- Slice 9b.2 complete. Slices 9b.3–9b.4 not yet started.

## Next Steps

- **Slice 9b.3** — Route `sys_read`, `sys_write`, `sys_seek`, `sys_readv`, `sys_writev` uniformly through `nx_slot_call_blocking(&task->caller_slot, entry->target, &msg, reply, sizeof reply)`.  Eliminates the `switch(handle_type)` in these syscalls.  Needs updated fs/char_device IDL calls.  ~80 lines changed in syscall.c; ~10 kernel tests.

---

**Files Changed:**
- `framework/handle.h` — NX_HANDLE_RESOURCE, union id/object, slot→target rename, new APIs
- `framework/handle.c` — all slot→target renames, nx_handle_alloc_resource, nx_handle_entry_get
- `framework/process.c` — console pre-install switched to nx_handle_alloc_resource
- `framework/syscall.c` — sys_open, sys_handle_close, sys_read, sys_write, sys_seek, sys_ioctl, sys_fork, sys_dup3, sys_fcntl, ppoll handler
- `test/host/handle_test.c` — slot→target renames
- `test/host/slice_8_0b_test.c` — pre-9b.1 fake ops updated to uint32_t id API
- `test/host/slice_9b_2_test.c` — NEW: 8 host tests for slice 9b.2
- `test/host/Makefile` — registered slice_9b_2_test.c
