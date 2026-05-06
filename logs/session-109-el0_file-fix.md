# Session 109: el0_file kernel test — root cause found and fixed

**Date:** 2026-05-05
**Phase:** Bug fix (no new slice)

---

## Goals

- Identify the root cause of `el0_file_open_write_close_reopen_read_roundtrip` kernel test failure.
- Fix it.

## What Was Done

### Debugging approach

Added targeted kprintfs to narrow down the failure:
1. `[el0_file] kthread started` / `[el0_file] dropping to EL0` — confirmed the kthread was reaching EL0.
2. `[sys_read] enter h=0x%x cap=%lu` — showed sys_read was entered with `h=0xfffffffc`.
3. `[scb] task=el0_file slot=vfs active=1 inflightbuf=0x0` in `nx_slot_call_blocking` — confirmed 4 blocking IPC calls fired, all with valid state.
4. `[sys_open] vfs_id=%u h=0x%x` — never fired, confirming sys_open always returned before `nx_handle_alloc_resource`.

### Key findings from instrumented run

```
[el0_file] kthread started base=0xx
[el0_file] dropping to EL0
[scb] task=el0_file slot=vfs active=1 inflightbuf=0x0000000000000000  ← stat #1
[scb] task=el0_file slot=vfs active=1 inflightbuf=0x0000000000000000  ← open #1
[scb] task=el0_file slot=vfs active=1 inflightbuf=0x0000000000000000  ← stat #2
[scb] task=el0_file slot=vfs active=1 inflightbuf=0x0000000000000000  ← open #2
[sys_read] enter h=0xfffffffc cap=64
```

Only 4 IPC calls (2 stat + 2 open), no `[sys_open]` output, then sys_read with garbage handle.  The EL0 program's sys_write and sys_handle_close receive the garbage handle from open#1 and return NX_ENOENT immediately (no IPC needed for invalid handles).

### Root cause

`nx_vfs_open` in `framework/vfs_call.c`:

```c
int rc = nx_slot_call_blocking(slot, &msg, &reply, sizeof reply);
if (rc != NX_OK) return 0;   // BUG
return reply.rc;
```

`nx_slot_call_blocking` returns `task->in_flight_reply_rc`, which `posix_shim_handle_msg` sets to `(int)hdr->rc` — the first 4 bytes of the reply payload interpreted as `int32_t`.

For `NX_VFS_OP_OPEN`, the reply payload is `struct nx_vfs_reply_open { uint32_t rc; }` where `rc` is the 1-based vfs_simple ID.  A successful open returns vfs_id=1, so `in_flight_reply_rc = 1`.

The old guard `if (rc != NX_OK)` treats 1 (success) as a framework error because `NX_OK = 0`.  It returns 0 (failure).  sys_open then returns `NX_ENOENT`.  Both opens fail.  EL0 stores `-4` in x19 (the r-handle), sys_read gets `h = 0xFFFFFFFC`, handle lookup fails, returns `NX_ENOENT`.  EL0 passes that as the `len` to `sys_debug_write`, which hits the 4096-byte safety cap.  `debug_write_calls` stays 0.  ktest fails.

### Additional finding: test order reversal within ktest_vfs.c

The linker places `kernel_test_registry` section entries in **reverse definition order** within `ktest_vfs.c`.  So `el0_readdir` (defined last, line 257) runs **before** `el0_file` (line 182).  This is why the commit message for `ad54fa2` said "ghost IPC traffic that starves el0_file" — el0_rdr's ghost IPCs were starving tests that came after el0_readdir in the section, including tests from subsequent ktest files, NOT el0_file directly.  El0_file was failing due to the `nx_vfs_open` bug regardless.

### Fix

`framework/vfs_call.c` — change the post-`nx_slot_call_blocking` guard from `rc != NX_OK` to `rc < 0`:

```c
KERNEL_INIT_MSG(msg, task, slot, NX_VFS_OP_OPEN, req);
int rc = nx_slot_call_blocking(slot, &msg, &reply, sizeof reply);
/* nx_slot_call_blocking returns in_flight_reply_rc = (int32_t)vfs_id.
 * A positive vfs_id indicates success; only negative rc means a
 * framework-level error.  Do NOT treat non-zero as failure here. */
if (rc < 0) return 0;
return reply.rc;
```

This correctly handles:
- `rc < 0` (NX_EINVAL=-1, etc.): framework error → return 0 (failure)
- `rc = 0` (NX_OK): vfs_id=0 = open failure → `reply.rc = 0` → return 0
- `rc > 0` (vfs_id=1,2,...): success → falls through to `return reply.rc = vfs_id`

The read/write wrappers (`nx_vfs_read`, `nx_vfs_write`) use `if (rc != NX_OK) return rc` which is correct for them — positive byte counts reach the early return and are returned directly.  Only open had the wrong guard.

## Test Results

- `make test-tools` → **102/102 pass**
- `make test-host` → **476/476 pass**
- `make test-kernel` → **138/151 pass** (was 75/151); el0_file now PASS (`[el0-file-ok]` on UART)
- `make verify-registry` → 0 findings; `make verify-iface-fresh` → 0 drift

**13 remaining failures (all pre-existing):**
- **10 from ktest_9b_3.c**: `nx_slot_lookup("char_device")` returns NULL because the registered slot name is "char_device.serial".  Fix: update the lookup string in ktest_9b_3.c (and verify process.c / syscall.c which use the same name).
- **3 from posix_musl tests**: `nx_console_write_calls() >= 1` assertion; pre-existing, unrelated to this session.

## Files Changed

- `framework/vfs_call.c` — 1-line guard fix in `nx_vfs_open`

## Next Steps

1. **Phase 9 — per-process MM rework** (L3 4 KiB pages, VMAs, demand paging, COW fork).
2. **ktest_9b_3 slot-name fix** — change `nx_slot_lookup("char_device")` to `"char_device.serial"` in ktest_9b_3.c; audit process.c and syscall.c for the same lookup.
