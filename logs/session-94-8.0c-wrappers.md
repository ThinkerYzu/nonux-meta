# Session 94: Slice 8.0c (partial) — VFS Wrapper Infrastructure

**Date:** 2026-05-01
**Phase:** Phase 8 — Runtime Recomposition, Group B (IPC migration)
**Branch:** master

---

## Goals

- Land slice 8.0c infrastructure: generated VFS blocking-call wrappers,
  updated headers, per-callsite equivalence tests.
- Migrate `syscall.c`'s 31 direct `vops->` callsites to `nx_vfs_*` wrappers.
- Verify `make test` baseline + `make test-interactive` 7/7 stay green.

## What Was Done

### 1. gen-iface.py — DEFAULT_PATH_MAX lowered to 128

`DEFAULT_PATH_MAX` reduced from 4096 to 128 to match `NX_PATH_MAX`
(framework/syscall.h).  User-task kstacks are 4 KiB; an inline 4096-byte
path array in the request struct would overflow the stack inside the kernel
wrapper.  128 bytes covers every path the syscall layer accepts.

Call-header generation updated: replaces the stale
`extern int nx_slot_call_blocking(struct nx_slot *, struct nx_ipc_message *)`
with `#include "framework/slot_call.h"` (the real 4-parameter signature).

`tools/tests/test_gen_iface.py` updated to match the new call-header shape.

All affected headers regenerated via `gen-iface.py all`.

### 2. framework/vfs_call.c — 9 wrapper implementations

New file.  Each wrapper (`nx_vfs_open`, `nx_vfs_close`, `nx_vfs_retain`,
`nx_vfs_read`, `nx_vfs_write`, `nx_vfs_seek`, `nx_vfs_readdir`,
`nx_vfs_mkdir`, `nx_vfs_stat`) has two code paths:

- **Host build (`__STDC_HOSTED__`)**: direct fast path — reads
  `slot->active->descriptor->iface_ops` and calls the op directly.
  Bypasses IPC round-trip so host unit tests keep working without a real
  scheduler.
- **Kernel build**: builds a kstack request struct, allocates a kstack reply
  struct, constructs `struct nx_ipc_message`, and calls
  `nx_slot_call_blocking(slot, &msg, &reply_buf, sizeof(reply_buf))`.
  The caller's task blocks on `reply_waitq`; posix_shim wakes it with the
  reply payload.

`copy_path()` static helper copies NUL-terminated paths into the fixed
128-byte request field (safe: all kernel paths bounded by `NX_PATH_MAX`).

### 3. Build system updates

- `Makefile`: `framework/vfs_call.c` added to `FW_C`.
- `test/host/Makefile`: `../../framework/vfs_call.c` and
  `slice_8_0c_test.c` added to `SRCS`.

### 4. test/host/slice_8_0c_test.c — 23 new host equivalence tests

Per-callsite equivalence tests in 9 sections:
1. **open**: wrapper opens existing file (OK), missing file (ENOENT),
   equiv-direct (both paths give same rc).
2. **close**: wrapper increments close_calls counter.
3. **read**: wrapper reads known content; seek-back/reread gives same bytes.
4. **write**: wrapper stores data, subsequent read verifies it.
5. **seek**: wrapper moves cursor; seek returns correct position.
6. **stat**: wrapper returns correct kind/size; missing path → ENOENT;
   direct-vs-wrapper rc/kind/size match.
7. **readdir**: first entry returned; past-end → ENOENT;
   direct-vs-wrapper entry matches.
8. **mkdir**: wrapper calls op, rc propagated; direct-vs-wrapper match.
9. **retain**: wrapper bumps refcount; two sequential retains each +1.
   Plus null-slot robustness (NULL slot → ENOENT/error) for open/read/stat.

All 23 tests use the fixture pattern from `file_syscall_test.c`:
vfs slot ← vfs_simple ← local fake_fs driver.

### 5. syscall.c migration — BLOCKED (not landed)

The planned migration of `syscall.c`'s 31 direct `vops->` callsites to
`nx_vfs_*` wrappers was attempted but reverted because the kernel's
`nx_slot_call_blocking` does not complete in the current QEMU environment:

**Root cause investigation:** The `hook_inspector_observe_only_does_not_alter
_blocking_call_result` kernel ktest — which performs an end-to-end blocking
call to the uart slot — hangs indefinitely.  This test was already hanging in
the Session 93 baseline (exact code from `git HEAD`, no new files added),
confirming the regression predates slice 8.0c.  The ktest
`cap_scan_rejects_forged_slot_ref_cap_during_blocking_call` (which returns
before enqueue) still passes, placing the failure somewhere in the
dispatcher→posix_shim→waitq wake path.

**Impact on slice 8.0c:** With `syscall.c` migrated to `nx_vfs_*`, EL0
programs that call `open`/`read`/`write` block on `reply_waitq` and never
resume — `make test-interactive` drops from 7/7 to 0/7.  Reverting
`syscall.c` to direct `vops->` calls restores interactive tests to 7/7.

**Next step (pre-8.0c migration):** Diagnose and fix the blocking-call
environmental issue.  Likely suspects: QEMU ARM64 memory ordering (MPSC
`nx_mpsc_pop` retry path), sched_rr runqueue corruption, or binary layout
sensitivity in the uart payload (`static const char payload_hi[] = "y"` is
interpreted as a `nx_char_device_msg_write` struct — adjacent bytes in
`.rodata` determine the `len` field, which changed with the build layout).

## Bugs Fixed / Investigated

- **verify-iface-fresh drift** — `gen-iface.py`'s `render_call_header` was
  still emitting the stale 2-param extern; fixed to emit the `slot_call.h`
  include, regenerated all call headers.

## Test Results

```
make test-tools:     93/93  python  PASS
make test-host:      397/397 host   PASS  (+23 vs. Session 93 baseline of 374)
make verify-iface-fresh:  0 drift
make verify-registry:     0 findings
make test-interactive:    7/7 PASS  (echo_cat, echo_hello, echo_pipe, ls_root,
                                     mkdir_tmp, ps_smoke, visible_prompt)
make test-kernel:    ~11 kernel tests run before hook_inspector hang (pre-existing
                     environmental issue — identical result with Session 93 baseline)
```

## Next Step

**Diagnose and fix the `nx_slot_call_blocking` hang** so the kernel's
IPC round-trip completes reliably.  Then re-apply the `syscall.c` migration
from this session (which is already written and tested on the host side) and
land slice **8.0c** properly.  Suspected fix area: `ktest_bootstrap.c`'s
`hook_inspector_kthread` sends a 1-byte payload that the char_device dispatch
interprets as `nx_char_device_msg_write` — the garbage `len` field drives
`uart_write` into a large or crashing loop depending on binary layout.  The
proper fix is to make `ktest_bootstrap.c` send a correctly-sized request struct.
