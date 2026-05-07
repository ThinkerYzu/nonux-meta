# Session 114: Reap-on-wait — process leak fix

**Date:** 2026-05-06
**Phase:** Post-Phase-9b (between Phase 9b and Phase 9)
**Branch:** master

---

## Goals

- Fix the process leak in busybox: every command forks a child that is never freed after `sys_wait` returns.

## What Was Done

### Root cause

`sys_wait` in `framework/syscall.c` was setting `target->reaped = true` after collecting the child's exit status but never releasing any resources.  The `reaped` flag only hid the child from subsequent `waitpid(-1)` scans; the process struct, handle table, process table slot, and 8 MiB MMU address space (L1/L2 page tables + user backing) all leaked.  Every busybox command left a full ghost process behind.

### `framework/syscall.c` — deliver exit status then destroy

In `sys_wait`, replaced `target->reaped = true` with `nx_process_destroy(target)`.  After writing the exit status to the caller's user-space pointer, `nx_process_destroy` frees:
- the handle table and its entries
- the process table slot
- the MMU address space (L1/L2 page tables + 8 MiB user backing)
- the `struct nx_process` itself

The task struct and kernel stack are freed in a follow-up change within the same session (see below).

### `framework/process.h` — removed `bool reaped`

The `reaped` field became dead code once `sys_wait` destroys the process immediately.  Removed from `struct nx_process`.

### `framework/process.c` — removed dead `reaped` guard

Removed the `if (p->reaped) continue` guard from `nx_process_find_exited_child` (no longer needed — destroyed processes are gone from the table entirely).  Updated the comment in `nx_process_exit` that said "Real reap-on-wait is still deferred" to reflect that it is now implemented.

### `core/mmu/mmu.c` — zero the entire 8 MiB user backing on creation

In `mmu_create_address_space`, expanded the TLS-only `memset` to zero the entire 8 MiB user backing:

```c
memset(user_pa, 0, USER_WINDOW_SIZE);
```

**Why:** once `sys_wait` frees the child's backing via `nx_process_destroy`, the kheap/PMM recycles those pages.  When the next process is created, `mmu_create_address_space` receives stale pages that still contain the previous process's stack data.  Before this fix, only the TLS region was zeroed because QEMU provides zero-filled pages on first allocation — after reap-on-wait the "fresh pages" assumption breaks.  The stale stack data caused musl's `crt1` to crash: non-zero `argc`/`argv` on entry → SIGSEGV → exit 139.  Zeroing the whole window restores the zero-fill contract POSIX requires for anonymous mappings.

### 8 kernel test files — removed post-wait secondary scans

After `sys_wait` returns, the child process struct no longer exists.  Eight test files had redundant "belt-and-suspenders" scans that looked up the child by PID in the process table to verify it had exited with the expected code.  These secondary scans now crash or produce spurious failures because the slot is gone.

The primary check — `debug_write_calls >= N`, where the N-th write is gated on the correct exit status — is sufficient.  The secondary scans were removed from:

- `test/kernel/ktest_wait.c` — removed `pid_before` scan + `found_child` scan
- `test/kernel/ktest_exec.c` — removed `exec_parent_pid` + `found` scan
- `test/kernel/ktest_posix.c` — removed `host_pid` + `found_child` scan
- `test/kernel/ktest_posix_signal.c` — removed `host_pid` + `found` scan
- `test/kernel/ktest_posix_pipe_xproc.c` — removed `host_pid` + `found` scan
- `test/kernel/ktest_argv_push.c` — removed `host_pid` + `found_child` scan
- `test/kernel/ktest_posix_segfault.c` — removed `child_found` scan
- `test/kernel/ktest_posix_undef.c` — removed `child_found` scan

Note: `KASSERT` does `return;` on failure, skipping cleanup — leaving tasks in the runqueue and cascading into subsequent test failures.  Removing the scans also eliminates that class of cascade failure.

## Key Findings

- **Stale-page crash:** the musl `crt1` SIGSEGV (exit 139) was not a POSIX or signal bug — it was stale stack data from a previous process bleeding into the new process's user window after reap-on-wait recycled the pages.  Zeroing the full window in `mmu_create_address_space` is the correct fix; it also matches what POSIX mandates for anonymous mappings.
- **Secondary scan fragility:** post-`sys_wait` PID lookups are inherently racy once reap is implemented.  The primary `debug_write_calls` counter approach is safer and sufficient; the secondary scans were defence-in-depth that became a liability.
- **Task struct and kernel stack also freed (same session):** `nx_task_destroy` already existed and freed both.  The missing link was a back-reference from process to task.  Added `main_task` pointer to `struct nx_process`, set it in `sys_fork` (`child->main_task = child_task`), and called `nx_task_destroy(zombie_task)` in `sys_wait` before `nx_process_destroy`.  Processes not created via fork (test fixtures, init) have `main_task == NULL`; `nx_task_destroy(NULL)` is a no-op.

## Decisions Made

- **Destroy immediately in `sys_wait`** — call `nx_process_destroy` right after collecting the exit status rather than setting a `reaped` flag.  Simpler: no hidden-zombie state to reason about.
- **Zero full 8 MiB window** — rather than only zeroing TLS on address-space creation, zero the entire `USER_WINDOW_SIZE` block.  This is O(8 MiB) on process creation but correct and matches the POSIX zero-fill contract; demand-paging (Phase 9) will eliminate the cost when it lands.
- **Remove secondary scans rather than fix them** — the scans tested a transient state that no longer exists post-destroy.  The primary write-count check is more robust and already the canonical correctness signal.

## Status at End of Session

- `make test-host` → **476/476 pass**
- `make test-kernel` → **151/151 pass**
- `make verify-registry` → **0 findings**
- `make verify-iface-fresh` → **0 drift**
- Full zombie reap complete: each `sys_wait` now frees the child's task struct, kernel stack, handle table, address space, and process table slot.

### `reap_task` scheduler IDL op (op_id 8)

Moved task destruction out of `sys_wait` and into the scheduler interface, following the same pattern as `runqueue_size` (session 113).

**`interfaces/idl/scheduler.json`** — added `reap_task` (op_id 8).  Unlike all other task-parameter ops (borrow semantics), this is a **transfer**: the scheduler takes ownership and must call `nx_task_destroy`.  The IDL doc describes the caller guarantees (task dequeued, single-CPU, process already destroyed).

**Generated files** — `gen-iface.py` updated `interfaces/scheduler.h` (new `void (*reap_task)(void *self, struct nx_task *task)` field), `interfaces/scheduler_msg.h` (enum + message structs), `framework/scheduler_call.h`, and `framework/scheduler_dispatch.h`.

**`sched_rr` and `sched_priority`** — each got a one-line `sched_*_reap_task` static function (`nx_task_destroy(t)`) registered in the ops table.  Trivial today; gives each policy an extension point for per-scheduler bookkeeping.

**`core/sched/sched.h` / `sched.c`** — `sched_reap_task(t)` core driver wrapper: delegates to `ops->reap_task` when available, falls back to `nx_task_destroy` directly for stubs/early-boot.

**`framework/syscall.c`** — `sys_wait` now calls `sched_reap_task(zombie_task)` instead of `nx_task_destroy` directly; `sys_wait` no longer needs to know about task internals.

## Next Steps

- Start Phase 9 — per-process memory management rework (L3 4 KiB pages, VMAs, demand paging, COW fork).  See `IMPLEMENTATION-GUIDE.md §Phase 9`.  Demand paging will also eliminate the O(8 MiB) zero-fill cost introduced this session.

---

**Files Changed:**
- `interfaces/idl/scheduler.json` — added `reap_task` op (op_id 8)
- `interfaces/scheduler.h` — generated: new `reap_task` function pointer
- `interfaces/scheduler_msg.h` — generated: enum + message structs
- `framework/scheduler_call.h` — generated: `nx_scheduler_reap_task` declaration
- `framework/scheduler_dispatch.h` — generated: dispatch case for reap_task
- `components/sched_rr/sched_rr.c` — `sched_rr_reap_task` + ops table entry
- `components/sched_priority/sched_priority.c` — `sched_priority_reap_task` + ops table entry
- `core/sched/sched.h` — `sched_reap_task` declaration
- `core/sched/sched.c` — `sched_reap_task` wrapper implementation
- `framework/syscall.c` — `sys_wait`: `sched_reap_task(zombie_task)` + `nx_process_destroy(target)`; `sys_fork`: `child->main_task = child_task`
- `framework/process.h` — removed `bool reaped`; added `struct nx_task *main_task` + forward decl
- `framework/process.c` — removed `if (p->reaped) continue` guard; updated comment in `nx_process_exit`
- `core/mmu/mmu.c` — `mmu_create_address_space`: zero full `USER_WINDOW_SIZE` instead of TLS only
- `test/kernel/ktest_wait.c` — removed post-wait secondary process scan
- `test/kernel/ktest_exec.c` — removed post-wait secondary process scan
- `test/kernel/ktest_posix.c` — removed post-wait secondary process scan
- `test/kernel/ktest_posix_signal.c` — removed post-wait secondary process scan
- `test/kernel/ktest_posix_pipe_xproc.c` — removed post-wait secondary process scan
- `test/kernel/ktest_argv_push.c` — removed post-wait secondary process scan
- `test/kernel/ktest_posix_segfault.c` — removed post-wait secondary process scan
- `test/kernel/ktest_posix_undef.c` — removed post-wait secondary process scan
