# Session 19: slice 4.4 — preemption + idle + kthread primitive

**Date:** 2026-04-22
**Phase:** 4 — Context Switch and Scheduler
**Branch:** master

---

## Goals

- Turn the scheduler on: stash the `sched_rr` policy's ops/self in the
  core driver via `sched_init`, call it from `framework/bootstrap.c`
  at the end of bring-up.
- Transition `boot_main`'s CPU context into an idle task via
  `sched_start()`; have `nx_task_current()` return something real
  from here on.
- Wire the timer ISR's `sched_tick` through `g_sched->tick`, add the
  IRQ-return reschedule shim to `core/cpu/vectors.S`.
- Ship `sched_spawn_kthread`, `nx_task_yield`.
- Add `timer_pause`/`timer_resume` (nested at the GIC) for the
  recomposition timer-quiescence rule from session 16.
- Five kernel tests demonstrating: sched_init populated,
  boot-context-promoted-to-idle, quantum drives need_resched,
  spawned kthread actually runs, two kthreads interleave
  cooperatively.

Phase 4 closes with this slice.

## What Was Done

### DAIF save/restore in `cpu_switch_to` (prerequisite)

Added a `daif` field to `struct nx_cpu_ctx` at offset 0x68. The
context-switch assembly now does `mrs x2, daif; str x2, [x0, #0x68]`
on the outgoing task and `ldr x2, [x1, #0x68]; msr daif, x2` on the
incoming. `nx_task_create` initialises `daif = 0` (IRQs enabled) for
new kthreads.

Why: without this, a task interrupted inside an IRQ handler (DAIF=I
masked by hardware) switching to a task that last yielded voluntarily
(DAIF=I enabled) would leak the masked state into the yielding task
— the CPU would never take another tick and the system would hang.
Caught while reasoning through the switch sequence; the alternative
of always toggling DAIF in the shim is racier and doesn't model the
invariant that each task carries its own exception-mask state.

### Core scheduler driver — `core/sched/sched.{h,c}`

New public API:

- `sched_init(ops, self)` — stashes the policy pair in file-scoped
  `g_sched_ops` / `g_sched_self`. R8 escape-hatch: from here on the
  ISR path can drive `pick_next` without dereferencing `slot->active`.
- `sched_is_initialized()` / `sched_ops_for_test()` /
  `sched_self_for_test()` — test introspection.
- `sched_start()` — statically allocates `struct nx_task g_idle_task`
  (name="idle", id=0), sets `TPIDR_EL1 = &g_idle_task`, enqueues idle
  as a fallback on the runqueue. Idempotent. The boot CPU becomes the
  idle task after this call.
- `sched_tick()` — forwards to `g_sched->tick(g_sched_self)`. Safe
  no-op until `sched_init` runs.
- `sched_check_resched()` — the IRQ-return + yield reschedule shim.
  Checks `curr->need_resched` + `preempt_count`, calls `yield()`
  (policy rotation), `pick_next()`, fires `NX_HOOK_CONTEXT_SWITCH`,
  then `cpu_switch_to(curr, next)`. Resumes the caller's context
  when some future switch picks `curr` again.
- `nx_task_yield()` — sets `need_resched=1`, calls `sched_check_resched`.
  Kthread bodies and tests use this for cooperative yield.
- `sched_spawn_kthread(name, entry, arg)` — thin wrapper over
  `nx_task_create` + `g_sched->enqueue`. Returns the task. Core
  primitive (not a component-spawned worker — the slice-3.8
  `spawns_threads ⇒ pause_hook` manifest rule does not apply).

### IRQ-return reschedule shim — `core/cpu/vectors.S`

`_irq_stub` gains a `bl sched_check_resched` between the `bl on_irq`
and `RESTORE_TRAPFRAME` + `eret`. On every IRQ-return, the shim reads
the current task's `need_resched` and, if set, switches on the
interrupted task's kernel stack. The switch may not return until
that task is picked again in a future reschedule — on resumption the
`RESTORE_TRAPFRAME` + `eret` bookend runs normally.

### Timer — `core/timer/timer.{h,c}`

`on_tick` now calls `sched_tick()` (the old "[tick] <n>" serial print
is gone; it was a boot-time debug line). Two new functions:

- `timer_pause()` — `__atomic_fetch_add` on a pause-nest counter; on
  the 0→1 edge, `gic_disable(TIMER_PPI)`.
- `timer_resume()` — CAS-decrement saturating at zero; on the 1→0
  edge, `gic_enable(TIMER_PPI)`. No-op if already at zero.

Nested calls compose cleanly. These land now so Phase 8's
recomposition path has the timer-quiescence primitive already in
place (DESIGN.md §Recomposition Protocol — Timer quiescence).

### `struct nx_component_descriptor` — `iface_ops` field

Added `const void *iface_ops` to the descriptor. Zero for most
components (uart_pl011, trivial_ops fixture); non-zero for
components that export an interface-specific op table alongside
their framework-facing `nx_component_ops`.

Two new macros sit alongside the originals for backward compatibility:

- `NX_COMPONENT_REGISTER_IFACE(NAME, CONTAINER, DEPS_FIELD, OPS,
  IFACE_OPS, DEPS_TABLE)` — full form.
- `NX_COMPONENT_REGISTER_NO_DEPS_IFACE(NAME, CONTAINER, OPS,
  IFACE_OPS)` — no-deps form.

The original 5-arg / 3-arg macros delegate to the IFACE variants
with `IFACE_OPS = NULL`, so existing callers (uart_pl011,
trivial_ops) keep working unchanged.

`sched_rr` now registers via `NX_COMPONENT_REGISTER_NO_DEPS_IFACE`
with `&sched_rr_scheduler_ops`. Its state struct drops the old
`current` field — sched_rr's `tick` now reads
`nx_task_current()` directly (the core driver is the sole source of
truth for "which task is running").

### Bootstrap handoff — `framework/bootstrap.c`

End of `nx_framework_bootstrap()`:

```c
struct nx_slot *sched_slot = nx_slot_lookup("scheduler");
if (sched_slot && sched_slot->active &&
    sched_slot->active->descriptor &&
    sched_slot->active->descriptor->iface_ops) {
    const struct nx_scheduler_ops *sops =
        sched_slot->active->descriptor->iface_ops;
    sched_init(sops, sched_slot->active->impl);
}
```

Called only on the freestanding branch (kernel build); host tests
don't include bootstrap.c in `SRCS`.

### Boot path — `core/boot/boot.c`

After `nx_framework_bootstrap()`:

- `sched_start()` — promotes boot context into idle task.
- `irq_enable_local()` — IRQs on for the first time.
- Under `NX_KTEST`: `ktest_main()` runs in the idle-task context
  (TPIDR_EL1 = &g_idle_task). Tests can spawn kthreads, yield, etc.
- Otherwise: `for(;;) wfi;` — the idle loop body.

### Kernel tests — `test/kernel/ktest_sched.c` (+5)

1. `sched_init_stashed_scheduler_ops_are_populated` — verifies
   `sched_is_initialized`, `sched_ops_for_test()` returns
   `&sched_rr_scheduler_ops`, self is non-NULL.
2. `sched_start_promoted_boot_context_into_idle_task` —
   `nx_task_current()` returns a task with id=0, name=="idle",
   state==RUNNING.
3. `sched_tick_flips_need_resched_after_quantum` — clear
   need_resched, call `sched_tick()` 32× bounded, assert it
   transitions to 1. Reset before return.
4. `sched_spawn_kthread_runs_entry_after_yield` — spawn a kthread
   that writes a sentinel, yield from idle, assert the sentinel.
   Dequeue + destroy on cleanup.
5. `sched_two_tasks_cooperate_via_yield` — spawn ta/tb that each
   loop `TWO_TASKS_ITERS=5` times (increment counter, yield). Idle
   yields up to 200 times; both counters reach 5. Dequeue + destroy.

### `ktest_main` — `reset_current_to_idle` between tests

The hardest bug in this slice hit kernel test #4. `ktest_context`
(slice 4.1) tests set `TPIDR_EL1` to their own static
`boot_task` struct via a local `set_current()` helper. They never
restore it. When slice-4.4 tests ran later, `nx_task_current()`
returned the stale `boot_task` pointer — `cpu_switch_to` saved
idle's register state into that unrelated struct, and when
something later tried to switch back "to idle", the real
`g_idle_task.cpu_ctx` was still zero. `ret` jumped to address 0
and QEMU spun.

Debugged by printing `curr->cpu_ctx.sp` / `.x30` pre-switch and
`live_lr` via inline asm post-switch. The saved LR for idle was
0 because we'd saved to the wrong task.

Fix: `ktest_main` now calls `reset_current_to_idle()` before every
test. Also promoted `g_idle_task` to a non-static symbol so
`ktest_main` can take its address. Clean fix; no test had to change.

### Test counts

- Python: **51** (unchanged).
- Host:   **153** (unchanged — slice 4.4 is kernel-side only).
- Kernel: **22** (17 → +5).
- Total:  **226 / 226 pass**, 0 leaks, 0 errors.
- `make verify-registry`: 0 findings.
- Non-ktest `make && qemu`: boots to "idle: waiting for work" and
  stays there — the idle loop is running as expected.

## Decisions

**DAIF saved per-task, not toggled in the shim.** Linux toggles DAIF
around the reschedule call; we save it in the context-switch primitive
instead. On single-CPU Phase 4 this is strictly simpler: every switch
path preserves the invariant "DAIF carries the task's last live
exception-mask state." No need for special cases for "switch from
ISR-context task to voluntary-yield task" etc.

**Idle uses the boot stack, kstack_base=NULL.** The idle task is the
ex-boot context — its SP already lives on the linker-allocated
boot stack. `nx_task_create` isn't called for idle, so no kstack is
separately allocated. `kstack_base=NULL, kstack_size=0` on
`g_idle_task` as a sentinel (never passed to `free_kstack` because
idle is never destroyed).

**`sched_start` is idempotent.** A boolean guards against double-entry
in case something funky calls it twice. Cheap safety net.

**`NX_HOOK_CONTEXT_SWITCH` fires inside `preempt_disable`.** Hook
handlers see `prev`/`next` after `pick_next` has selected the incoming
task and before `cpu_switch_to` fires. Preempt is disabled so the
handler sees the *actual* next task that runs.

**Host-side stub for `cpu_switch_to`.** The ARM64 asm can't be linked
into the x86-64 host build. Added a trivial `void cpu_switch_to(prev,
next) { abort(); }` under `#if __STDC_HOSTED__` in `task.c`. Host
tests never drive the scheduler into a real switch — `need_resched`
+ `preempt_count==0` is never reached — but `sched.c` references
the symbol unconditionally. If anything ever does hit it, `abort`
makes the failure loud.

**`iface_ops` is `const void *`, not a typed pointer.** The
descriptor in `framework/component.h` doesn't include every interface
header; consumers cast based on the slot's iface tag. Same pattern as
the existing `ops` field's per-callback-type dispatch.

**`reset_current_to_idle` is a `ktest_main` responsibility, not
per-test.** Putting it in the framework means every future ktest
author doesn't need to remember the invariant. The alternative (make
ktest_context restore TPIDR_EL1) works but requires test-author
discipline; frameworks beat discipline.

**Debug prints left in place during investigation.** The
`[dbg] resched:` trace was indispensable for diagnosing the
TPIDR_EL1 leak — without seeing `curr=0x...` and `idle=0x...` side
by side, the "saved state is zero" symptom was baffling. All debug
prints removed before commit; the session log is where they're
preserved.

## Phase 4 closed

All four slices landed:

- 4.1 — task + context switch primitive.
- 4.2 — scheduler interface + host conformance suite.
- 4.3 — `components/sched_rr/` + `NX_HOOK_CONTEXT_SWITCH` enum.
- 4.4 — preemption + idle + kthread primitive (this).

`make test` → **226/226** (51 python + 153 host + 22 kernel).

## What's Next

**Slice 3.9b** — the runtime dispatcher upgrade, now unblocked by
Phase 4's `sched_spawn_kthread` primitive. MPSC lock-free inbox;
per-CPU dispatcher kthread; `pause_hook` 1 ms wall-clock deadline
enforced against the ARM generic timer; pause-failure rollback;
runtime `NX_HOOK_SLOT_SWAPPED` dispatch from the registry's swap
path; `nx_ipc_enqueue_from_irq` ISR-safe enqueue.

**Phase 5** — virtual memory + handle system. `components/mm_buddy/`;
`framework/handle.{c,h}`; `framework/syscall.c`; first userspace (EL0)
process.

**Opportunistic polish**:
- MMU bring-up so `-mstrict-align` can be dropped (still unchanged).
- Linker LOAD-segment non-RWX (once MMU is up).
- Real AArch64 Linux Image header so `-kernel` self-describes the
  load offset.

## Files Touched

- `core/sched/sched.{h,c}` — g_sched stashed pair, sched_init/start/
  tick/check_resched, nx_task_yield, sched_spawn_kthread, idle.
- `core/sched/task.{h,c}` — daif field in cpu_ctx; host-only
  `cpu_switch_to` stub.
- `core/cpu/context.S` — save/restore daif.
- `core/cpu/vectors.S` — `bl sched_check_resched` at IRQ return.
- `core/timer/timer.{h,c}` — `sched_tick()` call; `timer_pause` /
  `timer_resume` nested pair.
- `core/boot/boot.c` — `sched_start()` + either ktest_main or wfi
  loop.
- `framework/component.h` — `iface_ops` field + `_IFACE` macros.
- `framework/bootstrap.c` — `sched_init` handoff at end of bring-up.
- `components/sched_rr/sched_rr.c` — use `NO_DEPS_IFACE` macro;
  `tick` reads `nx_task_current()`; state's `current` field dropped.
- `test/kernel/ktest_main.c` — `reset_current_to_idle` between tests.
- `test/kernel/ktest_sched_bootstrap.c` — state mirror updated
  (no `current` field).
- `test/kernel/ktest_sched.c` — 5 new tests.
- `Makefile` — `KTEST_C` gains `ktest_sched.c`.
- `proj_docs/nonux/HANDOFF.md` / `README.md` — status + slice 4.4
  check-off + Phase 4 closed.
- `proj_docs/nonux/logs/session-19-preemption.md` — this log.
