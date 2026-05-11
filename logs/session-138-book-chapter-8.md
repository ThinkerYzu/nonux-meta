# Session 138: Book chapter 8 — The scheduler

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–8 shipped (Part I + Part II + Part III + Part IV complete); Phase 9 not yet started
**Branch:** master

---

## Goals

- Draft the next book chapter, the second and final chapter of
  Part IV (Tasks).  Per
  [BOOK-OUTLINE.md](../BOOK-OUTLINE.md#chapter-list) that is
  **chapter 8: The scheduler** — the policy half of the task
  layer.  Chapter 7 ended with "we deliberately leave one big
  question for the next chapter: who decides which task runs
  next?"  This chapter is that answer.
- Cover both shipped policy components (`sched_rr` and
  `sched_priority`) line by line, the core/policy split,
  `sched_check_resched` end-to-end, the wait-queue primitive,
  and at least one worked timeline so a reader can see the
  mechanism running.

## What Was Done

### `sources/nonux/book/08-the-scheduler.md` (new, 1183 lines)

Inside the 600–1500-line sweet spot per
[BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md#length-and-pace).

Section structure:

1. **Intro** (3 paragraphs) — picks up chapter 7's loose end
   (we have a mechanism; this chapter adds the policy), promises
   the reader will know the core/policy split, what a runqueue
   is, the seven interface ops every policy must provide, the
   reschedule shim end-to-end, and a worked two-task timeline.
2. **Files** — `core/sched/sched.{h,c}`,
   `interfaces/scheduler.h`, the two shipped policy components,
   `core/sched/waitq.{h,c}` (one-paragraph cameo, full story
   deferred to later chapters that actually call it).
3. **Terms** (12 entries) — scheduler, core scheduler driver,
   scheduler policy, runqueue, quantum, preemption,
   `need_resched`, `preempt_count`, `g_idle_task`, WFI, wait
   queue.  All standard kernel-textbook terms; no invented
   metaphors per [BOOK-STYLE-GUIDE.md §Terminology rules](../BOOK-STYLE-GUIDE.md#terminology-rules).
4. **The architectural decision: core/policy split** — names
   the mechanism-vs-policy seam and the ASCII diagram of the
   two halves talking through `struct nx_scheduler_ops`.
   Forward-points to chapter 14 (live scheduler swap) without
   linking it yet — chapter 14 isn't shipped.
5. **The interface: `struct nx_scheduler_ops`** — quotes the
   header verbatim, then walks each of the eight pointers
   (`pick_next`, `enqueue`, `dequeue`, `yield`, `set_priority`,
   `tick`, `runqueue_size`, `reap_task`) with the per-op
   contract.  Calls out the *non*-requirement: the policy
   never tracks "current" — `TPIDR_EL1` is the single source
   of truth (back-references chapter 7).
6. **How the core driver finds the policy** — `sched_init`,
   the stashed `g_sched_ops`/`g_sched_self` pair, who calls
   `sched_init` (framework bootstrap at boot + the same
   `enable` hook again on live swap).  Side note on the
   two-step publish ordering.
7. **Booting into the idle task: `sched_start`** — the boot
   context has no identity until `sched_start` runs; the
   transition is a *rename*, not a creation.  Three-step
   walkthrough: `idle_task_init`, `msr tpidr_el1`, `enqueue`
   for the permanent fallback.  Connects to
   [chapter 7's idle-as-special-case](../book/07-kernel-threads-and-context-switch.md)
   section.
8. **The reschedule shim, line by line** — the centre of the
   chapter.  12-line abbreviated body of
   `sched_check_resched` (signal-delivery + MMU details
   deferred, called out explicitly with "covered in chapter
   17 / 15").  Numbered walkthrough of each line: get current,
   get policy, check `need_resched`, check `preempt_count`,
   clear the flag, policy yield, policy `pick_next`, the
   `wfi` branch, preempt-disable for hook chain, fire
   `NX_HOOK_CONTEXT_SWITCH` (forward-pointed to chapter 13),
   `cpu_switch_to`, `preempt_enable` running on the
   *incoming* task's stack.  Side note on clear-then-decide
   ordering.
9. **The timer half: `sched_tick`** — three-line body, the
   `g_sched_ops->tick(...)` call, `sched_rr_tick` as a
   concrete example.  Full chain diagram from hardware timer
   interrupt down to `cpu_switch_to`, with the
   `need_resched` flag as the seam between ISR (fast) and
   IRQ-return path (heavy).
10. **Voluntary yield: `nx_task_yield`** — the cooperative
    half.  Three-line body, identical machinery to involuntary
    preemption, must-not-call-from-ISR rule.  Names the
    unification: one place where switches happen, grep for
    `cpu_switch_to`.
11. **Policy one: round-robin** — `sched_rr_state`,
    `sched_rr_enqueue`, `sched_rr_pick_next` (with the
    `BLOCKED`-skip defence), `sched_rr_yield` (rotate head to
    tail), `sched_rr_set_priority` (returns `NX_EINVAL`
    uniformly, per the contract).
12. **The "wake idle" trick** — pulled out as its own section
    because it's the kind of small detail that separates
    "compiles and seems to work" from "feels right under
    load".  Worked latency case: without the trick, a UART
    RX wake of the dispatcher could sit up to 200 ms in the
    runqueue while idle burns its quantum in `wfi`.
13. **Policy two: fixed-priority** —
    `sched_priority_state` with 8 buckets,
    `sched_priority_pick_next` (high-to-low scan, per-bucket
    FIFO), `sched_priority_enqueue` (defaults to bucket 0),
    `sched_priority_set_priority` (the function call
    round-robin rejects with `NX_EINVAL` actually does
    something here), `sched_priority_yield` (rotate current
    *within its own bucket*).  Direct comparison to
    round-robin — same interface, different shape.
14. **Idle, WFI, and the empty-runqueue case** — three
    appearances of idle (`sched_start`, `pick_next` fallback,
    `wfi` branch in `sched_check_resched`).  Two trigger
    cases for `wfi`: empty runqueue, or `pick_next` returned
    the same task.  Low-power half explained.
15. **Wait queues: when tasks need to step off** — sketch the
    primitive: `nx_waitq_init`, `nx_waitq_wait_with_deadline`,
    `nx_waitq_wake_one`, `nx_waitq_wake_all`.  Four-step
    behaviour of `wait_with_deadline`.  Mention the global
    deadline list walked by `sched_tick`.  Defensive
    `BLOCKED`-on-runqueue side note.  Full chapter on
    blocking syscalls deferred to later chapters that
    actually call this primitive (the UART RX path, `sys_wait`,
    `sys_ppoll`).
16. **A worked timeline: two kthreads under sched_rr** —
    timestamped narrative from t=0 ms (boot, idle in `wfi`)
    through t=600 ms+ (steady-state rotation A → B → idle
    → A → ...) with ASCII runqueue snapshots at each step.
    Specifically calls out the moment the runqueue rotates
    *to* idle, then forward-references the "wake idle"
    section to explain why idle doesn't run its full quantum
    in practice.
17. **A few extra things to know** (7 side notes) — why
    policy is a component not a `#define` (forward-points to
    chapter 14), the dual-scheduler `kernel.json` build,
    `preempt_count` as a boolean disguised as an integer, no
    SMP locking yet, why `reap_task` exists (zombies on their
    own kernel stacks), why round-robin's `set_priority`
    rejects everything, the WFE cousin.
18. **Where to read more** — six source links (sched.{h,c},
    scheduler.h, both policy .c's, waitq.{h,c}), two
    chapter-internal back-references (chapter 4's "From flag
    to switch", chapter 7's `cpu_switch_to` walk-through),
    two external comparisons (Linux's `__schedule`, seL4's
    bounded-ops fixed-priority), one architecture spec (ARM
    ARM section on WFI).

### `sources/nonux/book/README.md`

Part IV row 8 promoted from `8. The scheduler` (no link, planned)
to `8. [The scheduler](08-the-scheduler.md)` (real link, shipped).
Part IV now complete.

### `proj_docs/nonux/BOOK-OUTLINE.md`

Chapter 8 row updated:
- Status: `planned` → `shipped`.
- "Reader sees" column rewritten from the one-line planning blurb
  (`\`core/sched/\`, runqueue, idle task, preemption, yield, plus
  the two real schedulers...`) to a real summary of what landed:
  the core/policy split, the four core-driver entry points
  (`sched_init` / `sched_start` / `sched_tick` /
  `sched_check_resched` / `nx_task_yield`), `need_resched`,
  `preempt_count`, idle as the renamed boot context, the "wake
  idle" trick on enqueue, the `wfi` branch on
  empty-runqueue/same-task, both policies covered line-by-line,
  wait queues sketched, and the worked two-kthread+idle timeline.

## Key Findings

- **The reschedule shim is the through-line for the chapter.**
  Once `sched_check_resched`'s 12 lines are read end to end,
  every other piece of the scheduler hangs off it cleanly:
  `need_resched` is "what makes the check return true",
  `preempt_count` is "what makes the check return early",
  `pick_next` is "step 7", `yield` is "step 6", the
  `wfi` branch is "step 8", the hook chain is "steps 9–10",
  and `cpu_switch_to` is the chapter-7 finale at step 11.
  Structuring around this single function (rather than around
  the policy ops in interface order) is what kept the chapter
  in the 1100s rather than the 1800s.
- **Round-robin and fixed-priority are almost identical at the
  source level once you know the interface.**  Both are ~300
  lines.  Both have the same `tick` body (a single counter
  decrement that sets `need_resched` on zero).  Both have the
  same wake-idle trick.  The differences are entirely
  contained in `pick_next`, the runqueue shape (one list vs
  eight), and what `set_priority` does.  The book leans on
  this symmetry: fixed-priority is presented as a one-page
  delta from round-robin rather than from scratch.
- **The wait-queue cameo paid off.**  Originally considered
  giving wait queues their own chapter, but the chapter
  structure works better with it as a one-section primer in
  chapter 8 (because the scheduler's `BLOCKED` state and
  `sched_tick`'s deadline walk both need it to be
  recognisable).  The full uses of wait queues — UART RX,
  `sys_wait`, `sys_ppoll` — get their treatment in the
  chapters that actually call the primitive (parts VI + VIII).
- **Forward references kept under control.**  Chapter
  explicitly defers TTBR0 flip + signal delivery + hook chain
  + live-swap to later chapters by *name* but never *links*
  them, because none of those chapters are shipped yet.
  Per [BOOK-STYLE-GUIDE.md §What not to do](../BOOK-STYLE-GUIDE.md#what-not-to-do)
  rule "Don't reference future chapters as if they exist" —
  named in prose, not linked.

## Decisions Made

- **Order both policies by simplicity, not by `kernel.json`
  default.**  Round-robin appears first even though
  `kernel.json` currently binds `sched_priority`.  Rationale:
  round-robin is the smaller, simpler shape, and once the
  reader has it, fixed-priority reads as "the same but with
  eight buckets".  The default-binding question (which
  policy actually runs at boot) is a chapter-14 / kernel.json
  topic — the book is describing the kernel, not the build
  configuration of any particular slice.
- **Pull "the wake idle trick" out as its own section.**
  Originally it was a paragraph inside the round-robin
  section.  Promoting it to a top-level section called it
  out as the kind of latency-floor design detail that's
  easy to miss and load-bearing under interactive workloads.
- **Timeline uses round-robin, not fixed-priority.**  Both
  would work; round-robin is shorter to narrate (you don't
  have to introduce priorities mid-timeline) and the lesson
  (200 ms quantum, rotate, idle visits) is the same shape.
- **Defer the live-swap discussion to chapter 14.**  Chapter
  8 mentions live swap twice (in "How the core driver finds
  the policy" and in "A few extra things to know"), but
  never explains how the swap actually works.  Two reasons:
  (1) live swap depends on pause/drain/resume which is
  chapter 14's material, (2) chapter 14 isn't shipped yet so
  there's no link to point at anyway.
- **No code-listing for `_irq_stub`.**  Chapter 3 already
  walks the IRQ-return vector-table assembly; chapter 7
  already shows `cpu_switch_to`.  Chapter 8 names them and
  cross-links, but does not re-quote the assembly.

## Status at End of Session

- **Book chapters shipped:** 1, 2, 3, 4, 5, 6, 7, 8 (Part I +
  Part II + Part III + Part IV all complete).  Next chapter
  starts Part V (Framework basics) with chapter 9 "Slots,
  components, and the registry".
- **Documentation web updated:** book/README.md row 8 promoted
  to a real link; BOOK-OUTLINE.md row 8 marked `shipped` with
  reader-sees rewritten.
- **Tests:** unchanged from session 137 (which was unchanged
  from session 123).  `make test-tools` **102/102 pass**;
  `make test-host` **485/485 pass**; `make test-interactive`
  **7/7 pass** (last run session 115); `make test-kernel`
  **152/152** (3600 s).  This session is documentation-only;
  no code changed.  Session 136's one-line `mmu_init`
  `l2_mmio_table[0] = 0` change is still waiting for a
  kernel-test re-run on the next code-bearing session.
- **`make verify-iface-fresh`:** 0 drift.
- **`make verify-registry`:** 0 findings (R2, R4, R9).

## Next Steps

- **Either start Phase 9** — per-process MM rework: L3 4 KiB
  pages, VMAs, demand paging, COW fork (per
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework))
  — and bundle a kernel-test re-run to validate session
  136's `mmu_init` change at the same time.  When this
  lands, the chapter-6 revision pass scheduled in session
  128 also becomes available.
- **Or continue the book with chapter 9** — Slots,
  components, and the registry (first chapter of Part V,
  Framework basics).  The natural reading order is
  9 → 10 (IDL) → 11 (Channels and IPC) → 12 (Filesystems).

---

**Files Changed:**
- `sources/nonux/book/08-the-scheduler.md` — new, 1183 lines.  The full chapter 8 draft.
- `sources/nonux/book/README.md` — Part IV row 8 promoted from "no link, planned" to a real link.
- `proj_docs/nonux/BOOK-OUTLINE.md` — chapter 8 row status `planned` → `shipped`, "Reader sees" column rewritten to reflect what actually landed.
