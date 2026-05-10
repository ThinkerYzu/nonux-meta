# Session 126: Book chapter 4 — The timer and ticks

**Date:** 2026-05-09
**Phase:** Phases 1–9b complete; book chapters 1–4 shipped (Part I + Part II complete); Phase 9 not yet started
**Branch:** master

---

## Goals

- Draft the next book chapter, picking up where chapter 3 left
  off (IRQ machinery is in hand; what is the *first* IRQ a kernel
  arranges for itself?).
- Per [BOOK-OUTLINE.md](../BOOK-OUTLINE.md#chapter-list), that's
  **chapter 4: The timer and ticks** — the second and final
  chapter of Part II (Time and interrupts).

## What Was Done

### `sources/nonux/book/04-timer-and-ticks.md` (new, 760 lines)

Walks the path from "what is the kernel measuring time *with*"
to "how does a tick become a context switch", reusing the IRQ
pipeline from chapter 3 as the substrate.

Section structure:

1. **Intro** (3 paragraphs) — picks up from chapter 3, names the
   kernel's need for an unprompted heartbeat (preemption,
   timeouts, bookkeeping), states the chapter's promise.
2. **Files** — `core/timer/timer.{h,c}`, `core/sched/sched.c`
   (only the slice this chapter needs), `core/cpu/vectors.S`
   (already covered in ch.3, referenced here), `core/boot/boot.c`.
3. **Terms** — 11 entries: timer tick, ARM Generic Timer, EL1
   physical timer, `CNTFRQ_EL0`, `CNTP_TVAL_EL0`,
   `CNTP_CTL_EL0`, PPI 30, preemption, time slice/quantum,
   `need_resched`, drift, recomposition.
4. **Why a kernel needs a steady heartbeat** — what dies
   without a tick: preemption, timeouts, bookkeeping. Side
   note on tick-rate trade-offs (Linux 100/250/300/1000;
   nonux 10).
5. **The ARM Generic Timer** — what's in it (free-running
   counter + countdown), the EL1 physical timer specifically,
   the three system registers we touch, the math
   (62.5 MHz / 10 Hz = 6 250 000 down-counter reload). Side
   note on aperiodic / "tickless" timers — possible because
   we manually rearm.
6. **`timer_init`** walked line-by-line: read `CNTFRQ`,
   compute interval, register IRQ in the chapter-3 table,
   `gic_enable`, write `TVAL`, write `CTL=1`. Subsection
   "Why the timer is a PPI" closes chapter 3's forward
   reference.
7. **The tick: what `on_tick` actually does** — the three
   responsibilities in order: (1) rearm first to avoid drift
   (with arithmetic showing the cumulative cost of a late
   rearm), (2) bump `g_ticks` (atomic / relaxed rationale),
   (3) `sched_tick` → policy `tick` op → `sched_rr_tick`
   walked in detail (decrement quantum, set `need_resched`
   on expiry).
8. **From flag to switch** — the IRQ-stub `bl
   sched_check_resched` from chapter 3, the simplified
   `sched_check_resched` body, the chain of guards
   (`need_resched`? `preempt_count`?), `cpu_switch_to`
   forward-pointed to the threads chapter. Big ASCII
   diagram showing the full timer-driven preemption pipeline
   end-to-end.
9. **`timer_ticks`** — monotonic ticks-since-boot, 60 billion
   years to overflow, used for deadline computations + waitq
   timeouts (`nx_waitq_tick_deadlines` glossed). Resolution
   note (one tick = 100 ms; `CNTPCT_EL0` for finer).
10. **`timer_pause` / `timer_resume`** — what they do (mask
    the PPI at the GIC), why they exist (recomposition's
    scheduler-pointer-update window), the nesting +
    saturate-at-zero patterns. Forward-pointed (no chapter
    number) to a future recomposition chapter.
11. **A few extra things** — six side notes: per-CPU timers,
    firmware-set frequency, signed 32-bit `TVAL`,
    level-triggered IRQ (rearm clears the line),
    monotonic-vs-wall-clock, ARMv7-A's pre-existing Generic
    Timer.
12. **Where to read more** — source links + chapter 3
    cross-refs to `gic_ack`/`gic_eoi` and the IRQ pipeline,
    plus ARM ARM, QEMU `virt`, Linux booting docs.

Length: 760 lines — shorter than chapter 3 (1211) because the
timer is conceptually simpler. Comfortably in the 600–1500
band.

### Chapter 3 cross-link patches (minimal)

Two small edits on `book/03-exceptions-gic-and-irqs.md` to
replace its "later chapter" placeholders with real links to
chapter 4 (the PPI subsection's "covered in a later chapter"
and the boot-order code-comment).

### Index / outline updates

- **`book/README.md`** — chapter 4 entry under Part II changed
  from plain text to `[The timer and ticks](04-timer-and-ticks.md)`.
- **`BOOK-OUTLINE.md`** — chapter 4 status `planned → shipped`,
  "Reader sees" column expanded.

## Key Findings

- **The chapter writes itself once chapters 2 and 3 exist.** The
  timer is a small driver (~80 lines) but its end-to-end story
  reaches into the GIC (chapter 3), the IRQ table (chapter 3),
  the IRQ-return shim (chapter 3), and the scheduler (later
  chapter). Without those scaffolds in place, the chapter would
  have to pre-explain three different topics. With them, the
  chapter can stay focused on what's specific to the timer:
  Generic-Timer system registers, the rearm-first rule, the
  tick→quantum→`need_resched` chain.
- **Scheduler scope had to be carefully bounded.** The chapter
  shows the slice of `sched_tick` and `sched_check_resched`
  the timer touches, but defers `cpu_switch_to`, runqueues,
  policy details, and `pick_next` to the threads + scheduler
  chapters. The pattern: state what each function *receives*
  and *produces*, name the `cpu_switch_to` call as the
  context-switch primitive without unpacking it, leave the
  reader confident about *flow* without claiming completeness
  on *mechanics*.
- **Drift gets a real arithmetic example.** The "rearm first"
  rule could be stated as a one-liner ("avoids drift") and
  left there. Spelling out the math (5 µs × 10 ticks/sec ×
  3600 sec = 0.18 s/hour) gives the reader something
  measurable to anchor the rule against. Same pattern as
  chapter 3's "32 instructions per vector entry" arithmetic.

## Decisions Made

- **Cover only the EL1 physical timer.** ARM's Generic Timer
  family includes virtual, hypervisor, and secure timers as
  well; nonux uses one. A side note acknowledges the others
  exist but the chapter doesn't enumerate them.
- **No chapter-number reference for the future "recomposition"
  chapter.** Per
  [BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md#what-not-to-do):
  unwritten chapters are referenced as "a later chapter" or
  "a later chapter on runtime composition", never by number.
  When chapter 14 lands, this can be patched up minimally.
- **No scheduler chapter-number reference either.** Same
  reason — the threads chapter (planned 7) and scheduler
  chapter (planned 8) are not shipped yet. Both are referenced
  by topic, not number.
- **Keep the three timer system-register helpers inlined in the
  walkthrough.** They're three single-line wrappers; reading
  them once saves the reader from cross-referencing on every
  later code block.
- **No content changes to chapters 1 or 2.** Their forward
  references didn't reach into timer/scheduler topics.

## Status at End of Session

- `book/04-timer-and-ticks.md` drafted, 760 lines, staged in
  `sources/nonux`.
- `book/README.md` chapter-4 link wired up, staged.
- `book/03-exceptions-gic-and-irqs.md` two minimal cross-link
  patches, staged.
- `BOOK-OUTLINE.md` chapter-4 row updated `planned → shipped`,
  staged in `proj_docs/nonux`.
- `HANDOFF.md` Current Status, Latest session log, Next
  Actions, Session Logs list — to be updated in this commit.
- `HANDOFF-ARCHIVE.md` — session 121 to be rolled into archive
  per the keep-last-5 convention.
- No code changes. Tests not re-run; test counts unchanged
  from session 125.

**Part II complete** — the book has a contiguous Part I + Part II
(four chapters: Boot and linker, Console and UART, Exceptions/GIC
/IRQs, Timer and ticks). Next chapter is Part III (Memory),
chapter 5 (PMM).

## Next Steps

- Either start Phase 9 (per-process MM rework — L3 4 KiB pages,
  VMAs, demand paging, COW fork; see
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework))
  or continue the book with chapter 5 (Physical memory and the
  page allocator), the first chapter of Part III.

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/04-timer-and-ticks.md` — new, 760 lines.
- `book/README.md` — chapter-4 entry now a real link.
- `book/03-exceptions-gic-and-irqs.md` — two minimal cross-link
  patches (forward references to chapter 4 made concrete).

`proj_docs/nonux` (this commit):
- `BOOK-OUTLINE.md` — chapter 4 status `planned → shipped`,
  "Reader sees" column expanded.
- `HANDOFF.md` — Current Status, Latest session log line, Next
  Actions, Session Logs list updated.
- `HANDOFF-ARCHIVE.md` — session 121 rolled into archive.
- `logs/session-126-book-chapter-4.md` — new (this file).
