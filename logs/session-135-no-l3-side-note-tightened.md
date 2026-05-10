# Session 135: Book chapter 6 — "no L3 tables" side note tightened

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–6 shipped (Part I + Part II + Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Tighten the side note added in session 134 and add the
  `Tbl/Blk` field name (the chapter's established vocabulary
  for descriptor bit 1).

## What Was Done

### Background — what the user caught

The session 134 side note was 16 lines and didn't use the
`Tbl/Blk` field name that the rest of the chapter has been
using since the descriptor-section table.  Reader feedback:
*"please mention 'Tbl/Blk' in the side note. Also make the
side note shorter."*

### Rewrite

Original (16 lines):

> Notice we never declare an `l3_*_table`.  The L2 entries
> are *block* descriptors (2 MiB blocks) rather than table
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

New (10 lines):

> Notice we never declare an `l3_*_table`.  The L2 entries we
> built above are block descriptors (`Tbl/Blk = 0`), so the
> walk ends at L2 — the MMU never asks for an L3.  The 4 KiB
> granule (TG0 = 0b00) defines a third level, but L3 is
> optional when 2 MiB granularity suffices, and nonux today
> is 2 MiB-aligned everywhere.  Phase 9 introduces L3 tables
> for the user window (`Tbl/Blk = 1` at L2, pointing at an L3
> of 4 KiB page descriptors); until then, the L3 rules in
> §"Descriptors" are prescriptive, not descriptive of today's
> code.

Improvements:

- Uses `Tbl/Blk = 0` and `Tbl/Blk = 1` to anchor the
  current-vs-Phase-9 contrast in the chapter's established
  field name.
- Drops the parenthetical "(every kernel mapping and every
  per-process user window is a multiple of 2 MiB)" and
  collapses to "nonux today is 2 MiB-aligned everywhere".
- "Phase 9 per-process MM rework is exactly the change that
  introduces them: it swaps..." → "Phase 9 introduces L3
  tables for the user window".  The mechanism still gets
  named (`Tbl/Blk = 1` at L2 → L3 of 4 KiB pages) but
  without the lead-in.
- "the bit-1 rule for L3 page descriptors in the previous
  section is *prescriptive* for a future reader of the
  rework — not something you'll see in today's tables" →
  "the L3 rules in §'Descriptors' are prescriptive, not
  descriptive of today's code".

The note is now 10 lines instead of 16, and the `Tbl/Blk`
references make the parallel between today's L2 block
descriptors and Phase 9's L2 table descriptors immediate.

## Status at End of Session

- `book/06-virtual-memory-and-mmu.md` patched in §"The
  tables themselves" — side note shortened from 16 to 10
  lines and reworded around `Tbl/Blk = 0` / `Tbl/Blk = 1`.
- No source-code changes; tests not re-run.

## Next Steps

- Same forward fork as sessions 128–134.

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/06-virtual-memory-and-mmu.md` — §"The tables
  themselves" side note "no L3 tables in this configuration"
  shortened and reworded around `Tbl/Blk = 0` (today) and
  `Tbl/Blk = 1` (Phase 9).

`proj_docs/nonux` (this commit):
- `HANDOFF.md` — Latest session log line, Session Logs list,
  Next Actions updated.
- `logs/session-135-no-l3-side-note-tightened.md` — new
  (this file).
