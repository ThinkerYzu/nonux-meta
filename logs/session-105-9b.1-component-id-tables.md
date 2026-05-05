# Session 105: 9b.1 — Component-Owned Object Tables

**Date:** 2026-05-04
**Phase:** 9b (Handles as Capabilities)
**Branch:** master

---

## Goals

- Implement slice 9b.1: replace the `void *file` pointer-passing API with a component-owned integer ID table for `fs` and `vfs` interfaces.
- Add `read` op to `char_device` IDL for uart_pl011.
- Keep all 459 existing host tests passing; add 10 new 9b.1 tests.

## What Was Done

### IDL changes (fs.json, vfs.json, char_device.json)

Changed `fs` and `vfs` IDL ops to use `uint32_t id` instead of `opaque_self_handle`:
- `open(path, flags) → u32` — returns 1-based component-local ID (0 = failure)
- `close(id: u32)`, `retain(id: u32)` — take ID instead of void pointer
- `read(id: u32, buf, cap)`, `write(id: u32, buf, len)`, `seek(id: u32, offset, whence)`

Added `read(id, buf, cap) → i64` op to `char_device` IDL (op_id 3), placed before the existing `rx_byte` (op_id 2) in the JSON — op_ids are stable, position doesn't matter for dispatch.

Regenerated all generated headers via `gen-iface.py all`.

### Manual wrapper updates (fs_call.c, vfs_call.c)

`nx_fs_open` now returns `uint32_t` (0 = failure); all per-open wrappers take `uint32_t id`. Same for `nx_vfs_*` wrappers in `vfs_call.c`. Error return for host-path NULL slot resolution changed from `NX_ENOENT` to `0` (matching the u32 convention).

### Component implementations (ramfs.c, procfs.c, vfs_simple.c)

Each component now maintains an internal `opens[]` array. `open` returns `index + 1` (1-based); `close/retain/read/write/seek` look up by `id − 1` via a `*_id_to_*` helper that validates bounds and `in_use`. `vfs_simple` stores `(mount_name, driver_id)` pairs and forwards to the underlying fs driver with its `driver_id`.

### char_device: uart_pl011 read op

Added `uart_pl011_read(self, id, buf, cap)` delegating to `nx_console_read` (kernel build) / returning 0 / EOF (host build). `id` is ignored — uart_pl011 is a singleton device.

### syscall.c adaptation

`sys_open`: calls `nx_vfs_open(slot, path, flags)` which returns `uint32_t vfs_id`; stores it as `(void*)(uintptr_t)vfs_id` in the handle entry's `object` field (temporary bridge until slice 9b.2 redesigns the handle entry).

`sys_read`, `sys_write`, `sys_seek`, `sys_handle_close`, `sys_dup3` (close/retain path), `sys_fcntl` (retain path): all extract the ID as `(uint32_t)(uintptr_t)object` before calling the VFS wrapper.

### Test suite updates

Updated 8 existing test files to use the new uint32_t API:
- `conformance/conformance_fs.c` — conformance suite
- `conformance_fs_test.c` — fs_stub fixture + local unit tests
- `component_ramfs_test.c` — ramfs-specific tests
- `component_vfs_simple_test.c` — vfs_simple tests
- `file_syscall_test.c` — syscall-level file tests
- `slice_8_0c_test.c` — VFS wrapper equivalence tests
- `slice_8_0d_test.c` — FS wrapper equivalence tests
- (fake driver stub functions in all files updated to new signatures)

Added new `slice_9b_1_test.c` with 10 tests covering:
- ramfs: open returns non-zero ID; missing-path returns 0; write/read round-trip; stale-ID error; independent cursors; table-full returns 0; seek via ID
- vfs_simple: open returns VFS-local ID; read/write round-trip
- char_device: read op present in struct; signature is `(self, id, buf, cap)`

## Key Findings

- **Error code loss from `open`.** The `u32` return type for `open` collapses all failure modes (ENOENT, ENOMEM, EPERM) to `0`. `sys_open` maps `id == 0` to generic `NX_ENOENT`. This is a deliberate trade-off in the 9b.1 design; specific error codes return in slice 9b.3 via the IPC reply header's `rc` field.

- **`(void*)(uintptr_t)id` bridge in handle entry.** Until slice 9b.2 redesigns `nx_handle_entry`, the `object` field stores the vfs open-ID cast to a pointer. This is safe on 64-bit (IDs are 1..108, all non-NULL) and clearly marked with `/* Slice 9b.1: */` comments at every cast site.

- **`nx_handle_alloc` rejects NULL object.** Since IDs are 1-based, `(void*)(uintptr_t)id` is always non-NULL — the alloc guard passes correctly.

## Status at End of Session

- `make test-host` → **469/469 pass** (10 new)
- `make verify-iface-fresh` → 0 drift
- `make verify-registry` → 0 findings (R2, R4, R9)
- Slice 9b.1 complete. Slices 9b.2–9b.4 not yet started.

## Next Steps

- **Slice 9b.2** — Handle entry redesign: replace `{ type, rights, void *object }` with `{ rights, uint32_t id, struct nx_slot *target }`. Update `nx_handle_alloc` signature; update `sys_open` to store id directly in the new field; pre-install stdin/stdout/stderr as `{ rights, id=0, target=&g_char_device_slot }`. ~8 host tests.

---

**Files Changed:**
- `interfaces/idl/fs.json` — change 6 ops to id:u32 API
- `interfaces/idl/vfs.json` — same changes
- `interfaces/idl/char_device.json` — add read op (op_id 3)
- `interfaces/fs.h`, `interfaces/fs_msg.h`, `framework/fs_call.h`, `framework/fs_dispatch.h` — regenerated
- `interfaces/vfs.h`, `interfaces/vfs_msg.h`, `framework/vfs_call.h`, `framework/vfs_dispatch.h` — regenerated
- `interfaces/char_device.h`, `interfaces/char_device_msg.h`, `framework/char_device_call.h`, `framework/char_device_dispatch.h` — regenerated
- `framework/fs_call.c` — updated wrapper signatures
- `framework/vfs_call.c` — updated wrapper signatures
- `components/ramfs/ramfs.c` — internal ID table (id_to_open helper + 1-based IDs)
- `components/procfs/procfs.c` — internal ID table
- `components/vfs_simple/vfs_simple.c` — internal ID table with driver_id
- `components/uart_pl011/uart_pl011.c` — add read op
- `framework/syscall.c` — bridge (void*)(uintptr_t)id at 7 sites
- `test/host/conformance/conformance_fs.c` — updated to uint32_t API
- `test/host/conformance_fs_test.c` — fs_stub updated + local tests
- `test/host/component_ramfs_test.c` — updated to uint32_t API
- `test/host/component_vfs_simple_test.c` — updated to uint32_t API + fake driver
- `test/host/file_syscall_test.c` — fake driver + one direct-alloc test updated
- `test/host/slice_8_0c_test.c` — fake8c driver updated to uint32_t API
- `test/host/slice_8_0d_test.c` — fd8 driver updated to uint32_t API
- `test/host/slice_9b_1_test.c` — NEW: 10 host tests for slice 9b.1
- `test/host/Makefile` — registered slice_9b_1_test.c
