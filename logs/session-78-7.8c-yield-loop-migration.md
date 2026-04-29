# Session 78 — slice 7.8c: yield-loop migration + busybox-patch revert

**Date:** 2026-04-29
**Slice:** 7.8c — migrate the three v1 yield-loop call sites onto the slice 7.8a wait-queue primitive + revert the slice 7.6d.N.final.e busybox `fflush(stdout)` patch.
**Outcome:** **Closes slice 7.8.**  All three yield-loops (`sys_read` CHANNEL arm slice 7.6d.N.6b, `sys_wait` slice 7.4b, `nx_console_read` slice 7.6d.N.final.a) now block via `nx_waitq_wait_unless` on the appropriate object's waitq.  busybox's stock upstream `lineedit.c` runs unmodified — the prompt-flush comes from busybox's intended `ask_terminal()` `\e[6n` + `fflush_all()` path, which now succeeds because `safe_poll(stdin, 0) == 0` returns success via the slice 7.8b `NX_SYS_PPOLL`.  `make test` 453/453; `make test-interactive` 7/7.

---

## Goal

Slice 7.8a landed the wait-queue primitive with no production callers; slice 7.8b lit it up via the `NX_SYS_PPOLL` syscall.  Slice 7.8c finishes the family by:

1. Promoting the three v1 yield-loop call sites onto the waitq primitive — replacing `nx_task_yield()` busy polls with real BLOCKED waits that only wake on the producer's state change (or a deadline).
2. Reverting the slice 7.6d.N.final.e tactical patch in `third_party/busybox/libbb/lineedit.c:put_prompt_custom` — the prompt visibility now comes from busybox's intended `ask_terminal` flush path, which works once `safe_poll(stdin, 0) == 0` returns success (which now happens because `__NR_ppoll = 73` is mapped to `NX_SYS_PPOLL = 41` per slice 7.8b).

The slice plan (HANDOFF.md Next Actions §0 + IMPLEMENTATION-GUIDE.md §Slice 7.8c) called for ~50 lines per site, 3 sites + the patch revert.  Real cost: ~140 lines added (kernel + waitq), ~20 lines deleted (busybox patch + the three yield-loop bodies), so net ~120 new lines.

---

## What landed

### `core/sched/waitq.{h,c}` — new `nx_waitq_wait_unless`

The slice 7.8a primitive only exposed unconditional `wait_with_deadline`.  Migrating yield-loops needs an atomic "register-then-check" entry path so a wake that fires between the caller's outer `cond` check and the wait isn't lost.

New API:

```c
int nx_waitq_wait_unless(struct nx_waitq *wq, uint64_t budget_ns,
                         int (*pred)(void *), void *ctx);
```

Re-evaluates `pred(ctx)` *inside* the IRQ-disabled + preempt-disabled critical section that registers the caller on `wq`.  If it returns non-zero, the caller is NOT enqueued and the function returns `NX_OK` immediately — treating the predicate as an already-fired wake.  Otherwise the standard wait path runs (dequeue from runqueue, link onto `wq->waiters`, optionally onto `g_deadline_list`, set `NX_TASK_BLOCKED`, yield).

Internal restructuring: the body of `wait_with_deadline` was extracted into a static `waitq_register_and_wait_locked(wq, t, budget_ns, flags)` helper that the new variant also calls — both functions diverge only in the predicate-check prologue.  +33 lines `waitq.c`, +21 lines `waitq.h`.

The caller pattern is now:

```c
while (!cond) {
    nx_waitq_wait_unless(&wq, 0, cond_pred, ctx);
}
```

A wake that fires between the outer `cond` check and the inner `cond_pred` check is caught by the predicate.  A wake that fires after the predicate but before the actual yield is caught by the existing wait_with_deadline machinery (the task is on the waitq when the wake fires).

### `framework/process.{h,c}` — per-process exit_waitq

`struct nx_process` grows `struct nx_waitq exit_waitq`.  Initialised in `nx_process_create`; the statically-defined `g_kernel_process` gets a designated initialiser making the waiters list self-pointing (nobody waits on the kernel process's exit, but a stray `wake_all` shouldn't crash).

`nx_process_exit` walks up to `parent_pid` (set at fork time) and calls `nx_waitq_wake_all(&parent->exit_waitq)` after the handle-table cleanup but before the exiting task's terminal `wfe` loop.  Processes spawned outside fork (`parent_pid == 0` — init, the test parents) skip the wake; nothing waits on them.

### `framework/syscall.c` — `sys_wait` migration

Was:

```c
while (target->state != NX_PROCESS_STATE_EXITED)
    nx_task_yield();
```

(plus a parallel yield loop in the `pid == -1` POSIX-waitpid-any branch.)

Now:

```c
while (target->state != NX_PROCESS_STATE_EXITED) {
    nx_waitq_wait_unless(&caller->exit_waitq, 0,
                         sys_wait_target_exited_pred, target);
}
```

(parallel migration in the `pid == -1` branch using `sys_wait_any_child_pred` which scans for a reapable EXITED child OR a no-children-at-all condition).  Two new static-inline predicate helpers above `sys_wait`; both are guarded `#if !__STDC_HOSTED__` because the host build can't reach the wait path (sys_wait early-returns `NX_EINVAL` on host).

### `framework/syscall.c` — `sys_read` CHANNEL arm migration

Was:

```c
for (;;) {
    got = nx_channel_recv(obj, staging, cap);
    if (got != NX_EAGAIN) break;
    nx_task_yield();
}
```

Now (kernel branch only — host stays as a single non-blocking attempt):

```c
struct nx_waitq             read_wq;
struct nx_pollset_listener  listener;
nx_waitq_init(&read_wq);
nx_pollset_listener_init(&listener, &read_wq);
nx_channel_endpoint_register_pollset(obj, &listener);
for (;;) {
    got = nx_channel_recv(obj, staging, cap);
    if (got != NX_EAGAIN) break;
    nx_waitq_wait_unless(&read_wq, 0,
                         sys_read_channel_ready_pred, obj);
}
nx_channel_endpoint_unregister_pollset(&listener);
```

Reuses the slice 7.8b pollset-listener mechanism — `nx_channel_send` and `nx_channel_endpoint_close` already walk `pollset_listeners` and wake every registered waitq.  No new producer-side wake calls needed.  The kstack `read_wq` + `listener` pair is a one-off "private waitq" that this call owns for its lifetime; multiple concurrent readers each get their own waitq, and a single producer wake fans out through the listener list.

Predicate (`sys_read_channel_ready_pred`) returns non-zero when `nx_channel_endpoint_depth(e) > 0` OR `nx_channel_endpoint_peer_closed(e)` — exactly the condition `nx_channel_recv` uses to return non-EAGAIN.

### `framework/console.c` — `nx_console_read` migration

Same pattern as `sys_read` CHANNEL: kstack waitq + listener, register on `g_console_pollset_listeners`, wait_unless predicate (`console_read_ready_pred`) covers ring-non-empty OR Ctrl-D EOF armed.  The existing producer side (`nx_console_rx_isr`, `nx_console_test_inject_bytes`, `nx_console_test_inject_eof` from slice 7.8b) already calls `nx_pollset_wake_all(&g_console_pollset_listeners)` so no producer changes needed.

The Ctrl-D EOF branch unregisters the listener before returning 0 (otherwise the kstack listener would persist past function return — undefined behaviour the moment the next caller's stack overlaps).  The drain-up-to-cap branch unregisters after the loop exits via `got > 0`.

### `third_party/busybox/libbb/lineedit.c` — patch revert

The `fflush(stdout);` line + ~20 lines of comment that slice 7.6d.N.final.e added to `put_prompt_custom` are removed.  `put_prompt_custom` reverts to upstream form: `fputs_stdout(prompt) + cursor = 0 + cmdedit_y/x assignments`.  The prompt now becomes visible because busybox's `ask_terminal()` (called once per prompt at `lineedit.c:1908-1916`) succeeds in its `safe_poll(stdin, 0) == 0` gate, fires the `\e[6n` cursor-position query, and calls `fflush_all()` — which flushes the prompt that was sitting in musl's tty-line-buffered FILE.

The `test/interactive/visible_prompt.{script,expected}` regression script stays as a guard against future re-introduction.

---

## Discoveries

1. **Lost-wakeup race is real on single-CPU.**  The classic single-CPU "no concurrency means no race" intuition fails here because timer interrupts can preempt between the caller's outer `cond` check and the wait's inner critical section.  Without `wait_unless`'s predicate re-check, a child exit that fires during the gap (its wake reaches an empty exit_waitq) would block sys_wait forever.  The predicate-inside-critical-section pattern is the simplest correct fix.

2. **Each yield-loop site needs its own private waitq.**  Initially I considered giving each `nx_channel_endpoint` a single `read_waitq` field that sys_read would share across concurrent readers.  That's strictly more efficient (no kstack allocation) but conflates "blocking readers waiting for data" with "producer wakes everyone via wake_all" — a wake_all on a shared waitq with N readers wakes all N, even though only one will succeed in `nx_channel_recv` (the others will see EAGAIN and re-block).  Per-call kstack waitq + listener gives the same wake fan-out via the existing pollset-listener mechanism without that thundering-herd issue: each caller is on its own waitq, and the producer's `nx_pollset_wake_all` only wakes the registered listeners (one per active reader).

3. **The Ctrl-D early-return path needs an explicit unregister.**  v1's `nx_console_read` had a single early-return for Ctrl-D EOF; with the kstack listener now registered before the loop, that early return must unregister the listener before returning, otherwise the per-object list is left pointing at a stack address that's about to be invalidated.  Caught in code review before testing — defensive programming pays.

4. **`nx_process` had to gain `core/sched/waitq.h` as a direct include.**  Embedding `struct nx_waitq` requires its definition; previously `process.h` only knew the forward decl.  Pure include shuffle, no behaviour change; verified the host build still compiles (waitq.h is host-safe by design).

---

## Test results

```
make test:           453/453 pass (51 python + 283 host + 119 kernel,
                                    0 leaks, 0 errors, exit 0)
make test-interactive: 7/7 pass
                       (echo_cat + echo_hello + echo_pipe + ls_root +
                        mkdir_tmp + ps_smoke + visible_prompt)
```

Same totals as session 77 baseline — the migration adds no new tests but verifies that every existing test case continues to pass with the yield-loops replaced.  Critically, `visible_prompt` still passes after reverting the slice 7.6d.N.final.e busybox patch, proving busybox's intended `ask_terminal` flush path runs end-to-end via `NX_SYS_PPOLL`.

---

## Cleanup status

Slice 7.8 closed.  No more yield-loops anywhere in the kernel.  No more vendored-busybox local diffs.  The three remaining `nx_task_yield()` calls in syscall.c are deliberate scheduler primitives, not blocked-on-state polls.

Phase 9 benchmarks no longer face the noise floor of yield-loop CPU waste under load — every blocking syscall now actually sleeps until its specific wake condition fires (or a deadline expires).  Future Phase 8 runtime-recomposition drain points can also depend on the waitq abstraction for clean quiesce semantics.

---

## Next steps

Phase 7 is done.  The next forward step is **Phase 8: Runtime Recomposition and Config Manager** — `framework/config.c`, runtime hot-swap (disable old → enable new, message draining), runtime connection mode switching, demonstrating a runtime scheduler swap.  See [IMPLEMENTATION-GUIDE.md §Phase 8](IMPLEMENTATION-GUIDE.md#phase-8-runtime-recomposition-and-config-manager).

The deferred items inside Phase 7 (7.6d.N.16 POSIX-fd-to-slot alignment; 7.6d.N.final.c-full signal-handler trampoline; reap-on-wait) remain available as focused follow-ups when a workload demands them.
