# Session 30: slice 6.3 — file syscalls + HANDLE_FILE destructor + EL0 file round-trip

**Date:** 2026-04-23
**Phase:** 6 slice 6.3
**Branch:** master

---

## Goals

Close the EL0-userspace-to-filesystem loop.  Slice 6.2 left two new
components active in the bootstrap composition but no way for a
userspace program to reach them; slice 6.3 adds three syscalls
(`open` / `read` / `write`), wires the handle layer to vfs_simple's
close via a new `HANDLE_FILE` branch, and proves the whole chain
end-to-end with a baked-in EL0 program.

## Scope choices

- **Fixed-size staging buffer + length cap.**  Read/write use a
  256-byte `NX_FILE_IO_MAX` staging buffer.  Consumers passing
  larger `cap` / `len` get at most that many bytes per call — users
  loop to transfer more.  Matches the channel shape
  (`NX_CHANNEL_MSG_MAX = 256`) so the two bulk-IO kernel paths share
  a size class and neither needs a runtime allocator in v1.
- **Per-byte path copy.**  `copy_path_from_user` reads one byte at a
  time through the user-window bounds check rather than trying to
  bulk-copy `NX_PATH_MAX` bytes.  Motivation: a path that genuinely
  ends before the user-window boundary would fail a bulk copy's
  bounds check even though it's valid.  Per-byte is slower per call
  (128 tiny memcpys vs. one 128-byte memcpy) but paths are
  fundamentally short and the bounds-check correctness matters more
  than the microseconds.
- **Rights derived from open flags.**  `NX_VFS_OPEN_READ →
  NX_RIGHT_READ`, `NX_VFS_OPEN_WRITE → NX_RIGHT_WRITE`.  `CREATE`
  is an open-time flag only and doesn't become a right (nothing
  consumes it after open).  `NX_RIGHT_SEEK` is already defined in
  handle.h but nothing populates it yet — slice 6.4 wires it when
  the seek syscall lands.
- **Close-after-unmount semantics.**  If the `vfs` slot is unmounted
  between `open` and `close`, the handle-close bails on the driver-
  close step (the object leaks in the driver) but still reclaims the
  handle-table slot.  Documented in the syscall body; a host test
  pins the behaviour so a future rewrite can't regress without
  explicit acknowledgement.
- **No per-task handle table yet.**  Still using the single global
  `g_kernel_handles` from slice 5.4.  Per-task tables remain a
  Phase 5 follow-up; no file-syscall feature ride-alongs with it.

## What Was Done

### `framework/syscall.h` — three new syscall numbers + constants

- `NX_SYS_OPEN = 6`, `NX_SYS_READ = 7`, `NX_SYS_WRITE = 8` appended
  before `NX_SYSCALL_COUNT`.  Keeps the ABI stable — prior numbers
  (including the channel syscalls) unchanged.
- `NX_PATH_MAX = 128` and `NX_FILE_IO_MAX = 256` public so EL0
  programs and tests can size their buffers without guessing.

### `framework/syscall.c` — file syscalls + close extension

- `resolve_vfs(out_ops, out_self)` helper.  Looks up the `vfs` slot
  and extracts `iface_ops` (cast to `nx_vfs_ops *`) + `impl`.  Same
  shape as the slice-5.6 `resolve_vfs` pattern vfs_simple itself
  uses to reach `filesystem.root` — one step up the stack.
- `copy_path_from_user(kdst, cap, user_src)`.  Per-byte
  `copy_from_user` loop, stops on NUL.  Returns NX_EINVAL if no NUL
  within `cap` or any byte fails the bounds check.
- `sys_handle_close` gains a `HANDLE_FILE` branch: after looking up
  (type, object) and confirming type == FILE with non-NULL object,
  it calls `resolve_vfs` and dispatches through `vops->close`.  If
  the vfs slot is gone, the driver-close is silently skipped and
  the handle slot still gets freed.
- `sys_open(path, flags)`.  Path copy → resolve_vfs → driver open
  → rights-derive → nx_handle_alloc for a `HANDLE_FILE`.  On
  handle-alloc failure, calls `vops->close` to roll back the
  driver-side open so the per-open pool stays balanced.  Returns
  the handle as `nx_status_t` (handles are uint32, stats are int64;
  the upcast is well-defined).
- `sys_read(h, buf, cap)` / `sys_write(h, buf, len)`.  Shared
  shape: `lookup_file_object(h, need_rights, &obj)` (validates
  type + rights, returns the per-open object pointer);
  `resolve_vfs`; read/write through a kernel staging buffer capped
  at `NX_FILE_IO_MAX`.
- Dispatch table grows three entries.

### `test/kernel/user_prog_file.S` — baked-in EL0 program

Follows `user_prog_chan.S`'s pattern:

1. `open("/hello", READ|WRITE|CREATE)` → handle in x19.
2. `write(w, "[el0-file-ok]", 13)`.
3. `close(w)` — runs the new HANDLE_FILE destructor.
4. `open("/hello", READ)` → handle in x19.
5. `read(r, recv_buf, 64)` → byte count in x20.
6. `close(r)`.
7. `debug_write(recv_buf, x20)` — produces `[el0-file-ok]` on UART.
8. `wfe` park.

All PC-relative; copies into the user window the same way the
channel program does.  Symbol markers `__user_file_prog_start /
_end` link into `.rodata` of `kernel-test.bin`.

### Tests

- **`test/host/file_syscall_test.c`** — 9 tests, same fake-fs
  injection pattern used by `component_vfs_simple_test.c` (slice
  6.2) but extended to stand up a full `vfs_simple` component bound
  to the `vfs` slot so the syscall dispatcher has a complete
  composition to call into:
  - `sys_open_create_write_close_roundtrip` — the headline happy
    path, also asserts `fake_fs.close_calls` grew (proves the
    HANDLE_FILE branch fired).
  - `sys_open_assigns_rights_from_flags` — open WRITE-only, read
    gets NX_EPERM.
  - `sys_read_on_non_file_handle_returns_einval` — alloc a VMO
    handle directly, sys_read rejects.
  - `sys_write_on_stale_handle_returns_enoent`.
  - `sys_open_with_no_vfs_slot_returns_enoent`.
  - `sys_open_null_path_returns_einval`.
  - `sys_open_path_without_nul_within_cap_returns_einval`.
  - `sys_open_then_write_caps_at_file_io_max` — write request
    larger than `NX_FILE_IO_MAX` returns a capped byte count.
  - `sys_handle_close_file_after_unmount_still_closes_handle_slot`
    — documented edge case where driver-close is skipped but
    handle slot is still reclaimed.
- **`test/kernel/ktest_vfs.c`** gains
  `el0_file_open_write_close_reopen_read_roundtrip`: spawns a
  kthread that copies `user_prog_file.S` bytes into the user
  window and drops to EL0.  Yields until the debug_write counter
  rises (the end-to-end signal — every earlier SVC must have
  succeeded for EL0 to reach it), then dequeues the stranded task.
  Live ktest log now shows `[el0-file-ok]` alongside `[el0] hello`
  and `[el0-chan-ok]`.

### Makefile wiring

- `test/host/Makefile`: SRCS += `file_syscall_test.c`.
- Top Makefile: `KTEST_S` += `test/kernel/user_prog_file.S`.

## Key Findings

- **`sys_handle_close` is the right centralisation point.**  By the
  time slice 6.3 landed its `HANDLE_FILE` branch, `sys_handle_close`
  was already doing a type-aware dispatch for `HANDLE_CHANNEL`
  (slice 5.6).  Adding a new type is a two-line change.  This
  suggests future handle types (VMO, PROCESS, ...) should all route
  through the same type-aware close rather than each syscall
  family rolling its own close path.
- **Per-byte path copy pays for itself.**  First draft did a bulk
  `copy_from_user(kpath, user_path, NX_PATH_MAX)` followed by a NUL
  search; any path within ~128 bytes of the user-window end would
  fail even though the path itself was valid.  Per-byte fixes it
  and costs essentially nothing (paths are NUL-terminated within
  a few bytes typically; the loop exits early).
- **Fake-fs fixture carry-over.**  The slice-6.2
  `component_vfs_simple_test.c` already had a fake_fs shape
  tailored to test vfs_simple's dispatch; slice 6.3's
  `file_syscall_test.c` reuses the same pattern but needs a
  fake_fs with actual per-open storage (not just call recording)
  so the `write → close → reopen → read` round-trip pattern can be
  reasoned about.  Two slightly different fakes in two test files
  is fine — consolidating would mean building a shared fixtures
  library, which is over-engineering for two callers.

## Decisions Made

- **Rollback on handle-alloc failure in sys_open.**  If
  `nx_handle_alloc` fails after `vops->open` succeeds, the syscall
  explicitly calls `vops->close` to release the per-open slot.
  Why: leaving it unreleased would leak a driver-side resource on a
  pathological failure path (handle table full).  How to apply:
  any future syscall that allocates both a handle and a driver-side
  resource must follow the "alloc handle AFTER driver resource;
  roll back driver on handle failure" pattern.
- **Close-after-unmount skips the driver.**  If the `vfs` slot is
  gone by the time close runs, the driver-close is silently
  skipped and the handle slot reclaimed.  Why: we have no way to
  reach the old driver once the slot is unbound; the handle slot
  still must be freed so the caller's table doesn't leak.  How to
  apply: any future handle type with an object-side destructor
  that dispatches through a slot needs the same "slot gone →
  reclaim handle, skip destructor" policy.
- **No seek/readdir in 6.3.**  Even though `nx_fs_ops.read / .write`
  advance a cursor and a seek is useful, adding `seek` here would
  double the slice's surface area.  Deferred to 6.4 where the
  directory support pays for the shared interface-header churn
  anyway.

## Status at End of Session

- `make test` → **348/348 pass (51 python + 243 host + 54 kernel), 0
  leaks, 0 errors, exit 0**.  +10 from slice 6.2.
- `make run` (plain kernel.bin under QEMU) boots through to the
  idle loop with the 5-slot composition log unchanged from slice
  6.2 — no regression in the bring-up path.
- `make test-kernel` ktest log shows three EL0 markers back to
  back: `[el0] hello` (slice 5.5), `[el0-chan-ok]` (slice 5.6),
  `[el0-file-ok]` (this slice).  Each one a complete end-to-end
  SVC round-trip through a distinct syscall family.

## Next Steps

Slice 6.4 closes Phase 6:

- Add `readdir` + `seek` to `interfaces/fs.h` and
  `interfaces/vfs.h`.  The two interfaces diverge here: VFS
  readdir will iterate across mount boundaries (Phase 8+),
  fs-driver readdir stops at the current fs.
- Extend the slice-6.1 `conformance_fs` suite with readdir + seek
  cases.  Ramfs passes automatically once it implements them.
- Ramfs grows a one-level directory inode.  v1: only `/` is a
  directory; ramfs's flat filename table becomes the `/` dentry
  list.
- `NX_SYS_READDIR(h, &entry, cap)` + `NX_SYS_SEEK(h, offset,
  whence)` syscalls.  `NX_RIGHT_SEEK` populated on open.
- EL0 demo opens `/`, iterates its dentries, writes each name to
  UART.  Phase 6 closes.

---

**Files Changed:**
- `sources/nonux/framework/syscall.h` — three new syscall numbers; `NX_PATH_MAX` / `NX_FILE_IO_MAX`
- `sources/nonux/framework/syscall.c` — `resolve_vfs`, `copy_path_from_user`, `sys_open`, `sys_read`, `sys_write`; HANDLE_FILE branch in `sys_handle_close`
- `sources/nonux/test/kernel/user_prog_file.S` — new (EL0 round-trip program)
- `sources/nonux/test/kernel/ktest_vfs.c` — add EL0 file round-trip ktest
- `sources/nonux/test/host/file_syscall_test.c` — new (9 syscall-level tests with full vfs_simple + fake_fs composition)
- `sources/nonux/test/host/Makefile` — add file_syscall_test.c
- `sources/nonux/Makefile` — add `test/kernel/user_prog_file.S` to `KTEST_S`
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 6.3 rewritten with as-built details
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 25 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
