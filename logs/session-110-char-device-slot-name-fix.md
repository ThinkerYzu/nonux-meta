# Session 110: char_device slot name fix + posix_musl passing

**Date:** 2026-05-05
**Phase:** Bug fix (no new slice)

---

## Goals

- Fix `posix_musl_prog_runs_main_through_init_libc_and_exits_57` kernel test.
- Identify and fix the underlying root cause shared with `musl_exec` and `posix_musl_printf`.

## Root Causes Found

### 1. Wrong slot name: "char_device" instead of "char_device.serial"

The registered slot name is `"char_device.serial"` (namespace-qualified by kernel.json), not `"char_device"`.  Every call to `nx_slot_lookup("char_device")` returned NULL.

**Impact:**

- `process.c` — `nx_process_create` looked up `char_device` to pre-install stdout/stderr/stdin RESOURCE handles.  Got NULL → handles installed with `target=NULL`.
- `syscall.c sys_write` — `char_slot = NULL`; `res->target == char_slot` is `NULL == NULL` → TRUE for all pre-installed console handles.  Routes to `nx_char_device_write(NULL, ...)` → `nx_slot_call_blocking(NULL, ...)` → NX_EINVAL.  Write fails silently.
- `syscall.c sys_read` — same pattern; console reads fail with NX_EINVAL.
- `syscall.c sys_ioctl` — console detection fails → ENOTTY on `tcgetattr`.
- `syscall.c sys_fork` — handle inheritance skips char_device close logic.
- `ktest_9b_3.c` — all char_device IPC tests failed at `KASSERT_NOT_NULL(cs)`.

**Fix:** Replace `"char_device"` with `"char_device.serial"` in all production code (`process.c`, `syscall.c`) and in `ktest_9b_3.c`.  Update stale comment in `file_syscall_test.c`.

### 2. `uart_pl011_write` bypassed `nx_console_write`

`uart_pl011_write` wrote bytes via `uart_putc()` directly, never calling `nx_console_write()`.  The test `posix_musl_prog` checks `nx_console_write_calls() >= 1` (incremented only by `nx_console_write()`).  Because the console handles had NULL target (root cause 1), writes took the `target==NULL==char_slot` path to `nx_char_device_write(NULL,...)` which failed.  Even after fixing the slot name, `uart_pl011_write` still needed to route through `nx_console_write` so the counter is incremented.

**Fix:** Change `uart_pl011_write` to call `nx_console_write(buf, len)` instead of iterating `uart_putc`.  `nx_console_write` loops `uart_putc` internally and increments the counter.

## What Was Done

- `framework/process.c:139` — `"char_device"` → `"char_device.serial"`
- `framework/syscall.c` — all 9 occurrences of `nx_slot_lookup("char_device")` → `"char_device.serial"`
- `components/uart_pl011/uart_pl011.c` — `uart_pl011_write`: loop over `uart_putc` → `nx_console_write(buf, len)`
- `test/kernel/ktest_9b_3.c` — all `"char_device"` → `"char_device.serial"`
- `test/host/file_syscall_test.c` — stale comment updated

## Test Results

- `make test-host` → **476/476 pass** (unchanged)
- `make test-kernel` → **142/151 pass** (was 138/151)
  - `posix_musl_prog_runs_main_through_init_libc_and_exits_57` → **PASS** (`[musl-ok]` on UART)
  - `musl_exec_parent_forks_and_execs_musl_child_returns_57` → **PASS**
  - `posix_musl_printf_emits_every_conversion_and_exits_67` → **PASS**
  - `char_device_slot_is_active_after_bootstrap` → **PASS** (slot lookup now works)
- 9 remaining FAILs: all from `ktest_9b_3.c` failing at `KASSERT(atomic_load(&g_9b3_done))`

## Remaining ktest_9b_3 Failures — Root Cause Analysis

8 ktest_9b_3 tests still fail. `char_device_slot_is_active_after_bootstrap` (no kthread, just slot lookup) now PASSES. All failing tests spawn kthreads that call `nx_char_device_read` or `nx_vfs_*`.

**Root cause:** Stale stdin-read IPCs from previous busybox tests block the dispatcher.

1. Late busybox tests (e.g. `ktest_posix_busybox_sh_stdin`) spawn EL0 processes that call `read(0, ...)`.  The EL0 `read` goes through `sys_read` → `nx_char_device_read` → IPC to char_device.serial.
2. The ktest dequeues the EL0 task before the dispatcher processes the read IPC.
3. The dispatcher eventually processes the stale IPC: calls `uart_pl011_read` → `nx_console_read`.  No bytes in ring → `nx_console_read` calls `nx_waitq_wait_unless`, blocking the **dispatcher kthread**.
4. When `char_device_read_exact_byte_content_matches` (first 9b3 test) injects bytes via `nx_console_test_inject_bytes`, those bytes wake the stale blocking `nx_console_read` (via `nx_pollset_wake_all`), consuming the bytes.
5. The 9b3-r10 kthread's own IPC is now processed with an empty ring: `nx_console_read` blocks the dispatcher again.
6. All subsequent IPC calls (VFS writes, char_device writes from later 9b3 tests) queue up behind the blocked dispatcher and time out.

**Fix direction:** Make `uart_pl011_read` non-blocking in dispatcher context: return available bytes or 0 immediately (never call the blocking `nx_waitq_wait_unless` path from within a dispatcher handler).  Or: drain leftover busybox stdin IPCs as part of ktest teardown before ktest_9b_3 runs.

## Files Changed

- `framework/process.c`
- `framework/syscall.c`
- `components/uart_pl011/uart_pl011.c`
- `test/kernel/ktest_9b_3.c`
- `test/host/file_syscall_test.c` (comment only)
