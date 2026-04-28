# Session 76 — slice 7.8a: wait-queue primitive + `NX_TASK_BLOCKED`

**Date:** 2026-04-29
**Slice:** 7.8a — wait-queue primitive (foundation for slice 7.8b's `NX_SYS_PPOLL` and 7.8c's yield-loop migration)
**Outcome:** **Closed.**  New `core/sched/waitq.{h,c}` lands a real kernel blocking primitive on top of the existing `nx_deadline` monotonic-timer primitive.  No callers yet — pure framework addition, scheduled exactly per the IMPLEMENTATION-GUIDE plan.  `make test` 447 → **452/452** (+5 new universal cases); `make test-interactive` unchanged at 7/7.

---

## Goal

Slice 7.7 is closed.  Slice 7.8 promotes the v1 yield-loop pattern (slice 7.6d.N.6b `sys_read` CHANNEL arm; slice 7.4b `sys_wait`; slice 7.6d.N.final.a `nx_console_read`) into a real kernel blocking primitive, then lands `NX_SYS_PPOLL` on top of it (7.8b) and migrates the three call sites + reverts the slice-7.6d.N.final.e busybox `fflush(stdout)` patch (7.8c).

7.8a is the foundation slice: the wait-queue primitive itself, the `NX_TASK_BLOCKED` task state in the scheduler, and a deadline-expiry path hooked into the per-tick path.  Sequenced *before* 7.8b/c so the abstraction is fully validated by conformance ktests with no production caller pressure.

The slice plan (HANDOFF.md Next Actions §0 + IMPLEMENTATION-GUIDE.md §Slice 7.8a) called for:

1. New `core/sched/waitq.{h,c}` with `nx_waitq_init`, `nx_waitq_wait_with_deadline`, `nx_waitq_wake_one`, `nx_waitq_wake_all`.
2. Deadline plumbing through `core/cpu/monotonic.h` (the same `nx_deadline_start` / `_exceeded` primitive slice 3.9b.2 added for `pause_hook`).
3. New `NX_TASK_BLOCKED` state in `enum nx_task_state` + `sched_rr_pick_next` skips BLOCKED.  Wake transitions BLOCKED → READY and re-inserts on the runqueue.
4. 4-5 universal conformance cases (`waitq_wake_one_releases_one`, `wake_all_releases_all`, `wait_returns_on_deadline`, `wake_after_wait_no_lost_wakeup`, `multi_waiter_fifo_order`).
5. No callers — `make test` 447 + 5 new conformance cases.

All landed.  Total cost ~190 lines kernel + ~210 lines ktest, slightly over the plan's ~150 estimate because the IRQ-safe wake path needed a short save/restore helper (waitq's wake is callable from ISR context, e.g. console RX in 7.8b) and the ktest harness needed a `WAIT_FOR(cond, budget)` helper to coordinate kthread-driven test cases without flakiness.

---

## What landed

### `core/sched/task.h` + `task.c` — task wait state

`enum nx_task_state` already had `NX_TASK_BLOCKED = 2` (added in an earlier slice as a placeholder).  This slice activates it.  `struct nx_task` grows:

```c
struct nx_waitq    *wait_q;            /* current waitq, NULL if not waiting */
struct nx_deadline  wait_deadline;     /* monotonic deadline (24 bytes) */
int                 wait_has_deadline; /* 0 = block forever, 1 = on g_deadline_list */
int                 wait_woken;        /* 1 if wake delivered, 0 if deadline expired */
struct nx_list_node deadline_node;     /* link into the global deadline list */
```

`sched_node` is reused for the waitq linkage — a task is on either the scheduler runqueue or `wq->waiters`, never both.  This keeps `sizeof(struct nx_task)` growth bounded to the four new scalars + 16 bytes for `deadline_node` (~72 bytes total).

`nx_task_create` and `nx_task_create_forked` initialize all five fields to inert defaults; `idle_task_init` in `core/sched/sched.c` does the same for the statically-allocated idle task.  `task.h` now `#include`s `core/cpu/monotonic.h` directly so the embedded `struct nx_deadline` is sized correctly across host + kernel builds.

### `core/sched/waitq.{h,c}` — the primitive (~190 lines)

Public API in `waitq.h`:

```c
struct nx_waitq { struct nx_list_head waiters; };

void nx_waitq_init(struct nx_waitq *wq);
int  nx_waitq_wait_with_deadline(struct nx_waitq *wq, uint64_t budget_ns);
void nx_waitq_wake_one(struct nx_waitq *wq);
void nx_waitq_wake_all(struct nx_waitq *wq);
void nx_waitq_tick_deadlines(void);
```

Return contract for `nx_waitq_wait_with_deadline`:

| Return       | Meaning                                                                |
|--------------|------------------------------------------------------------------------|
| `NX_OK`      | Woken via `wake_one` or `wake_all`.                                    |
| `NX_EDEADLINE` | Deadline elapsed before any wake (only with `budget_ns > 0`).        |
| `NX_EINVAL`  | `wq == NULL` or no current task.                                       |

`budget_ns == 0` blocks indefinitely (returns only on wake).  Reused `NX_EDEADLINE = -9` rather than introducing `NX_ETIMEDOUT`; the existing code's comment ("`pause_hook` (or similar) exceeded its wall-clock budget") covers waitq-style timeouts cleanly.

Implementation in `waitq.c`:

1. **`wait_with_deadline`** disables IRQs (`daifset, #2` save/restore) + calls `nx_preempt_disable`, then:
   - Pulls `current` off the scheduler runqueue via the stashed `g_sched_ops->dequeue`.
   - Appends to `wq->waiters` via `sched_node` (FIFO — `nx_list_add_tail`).
   - If `budget_ns > 0`: starts a deadline (`nx_deadline_start`) and links onto a singleton `g_deadline_list` via `deadline_node`.  Tasks waiting indefinitely are NOT on the deadline list, so the per-tick walk is O(N_deadline_waiters) not O(N_blocked).
   - Sets `state = NX_TASK_BLOCKED`, `wait_woken = 0`.
   - Re-enables preempt + IRQs and calls `nx_task_yield()`.

   When `current` resumes (after wake or deadline expiry), it returns `NX_OK` if `wait_woken == 1`, else `NX_EDEADLINE`.

2. **`wake_one` / `wake_all`** (callable from ISR context — save/restore DAIF.I) pull each waiter off `wq->waiters`, unlink from `g_deadline_list` if `wait_has_deadline`, set `wait_woken = 1`, set `state = NX_TASK_READY`, and re-enqueue on the scheduler runqueue.

3. **`tick_deadlines`** walks `g_deadline_list` once per call, expiring any waiter whose `nx_deadline_exceeded` returns true: unlinks from waitq + deadline list, sets state READY, leaves `wait_woken = 0` so `wait_with_deadline` returns `NX_EDEADLINE`, and re-enqueues on the runqueue.

The IRQ save/restore is kernel-only (`#if !__STDC_HOSTED__` guarded around the `mrs/msr daif` asm); host build stubs it to a no-op.  Host's `nx_deadline_*` already uses `clock_gettime(CLOCK_MONOTONIC)` so the deadline math is portable.

### `components/sched_rr/sched_rr.c` — pick_next skips BLOCKED

Was:

```c
static struct nx_task *sched_rr_pick_next(void *self) {
    if (nx_list_empty(&s->runqueue)) return NULL;
    return nx_list_entry(s->runqueue.n.next, struct nx_task, sched_node);
}
```

Now walks the list, skipping `NX_TASK_BLOCKED` tasks.  Defensive: in normal flow `wait_with_deadline` dequeues a blocking task before flipping its state, and wake/deadline-expire re-enqueues it as READY — so a BLOCKED task on the runqueue should not occur.  But the guard is cheap and prevents a future caller's missed-dequeue bug from livelocking the system (a `pick_next` that returned a BLOCKED task would never wake).

### `core/sched/sched.c` — sched_tick wires deadline expiry

```c
void sched_tick(void) {
    if (!g_sched_ops) return;
    g_sched_ops->tick(g_sched_self);
    nx_waitq_tick_deadlines();   // slice 7.8a
}
```

Cheap when the deadline list is empty (a single `nx_list_empty` check).  Idle init grew the same five wait-state field initializers as the dynamically-allocated tasks.

### `Makefile` + `test/host/Makefile`

Both gain `core/sched/waitq.c` in their build lists.  Root Makefile's `KTEST_C` gains `test/kernel/ktest_waitq.c`.  Host `vpath` already covered `core/sched/`.

### `test/kernel/ktest_waitq.c` — 5 universal cases (~210 lines)

Five ktests covering the primitive's contract:

1. **`waitq_wake_one_releases_one`** — spawns 2 waiters, both park, `wake_one` releases exactly one (asserts `c1.done + c2.done == 1` after the first wake).  Drains the second waiter so reap is clean.
2. **`waitq_wake_all_releases_all`** — spawns 3 waiters, all park, `wake_all` releases all three.
3. **`waitq_wait_returns_on_deadline`** — single waiter with 50 ms budget, no wake call.  Asserts `rc == NX_EDEADLINE` after up to 1024 yields (the deadline check fires from `sched_tick`, which runs on the 100 ms timer; 1024 yields on idle covers many ticks).
4. **`waitq_wake_after_wait_no_lost_wakeup`** — single waiter parks, then main task wakes.  Tests that the wake is observed (no lost-wakeup race between pre-yield bookkeeping and the wake).
5. **`waitq_multi_waiter_fifo_order`** — three waiters spawn in order A → B → C, yielding after each spawn so each parks before the next arrives.  Three successive `wake_one` calls release them in spawn order; asserts `ca.order == 1`, `cb.order == 2`, `cc.order == 3` via a shared `g_completion_seq` counter.

Test harness pattern: each waiter is a kthread that calls `nx_waitq_wait_with_deadline(wq, budget_ns)`, then on return records `rc`, increments `g_completion_seq`, sets `done = 1`, and parks in `for (;;) nx_task_yield()` until reaped.  A `WAIT_FOR(cond, budget)` macro yields in a bounded loop until `cond` becomes true — keeps the test terminating even if scheduling is broken.  Reap path mirrors slice 4.4's pattern: `ops->dequeue` + `nx_task_destroy`.

---

## Discoveries

1. **`NX_ETIMEDOUT` doesn't exist; `NX_EDEADLINE` does.**  First-iteration `waitq.c` returned `NX_ETIMEDOUT` to match POSIX vocabulary.  Host build failed to compile (`error: 'NX_ETIMEDOUT' undeclared`).  `framework/registry.h` already has `NX_EDEADLINE = -9` with the comment "`pause_hook` (or similar) exceeded its wall-clock budget" — generic enough to cover waitq.  Reused it across waitq.h doc + waitq.c return path + ktest assertion (`NX_EDEADLINE` rather than introducing a redundant new code).  Rejected adding `NX_ETIMEDOUT = -12`: would force every translation table to differentiate NX_E* timeout from NX_E* deadline for a distinction the kernel doesn't actually make.

2. **Idle task needs explicit wait-state init.**  `g_idle_task` in `sched.c` is statically allocated and zero-initialized by C semantics, BUT `nx_list_remove`'s sanity assumes nodes are self-pointing (so `nx_list_empty` works).  Without explicit init of `deadline_node.next/prev`, a future call to `nx_list_remove(&g_idle_task.deadline_node)` would corrupt zero-pointed prev/next links.  Added `g_idle_task.deadline_node.next = &g_idle_task.deadline_node; .prev = &g_idle_task.deadline_node;` to `idle_task_init` for symmetry with the rest of the idle init.

3. **`pick_next` walking O(n) of runqueue rather than peeking head is the right call now.**  Plan said "skip BLOCKED tasks" — could be implemented as either (a) walk past BLOCKED in `pick_next`, or (b) leave BLOCKED tasks on the runqueue but skip them.  Went with (a) (linear walk), but combined with (b) (the wait function dequeues BLOCKED tasks before yielding).  So in normal flow `pick_next` returns the head in O(1); the linear walk only kicks in if some future caller forgets to dequeue.  Defence-in-depth at a fractional cost.

4. **`g_deadline_list` static initializer.**  Used C-struct designated initializer to make `g_deadline_list` self-pointing without a runtime init call:

   ```c
   static struct nx_list_head g_deadline_list = {
       { &g_deadline_list.n, &g_deadline_list.n }
   };
   ```

   Avoids the "first call to nx_waitq_init forgets to init the global list" footgun that an explicit runtime init would invite.

---

## Test results

```
make test            451/452 pass: ... wait, recount
                     51 python + 283 host + 118 kernel = 452/452 pass
                     0 leaks, 0 errors, exit 0
make test-interactive 7/7 pass (echo_cat + echo_hello + echo_pipe + ls_root +
                                  mkdir_tmp + ps_smoke + visible_prompt)
```

Up from session 75's baseline of `make test` 447/447 (51 python + 283 host + 113 kernel) and `make test-interactive` 7/7.

The 5 new kernel tests:

```
waitq_wake_one_releases_one       PASS
waitq_wake_all_releases_all       PASS
waitq_wait_returns_on_deadline    PASS
waitq_wake_after_wait_no_lost_wakeup PASS
waitq_multi_waiter_fifo_order     PASS
```

`make test-interactive` confirms no regression — the existing yield-loop call sites are still in place (their migration is slice 7.8c), so production behaviour is unchanged.

---

## Next steps

**7.8b — `NX_SYS_PPOLL` built on waitqs** (~150 lines, 1 session).  Per-handle-type `add_to_pollset(handle, pollset)` op for CHANNEL / FILE / DIR / CONSOLE; `sys_ppoll` walks the user `pollfd[]` (cap 32), allocates a kstack pollset, registers, blocks on the pollset waitq with deadline, copies `revents` back.  Producer-side wakes: `nx_channel_send` → wake_all read waitq; `nx_channel_close` → wake_all both; console RX ISR → wake_all RX waitq.  musl `__NR_ppoll = 73 → NX_SYS_PPOLL`.  Closes the busybox-prompt root cause for real (busybox's `ask_terminal()` `safe_poll(stdin, 0) == 0` test now succeeds → `\e[6n` + `fflush_all()` branch fires → prompt visible without our patch).

**7.8c — Migrate yield-loops + revert slice 7.6d.N.final.e patch** (~50 lines per site).  Replace the three yield-loops (`sys_read` CHANNEL arm, `sys_wait`, `nx_console_read`) with `nx_waitq_wait_with_deadline` calls; revert the busybox `fflush(stdout)` patch in `third_party/busybox/libbb/lineedit.c:put_prompt_custom`.  The visible-prompt regression script (`test/interactive/visible_prompt.{script,expected}`) stays as a guard.
