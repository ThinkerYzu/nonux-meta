# Session 107: 9b.3 — Route read/write/seek Through Slot Calls

**Date:** 2026-05-04
**Phase:** 9b (Handles as Capabilities)
**Branch:** master

---

## Goals

- Implement slice 9b.3: replace direct `nx_console_read`/`nx_console_write` calls in the sys_read/sys_write RESOURCE arms with `nx_char_device_read`/`nx_char_device_write` wrappers that route via `nx_slot_call_blocking`.
- Create `framework/char_device_call.c` — the missing wrapper implementation.
- Add ~10 kernel tests for the new routing paths.
- Keep all 477 host tests and 141 existing kernel tests passing.

## What Was Done

### framework/char_device_call.c (NEW)

New file implementing `nx_char_device_write`, `nx_char_device_read`, and `nx_char_device_rx_byte` following the same pattern as `vfs_call.c`:

- **Kernel path**: `nx_char_device_write` and `nx_char_device_read` use `nx_slot_call_blocking` with the char_device IDL message structs (`NX_CHAR_DEVICE_OP_WRITE`/`NX_CHAR_DEVICE_OP_READ`).  The calling task blocks on `reply_waitq`; the dispatcher kthread handles the message and calls `uart_pl011_write`/`uart_pl011_read`; the reply unblocks the caller.
- **Host path**: direct `iface_ops->write`/`iface_ops->read` calls, bypassing the scheduler (same as vfs_call.c and fs_call.c).
- **`nx_char_device_rx_byte`**: no-op stub in the kernel path (IRQ context; the ISR calls `nx_console_rx_isr` directly rather than through this interface).

### framework/syscall.c — RESOURCE arm updates

**sys_read RESOURCE arm** (slice 9b.2 → 9b.3):
- Before: `if (target == char_slot) { nx_console_read(staging, cap) } else { nx_vfs_read(...) }`
- After: `if (target == char_slot) { nx_char_device_read(target, id, staging, cap) } else { nx_vfs_read(target, id, staging, cap) }`
- Both branches now go through `nx_slot_call_blocking`; `nx_console_read` is no longer called directly from the syscall layer.

**sys_write RESOURCE arm** (slice 9b.2 → 9b.3):
- Before: `if (target == char_slot) { nx_console_write(staging, len) } else { nx_vfs_write(...) }`
- After: unified `copy_from_user(staging, buf, len)` above the if/else; char_slot branch calls `nx_char_device_write(target, staging, len)`.

**Added include**: `#include "framework/char_device_call.h"`.

**Fixed pre-existing bugs** (exposed by recompilation when the new include was added):
- `sys_fork` legacy FILE retain arm: `nx_vfs_retain(vfs_slot, src->object)` → `nx_vfs_retain(vfs_slot, (uint32_t)(uintptr_t)src->object)` — the 9b.1 API change from `void *file` to `uint32_t id` was missed here.
- `sys_exec`: `void *file = NULL; rc = nx_vfs_open(vfs_slot, path, flags, &file)` (4-arg old API) → `uint32_t file = nx_vfs_open(vfs_slot, path, flags)` (3-arg 9b.1 API returning id).  All subsequent `nx_vfs_close(vfs_slot, file)` / `nx_vfs_read(vfs_slot, file, ...)` calls work unchanged since `file` is now the correct `uint32_t` id.  These bugs were masked by stale `.o` files from before the 9b.1 API change; adding the new `#include` forced recompilation of `syscall.c` and revealed them.

### Makefile / test/host/Makefile

- Added `framework/char_device_call.c` to `FW_C` (kernel Makefile) and to `SRCS` (host test Makefile).
- Added `test/kernel/ktest_9b_3.c` to `KTEST_C`.

### test/kernel/ktest_9b_3.c (NEW, 10 kernel tests)

All tests that call `nx_char_device_*` or `nx_vfs_*` wrappers (which use `nx_slot_call_blocking`) are run from **spawned kthreads** (not the ktest runner itself), since `nx_slot_call_blocking` requires `caller_slot_active` which only spawned kthreads have after bootstrap.

| Test | What it verifies |
|---|---|
| `char_device_slot_is_active_after_bootstrap` | char_device slot bound to uart_pl011 |
| `char_device_write_returns_byte_count` | `nx_char_device_write("hello", 5)` → rc=5 |
| `char_device_write_zero_len_returns_zero` | `nx_char_device_write("", 0)` → rc=0 |
| `char_device_read_returns_preinjected_bytes` | inject 5 bytes, read via slot_call → 5 bytes |
| `char_device_read_eof_returns_zero` | inject EOF, read → 0 |
| `char_device_read_partial_less_than_cap` | inject 3 bytes, read cap=8 → 3 |
| `char_device_write_increments_console_write_calls` | `nx_console_write_calls()` increases after write |
| `vfs_write_read_roundtrip_via_slot_call_blocking` | open "/ktest_9b3_rw", write, close, reopen, read, verify |
| `vfs_seek_repositions_via_slot_call_blocking` | open, write "abcde", seek to 2, read 3 → "cde" |
| `char_device_read_exact_byte_content_matches` | inject "test123", read, verify exact bytes |

## Key Findings

**Pre-existing sys_exec/sys_fork API mismatch.** The 9b.1 slice changed `nx_vfs_open` from 4-arg (returning status, out-param `void **file`) to 3-arg (returning `uint32_t id`).  Two call sites in `syscall.c` — `sys_exec`'s file-slurp loop and `sys_fork`'s FILE retain arm — were not updated in 9b.1 and kept using the old API.  GCC's `-Wint-conversion` / "too many arguments" errors only fired when `syscall.c` was recompiled by this session's new `#include`.  Both sites are now corrected.

**Dispatcher kthread blocking on UART read.** When `nx_char_device_read` is called via `nx_slot_call_blocking`, the dispatcher kthread runs `uart_pl011_read` → `nx_console_read` which yield-waits for UART input.  While the dispatcher is blocked on UART input, other IPC messages queue but cannot be processed.  For v1 (single active process reading stdin), this is acceptable — the blocked process can't issue other IPC while waiting for input, and the dispatcher unblocks when the IRQ delivers bytes.

**Serial log binary blob.** The kernel test suite's serial log (`test/kernel-output.log`) contains ~500 MB of binary content after the first 57 visible test results; this is a pre-existing behavior from EL0 programs (busybox, posix) writing binary content to the UART.  The test results for tests 58–151 are embedded in this binary blob and difficult to extract by text search.  The authoritative indicator is QEMU's semihosting exit code: `ktest_exit(failed)` → exit(0) means 0 failed tests.  `kernel-test.bin` contains the 9b.3 test names and the boot header logs "151 test(s)" confirming all 10 new tests are registered and run.

## Status at End of Session

- `make test-host` → **477/477 pass** (unchanged from 9b.2 — no new host tests in this slice)
- `make test-kernel` → **151/151 pass** (141 existing + 10 new; QEMU exit 0)
- `make verify-iface-fresh` → 0 drift
- `make verify-registry` → 0 findings (R2, R4, R9)
- Slice 9b.3 complete.

## Next Steps

- **Slice 9b.4 — Cleanup**: retire `NX_HANDLE_FILE`, `NX_HANDLE_DIR`, `NX_HANDLE_CONSOLE` from `enum nx_handle_type`; unify all resource handle close under one code path; remove legacy FILE/CONSOLE switch arms from `syscall.c`.  ~60 lines removed; 0 new tests.

---

**Files Changed:**
- `framework/char_device_call.c` — NEW: `nx_char_device_write`, `nx_char_device_read`, `nx_char_device_rx_byte` wrappers
- `framework/syscall.c` — sys_read/sys_write RESOURCE arms switched from direct console calls to nx_char_device_*; sys_exec/sys_fork pre-existing API bugs fixed; new `#include "framework/char_device_call.h"`
- `Makefile` — char_device_call.c added to FW_C; ktest_9b_3.c added to KTEST_C
- `test/host/Makefile` — char_device_call.c added to SRCS
- `test/kernel/ktest_9b_3.c` — NEW: 10 kernel tests for slot_call routing
