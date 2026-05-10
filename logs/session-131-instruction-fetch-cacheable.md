# Session 131: Book chapter 6 — instruction-fetch nuance in §"Why turn on the MMU at all"

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–6 shipped (Part I + Part II + Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Tighten the chapter-6 §"Why turn on the MMU at all" bullet
  point on caching and memory ordering, surfaced during the
  web-search validation pass on the Session 130 side note.

## What Was Done

### Background — what the validation pass found

When validating Session 130's rewritten side note against ARM
documentation, the secondary search confirmed an
ARM-architecture detail that the parent bullet point had
glossed over.

The chapter's bullet read:

> **Caching and memory ordering.** When the MMU is off, ARM
> treats *all* of RAM as Device-nGnRnE memory: no caches,
> strict ordering, every load and store goes to the bus.

Per ARM's *Describing memory in AArch64* and *VMSA behavior
when a stage 1 MMU is disabled* (DDI0406C), with MMU off:

- **Data accesses** are Device-nGnRnE.  ✓ The chapter's
  "every load and store goes to the bus" matches this for
  data.
- **Instruction fetches** are architecturally treated as
  *cacheable*.  ✗ The chapter's "*all* of RAM as Device"
  / "no caches" framing was too strong.

In nonux this nuance is *practically* hidden — `SCTLR.I` is
also clear until `mmu_init` enables M+C+I together, so
I-cache fills don't happen — but the architectural claim was
sloppier than necessary.

### Bullet rewrite

The bullet now says:

> **Caching and memory ordering.** When the MMU is off, the
> architecture treats every data access as Device-nGnRnE: no
> caches, strict ordering, every data load and store goes to
> the bus.  (Instruction fetches are architecturally
> *cacheable* in this state, but in practice the I-cache is
> also disabled until `mmu_init` turns it on alongside
> `SCTLR.M` and `SCTLR.C`, so nothing actually fills.)
> That's catastrophically slow.  Turning the MMU on lets us
> tag RAM as **Normal** memory — cacheable, write-back, free
> to be reordered for performance — while keeping MMIO
> regions (the GIC, the PL011) tagged as **Device** so writes
> to them stay strictly ordered.  This is the *single*
> biggest reason real kernels turn the MMU on as early as
> possible.

The conclusion (caching is the biggest reason to turn on the
MMU early) is unchanged.  The first sentence is now precise
about *data* accesses; the parenthetical surfaces the
architectural exception (instruction fetches are cacheable)
and immediately notes why it doesn't matter in nonux's
specific configuration.

## Key Findings

- **The Session 130 web-search validation pass was high-yield.**
  Validating one side note's claims against ARM documentation
  surfaced an unrelated minor inaccuracy in the same section.
  The validation methodology is therefore worth applying to
  other "we explain X" passages in shipped chapters — broken
  links, wrong section numbers, and overshoots like this one
  are exactly what a search-the-architecture pass catches.
- **The "I-cache architecturally cacheable with MMU off" rule
  is more of a footnote than a load-bearing fact for nonux.**
  Nothing in nonux relies on the I-cache being filled before
  `mmu_init`, and `SCTLR.I` is clear at the relevant point
  anyway.  The fix is precision in the prose, not a behaviour
  change.

## Decisions Made

- **Rewrite as a parenthetical inside the bullet, not a
  separate side note.** A new side note for one architectural
  footnote would be heavier than necessary; a parenthetical
  keeps the reader oriented in the bullet's main argument
  (caching is the biggest reason).
- **Surface the practical-vs-architectural distinction.**
  Saying "architecturally cacheable but in practice the
  I-cache is also disabled" gives the reader the right model:
  the architecture leaves a door open; nonux closes it
  immediately.

## Status at End of Session

- `book/06-virtual-memory-and-mmu.md` patched in §"Why turn
  on the MMU at all": one bullet rewritten (~10 lines → ~13
  lines).
- No source-code changes; tests not re-run.

## Next Steps

- Same forward fork as sessions 128–130: continue the book
  with chapter 7 (Kernel threads and context switching, Part
  IV) or start Phase 9 (per-process MM rework).

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/06-virtual-memory-and-mmu.md` — §"Why turn on the
  MMU at all" caching/memory-ordering bullet rewritten:
  scopes the "MMU off → Device-nGnRnE" claim to data
  accesses, adds a parenthetical noting that instruction
  fetches are architecturally cacheable but in practice the
  I-cache stays disabled until `mmu_init`.

`proj_docs/nonux` (this commit):
- `HANDOFF.md` — Latest session log line, Session Logs list,
  Next Actions updated.
- `logs/session-131-instruction-fetch-cacheable.md` — new
  (this file).
