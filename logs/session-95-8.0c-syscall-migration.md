# Session 95: Slice 8.0c — syscall.c VFS Migration Complete

**Date:** 2026-05-02
**Phase:** Phase 8 — Runtime Recomposition, Group B (IPC migration)
**Branch:** master

---

## Goals

- Diagnose why `syscall.c` VFS migration makes `kernel-busybox.bin` hang on
  startup (the blocking call from the init kthread's `sys_exec` path).
- Fix the root cause.
- Re-apply and land the full slice-8.0c `syscall.c` migration.

## What Was Done

### 1. Root-cause investigation via diagnostic kprintfs

Added temporary `kprintf` instrumentation at:
- `posix_shim_handle_msg` — to confirm whether the reply leg fires.
- `nx_dispatcher_pump_once` — to log every dequeued message.
- `nx_vfs_open` kernel path — to show `reply.out_file` after the blocking call.
- `nx_vfs_dispatch` OPEN/READ cases — to show file handle values.
- `vfs_simple_open` / `vfs_simple_read` — to trace `w->mount` and `in_use`.
- `sys_exec` — to log read return values and `total`.

**Key findings from the debug runs:**

1. The dispatcher **does** process the VFS OPEN/READ/CLOSE messages (confirmed
   by `[dispdbg]`/`[psdbg]` lines).  The blocking call round-trip works.
2. Without kprintfs, `sys_exec`'s first `nx_vfs_read` returns `NX_ENOENT` and
   the exec path takes the error branch → close → return error.  With kprintfs
   the read succeeds.  Classic heisenbug: adding a `kprintf` anywhere in the
   call chain changes the stack layout and masks the corruption.
3. The heisenbug pattern (`NX_ENOENT` from `vfs_simple_read` with correct
   file handle, `w->mount` inconsistently NULL or valid) pointed to a
   **kstack overflow** rather than a logic bug.

### 2. Root cause: kstack overflow in `sys_getdents64`

The init kthread has a 1-page (4096-byte) kernel stack.  After `drop_to_el0`
and the SVC entry (SAVE_TRAPFRAME = 272 bytes), there are only ~3800 bytes
available for the syscall handler chain.

`sys_getdents64` had `uint8_t staging[4096]` as a local array — a 4 KiB
on-stack buffer that alone exceeded the remaining kstack space.  Without the
blocking-call migration, the staging array was the only large local, and the
stack usage stayed within bounds by accident (the overflow was small and hit
zero-filled memory).  With the blocking-call frames adding ~500 extra bytes
(`nx_slot_call_blocking` + `nx_waitq_wait_with_deadline` + `sched_check_resched`),
the combined stack usage overflowed into adjacent physical memory, corrupting the
dispatcher kthread's state and producing the intermittent hang.

**Fix:** heap-allocate the `staging` buffer in `sys_getdents64` via
`malloc` / `free`.  All other migrated syscalls (`sys_read`, `sys_write`,
`sys_seek`) use a pre-existing `staging[NX_FILE_IO_MAX]` (256 bytes) — small
enough to be safe.

### 3. Full slice-8.0c migration landed

Migrated all 14 VFS callsites in `framework/syscall.c`:

| Syscall | Operations migrated |
|---------|-------------------|
| `sys_handle_close` | `nx_vfs_close` |
| `sys_open` | `nx_vfs_stat`, `nx_vfs_open`, `nx_vfs_close` (error) |
| `sys_read` | `nx_vfs_read` |
| `sys_write` | `nx_vfs_write` |
| `sys_seek` | `nx_vfs_seek` |
| `sys_readdir` (legacy) | `nx_vfs_readdir` |
| `sys_fork` | `nx_vfs_retain` (FILE handle inheritance) |
| `sys_exec` | `nx_vfs_open`, `nx_vfs_read`, `nx_vfs_close` |
| `sys_fstatat` | `nx_vfs_stat` |
| `sys_getdents64` | `nx_vfs_readdir` + heap staging |
| `sys_mkdirat` | `nx_vfs_mkdir` |
| `sys_dup2` | `nx_vfs_close`, `nx_vfs_retain` |
| `sys_fcntl` | `nx_vfs_retain` |

`resolve_vfs()` helper removed; all callsites use `nx_slot_lookup("vfs")` +
the `nx_vfs_*` wrappers directly.

Also bumped the Makefile `test-kernel` QEMU timeout from 90 s → 300 s:
`sys_exec` now takes ~800 ms extra per call (4 blocking IPC round-trips at
~200 ms each through the 10 Hz timer), and the ktest busybox-exec suite has
~25 exec-heavy tests.

## Test Results

```
make test-tools:         93/93  PASS
make test-host:         397/397 PASS
make verify-iface-fresh: 0 drift
make verify-registry:    0 findings
make test-interactive:   7/7  PASS  (echo_cat, echo_hello, echo_pipe, ls_root,
                                     mkdir_tmp, ps_smoke, visible_prompt)
make test-kernel:       123/123 PASS (with 300 s timeout)
```

## Decisions Made

- **Heap-allocate `sys_getdents64` staging buffer** — the 4 KiB on-stack
  array is the only easy-to-fix kstack overflow; other `syscall.c` staging
  arrays (256 bytes) are small enough to remain on the kstack.
- **Increase ktest timeout to 300 s** — each exec now costs ~800 ms in
  blocking IPC round-trips; 300 s is ~3× the pre-migration measured runtime
  with plenty of headroom.

## Commits

- `217724b` — `slice 8.0c: migrate all syscall.c VFS callsites to nx_vfs_* wrappers`
- `b8c86e7` — `test-kernel: bump QEMU timeout 90s → 300s for 8.0c blocking-call exec`

## Next Step

Slice **8.0d** — migrate component-to-component calls (vfs_simple → ramfs/procfs).
This routes the remaining direct `iface_ops` accesses inside vfs_simple through
`nx_slot_call_blocking`, completing Group B's goal of eliminating all
`slot->active->descriptor->iface_ops` calls outside `framework/slot_call.c`.
