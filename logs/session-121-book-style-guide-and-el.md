# Session 121: Book style guide + EL0/EL1/EL2 section

**Date:** 2026-05-09
**Phase:** Phases 1–9b complete; post-9b fixes through Session 119; Phase 9 not yet started
**Branch:** master

---

## Goals

- Capture the style/terminology rules that emerged during Session
  120 (chapter 1) into a permanent guide so future chapters stay
  consistent.
- Extend chapter 1 with a dedicated section on ARM64 exception
  levels (EL0–EL3), which the chapter previously assumed without
  defining.

## What Was Done

### `BOOK-STYLE-GUIDE.md` added

New top-level document in `proj_docs/nonux/` (peer to SPEC,
DESIGN, IMPLEMENTATION-GUIDE, etc.).  368 lines covering:

- **Audience.** Comfortable with C, never looked inside a kernel,
  possibly non-native English readers.  8th-grade prose target on
  graduate-level concepts.
- **Voice and tone.** Plain English, short sentences (~15–20
  words), active voice, no filler, no emoji.
- **Terminology rules** *(strictest part of the guide)*: use
  standard terms not invented metaphors, with a "Use this / Not
  this" table codifying the Session 120 corrections (ELF header
  not "packing list"; raw binary not "dumped on the floor"; boot
  ROM not "sticky note"; interrupt not "tap on the shoulder"; CPU
  not "the brain"; bootloader not "morning shift program").
  Explain on first use, define once per chapter, maintain a Terms
  section.
- **When analogies work — and when they don't.**  OK: standard
  textbook analogies (pile-of-plates stack), illustrative
  comparisons (house number for address), one whole-chapter
  framing analogy.  Not OK: invented terms, untrue analogies (the
  Session 120 freezer incident is cited as the canonical
  fact-check reminder), strained ones.
- **Formatting conventions.** Bold for first use, monospace for
  code/paths/registers/addresses, code-block guidance, tables,
  block quotes, ASCII vs real diagrams, relative-link rule.
- **Chapter structure.** Skeleton: intro → optional analogy →
  Terms → optional background → main content → optional extras →
  "Where to read more".  Length 600–1500 lines.
- **Chapter naming and numbering.** `NN-topic-slug.md`; numbers
  are reading order, not creation order.
- **Workflow.** Outline → identify standard terms → write → read
  aloud → fact-check → update TOC → update cross-refs.
- **What not to do.** Patterns already tried and rejected
  (metaphor pile-on, over-explanation, dead forward references,
  buried lede, inside-baseball prose).

`proj_docs/nonux/README.md` updated with a new "Authoring"
subsection in the doc index and a "Book Style Guide" entry in
Quick Links.

### Chapter 1 — new "Privilege levels: EL0, EL1, EL2, and EL3" section

Inserted between "How QEMU hands the kernel a 'ready' machine"
and "What the CPU is doing the moment our code starts" — so the
reader has the privilege-model vocabulary fresh before reading
about the EL2→EL1 drop in `start.S`.  Six subsections:

1. **Why CPUs have privilege levels at all** — motivation:
   dangerous instructions need to be gated.
2. **What each EL is for** — permissions table (EL0/EL1/EL2/EL3)
   with the "higher = more powerful" rule.
3. **Why four levels?** — comparison to x86's two rings, then
   each ARM EL motivated (TrustZone, virtualization, kernel,
   user).
4. **Where nonux fits** — "nonux runs at EL1"; explains why QEMU
   hands us EL2 and references `start.S`.  Mentions the
   `-machine virtualization=off` alternative.
5. **How transitions between ELs happen** — `eret` for
   higher-to-lower (voluntary), exceptions for lower-to-higher
   (involuntary).  Explicit security-asymmetry callout.
6. **What this means for nonux in practice** — three
   consequences: `start.S` drops EL, privileged instructions are
   EL1-only, processes run at EL0 with syscall round-trip.

The downstream "Two things to notice" bullet about EL2/EL1 was
trimmed — six lines re-explaining `eret` collapsed to a one-line
back-reference, since the new section now covers it upstream.

Chapter length: 859 → 968 lines.

## Key Findings

- **Style rules are easier to enforce when written down.**  The
  Session 120 conversations re-derived the same rules (standard
  terms, no invented metaphors, fact-check analogies) several
  times.  Codifying them in `BOOK-STYLE-GUIDE.md` means the
  rationale is preserved and future chapters won't have to relearn
  the lessons.
- **"Use this / Not this" tables work.**  The terminology table
  in the style guide is more useful than prose — it shows the
  rejected metaphor next to the accepted standard term, so future
  authors can see at a glance what *not* to write.
- **The freezer incident is a useful canonical example.**  It's
  cited in the style guide as the fact-check reminder; concrete
  past mistakes are stickier than abstract advice.

## Decisions Made

- **Style guide goes in `proj_docs/nonux/`, not in the source
  repo.**  Reasoning: it's an authoring rule for the book, not
  part of the kernel.  Lives peer to SPEC/DESIGN/etc.  The book
  itself stays in the source repo.
- **EL section placement: just before `start.S` is described.**
  Putting it earlier (e.g., right after the Terms section) would
  separate the conceptual model from where it's first used; the
  current placement minimizes the gap between "EL section" and
  "start.S drops from EL2 to EL1".

## Status at End of Session

- Style guide written and indexed.
- Chapter 1 has a dedicated EL section; downstream prose trimmed
  to remove the now-redundant EL2/EL1 inline explanation.
- No code changes; tests not re-run.  Test counts unchanged from
  Session 118: 102/102 tools · 485/485 host · 152/152 kernel.

## Next Steps

- Phase 9 — per-process MM rework (L3 4 KiB pages, VMAs, demand
  paging, COW fork).  See
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework).
- Future book chapters per the style guide.

---

**Files Changed:**

`proj_docs/nonux`:
- `BOOK-STYLE-GUIDE.md` — new (368 lines).
- `README.md` — Authoring subsection added in doc index; Book
  Style Guide entry in Quick Links.

`sources/nonux`:
- `book/01-boot-and-linker.md` — new "Privilege levels: EL0,
  EL1, EL2, and EL3" section (~110 lines); downstream "Two things
  to notice" bullet trimmed.  859 → 968 lines.
