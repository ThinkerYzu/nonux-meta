# Session 95: Slice 8.0c — `syscall.c` Migration Complete

**Date:** 2026-05-02
**Phase:** Phase 8 — Runtime Recomposition, Group B (IPC migration)
**Branch:** master

---

## Goals

- Investigate and fix the `nx_slot_call_blocking` hang that blocked the
  `syscall.c` migration in Session 94.
- Migrate all remaining `vops->` callsites in `syscall.c` to `nx_vfs_*`
  blocking wrappers — closing slice 8.0c.

---

## Root Cause Investigation

### Hypothesis: kstack overflow in `sys_getdents64`

The Session 94 blocker: full migration broke `make test-interactive` 0/7
(QEMU 120-second timeout, no busybox output), even though 123/123 kernel
tests passed and the partial migration (sys_exec only) worked fine.

**Root cause identified:** `sys_getdents64` allocated `uint8_t staging[4096]`
on the kernel stack (`enum { STAGING_MAX = 4096 };` + `uint8_t staging[STAGING_MAX];`).

Stack budget analysis for the init kthread:
- Initial kthread frames (`nx_init_busybox_kthread` → `drop_to_el0`): ~80 bytes
- SAVE_TRAPFRAME on SVC entry: 272 bytes
- `on_sync` + `sys_getdents64` frame (staging[4096] + locals): ~4200 bytes

Total peak: **~4552 bytes** — overflows the 4 KiB (`pmm_alloc_pages(1)`)
kernel stack by ~456 bytes.

**Without the blocking-call migration**: the overflow was small (~456 bytes)
and the adjacent physical memory (typically dead kthread-entry frames at the
top of the same page) wasn't live, so the overflow was silent.

**With the blocking-call migration**: each `nx_vfs_readdir` call in the
`getdents64` loop added ~500 bytes of frame (`nx_vfs_readdir` →
`nx_slot_call_blocking` → `nx_waitq_wait_with_deadline` → `sched_check_resched`).
The combined overflow (~956+ bytes) corrupted adjacent physical memory —
likely the dispatcher kthread's kstack — causing the dispatcher to behave
incorrectly and never deliver the VFS reply, hanging the init task on
`reply_waitq` forever.

**Why the partial migration (sys_exec only) worked**: `sys_exec` doesn't call
`sys_getdents64` — its VFS calls are `nx_vfs_open` + `nx_vfs_read` (loop)
+ `nx_vfs_close`, all with small frame overhead. The kstack budget for
`sys_exec` was within the 4 KiB limit.

### Fix: heap-allocate the staging buffer

Changed `uint8_t staging[STAGING_MAX]` to `uint8_t *staging = malloc(STAGING_MAX)`
with `free(staging)` at all three exit paths in `sys_getdents64`:
- Loop error: `if (rc != NX_OK) { free(staging); return NX_LINUX_EINVAL; }`
- `copy_to_user` failure: `if (rc != NX_OK) { free(staging); return NX_LINUX_EINVAL; }`
- Normal exit: `free(staging); return result;`

---

## What Was Done

### `framework/syscall.c` — full migration (commit `217724b`)

14 callsites migrated; `resolve_vfs()` helper removed (now dead code,
causing `-Werror=unused-function`, so removed along with the migration).

Sites migrated:

| Syscall | Ops migrated |
|---------|-------------|
| `sys_handle_close` | `vops->close` → `nx_vfs_close` |
| `sys_open` | `vops->stat` → `nx_vfs_stat`; `vops->open` → `nx_vfs_open`; `vops->close` (rollback) → `nx_vfs_close`; removed `resolve_vfs` call (kept `vfs_slot` lookup already present) |
| `sys_read` | `vops->read` → `nx_vfs_read` |
| `sys_write` | `vops->write` → `nx_vfs_write` |
| `sys_seek` | `vops->seek` → `nx_vfs_seek` |
| `sys_readdir` (legacy) | `vops->readdir` → `nx_vfs_readdir` |
| `sys_fork` | FILE retain loop: `vops->retain` → `nx_vfs_retain` |
| `sys_fstatat` | `vops->stat` → `nx_vfs_stat` |
| `sys_getdents64` | `vops->readdir` → `nx_vfs_readdir` (in loop) **+ heap-allocate staging** |
| `sys_mkdirat` | `vops->mkdir` → `nx_vfs_mkdir` |
| `sys_dup2` | `vops->close` → `nx_vfs_close`; `vops->retain` → `nx_vfs_retain` |
| `sys_fcntl` | `vops->retain` → `nx_vfs_retain` |

Pattern for each site: `nx_slot_lookup("vfs")` replaces `resolve_vfs()`;
`nx_vfs_*` wrappers route through `nx_slot_call_blocking` in the kernel
build and use the host fast-path in host tests.

### Removed

- `resolve_vfs()` static helper (dead after migration).

---

## Test Results

```
make test-tools:        93/93  PASS
make test-host:        397/397 PASS  (same as Session 94 baseline)
make verify-iface-fresh: 0 drift
make verify-registry:    0 findings
make test-interactive:   7/7 PASS
  echo_cat, echo_hello, echo_pipe, ls_root, mkdir_tmp, ps_smoke, visible_prompt
make test-kernel:       123/123 PASS  (ktest: running 123 test(s))
```

Notable ktest improvement: `exec_fork_child_execs_init_parent_waits_for_exit_17`
now shows `[exec-parent][el0-elf-ok][exec-ok]PASS` — the ELF exec round-trip
via blocking VFS calls works end-to-end in the fork-exec path.

---

## Next Step

**Slice 8.0d** — migrate component-to-component calls: `vfs_simple` →
ramfs/procfs mount-table dispatch.  The `vfs_simple` component currently
calls into the filesystem driver via direct `ops->` calls; these should
go through `nx_slot_call_blocking` just like the syscall-layer calls now do.
