# Session 93: Slice 8.0b — Activate Generated handle_msg Shims

**Date:** 2026-05-01
**Phase:** Phase 8 — Runtime Recomposition, Group B (IPC migration)
**Branch:** master

---

## Goals

- Land slice 8.0b: activate generated `handle_msg` shims on all 5 production
  components (`vfs_simple`, `ramfs`, `procfs`, `mm_buddy`, `sched_rr`) and
  replace `uart_pl011`'s smoke-test stub with the real generated shim.
- Components remain dual-callable (direct ops + `handle_msg`) for 8.0c
  equivalence testing.
- `make test` strictly higher than Session 92 baseline (566).

## What Was Done

### 1. framework/ipc.h — reply_payload_len field

Added `uint32_t reply_payload_len` to `struct nx_ipc_message`.  Set by the
receiver's `handle_msg` (via `nx_<iface>_dispatch`) to the number of reply
bytes written in-place at `msg->payload`.  Zero means the dispatcher sends a
header-only reply carrying just the int rc.  Also changed `payload` from
`const void *` to `void *` so dispatch can write the reply struct back into
the same buffer.

### 2. framework/dispatcher.c — build_reply updated for in-place replies

`build_reply` now checks `req->reply_payload_len`.  If > 0, it copies that
many bytes from `req->payload` into the pool entry (the dispatch wrote the
per-op reply struct there).  If 0, falls back to the legacy header-only path
(`nx_reply_header.rc = rc`, `plen = sizeof *hdr`).  The legacy path keeps all
existing handlers that don't use the generated dispatch working transparently.

### 3. tools/gen-iface.py — _dispatch_case() and pointer encoding

Two related changes:

- **`bytes_in` / `bytes_out` encoding:** msg struct fields are now
  `uint64_t <name>; /* pointer encoded as u64 */` instead of inline byte
  arrays.  The caller encodes the pointer as u64; the dispatch casts it back
  to `(const void *)` / `(void *)` for the handler.  This avoids copying
  potentially large buffers through the IPC message.

- **`_dispatch_case()` generator function:** new helper that renders one
  switch-case body for `nx_<name>_dispatch`.  Handles all param types
  (scalar, string_in, bytes_in, bytes_out, opaque_self_handle, struct_in/out,
  struct_inout, void_ptr returns, etc.), writes the reply struct in-place at
  `msg->payload`, and sets `msg->reply_payload_len = sizeof *_r`.  The
  generated `nx_<iface>_dispatch` function is now a proper `static inline`
  function rather than a macro.

- **`bytes_out` omitted from reply structs:** since the handler writes
  directly into the caller's buffer via the encoded pointer, there is no data
  to return in the reply struct for `bytes_out` params.

### 4. framework/*_dispatch.h — regenerated

All five dispatch headers (`char_device_dispatch.h`, `fs_dispatch.h`,
`mm_dispatch.h`, `scheduler_dispatch.h`, `vfs_dispatch.h`) regenerated from
the updated `gen-iface.py`.  Each now contains a full `static inline int
nx_<iface>_dispatch(void *self, const struct nx_<iface>_ops *ops, struct
nx_ipc_message *msg)` function with a switch on `msg->msg_type`.

### 5. interfaces/*_msg.h — regenerated

`char_device_msg.h`, `fs_msg.h`, `vfs_msg.h` regenerated with the updated
`bytes_in`/`bytes_out` pointer encoding.  `mm_msg.h` and `scheduler_msg.h`
were already consistent (no `bytes_*` params in those interfaces).

### 6. Production components — handle_msg wired

All six components updated to include their dispatch header and set `handle_msg`:

- `components/vfs_simple/vfs_simple.c` → `nx_vfs_dispatch`
- `components/ramfs/ramfs.c` → `nx_fs_dispatch`
- `components/procfs/procfs.c` → `nx_fs_dispatch`
- `components/mm_buddy/mm_buddy.c` → `nx_mm_dispatch`
- `components/sched_rr/sched_rr.c` → `nx_scheduler_dispatch`
- `components/uart_pl011/uart_pl011.c` → `nx_char_device_dispatch`

Each component's `component_ops` struct now has `handle_msg` pointing to a
thin wrapper that calls the generated dispatch with the component's ops table
and self pointer.

### 7. test/host/slice_8_0b_test.c — new host test file (+24 tests)

Six test sections:
1. **handle_msg non-NULL checks** (5 tests) — `vfs_simple`, `ramfs`, `procfs`,
   `mm_buddy`, `sched_rr` component_ops all have non-NULL `handle_msg`.
2. **char_device dispatch** (3 tests) — `write` routes correctly, unknown op
   returns `NX_EINVAL`, negative rc sets `bytes_actual = 0`.
3. **mm dispatch** (3 tests) — `page_size` returns value via reply,
   `alloc_pages` returns pointer encoded as u64 (union buf fix for in-place
   write overflow), `max_order` returns value.
4. **scheduler dispatch** (2 tests) — `pick` returns task pointer, `enqueue`
   routes and returns rc.
5. **VFS + FS dispatch** (5 tests) — `open`, `read`, `write`, `close`, `readdir`
   route to the correct op.
6. **Equivalence** (6 tests) — `ASSERT_CALL_EQUIVALENT` confirms direct-vs-dispatch
   call count and rc match for representative ops across all interfaces.

### 8. tools/tests/test_gen_iface.py — test updated

`TestDispatchHeader.test_dispatch_macro_covers_every_op` updated from the old
`#define NX_DEMO_DISPATCH(self, ops, msg)` macro assertion to the new
`static inline int nx_demo_dispatch(` function signature.

## Bugs Fixed

- **test_dispatch_macro_covers_every_op** — test expected old macro form;
  updated to match the new function form.
- **mm_dispatch_alloc_returns_pointer_in_reply** — payload buffer was
  `sizeof(struct nx_mm_msg_alloc_pages)` = 4 bytes but the reply struct
  `nx_mm_reply_alloc_pages` is 8 bytes (`uint64_t rc`); fixed with a
  `union { req; rep; } buf` to ensure the buffer fits both.

## Test Results

```
make test → 590/590 pass (93 python + 374 host + 123 kernel)
make test-interactive → 7/7 pass
make verify-iface-fresh → 0 drift
make verify-registry → 0 findings
```

+24 host tests vs. Session 92 baseline of 566.

## Next Step

Slice **8.0c** — migrate framework production paths to wrappers.  `syscall.c`
(~30 direct `vops->` callsites), `dispatcher.c`, `bootstrap.c`, `component.c`,
`process.c` all bypass the IPC router.  Per-callsite equivalence tests land
*before* the migration; wrappers swap in; equivalence asserted again.
Hot-path perf checkpoint gated to ≤10 µs round-trip on QEMU virt aarch64.
