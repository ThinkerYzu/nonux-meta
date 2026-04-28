# Session 27: slice 5.6 — channels (closes Phase 5)

**Date:** 2026-04-23
**Phase:** 5 slice 5.6 (closes Phase 5)
**Branch:** master

---

## Goals

Ship the first real EL0-consumable API: `HANDLE_CHANNEL` +
`nx_channel_create / _send / _recv` syscalls. An EL0 userspace
program creates a channel, sends bytes on one endpoint, receives them
on the peer, and forwards the received payload to UART — end-to-end
proof that the handle framework, syscall dispatcher, rights
attenuation, and IPC-style message passing all work together.

## Scope choices

Narrowed vs the original plan in two places:

- **Self-loop, not cross-process.** The plan envisioned an in-kernel
  echo component bound to a channel. That requires wiring the IPC
  router into the channel framework — a separate integration layer
  that's not on the Phase 5 critical path. The EL0 test instead owns
  both endpoints: `end0` sends, `end1` receives, same address space.
  Still validates every syscall + every framework layer end-to-end.
- **Non-blocking recv (NX_EAGAIN when empty).** Blocking recv needs
  scheduler cooperation (wake-on-send queues); deferred. For the v1
  test, the program sends before it reads, so recv sees data and
  never blocks.

## What Was Done

### `framework/channel.{h,c}` — new

- `struct nx_channel_endpoint` is opaque to callers. Internally, a
  pair of endpoints share one `struct nx_channel` allocation with an
  atomic refcount starting at 2; closing an endpoint decrements, and
  the allocation is freed when refcount hits 0.
- `peer_of(e)` is `&e->chan->e[1 - e->idx]` — no extra field; each
  endpoint knows its index at create time.
- Fixed-size messages: 256 bytes per message × 8 ring slots per
  inbox. `nx_channel_send(e, ...)` writes into `peer_of(e)`'s ring
  (so the peer receives it); `nx_channel_recv(e, ...)` reads from
  `e`'s own ring.
- Error surface:
  - `NX_OK` on success (`_send` returns byte count; `_recv` returns
    byte count).
  - `NX_EINVAL` for NULL args, oversized payload, zero cap.
  - `NX_EBUSY` if either endpoint is closed or the peer's ring is
    full.
  - `NX_EAGAIN` (new; added to `registry.h`) if recv finds the ring
    empty.
  - `NX_ENOMEM` if recv's cap is smaller than the queued message (the
    message stays queued for the next attempt).
- `nx_channel_endpoint_close` is idempotent and free-on-last-ref.

### `framework/syscall.{h,c}` — channel syscalls + copy_*_user helpers

- Three new syscall numbers: `NX_SYS_CHANNEL_CREATE`,
  `NX_SYS_CHANNEL_SEND`, `NX_SYS_CHANNEL_RECV`.
- `copy_from_user` / `copy_to_user` helpers. On kernel, they bounds-
  check the user pointer against `mmu_user_window_base / _size` and
  reject anything outside — catches the "kernel-pointer passed in as
  user buffer" class of misuse. On host they're unchecked memcpy
  (host syscall tests hand-construct kernel-addressed trap frames).
- `sys_channel_create(end0_user_ptr, end1_user_ptr)` allocates the
  channel, allocs two CHANNEL handles (each with
  `READ|WRITE|TRANSFER` rights), and `copy_to_user`s the handle IDs.
  Error paths unwind cleanly: endpoints closed, handles freed.
- `sys_channel_send(h, user_buf, len)` looks up the handle, checks
  the type is CHANNEL and rights include WRITE, copies the user
  payload into a kernel staging buffer (so a racing mutator on the
  userspace side can't change the bytes after our bounds check), then
  calls `nx_channel_send`.
- `sys_channel_recv(h, user_buf, cap)` — symmetric: staging buffer,
  bounds-check, RIGHT_READ required.
- `sys_handle_close` is now **type-aware**: before calling
  `nx_handle_close`, it looks up the slot; if type is
  `NX_HANDLE_CHANNEL`, it calls `nx_channel_endpoint_close(object)`
  first. Future handle types (VMO, PROCESS, IRQ, FILE) hook in the
  same way.

### Host tests (+11 in `test/host/channel_test.c`)

- `channel_create_returns_two_distinct_endpoints`.
- `channel_send_on_a_arrives_on_b` — the cross-endpoint routing
  (send on A goes to B's inbox, not A's).
- `channel_send_and_recv_are_fifo` — 4-message ordering.
- `channel_recv_on_empty_returns_eagain`.
- `channel_send_too_large_returns_einval`.
- `channel_send_full_ring_returns_ebusy` + recover after recv.
- `channel_recv_with_cap_too_small_returns_enomem_and_keeps_message`
  (message stays queued for retry with bigger buffer).
- `channel_send_after_peer_close_returns_ebusy` + tests the refcount
  lifetime (one endpoint close keeps channel alive; second close
  frees it).
- `channel_double_close_on_same_endpoint_is_idempotent`.
- `channel_endpoint_depth_tracks_pending_messages`.
- `channel_null_args_rejected_cleanly` for every API entry point.

### Kernel test (+1 in `test/kernel/ktest_channel.c`)

- `el0_channel_create_send_recv_round_trip` — the flagship test.
  Spawns a kthread whose entry memcopies the baked-in channel-using
  EL0 program (`test/kernel/user_prog_chan.S`) into the MMU's user
  window and calls `drop_to_el0`. The EL0 program does four SVCs in
  a row: CHANNEL_CREATE, CHANNEL_SEND, CHANNEL_RECV, DEBUG_WRITE
  (with the received bytes). The kernel test yields until the
  debug_write counter rises — a counter increment proves every
  earlier SVC succeeded end-to-end. Live ktest log shows
  `[el0-chan-ok]` — the channel payload traveling from EL0 through
  the kernel's IPC framework back out to UART.

### `test/kernel/user_prog_chan.S` — new EL0 program

PC-relative assembly so the test can memcpy the bytes into the user
window at a different VA and they execute correctly. Storage slots
for the two channel handles and the recv buffer live inside the
program, which means the kernel's `copy_to_user` writes directly
into the relocated user-window allocation — no separate stack
manipulation.

### Collateral: pre-existing test fixes

Two earlier tests (from slices 5.3/5.4) used `NX_HANDLE_CHANNEL` as
an opaque type tag with a dummy `&local_int` "object":

- `test/host/syscall_test.c :: syscall_handle_close_routes_to_current_table`
- `test/kernel/ktest_syscall.c :: syscall_handle_close_through_svc_closes_handle_in_kernel_table`

Both went through `sys_handle_close` which now runs the channel
destructor on CHANNEL objects — dereferencing the dummy int as a
channel endpoint segfaults. Fix: use `NX_HANDLE_VMO` instead (no
destructor yet), with an explanatory comment.

Net: `make test` → **306/306** (51 python + 208 host + 47 kernel),
up from 295 → **+11 host + 1 kernel = +12 tests**. Phase 5 closes.

## Key Decisions

- **Fixed 256-byte max message, 8-slot ring.** Power-of-two sizes
  keep the circular-index math one `& (N-1)` away, and the total
  per-endpoint footprint is ~2 KiB (manageable for dozens of channels
  on a real system). Variable-length messages + size prefix land
  with the first real protocol consumer (VFS, likely Phase 6).
- **Staging buffer for copy_from_user -> nx_channel_send.** The
  alternative is passing the user pointer straight to
  `nx_channel_send` which then `memcpy`s into its ring. That skips
  one memcpy but creates a TOCTOU: between copy_from_user's bounds
  check and the eventual memcpy, a multi-threaded EL0 could mutate
  the buffer. 256 bytes of stack staging is cheap insurance.
- **Bounds-checking `copy_*_user` against the user window, not page
  tables.** v1 is single-window; a per-task TTBR0 world will need a
  proper "is this user VA mapped readable for this task" check. The
  shape of the helper doesn't change — only its predicate does.
- **`sys_handle_close` runs object destructors based on `type`.**
  Open question: what about `nx_handle_duplicate` — if two handles
  point at the same channel endpoint, closing one would now close
  the underlying object for both. The slice-5.3 handle semantics
  say "duplicate is a new handle to the same object"; combined with
  channel refcounting, each duplicated handle needs to hold its own
  reference. Today `duplicate` doesn't touch the channel refcount,
  so a duplicate-then-close pair breaks the invariant. **Follow-up**:
  either make duplicate type-aware too (bump the channel refcount)
  or separate handle-close from object-close more cleanly. Not yet
  exercised because no test duplicates a CHANNEL handle.
- **Refcount on the shared channel, not per endpoint.** Alternative:
  each endpoint has its own refcount, and "peer closed" propagates
  over the pointer link. Simpler with shared refcount: one atomic
  per channel, single free site.

## Known Issues / Follow-ups

- **Handle-duplicate + channel-close interaction**, per above. Safe
  only as long as no slice 5.x test duplicates CHANNEL handles.
  Before EL0 code starts sharing channels across contexts, this
  needs to land.
- **Blocking recv** — scheduler-backed wait queue on the endpoint.
  Pairs with a real service that needs to sleep pending a message.
- **Per-task handle table + per-task TTBR0** — the slice-5.5
  deferred items remain. Once a second EL0 task exists, the global
  `g_kernel_handles` breaks down.
- **Variable-length messages + size prefix** — driven by the first
  real protocol consumer (Phase 6 VFS).
- **`NX_RIGHT_TRANSFER` enforcement when passing a channel handle
  over another channel** — the right exists but nothing checks it
  yet. Lands with cross-process channels.

## Phase 5 closes

The full Phase 5 arc across six sessions (22–27):

1. Slice 5.1 — MMU on, identity map (Session 22).
2. Slice 5.2 — `components/mm_buddy/` + `interfaces/mm.h` (Session 23).
3. Slice 5.3 — handle framework (Session 24).
4. Slice 5.4 — SVC from EL1 (Session 25).
5. Slice 5.5 — first EL0 process (Session 26).
6. Slice 5.6 — channels (this session).

Validation per IMPLEMENTATION-GUIDE Phase 5: *"Userspace process runs
in EL0, creates a channel, sends/receives a message via handle
syscalls."* Met end-to-end. Phase 5 complete.

Deferred items across the phase:
- Per-task TTBR0 (kernel to high half).
- `nx_process_exit` + TASK_ZOMBIE.
- Blocking recv.
- `copy_from_user` with page-fault fixup.
- Handle-duplicate ↔ channel-refcount interaction.
- User window PMM reservation.

## Next Actions

**Phase 6 — VFS and Ramfs.** `components/vfs_simple/` with
mount/open/read/write/readdir; `components/ramfs/` in-memory
filesystem; `HANDLE_FILE` destructor on close; first real VFS-
consuming EL0 program that creates, writes, and reads a file. See
[IMPLEMENTATION-GUIDE.md §Phase 6](../IMPLEMENTATION-GUIDE.md#phase-6-vfs-and-ramfs).

## Files Touched

- `framework/channel.h` — **new**
- `framework/channel.c` — **new**
- `framework/registry.h` — `NX_EAGAIN = -11`
- `framework/syscall.h` — `NX_SYS_CHANNEL_CREATE/_SEND/_RECV`
- `framework/syscall.c` — three syscall bodies, copy_*_user helpers,
  type-aware `sys_handle_close`
- `test/host/channel_test.c` — **new**, 11 cases
- `test/host/syscall_test.c` — fix `syscall_handle_close_routes_to_current_table`
- `test/kernel/ktest_channel.c` — **new**, 1 case
- `test/kernel/ktest_syscall.c` — fix matching case
- `test/kernel/user_prog_chan.S` — **new**, EL0 program
- `Makefile` — `FW_C += channel.c`; `KTEST_C += ktest_channel.c`;
  `KTEST_S += user_prog_chan.S`
- `test/host/Makefile` — SRCS += channel_test.c + framework/channel.c
