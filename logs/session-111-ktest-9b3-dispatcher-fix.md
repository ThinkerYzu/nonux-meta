# Session 111: Fix 9 Pre-Existing ktest_9b_3 Failures (Dispatcher + Console + RAMFS)

**Date:** 2026-05-05
**Phase:** Phase 9b closed; pre-Phase-9 cleanup
**Branch:** master

---

## Goals

- Fix all 9 pre-existing failing kernel tests in `ktest_9b_3` (tests 2–10 out of 10).
- Achieve 151/151 kernel tests passing.

---

## What Was Done

### Root Cause Investigation

All 9 failures in `ktest_9b_3` had the same symptom: tests blocked indefinitely (> 4096 yields) waiting for `atomic_load(&g_9b3_done)` to become non-zero. The dispatcher was stalled and no IPC was completing.

Three independent root causes were found and fixed in sequence.

---

### Issue 1: Dispatcher kthread killed by the live_swap test

**Root cause:** `ktest_live_swap_tasks_survive_and_behavior_changes` (in `test/kernel/ktest_live_swap.c`) correctly dequeues the three test kthreads (ta/tb/tc) from `sched_priority` before swapping the scheduler to `sched_rr`, then re-enqueues them into `sched_rr` after. However, it never dequeued or re-enqueued the **dispatcher kthread**. The dispatcher was left stranded in `sched_priority`'s runqueue when that scheduler was swapped out and destroyed. The dispatcher was effectively dead for all subsequent tests in the `ktest_9b_3` suite.

**Fix:**
- Added `nx_dispatcher_task_for_test()` to `framework/dispatcher.c`: stores `g_disp_task` when the dispatcher kthread starts, exposes it via a `#ifndef __STDC_HOSTED__` guarded function.
- Declared `nx_dispatcher_task_for_test()` in `framework/dispatcher.h` (kernel build only).
- Modified `test/kernel/ktest_live_swap.c` to call `nx_dispatcher_task_for_test()`, dequeue the dispatcher from `sched_priority` before the swap, and re-enqueue it into `sched_rr` after (alongside ta/tb/tc).

---

### Issue 2: `uart_pl011_read` blocks the dispatcher (pre-existing, became visible after Issue 1 fix)

**Root cause:** `uart_pl011_read` called `nx_console_read` which calls `nx_waitq_wait_unless` in dispatcher context when the RX ring is empty. This is a violation of the bounded-handler rule: a dispatcher handler must never block. With Issue 1 present the dispatcher was already dead so this never triggered; with Issue 1 fixed, the dispatcher survived the live_swap test but was then immediately killed by the first `read` IPC that arrived when no bytes were available in the RX ring.

**Fix:**
- Added `nx_console_read_nonblocking()` to `framework/console.c` (kernel build only): returns bytes from the ring if available, or returns `NX_EAGAIN` immediately if the ring is empty — no `nx_waitq_wait_unless` call.
- Added `nx_console_read_ready()` as a companion predicate.
- Declared both in `framework/console.h`.
- Changed `uart_pl011_read` in `components/uart_pl011/uart_pl011.c` to call `nx_console_read_nonblocking` in kernel builds. Updated the docstring comment from "blocks until bytes available" to reflect the non-blocking behavior.
- Added `nx_console_reset_for_test()` calls at the start of tests 2 and 3 in `test/kernel/ktest_9b_3.c` (the write-test kthreads that previously started without draining stale RX state from earlier busybox tests).

**Design note captured:** Dispatcher handlers must never block — any operation that may wait (e.g., waiting for console RX bytes) must either use non-blocking access and return `NX_EAGAIN`, or be moved to a task context (e.g., `sys_read`'s EL0 task kthread). Since EL0 `read(0)` calls in busybox ktests complete immediately (bytes are pre-loaded into the ring before the call), this change is transparent to all existing tests.

---

### Issue 3: RAMFS inode table exhaustion (pre-existing, became visible after Issues 1+2 fixed)

**Root cause:** `RAMFS_MAX_FILES = 24`. After running approximately 150 kernel tests (each creating files in the ramfs), the inode table is exhausted when `vfs_write_read_roundtrip_via_slot_call_blocking` (test 8 in ktest_9b_3, which runs after test 9 consumes slot 24) tries to create `/ktest_9b3_rw`. The `nx_vfs_open` call returns `NX_ENOMEM` and the test blocks waiting for a completion IPC that never arrives.

**Fix:**
- Bumped `RAMFS_MAX_FILES` from 24 → 32 in `components/ramfs/ramfs.c`.
- Updated expected-exhaustion `ATTEMPTS` constants in two host test files to match the new table size:
  - `test/host/component_ramfs_test.c`: file exhaustion 32→48 attempts, open exhaustion 128→256 attempts.
  - `test/host/slice_9b_1_test.c`: 128→256 attempts.

**Why not 48?** Bumping to 48 was tried first (8 files × 4 MiB = 32 MiB per additional file × 24 extra = 96 MiB additional RAM reservation). This caused 3 unrelated kernel tests to fail with `NX_ENOMEM` because the extra RAM reserved for RAMFS inline storage starved the MMU address space pool. 32 files (adding 8 × 4 MiB = 32 MiB) fits without crowding other allocations.

---

## Key Findings

- **Dispatcher kthread must be treated like any other schedulable entity during a scheduler swap.** If test code dequeues user tasks from the outgoing scheduler, it must also dequeue all framework kthreads (dispatcher, idle task) that were running under that scheduler.
- **Dispatcher handlers must never block.** Calling `nx_waitq_wait_unless` (or any other indefinitely-waiting primitive) from inside a component handler kills the dispatcher. All blocking must be pushed to EL0 task context (`sys_read` path) or handled via non-blocking + `NX_EAGAIN` + retry at the caller.
- **RAMFS_MAX_FILES is a hard limit on total files ever created, not just concurrently open.** Inodes are never reclaimed (no `unlink` in the current ramfs). Tests that run late in a long test suite can exhaust the table even if individual tests clean up their file descriptors.
- **Issue cascade:** The three root causes were independent but masked each other. Issue 1 killed the dispatcher before Issue 2 could trigger. Issue 2 killed the dispatcher (after Issue 1 was fixed) before Issue 3 could be observed. Each fix revealed the next layer.
- **Drain loop vs. reset for console:** A drain loop in `nx_console_reset_for_test()` was attempted but caused excessive yields that hit idle WFI (wait-for-interrupt), introducing its own hang. The simpler `nx_console_reset_for_test()` ring-pointer reset (without a drain loop) was retained as-is; the kthreads in the relevant tests do not consume leftover bytes.

---

## Decisions Made

- **`nx_dispatcher_task_for_test()` exposed via header guard** — Only visible in kernel builds (`#ifndef __STDC_HOSTED__`). Host tests never need to touch the dispatcher kthread pointer. Keeps the function out of the host-side mock harness entirely.
- **Non-blocking console read in kernel builds, blocking only in EL0 task context** — The EL0 `sys_read` path already runs on a task kthread (not the dispatcher), so it is safe to block there. The dispatcher-side handler is not the right place to block; `NX_EAGAIN` is the correct signal.
- **RAMFS_MAX_FILES = 32, not higher** — 48 was tried and caused MMU starvation (3 NX_ENOMEM failures in unrelated tests). 32 gives enough headroom for the full 151-test suite without crowding address space reservations.

---

## Status at End of Session

- `make test-tools` → **102/102 pass**
- `make test-host` → **476/476 pass**
- `make test-kernel` → **151/151 pass** (was 142/151; all 9 previously failing ktest_9b_3 tests now pass)
- `make verify-registry` → 0 findings (R2, R4, R9)
- `make verify-iface-fresh` → 0 drift

All tests pass. No known blockers. Ready to proceed to Phase 9 (per-process MM rework).

---

## Next Steps

- **Phase 9 — per-process MM rework.** L3 4 KiB pages, per-process VMAs, demand paging, COW fork. See [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework).

---

**Files Changed:**
- `framework/dispatcher.c` — added `g_disp_task` field and `nx_dispatcher_task_for_test()` (stores and exposes the dispatcher kthread pointer)
- `framework/dispatcher.h` — declared `nx_dispatcher_task_for_test()` (kernel build only, `#ifndef __STDC_HOSTED__`)
- `test/kernel/ktest_live_swap.c` — dequeue dispatcher before scheduler swap, re-enqueue into sched_rr after; added `#include "framework/dispatcher.h"`
- `framework/console.c` — added `nx_console_read_nonblocking()` and `nx_console_read_ready()` (kernel build only); kept `nx_console_reset_for_test()` ring-reset as-is
- `framework/console.h` — declared `nx_console_read_nonblocking()` and `nx_console_read_ready()`
- `components/uart_pl011/uart_pl011.c` — use `nx_console_read_nonblocking` in kernel build; updated handler comment to reflect non-blocking behavior
- `test/kernel/ktest_9b_3.c` — added `nx_console_reset_for_test()` at start of tests 2 and 3
- `components/ramfs/ramfs.c` — `RAMFS_MAX_FILES` 24 → 32
- `test/host/component_ramfs_test.c` — ATTEMPTS: file exhaustion 32→48, open exhaustion 128→256
- `test/host/slice_9b_1_test.c` — ATTEMPTS 128→256
