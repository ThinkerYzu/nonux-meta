# Session 16: Phase 4 planning + slice 4.1 (task + context switch)

**Date:** 2026-04-21
**Phase:** 4 — Context Switch and Scheduler
**Branch:** master

---

## Goals

- Align on Phase 4's shape before any code lands: cut the phase into
  four reviewable slices, commit the design decisions behind each, and
  write them down in [DESIGN.md](../DESIGN.md) and
  [IMPLEMENTATION-GUIDE.md](../IMPLEMENTATION-GUIDE.md).
- Ship slice 4.1 — the ARM64 context-switch primitive and the
  task representation — as a pure core-side change. No scheduler
  policy, no timer hookup, no preemption.

## What Was Done

### Design decisions (landed in DESIGN.md)

**Scheduler split: core driver + policy component.** The scheduler is
layered across the core/component boundary:

- `core/sched/` owns `struct nx_task`, `cpu_switch_to`, preempt
  counters, and the (slice-4.4) IRQ-return reschedule shim. It's
  frozen — never swappable at runtime.
- `components/sched_rr/` (slice 4.3) owns the runqueue, the quantum,
  and the `struct scheduler_ops` surface. Swappable in principle.

The two halves connect through a single stashed pointer (`g_sched`,
`g_sched_self`) that `framework/bootstrap.c` writes at the end of
boot bring-up via `sched_init(ops, self)`. The timer ISR never
dereferences `slot->active`; it flips `current->need_resched`, and
the IRQ-return shim reads `g_sched` on the interrupted task's kernel
stack. This keeps Invariant #7 (slot-resolve locality) intact while
enabling preemption before the per-CPU dispatcher thread — which
needs kthread support — arrives in slice 3.9b.

**Phase 4 = kernel threads only.** EL1h only, shared address space,
separate kernel stack per task. MMU / per-process AS stays Phase 5;
EL0 stays Phase 7.

**Timer quiescence across recomposition.** Every timer source whose
tick feeds a component must be paused across the PAUSE → REWIRE →
RESUME window. Implemented by a new `timer_pause()` / `timer_resume()`
nesting pair at the GIC-mask level (slice 4.4). Closes the race
where a tick mid-swap would read a torn `g_sched`.

**`sched_priority` deferred to Phase 8.** Phase 4 proves swappability
through the conformance suite + manifest-driven binding, not via a
runtime swap demo. A second scheduler implementation pairs better
with Phase 8's real recomposition API.

**Kthread primitive is a core/sched symbol.** `sched_spawn_kthread`
(slice 4.4) is not a public `nx_*` framework API — it's a core
primitive. Kernel threads are not "component-spawned workers" in the
sense slice 3.8's `spawns_threads ⇒ pause_hook` rule means, so
`sched_rr` declares both manifest fields `false`.

**`NX_HOOK_CONTEXT_SWITCH` hook-point.** Renamed the Phase-4+ entries
in the hook-points table to the `NX_HOOK_*` convention. Extended
`struct nx_hook_context.u` with a `csw { prev, next }` arm. Enum
addition lands in slice 4.3; dispatch is live in slice 4.4.

### Slice plan (IMPLEMENTATION-GUIDE.md §Phase 4)

| Slice | Deliverable | Blocks |
|---|---|---|
| **4.1** (this session) | `struct nx_task` + `cpu_switch_to` + preempt counters | 4.4 |
| **4.2** | `interfaces/scheduler.h` + host conformance suite | 4.3 |
| **4.3** | `components/sched_rr/` + `NX_HOOK_CONTEXT_SWITCH` enum | 4.4 |
| **4.4** | Timer→reschedule shim, idle task, kthread primitive, interleave demo | 3.9b |

Critical path: 4.1 → 4.3 → 4.4. 4.1 & 4.2 are independent.

### Slice 4.1 — implementation

New files:

- **`core/lib/list.h`** — intrusive doubly-linked-list primitives
  (`struct nx_list_head` / `struct nx_list_node`, add/remove,
  pop_front, `nx_list_for_each*`, `nx_list_entry`). Will back the
  scheduler's runqueue in slice 4.3.
- **`core/sched/task.{h,c}`** — `struct nx_cpu_ctx` (12 callee-saved
  regs + SP, 16-byte-aligned), `struct nx_task` (id, name, state,
  preempt_count, need_resched, kstack, sched_node). The static_assert
  `offsetof(nx_task, cpu_ctx) == 0` lets the asm treat a task pointer
  as a context pointer directly. `nx_task_create(name, entry, arg,
  kstack_pages)` allocates + hand-crafts `cpu_ctx` so the first
  cpu_switch_to into the new task lands at `entry(arg)` via a small
  first-switch thunk. `nx_task_destroy` releases the kstack and
  struct. `nx_task_current()` reads TPIDR_EL1 on the kernel build,
  a host-only static on the host build. Host-only
  `nx_task_set_current_for_test()` lets preempt-counter tests exercise
  nesting without a real scheduler.
- **`core/sched/sched.{h,c}`** — `nx_preempt_disable` /
  `nx_preempt_enable` / `nx_preempt_count`. Per-task counter on
  `current->preempt_count`. The "when count hits zero, consult
  need_resched" action point is stubbed here; slice 4.4 wires it to
  the reschedule shim.
- **`core/cpu/context.S`** — `cpu_switch_to(prev, next)`: save
  callee-saved regs + SP into `prev` (via stp pairs on 16-byte
  offsets), load same from `next`, `msr tpidr_el1, x1`, `ret`. First
  `ret` for a newly-created task lands at `nx_task_bootstrap`, a
  small thunk that restores x19 (entry) / x20 (arg) and calls
  `entry(arg)` — the classic ARM64 coroutine-switch pattern where the
  saved LR encodes "which function does this task resume into."
  `blr x19` rather than `br x19` so a debugger sees a sensible call
  stack on the first entry.

New tests:

- **`test/host/task_test.c`** (+6 tests):
  - `task_create_populates_cpu_ctx_for_first_switch` — x19 = entry,
    x20 = arg, SP at top of kstack, 16-byte aligned, initial state
    fields zero.
  - `task_create_truncates_long_name` — names longer than
    `NX_TASK_NAME_MAX` truncate with a terminating NUL.
  - `task_create_assigns_unique_nonzero_ids` — the atomic ID seq
    produces distinct nonzero IDs.
  - `task_create_rejects_null_entry` — defensive.
  - `preempt_disable_and_enable_nest_on_current` — nesting 1/2/1/0;
    underflow guard.
  - `preempt_count_without_current_is_zero` — safe no-op when no
    "current" task is set.

- **`test/kernel/ktest_context.c`** (+2 tests):
  - `context_switch_round_trip` — boot context (A) → new task (B) →
    sentinel write → `cpu_switch_to(B, A)` → back in A with the
    sentinel visible. Proves save/restore + first-switch thunk
    actually work under QEMU.
  - `context_switch_preserves_callee_saved` — load recognisable
    values into x19 and x28 in A, switch into B (which stomps every
    callee-saved), switch back, verify x19 / x28 still carry A's
    values. Bookend check for the `stp`/`ldp` pair block.

### Makefile wiring

- `CORE_S` gains `core/cpu/context.S`.
- `CORE_C` gains `core/sched/task.c core/sched/sched.c`.
- `KTEST_C` gains `test/kernel/ktest_context.c`.
- `test/host/Makefile` adds `task_test.c` to `SRCS`, adds
  `core/sched/{task,sched}.c` to the cross-compiled list, and
  extends `vpath` to include `../../core/sched`.

### Test counts

- Python: **51** (unchanged).
- Host:   **134** (128 → +6 task/preempt tests).
- Kernel: **13** (11 → +2 context tests).
- Total:  **198 / 198 pass**, 0 leaks, 0 errors, exit 0.
- `make verify-registry`: 0 findings.
- QEMU boot unchanged: composition JSON emits, then ktest runs 13/13
  including the two new context tests.

## Decisions

**`struct nx_cpu_ctx` layout placed at offset 0 of `struct nx_task`.**
Lets the asm treat task pointer as ctx pointer — no offset constants
threaded through a generated header, no risk of C and asm disagreeing
on layout. `_Static_assert` pins it.

**First-switch thunk over "fake return frame on the kstack."** Two
candidate shapes: (a) hand-craft a return frame on the new kstack so
`ret` from cpu_switch_to lands directly at entry, or (b) stash entry
in x19 / arg in x20 and set LR to a small thunk. (b) is simpler and
gives a cleaner debugger stack; (a) requires matching the C ABI's
frame-record layout byte-for-byte. Went with (b).

**SP alignment at kstack top, no red-zone.** AArch64 AAPCS requires
16-byte alignment at public interface boundaries. `sp_top &= ~0xf`.
No red zone reserved; the kernel's compiled code doesn't assume
one, and we're not running leaf functions below SP anywhere critical.

**Preempt counter is per-task, not global.** The kernel is
preemptive, so two tasks with different disable nesting can coexist;
a global counter would lose that information at every task switch.
Cost is a field on `struct nx_task` and two loads in
preempt_{dis,en}able — negligible.

**Host tests use a host-only `nx_task_set_current_for_test()`.** The
host build has no TPIDR_EL1; the scheduler-less preempt-counter tests
still need *some* "current" task to act on. The override is clearly
named, `#if __STDC_HOSTED__` guarded, and never linked in kernel
builds — won't leak into production.

**Host kstack from `malloc`, not PMM.** Host tests don't execute on
the allocated stack (cpu_switch_to isn't exercised on host), so a
plain `malloc(pages * 4096)` is enough to prove `cpu_ctx->sp` lands
inside the region. Keeps `test/host/` off the PMM state machine for
slices where it's not needed.

**No exit semantics yet.** If a task's entry function ever returns,
the bootstrap thunk drops into a `wfe` park. Slice 4.4 adds
`nx_task_exit()` once there's a scheduler to dequeue the dead task.

## What's Next

**Slice 4.2 — conformance suite + `interfaces/scheduler.h`.** Define
the scheduler operations, ship the host conformance harness, exercise
it against a `sched_null` fixture. ~10 new host tests expected.

**Slice 4.3 — `components/sched_rr/`.** First real policy component.
Manifest + impl + README. Adds `NX_HOOK_CONTEXT_SWITCH` enum (dispatch
still stubbed until 4.4).

**Slice 4.4 — preemption + idle + kthread primitive.** Biggest of the
four; this is where the reschedule shim, `timer_pause()`/
`timer_resume()`, idle task, and the bootstrap handoff
(`sched_start()` replacing `wfi`) land.

**Opportunistic, not blocking:**
- MMU bring-up so `-mstrict-align` can be dropped (unchanged from
  session 15).
- Linker script LOAD-segment non-RWX (unchanged).

## Files Touched

- `core/cpu/context.S` — new.
- `core/lib/list.h` — new.
- `core/sched/task.{h,c}` — new.
- `core/sched/sched.{h,c}` — new.
- `test/host/task_test.c` — new.
- `test/kernel/ktest_context.c` — new.
- `Makefile` — `CORE_S` / `CORE_C` / `KTEST_C` wiring.
- `test/host/Makefile` — host test wiring + vpath.
- `proj_docs/nonux/DESIGN.md` — new `#### Scheduler: Core Driver +
  Component` under §Execution Model; §Recomposition Protocol gains a
  "Timer quiescence" block; hook-points table renamed to `NX_HOOK_*`;
  hook-context union gains the `csw` arm; Design Evolution entry
  added for 2026-04-21.
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — Phase 4 section rewritten
  with 4.1 → 4.4 slice breakdown.
- `proj_docs/nonux/HANDOFF.md` — status + session logs.
- `proj_docs/nonux/README.md` — status + last-updated.
- `proj_docs/nonux/logs/session-16-task-context.md` — this log.
