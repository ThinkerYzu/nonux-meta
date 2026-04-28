# Session 32: slice 7.1 — `struct nx_process` + per-process handle tables

**Date:** 2026-04-24
**Phase:** 7 slice 7.1 (opens Phase 7)
**Branch:** master

---

## Goals

Open Phase 7 by introducing the process abstraction.  Before this
slice, every EL0 program (channels, files, readdir) shared a single
global `g_kernel_handles` table — meaning "closing a handle" was
semantically a kernel-wide operation, not a userspace-isolated one.
Slice 7.1 fixes that: each task now belongs to a `struct nx_process`
that owns its own handle table.  The syscall dispatcher routes through
`nx_task_current()->process->handles` so handles allocated by one
process are invisible to another.

Address-space isolation (per-process TTBR0) and real scheduler-
integrated `wait()` are the next two slices (7.2 and 7.4).  This
slice's deliverable is the *structure* that those subsequent slices
will populate.

## Scope choices

- **Slice out TTBR0 switching.**  Per-process address space is a
  substantial MMU rework (kernel moves to TTBR1 high half; TTBR0
  flips on every cross-process context switch).  Packing it into 7.1
  would make the slice too big and would conflate "add process
  abstraction" with "make address spaces private".  Deferred to 7.2.
- **No real `wait()`.**  Pairs with `fork`/`exec` — slice 7.4.  For
  7.1, `NX_SYS_EXIT` marks the process EXITED and parks the task in
  `wfe`; the ktest harness dequeues the stranded task externally, as
  it already does for the existing EL0 programs.
- **Static kernel process.**  `g_kernel_process` (pid 0) is a BSS-
  allocated struct, not heap-allocated.  Why: it must exist from the
  very first moment of bootstrap, before any allocator is usable;
  it must be safe to reference from host tests that have no
  scheduler state; and the "kernel owns pid 0" convention is
  standard enough that giving it a fixed address simplifies every
  consumer.
- **Don't migrate the existing EL0 ktests.**  The channel / file /
  readdir / handle ktests keep running in the kernel process's
  table — which, under the new world, means they're still sharing
  with the idle task and every kthread.  Their test semantics
  haven't changed; adding `nx_process_create` + `task->process =
  new_proc` calls to each would be mechanical churn for zero test-
  value delta.  Isolation is proved separately via the new process-
  focused tests.

## What Was Done

### `framework/process.{h,c}` — new

- `enum nx_process_state { ACTIVE, EXITED }`.
- `struct nx_process` — `{pid, name[16], state, exit_code, handles}`.
- `g_kernel_process` — static, pid 0, BSS-allocated, handle table
  zero-initialised (`handle_table_init`'s invariant is that
  all-zeros is an empty table).
- PID allocation: monotonic `g_pid_next` counter starting at 1.  No
  reuse in v1 — pid 0 is reserved for the kernel, so user pids are
  1..(2^32 - 1).
- Process registration table: fixed 16-slot array.  Linear scan for
  lookup / add / remove.  O(n) but n ≤ 16.
- `nx_process_current()` — reads `nx_task_current()->process` on
  kernel; falls back to `&g_kernel_process` if there's no current
  task or the task has no process.  Host build always returns
  `&g_kernel_process` (no task currency concept on host).  Never
  returns NULL — callers can safely deref.
- `nx_process_destroy` walks the handle table and calls
  `nx_handle_close` on every live entry (so stale handle values
  from other parts of the kernel stop resolving).  Note: this does
  NOT run type-aware destructors (channel endpoint close, etc.) —
  that's `sys_handle_close`'s job.  Documented gap; 7.4 addresses it
  when fork/exec/wait need it.
- `nx_process_reset_for_test` — test fixture helper.  Destroys every
  user process (freeing their structs) and clears the kernel
  process's handle table + state.  Called from the top of most
  host tests for isolation.

### `struct nx_task` gains a `process` field

- Forward-declared in `core/sched/task.h` (full definition in
  `framework/process.h`).  Placed after the existing
  `sched_node` — `cpu_ctx` stays at offset 0 (the `_Static_assert`
  still passes).
- `nx_task_create` in `core/sched/task.c` now includes
  `<framework/process.h>` and populates
  `t->process = caller ? caller->process : &g_kernel_process`
  where `caller = nx_task_current()`.  This "inherit from caller"
  semantics matches the Unix tradition and works for every
  bootstrap path: the very first `nx_task_create` call (from
  bootstrap, before `sched_start`) has no current task yet, so the
  fallback kicks in and the new task gets the kernel process.
- `sched_start` in `core/sched/sched.c` explicitly sets
  `g_idle_task.process = &g_kernel_process` — the idle task never
  went through `nx_task_create` (it's a BSS-allocated static), so
  we set the field directly during init.

### `framework/syscall.c` — retires `g_kernel_handles`

- `static struct nx_handle_table g_kernel_handles;` deleted.
- `nx_syscall_current_table()` now reads
  `&nx_process_current()->handles`.
- `nx_syscall_reset_for_test()` now resets the CURRENT process's
  table (which, for host tests with no scheduled task, is the
  kernel process — same effective behaviour as the retired global).
- New `sys_exit(code)` handler calls `nx_process_exit(code)` which
  is `noreturn`.  Dispatch table gets `[NX_SYS_EXIT] = sys_exit`.

### `framework/syscall.h` — syscall number

- `NX_SYS_EXIT = 11` appended before `NX_SYSCALL_COUNT`.

### Tests

- **`test/host/process_test.c`** (8 tests):
  1. `kernel_process_is_pid_zero_and_always_present` — even after a
     reset, `g_kernel_process` stays at pid 0.
  2. `process_current_defaults_to_kernel_process_with_no_task`.
  3. `process_create_allocates_fresh_pid_and_empty_handle_table` —
     pids are 1, 2, 3... and each process's table starts empty.
  4. `process_destroy_closes_outstanding_handles` — destroy
     releases handles.
  5. `process_destroy_on_kernel_process_clears_table_but_keeps_
     process` — the static kernel process can't be freed, so
     destroy just clears its table.
  6. `two_processes_have_independent_handle_tables` — the core
     isolation property.  A handle allocated in one table doesn't
     resolve (or at least doesn't yield the same object) in the
     other.
  7. `syscall_current_table_points_at_current_processs_table`.
  8. `reset_for_test_wipes_user_processes_and_kernel_handles` +
     pid counter restarts at 1.
- **`test/kernel/ktest_process.c`** (4 tests):
  1. `current_returns_kernel_process_on_idle_task` — bootstrap
     invariant check.
  2. `current_same_pointer_as_syscall_current_table_owner` —
     slice-7.1 headline: the syscall dispatcher's handle-table
     lookup goes through the current process.
  3. `create_allocates_fresh_pid_on_kernel` — parity with the host
     test for the same path under the real kernel + kheap.
  4. `exit_marks_current_process_exited_with_code` — spawns a
     kthread that reassigns its `process` to a test target, calls
     `nx_process_exit(42)`, parks.  The test yields until the
     target's state flips to EXITED, asserts exit_code == 42, then
     externally dequeues the parked task.

### Wiring

- Top Makefile: `FW_C` += `framework/process.c`; `KTEST_C` +=
  `test/kernel/ktest_process.c`.
- `test/host/Makefile`: SRCS += `../../framework/process.c` +
  `process_test.c`.

## Key Findings

- **`g_kernel_handles` disappearing is invisible to the existing
  test suite.**  Every pre-7.1 test that used
  `nx_syscall_current_table()` or `nx_syscall_reset_for_test()`
  keeps passing unchanged — the indirection through the kernel
  process is semantically equivalent.  That's the cleanest
  possible outcome of the refactor; no test-churn needed.
- **Per-test reset cascades.**  The first draft of
  `reset_for_test_wipes_user_processes_and_kernel_handles` failed
  because a handle from an earlier test leaked into the kernel
  process's table (tests share state across the `TEST_REGISTRY`
  list in order).  Fix: call `nx_process_reset_for_test()` at the
  top of each process test.  The same discipline applies to any
  test that inspects the kernel process's handle count — future
  tests should reset up front.

## Decisions Made

- **Single `g_kernel_process` for the lifetime of the build.**
  Alternative: dynamically allocate it at bootstrap.  Chose static
  because it needs to exist before any allocator is live, and
  giving it a fixed address simplifies reasoning about it in host
  tests that don't run bootstrap.  Why: the kernel process is a
  singleton by nature — there's only ever one, and it never goes
  away.  How to apply: any future always-present framework object
  (dispatcher, idle, kernel process) follows this same pattern.
- **`nx_task_current()->process` inherited, not explicitly
  passed.**  Alternative: thread the `process` parameter through
  `sched_spawn_kthread` / `nx_task_create`.  Chose implicit
  inheritance because it matches Unix semantics ("kids inherit
  parent's context") and means existing callers don't change.
  Explicit reassignment after create is still available (ktests
  use it to give the exit-test kthread a custom process).
- **PID 0 is the kernel.**  Unix convention — kernel code is often
  described as "pid 0" in userspace tooling.  Cost is that user
  pids start at 1, which is fine.
- **Exit doesn't integrate with the scheduler yet.**  Alternative:
  mark the task ZOMBIE, dequeue it via `g_sched_ops->dequeue`,
  and call `cpu_switch_to` to the next runnable.  Chose the
  simpler wfe park for v1 because real exit semantics (wait,
  reaping, zombie cleanup) all need to land together — partial
  coverage creates half-correct state that's easy to misremember.
  How to apply: 7.4 does the full story.

## Status at End of Session

- `make test` → **376/376 pass (51 python + 266 host + 59 kernel),
  0 leaks, 0 errors, exit 0**.  +12 from slice 6.4.
- `make run` (plain `kernel.bin`) boots clean — composition log
  unchanged from before 7.1 (the refactor doesn't affect boot
  output, just the syscall dispatcher's table-lookup path).
- Phase 7 opened: slice 7.1 shipped.

## Next Steps

Slice 7.2 — per-process address space.

- Kernel mappings move into TTBR1 (high VAs starting around
  `0xFFFF_0000_0000_0000`).  TTBR0 becomes exclusively the per-
  process user address space.
- `framework/process` grows an `address_space` field (pointer to a
  TTBR0 root page-table page) allocated via `memory.page_alloc`
  at `nx_process_create` time.
- `cpu_switch_to` flips TTBR0 via
  `msr ttbr0_el1, Xn; isb; tlbi vmalle1; dsb ish; isb` when the
  incoming task's process differs from the outgoing task's.
- Each existing EL0 ktest creates its own process + user window
  before `drop_to_el0`, demonstrating that two EL0 programs can
  hold different bytes at the same VA.
- Kernel can still access user memory through the high-half view.

Closes the long-standing "per-task TTBR0" Phase 5 follow-up.

---

**Files Changed:**
- `sources/nonux/framework/process.h` — new (API: struct + lifecycle functions)
- `sources/nonux/framework/process.c` — new (implementation + `g_kernel_process`)
- `sources/nonux/framework/syscall.h` — add `NX_SYS_EXIT = 11`
- `sources/nonux/framework/syscall.c` — retire `g_kernel_handles`; route through `nx_process_current`; add `sys_exit`
- `sources/nonux/core/sched/task.h` — forward-decl `struct nx_process`; add `process` field
- `sources/nonux/core/sched/task.c` — include process.h; `nx_task_create` inherits `caller->process`
- `sources/nonux/core/sched/sched.c` — include process.h; `idle_task_init` sets `g_idle_task.process`
- `sources/nonux/Makefile` — `FW_C += framework/process.c`; `KTEST_C += ktest_process.c`
- `sources/nonux/test/host/Makefile` — SRCS += process.c + process_test.c
- `sources/nonux/test/host/process_test.c` — new (8 tests)
- `sources/nonux/test/kernel/ktest_process.c` — new (4 tests)
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Phase 7 gets sliced plan (7.1–7.7); §Slice 7.1 marked complete
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 27 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
