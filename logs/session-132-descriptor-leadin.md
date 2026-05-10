# Session 132: Book chapter 6 — descriptor section lead-in fix

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–6 shipped (Part I + Part II + Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Fix a meaningless lead-in sentence in §"Descriptors: the
  bit-level layout" surfaced by the user asking what "top
  address part of the entry" was supposed to mean.

## What Was Done

### Background — what the user caught

The lead-in into the descriptor table read:

> A descriptor is one 64-bit value. Most of the bits at the
> "top" address part of the entry are the same across L1, L2,
> and L3.

The user asked what "top address part" meant.  The honest
answer is that it doesn't refer to anything architecturally
real — there's no "top address" concept in ARM stage-1
descriptors.  The phrasing was a leftover from an earlier
draft where the lead-in tried to say "the high-order bits
encode the address; the low-order bits encode permissions and
attributes," and didn't get rewritten when session 129
replaced the broken ASCII descriptor diagram with a markdown
table.

### Lead-in rewrite

Replaced with:

> A descriptor is one 64-bit value.  The bit layout is the
> same across L1, L2, and L3 except that bit 1 tells the MMU
> "this is a table descriptor; keep walking" (`Tbl` = 1 at
> L1/L2) versus "this is a block / page descriptor; the walk
> ends here" (`Blk` = 0 at L1/L2; bit 1 must be 1 for valid
> page descriptors at L3).  The fields nonux uses, ordered
> from low bit to high:

The new lead-in:

- Drops the meaningless "top address part" phrase.
- Introduces the one *real* level-dependent difference (bit 1's
  meaning) before the table that lists fields shared across
  levels.
- Cleanly hands off to the table.

The bullet list of bit-by-bit detail that follows the table
already explains `Tbl/Blk` at line 290–293, so this lead-in
preview doesn't introduce anything that isn't elaborated below.

## Key Findings

- **Multi-pass edits leave artifacts.** Session 128 wrote the
  original lead-in to introduce a (broken) ASCII descriptor
  diagram; session 129 replaced the diagram with a markdown
  table but didn't rewrite the lead-in.  The leftover phrase
  ("top address part") was never re-grounded in the new
  structure.  A useful sweep on any future doc edit:
  re-read the *introductions* of any sections whose bodies
  changed.
- **Reader questions surface editing artifacts that
  proofreaders miss.** "What is X?" for any X in the prose
  is a fast way to find leftovers — if the answer is "it's
  nothing real, just a phrase I left in," the prose has
  drifted from what it's trying to say.

## Decisions Made

- **Use the lead-in to introduce the one level-dependent
  difference.** The table that follows lists fields shared
  across all three levels.  Naming the *exception* in the
  lead-in (bit 1 changing role between table and block / page
  descriptors) makes the table's "this is what every level
  has" framing make sense, without needing a second
  level-by-level comparison further down.
- **Don't restructure further.** The descriptor section
  already has the table + the bullet-list elaboration + the
  three descriptor-builder helpers in `mmu.c`.  The
  level-dependent bit 1 detail belongs in the lead-in to set
  up the rest, not in a separate sub-section.

## Status at End of Session

- `book/06-virtual-memory-and-mmu.md` patched in
  §"Descriptors: the bit-level layout" — three-line lead-in
  rewritten (~3 lines → ~7 lines).
- No source-code changes; tests not re-run.

## Next Steps

- Same forward fork as sessions 128–131: continue the book
  with chapter 7 (Kernel threads and context switching, Part
  IV) or start Phase 9 (per-process MM rework).

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/06-virtual-memory-and-mmu.md` — §"Descriptors: the
  bit-level layout" lead-in rewritten.  Removed the leftover
  "top address part of the entry" phrase that didn't refer
  to anything architecturally real; replaced with a sentence
  that names the one level-dependent difference (bit 1's
  meaning) before introducing the table of shared fields.

`proj_docs/nonux` (this commit):
- `HANDOFF.md` — Latest session log line, Session Logs list,
  Next Actions updated.
- `logs/session-132-descriptor-leadin.md` — new (this file).
