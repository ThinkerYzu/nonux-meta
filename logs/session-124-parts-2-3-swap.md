# Session 124: Chapter 2 polish + Part II/III swap

**Date:** 2026-05-09
**Phase:** Phases 1–9b complete; book chapters 1–2 shipped; outline restructured (Parts II/III swapped); Phase 9 not yet started
**Branch:** master

---

## Goals

- Address remaining gaps the user flagged in chapter 2 after
  session 123's Terms-section amendment had already landed.
- Reconsider the book's Part II / Part III order in light of
  what chapter 2 leaves the reader with.

## What Was Done

### Chapter 2 polish (three follow-up source-repo commits)

Each commit small, contained, separately pushed.

1. **MMIO framing fix** (`sources/nonux` `e607f26`) — the opening
   paragraph of "How software talks to hardware: MMIO" said
   "the CPU and devices share the same address space".  That's
   backwards: the CPU is the thing *issuing* addresses, not a
   participant in the space.  Reworded to "**RAM and devices
   share the same address space**" with a one-line gloss on the
   address-issue/range-respond model.

2. **IRQ properly introduced on first body use**
   (`sources/nonux` `aad8477`) — the IRQ Terms entry hedged with
   "Chapter 1 introduced the term", and the first body
   appearance ("the IRQ-driven ring buffer") wasn't bolded, so a
   reader seeing IRQ for the first time had no inline cue.
   Strengthened the Terms entry to stand on its own (CPU pauses,
   runs handler, resumes) and bolded **IRQ** at first body use
   with a parenthetical inline gloss, matching the syscall /
   kthread / signal pattern from session 123's amendment.

3. **Unintroduced function names** (`sources/nonux` `4d92341`) —
   three families of function names appeared in code blocks
   without prose introducing them: `uart_rd` / `uart_wr` (typed
   MMIO helpers), `nx_console_write` / `nx_console_read_nonblocking`
   (the wrappers `uart_pl011_*` forward to), and
   `console_read_ready_pred` plus `nx_console_{,un}register_pollset`
   (the wait/wake plumbing in `nx_console_read`).  Added one
   short paragraph for each, leaving the deeper waitq/pollset
   mechanics deferred to later chapters.

   Final chapter 2 length: 1163 lines (still within the 600–1500
   band per [BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md#length-and-pace)).

### Part II / Part III swap

**Before:** Part II — Memory (PMM, MMU); Part III — Time and
interrupts (Exceptions/GIC/IRQs, Timer).

**After:** Part II — Time and interrupts (Exceptions/GIC/IRQs,
Timer); Part III — Memory (PMM, MMU).

Reasoning the user agreed with: chapter 2 leaves the reader hot
on the PL011 RX IRQ.  Going straight into "how exceptions and
the GIC actually deliver IRQ 33" (now chapter 3) keeps that
thread live.  Pivoting to PMM (the previous chapter 3) was a
hard topic shift coming off chapter 2's payoff.

Side benefit: the [Phase 9 caveat](../BOOK-OUTLINE.md#phase-9-dependency)
on the MMU chapter now sits later in the queue (chapter 6
instead of chapter 4), so the early book is unblocked while the
MMU chapter can be drafted post-Phase-9.

Tradeoff: mild departure from the standard textbook order
(memory first).  Chapter 7 (threads) still needs stacks, but
the threads chapter sits in Part IV — Memory in Part III still
lands before Part IV, so the dependency is satisfied.

### Outline / index updates

- **`BOOK-OUTLINE.md`**: Part II and Part III table contents
  swapped; chapters 3–6 renumbered (PMM/MMU 3–4 → 5–6,
  Exceptions/Timer 5–6 → 3–4).  Reading-dependencies block
  updated: chapter 15 (Processes) now points at chapter 6 (MMU);
  chapter 16 (Syscalls) now points at chapter 3 (Exceptions/GIC).
  Phase 9 dependency section updated: "Chapter 4" → "Chapter 6".
- **`book/README.md`**: same Part II / III swap in the index.
- Session 122's historical log (the original outline session)
  intentionally left as-is — it records the decision in force at
  *that* time; the swap happened later and is recorded *here*.

## Key Findings

- **Continuity beats convention.**  The standard textbook order
  is memory before interrupts, but for a tutorial-style book
  whose chapter 2 just walked an IRQ-driven console end-to-end,
  the IRQ chapter naturally follows.  Convention isn't worth
  the handoff awkwardness when the chapters before and after
  are the relevant context.
- **Cross-references in the outline are the only lock-in.**  The
  outline's Reading-dependencies block names specific chapter
  numbers ("chapter 4 (MMU)", "chapter 5 (Exceptions/GIC)").
  Once chapters are written and link to each other by number,
  re-numbering would touch many files.  Doing the swap *before*
  chapters 3–6 are drafted means the only files touched are the
  outline + book index + this session log.

## Decisions Made

- **Chapter 2's body now answers "what's this function?" for
  every function name that appears in a code block** — the
  follow-up commits closed that gap.  The deeper mechanics of
  waitq + pollset stay deferred (later chapters own them); but
  the Terms-section / inline-gloss pattern from session 123's
  amendment now extends to function names too.
- **Parts II and III swapped, with the MMU chapter sliding to
  chapter 6.**  No content drafted for any of these yet, so the
  swap is a pure outline change.
- **No content for chapter 1.**  The MMU/PMM forward references
  in chapter 1 (the "Terms you'll see" section mentions PMM and
  MMU) don't carry chapter numbers, so the swap doesn't ripple
  back into chapter 1.

## Status at End of Session

- Chapter 2 polished (three small commits, all pushed).
- Outline restructured (Parts II/III swapped + cross-refs
  updated).  `BOOK-OUTLINE.md`, `book/README.md`, `HANDOFF.md`
  modified — staged for commit.
- No code changes; tests not re-run.  Test counts unchanged
  from session 118 / 122 / 123.

## Next Steps

- Either start Phase 9 (per-process MM rework) or draft
  chapter 3 (now "Exceptions, the GIC, and IRQs") — the latter
  is the natural follow-on to chapter 2.

---

**Files Changed:**

`sources/nonux` (already pushed):
- `book/02-console-and-uart.md` — three polish commits (MMIO
  framing, IRQ intro, function-name introductions); 1140 →
  1163 lines.
- `book/README.md` — Part II / III index swap (this commit).

`proj_docs/nonux` (this commit):
- `BOOK-OUTLINE.md` — Part II / III tables swapped; chapters
  3–6 renumbered; Reading-dependencies + Phase 9 dependency
  cross-refs updated.
- `HANDOFF.md` — Next Actions forward step updated for new
  chapter 3; Latest session log line + Session Logs list
  updated for session 124.
- `HANDOFF-ARCHIVE.md` — session 119 rolled into archive.
- `logs/session-124-parts-2-3-swap.md` — new (this file).
