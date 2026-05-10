# Session 137: Book chapter 7 — Kernel threads and context switching

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–7 shipped (Part I + Part II + Part III + first chapter of Part IV); Phase 9 not yet started
**Branch:** master

---

## Goals

- Draft the next book chapter, the first one of Part IV (Tasks).
  Per [BOOK-OUTLINE.md](../BOOK-OUTLINE.md#chapter-list), that's
  **chapter 7: Kernel threads and context switching** — the
  thread-and-cpu-switch half of the task layer, with the
  scheduler proper deferred to chapter 8.
- Frame the chapter as the answer to chapter 4's loose end
  ("who calls `cpu_switch_to`?") without spilling into the
  scheduler policy chapter.

## What Was Done

### `sources/nonux/book/07-kernel-threads-and-context-switch.md` (new, 1304 lines)

Walks the path from "what is a task" through the saved-context
layout, the ARM64 context switch assembly, the
`TPIDR_EL1` per-CPU identity register, task creation and the
first-switch thunk, the public `sched_spawn_kthread` API, the
idle task as a special case (the boot context retroactively
adopted), and the two paths into `cpu_switch_to` (involuntary
IRQ-return preemption and voluntary yield), closing with a
worked two-task example.

Section structure:

1. **Intro** (4 paragraphs) — picks up chapter 4's dangling
   `cpu_switch_to` reference, names the chapter's main claim
   (mechanism only — policy is chapter 8), promises the reader
   will know what a task is, what its kstack looks like, what
   a context switch saves, and where the "current task"
   identity actually lives on the CPU.
2. **Files** — `core/sched/task.{h,c}`, `core/cpu/context.S`,
   `core/sched/sched.c`, `framework/dispatcher.c` (one real
   in-tree kthread), `core/boot/boot.c` (idle adoption + init
   kthread).
3. **Terms** — 14 entries: task, kthread, kstack, context,
   context switch, callee-saved register, caller-saved
   register, AAPCS, TPIDR_EL1, TPIDR_EL0 (one-line mention,
   full story deferred), first-switch thunk, need_resched,
   preempt_count, voluntary yield, involuntary preemption.
4. **What is a task?** — plain-English framing: one CPU, one
   thread of execution, "tasks" are pause-states the kernel
   swaps in and out. Sets up the rest of the chapter as
   "make that swap correct, fast, safe".
5. **`struct nx_task`** — trimmed definition with a field-by-
   field reading; flags the offset-0 invariant for `cpu_ctx`
   without explaining yet *why* (the why lands in the next
   section). Side note on task/thread/process naming
   conventions across systems.
6. **The saved context: `struct nx_cpu_ctx`** — eight slots
   total. Five sub-sections, one per design decision: why
   only callee-saved, why SP, why DAIF (with the bug class
   it prevents — voluntarily yielding with IRQs enabled and
   coming back with them masked), why no FP/SIMD
   (`-mgeneral-regs-only`), why 16-byte aligned. Side note on
   lazy-FP as the real-world-kernel alternative.
7. **`cpu_switch_to`: the assembly** — the whole 22-instruction
   ARM64 routine, then five sub-sections: the arguments
   (AAPCS x0/x1), why cpu_ctx is at offset zero (with the
   `_Static_assert` quoted), the save half (stp pairs +
   mrs daif), the load half (ldp + msr daif), publishing the
   new identity via `msr tpidr_el1, x1`, and the `ret`'s
   two-case behavior (resume into C vs. branch to bootstrap
   thunk for first-time entry).
8. **`TPIDR_EL1`: who am I?** — `nx_task_current()` quoted in
   full (3 lines), then two reasons for register-rather-than-
   global: per-CPU for free on SMP, and `mrs` is cheap. Lists
   the two places that *write* `TPIDR_EL1` (`cpu_switch_to`
   and `sched_start`).
9. **Creating a task: `nx_task_create`** — the cpu_ctx
   fabrication block walked in three steps: pick a starting
   stack pointer (sp_top, 16-aligned), stash entry+arg in
   x19/x20 (with the clear explanation of why not x0 — the
   compiler-saved-restore distinction), set x30 = bootstrap.
   Notes x29=0 as the AAPCS frame-bottom sentinel and daif=0
   for IRQs-enabled-on-first-run.
10. **The first-switch thunk: `nx_task_bootstrap`** — the
    whole 4-instruction stub, the role (register-shuffle
    adapter between "what cpu_switch_to delivers" and "what
    AAPCS expects at a call site"), then a 20-line ASCII
    diagram showing the first ten events in a brand new
    kthread's life, from `nx_task_create` through the
    bootstrap to `entry(arg)`. Closes with "the thunk is
    invisible after the first switch" — every subsequent
    save records `x30` as a return-into-C address.
11. **`sched_spawn_kthread`: the public door** — the 3-step
    body (create + set process + enqueue), with pointers to
    the three real callers: `framework/dispatcher.c` (the IPC
    pump), `core/boot/boot.c` (the busybox-init runner), and
    the kernel test suite.
12. **The idle task: a switch *out of*, never *into* first** —
    `g_idle_task` is a static global with no heap struct and
    no allocated kstack. `sched_start` adopts the boot
    context as idle by setting `TPIDR_EL1 = &g_idle_task`
    and enqueueing it. The first context switch the kernel
    ever performs goes `idle → some_kthread`; idle's cpu_ctx
    is first populated at that save half.
13. **Voluntary and involuntary switches** — re-closes
    chapter 4's loose end with the full `sched_check_resched`
    body quoted (signal-delivery elided, with a pointer
    forward). Three sub-sections: IRQ-return path,
    voluntary yield, `preempt_count` as a counter not a flag
    (with `nx_preempt_disable`/`_enable` quoted).
14. **A worked example: two tasks taking turns** — 35-line
    ASCII timeline showing two kthreads A/B + idle on a 10 Hz
    tick, four switches in 300 ms. Demonstrates how the
    bootstrap thunk runs once per task and how subsequent
    resumes go back into the task's own C code.
15. **A few extra things to know** — 8 bullets:
    - per-task kstack vs per-CPU kstack tradeoff (memory vs
      blocking-safe), with Linux x86 fixmap as the
      per-CPU example;
    - one page is small (deliberate — overflow becomes a
      fault we can investigate);
    - `cpu_switch_to(curr, curr)` is wasteful → shim's
      `wfi` guard;
    - trap frame vs cpu_ctx are different things (different
      sizes, different scopes, both can coexist on the same
      task's kstack);
    - the offset-zero invariant is load-bearing — the
      `_Static_assert` catches the easy regression;
    - host tests stub the assembly (with the `abort()` body
      quoted);
    - "no thread of execution before sched_start" — every
      function that calls `nx_task_current` must handle a
      NULL return;
    - the DAIF save/restore prevents a specific bug class
      (the comment in `task.h` is what kept it fixed).
16. **Where to read more** — 7 links: `task.h`, `task.c`,
    `context.S`, `sched.c`, chapter 4 §"From flag to switch",
    chapter 3 §"The IRQ pipeline", ARM ARM §"AArch64 System
    Register Descriptions" + AAPCS doc, and Linux's
    `arch/arm64/kernel/entry.S` as a "compare against" link.

Style: every standard term bolded on first use with an inline
plain-English definition (per the style guide). No invented
terms — followed the guide's "use the industry name" rule,
e.g. "kernel stack" not "scratchpad", "context switch" not
"swap-in-swap-out", "callee-saved register" not "preserved
register". Sentence length aimed at 15–20 words on average,
8th-grade vocabulary. One whole-chapter framing analogy (none
used; the topic doesn't lend itself to one — just plain
technical English).

### Doc updates

- **`sources/nonux/book/README.md`** — Part IV row for
  chapter 7 promoted from "Kernel threads and context
  switching" (planned, no link) to a real link
  `[Kernel threads and context switching](07-kernel-threads-and-context-switch.md)`.
- **`proj_docs/nonux/BOOK-OUTLINE.md`** — chapter 7 row in
  the chapter list: status `planned → shipped`, "Reader
  sees" line expanded to reflect the actual scope
  (`cpu_switch_to` instead of the placeholder
  `nx_task_switch_to`, plus first-switch thunk,
  preempt_count vs need_resched, idle-as-special-case).

## Key Findings

- **The chapter 4 / chapter 7 handoff worked cleanly.** Chapter 4
  established that `sched_tick` sets `need_resched` and that
  *somebody* later consumes the flag. Chapter 7 picks up
  exactly there with `sched_check_resched` and `cpu_switch_to`.
  No re-grounding needed — the reader already knows about the
  timer ISR, the IRQ-return shim, `_irq_stub`, DAIF.I — and
  chapter 7 leans on all of those without re-introducing them.
  Good evidence that the part ordering (Time/IRQs → Memory →
  Tasks) holds together for the reader's threading.
- **Splitting "tasks" and "scheduler policy" across chapters 7
  and 8 is the right cut.** The mechanism (context switch +
  TPIDR_EL1 + kthread spawn) is one focused topic. The policy
  (runqueue, idle as fallback, RR vs priority, the
  `g_sched_ops` indirection through the scheduler component)
  is another. Trying to do both in one chapter would either
  overshoot 2000 lines or hand-wave one of them. As it stands,
  chapter 7 mentions `g_sched_ops->enqueue` and
  `g_sched_ops->pick_next` by name without explaining what
  drives them — chapter 8 will own that.
- **The "first switch is into the bootstrap thunk, all later
  switches are into C" distinction needed an ASCII diagram.**
  It's a subtle invariant that's hard to convey in prose
  alone. The two diagrams (single-task lifecycle + two-task
  timeline) carry it without any clever phrasing required.
- **DAIF save/restore is more interesting in print than I
  expected.** The bug class it prevents (voluntarily yield
  with IRQs on, come back with them masked, kernel hangs) is a
  good worked example for "why does the saved context include
  what it includes". Made it a section rather than a side
  note.

## Decisions Made

- **Defer the full TPIDR_EL0 story to a later chapter.**
  Chapter 7 mentions `TPIDR_EL0` once in the Terms list (as
  "TPIDR_EL1's userspace sibling, per-CPU, saved/restored on
  switch, used by EL0 libcs to point at TLS") but does not
  walk the save/restore code in `sched_check_resched` or the
  per-task `tpidr_el0` field. Rationale: TPIDR_EL0 is a
  userspace-TLS feature; without an EL0 context already
  established, the value is opaque. The natural home is
  chapter 15 (Processes and the user/kernel boundary) or
  chapter 17 (POSIX shim, libnxlibc, busybox). Marked
  internally as a forward link target; no explicit
  `[chapter 15](...)` cross-ref written yet because
  chapter 15 doesn't exist.
- **Defer the TTBR0 flip during context switch.** Same
  reasoning as TPIDR_EL0 — TTBR0 is a per-process MMU root,
  which is meaningless until the reader has the process layer
  in mind. The `sched_check_resched` quote in §"Voluntary and
  involuntary switches" elides the TTBR0 block with `…
  context-switch hook, TTBR0 flip, TPIDR_EL0 swap; later
  chapters …` and moves on.
- **Defer the context-switch hook (`NX_HOOK_CONTEXT_SWITCH`).**
  Hooks get their own chapter (13). Mentioning them inline
  would force a digression about the hook framework that
  doesn't belong here.
- **Keep the worked two-task example.** It's the longest
  diagram in the chapter (~35 lines) but it ties every concept
  together — bootstrap-thunk-on-first-switch, every-subsequent-
  switch-is-resume-into-C, DAIF differences between the two
  entry paths, the role of `pick_next`. Worth the lines.
- **Choose chapter 7 over starting Phase 9.** Both were
  options at session start per HANDOFF.md's "Next Actions";
  picked the book continuation because (a) the kernel-test
  re-run from session 136 is still pending, so any new
  kernel change should bundle that re-run, (b) Phase 9 is a
  multi-session effort (L3 + VMAs + demand paging + COW fork)
  that's better started with a fresh agenda, and (c) shipping
  chapter 7 unblocks the rest of Part IV.

## Status at End of Session

- **What's working:** Chapter 7 drafted at 1304 lines (inside
  the 600–1500 line sweet spot). `book/README.md` and
  `BOOK-OUTLINE.md` updated. No code changes.
- **What's not yet working:** Chapter 8 (The scheduler) — the
  natural next chapter, the policy/runqueue/idle-fallback
  half of Part IV.
- **Tests:** unchanged from session 136 — `make test-tools`
  **102/102 pass**; `make test-host` **485/485 pass**;
  `make test-interactive` **7/7 pass** (last run Session 115);
  `make test-kernel` **152/152** (3600 s, last full run before
  session 136's `mmu_init` one-liner). Documentation-only
  session; no code changed so no tests re-run.
- `make verify-iface-fresh`: 0 drift (unchanged).
- `make verify-registry`: 0 findings (R2,R4,R9; unchanged).

## Next Steps

- **Either** start Phase 9 (per-process MM rework — L3
  4 KiB pages, VMAs, demand paging, COW fork; see
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework)),
  **or** continue with chapter 8 (The scheduler).
- The Session 136 `mmu_init` change still wants a kernel-test
  re-run on the next code-bearing session.

---

**Files Changed:**

- `sources/nonux/book/07-kernel-threads-and-context-switch.md` (new) — 1304-line chapter draft
- `sources/nonux/book/README.md` — Part IV row for chapter 7 promoted from planned (no link) to a real link
- `proj_docs/nonux/BOOK-OUTLINE.md` — chapter 7 row marked `shipped`, "Reader sees" expanded
- `proj_docs/nonux/HANDOFF.md` — Current Status + Session Logs + Latest Session updated to point at this session
- `proj_docs/nonux/logs/session-137-book-chapter-7.md` (new) — this log
