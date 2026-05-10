# Session 134: Book chapter 6 — "no L3 tables" side note in §"The tables themselves"

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–6 shipped (Part I + Part II + Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Surface the "nonux doesn't currently use L3 tables" fact at
  the *table-construction* point in chapter 6, where a reader
  watching the tables get built is most likely to wonder.

## What Was Done

### Background — what the user caught

While reading chapter 6 the user asked: *"nonux doesn't have
L3 table?"* — referring to the descriptor-section lead-in
(session 132) that mentions L3's bit-1 rule for valid page
descriptors.

The answer is yes, nonux doesn't build any L3 tables today;
the kernel identity map and per-process address spaces both
stop at L2 with 2 MiB block descriptors.  This *is* covered
in §"What we're going to build" earlier in the chapter
("**We never touch L3 in the kernel-side identity map** —
2 MiB blocks are big enough for the whole kernel"), but a
reader who reaches §"The tables themselves" ten paragraphs
later may have forgotten — the descriptor-format walk-through
between those two sections names L3 several times.

The user's request: *"Readers may not remember every
details. So, please address it again in the side note of the
section 'The tables themselves'."*

### Side note added

Appended a new side note to §"The tables themselves",
immediately after the existing "where did the table addresses
come from?" side note:

> **Side note: no L3 tables in this configuration.** Notice
> we never declare an `l3_*_table`.  The L2 entries are
> *block* descriptors (2 MiB blocks) rather than table
> descriptors, so the walk reaches its answer at L2 and the
> MMU never asks for an L3.  The architecture *defines* a
> third level for the 4 KiB granule we configured (TG0 = 0b00
> in TCR), but using it is optional — block descriptors at L2
> short-circuit the walk whenever 2 MiB granularity is good
> enough.  nonux today doesn't need finer than 2 MiB anywhere
> (every kernel mapping and every per-process user window is
> a multiple of 2 MiB), so we don't pay for L3 tables.  The
> Phase 9 per-process MM rework is exactly the change that
> introduces them: it swaps the user-window L2 entries from
> block descriptors to table descriptors that point at L3
> pages, so each 4 KiB user page can be allocated, mapped,
> and protected individually.  Until then, the bit-1 rule for
> L3 page descriptors in the previous section is
> *prescriptive* for a future reader of the rework — not
> something you'll see in today's tables.

The note:

1. Names the concrete observation (no `l3_*_table` declared).
2. Explains *why* (block descriptors at L2 end the walk; L3
   is optional given the architecture).
3. Reconnects to the configuration (TG0 = 0b00 → 4 KiB
   granule = the granule that *would* govern L3).
4. Explains *why nonux doesn't need it today* (everything is
   2 MiB-aligned).
5. Forward-points to Phase 9 as the change that introduces
   L3 tables.
6. Closes the circle on the descriptor-section's L3 mention
   with one sentence reframing it as "prescriptive for a
   future reader" rather than "describing today's code."

## Key Findings

- **Long chapters need re-establishing the same fact at the
  reader's *next* point of confusion, not just at the
  earliest correct point.**  §"What we're going to build"
  states "no L3" at a high level; §"Descriptors: the
  bit-level layout" describes L3's bit-1 rule for
  completeness; the reader can lose track of which is
  current code and which is for-completeness.  Naming the
  fact again at the table-construction site makes it
  unambiguous.
- **A reader question is the cleanest signal that a fact has
  faded.**  The user reading from session 132's lead-in to
  the descriptor-format detail asked the question — exactly
  the journey a fresh reader would take.

## Decisions Made

- **Side note rather than prose-rewrite of an existing
  paragraph.**  The "tables themselves" prose walks the
  reader through `l1_table` / `l2_mmio_table` / `l2_ram_table`
  declarations, the L2 population loop, the L1 wiring.
  Inserting "and we don't have L3" mid-walk would interrupt
  the flow.  A side note at the end picks up cleanly after
  the construction is described.
- **Forward-link to Phase 9 in the side note, not just a
  vague "future rework".**  Phase 9 is named in §"What we're
  going to build" and elsewhere already; using the same name
  here keeps the "this is what triggers the change" pointer
  unambiguous.
- **Reframe the descriptor-section's L3 bit-1 rule as
  prescriptive.**  Saves the reader from feeling that they
  missed something — *of course* they didn't see L3 tables
  in the construction; the descriptor format documents the
  rule for the architecture, not for today's code.

## Status at End of Session

- `book/06-virtual-memory-and-mmu.md` patched in §"The
  tables themselves" — one new side note appended (~16 lines).
- No source-code changes; tests not re-run.

## Next Steps

- Same forward fork as sessions 128–133: continue the book
  with chapter 7 (Kernel threads and context switching, Part
  IV) or start Phase 9 (per-process MM rework — once Phase 9
  lands the chapter 6 revision pass becomes "actually delete
  the 'no L3 today' qualifications" rather than "rewrite from
  scratch").

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/06-virtual-memory-and-mmu.md` — §"The tables
  themselves" gained a side note "no L3 tables in this
  configuration" after the existing "where did the table
  addresses come from" side note.  Names the observation
  (no `l3_*_table` declared), explains why (2 MiB block
  descriptors at L2 end the walk), notes that L3 is
  prescribed by the granule choice but optional in the walk,
  and forward-points to Phase 9 as the change that
  introduces L3 tables.  Reframes the descriptor-section's
  L3 bit-1 rule as prescriptive for a future reader.

`proj_docs/nonux` (this commit):
- `HANDOFF.md` — Latest session log line, Session Logs list,
  Next Actions updated.
- `logs/session-134-no-l3-side-note.md` — new (this file).
