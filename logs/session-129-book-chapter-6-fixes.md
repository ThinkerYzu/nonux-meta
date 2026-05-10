# Session 129: Book chapter 6 — review fixes

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–6 shipped (Part I + Part II + Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Self-review chapter 6 against the source code and chapter 1 / 5
  cross-links, fix any defects found.
- No new content; polish only.

## What Was Done

### Review pass on `book/06-virtual-memory-and-mmu.md`

Cross-checked the chapter against `core/mmu/mmu.c`,
`core/cpu/el0_entry.S`, `core/boot/boot.c`, and the chapter-5
"Reserving a range" anchor. Verified:

- Descriptor builders (`device_block` / `normal_block` /
  `user_block`) match `mmu.c:74–112` bit-for-bit.
- `MAIR_VALUE`, `TCR_VALUE`, the `mmu_init` SCTLR-enable
  sequence, and the `CPACR_EL1.FPEN` tweak all match
  `mmu.c:50–225`.
- The two worked walks resolve correctly: VA `0x4009_0000`
  → bits 38..30 = 1 (L1[1]) → bits 29..21 = 0 (L2[0]) → PA
  `0x40090000`; VA `0x0900_1000` → L1[0] → bits 27 & 24 set
  in `0x09000000` give L2[72] → PA `0x09001000`.
- `USER_WINDOW_INDEX = 64`, kernel L2_ram exposes 1 block
  (2 MiB), per-process exposes 4 blocks (8 MiB) — all match
  `mmu.c:142–146`.
- Chapter 5 cross-link `#reserving-a-range` resolves to the
  real `## Reserving a range` heading at
  `book/05-physical-memory-and-pmm.md:561`.

### Five fixes applied

1. **Broken descriptor diagram replaced with markdown table.**
   The original ASCII box had an unclosed `AP[2:0|...|1]`
   bracket (typo: should be `AP[2:1]`), and the column headers
   (15 boundary positions) didn't align with the cell count
   (~13 cells in body). Replaced with a markdown table
   (`Bits | Name | Meaning`) listing the nine fields nonux
   actually uses (V, Tbl/Blk, AttrIdx, AP[2:1], SH, AF, PA,
   PXN, UXN), plus a parenthetical noting which bits stay zero
   (nG, contiguous-hint, reserved fields). Style matches
   chapter 1's existing markdown tables (sections "Sections"
   and "Privilege levels"). The bullet list of bit-by-bit
   detail that originally followed the diagram is kept — the
   table is the at-a-glance summary, the bullets give the
   prose elaboration. Lead-in changed to *"In more detail:"*
   to flag the relationship between the two.

2. **`AP` bullet wording fix.** The original said
   *"nonux uses three combinations: kernel R/W (`AP = 0b00`),
   user R/W (`AP = 0b01`), and read-only is currently
   unused"* — claims three but lists two. Rewritten as
   *"Of the four AP encodings, nonux uses two: kernel R/W
   (`AP = 0b00`) and user R/W (`AP = 0b01`). Read-only
   mappings would be encoded by setting bit 7, but no
   current call site needs one."*

3. **Pluralization fix in VA-layout diagram.**
   *"L3 idx (9 bit) | offset(12 bit)"* → *"L3 idx (9 bits) |
   offset(12 bits)"*. The first two cells already said *9
   bits*; the right two columns drifted singular.

4. **Chapter 1 cross-link fixed.** §"Where to read more"
   pointed at *Chapter 1 §"Exception levels and modes"*,
   but chapter 1's actual section is `## Privilege levels:
   EL0, EL1, EL2, and EL3`. The link went to the file root
   (no anchor) so it worked, but landed the reader at the
   top of chapter 1 instead of the EL section. Updated to
   `[Chapter 1 §"Privilege levels: EL0, EL1, EL2, and
   EL3"](01-boot-and-linker.md#privilege-levels-el0-el1-el2-and-el3)`
   — anchor verified against the actual heading.

5. **Belt-and-suspenders side note added in §"Fork's
   user-window memcpy".** The chapter explains the TTBR0
   switch in `mmu_copy_user_backing` as the fix for the
   "dst PA aliases the user-window VA under parent's TTBR0"
   bug, then later (in §"A boot-time consequence: PMM
   reservation") explains the PMM reservation as preventing
   the *same* bug. With the reservation in place, dst's
   8 MiB chunk strictly cannot overlap the user window, so
   the TTBR0 switch is defense-in-depth. Added a side note
   acknowledging the relationship: the reservation removes
   the bug at the source, and the local TTBR0 switch keeps
   the safety property local to one function instead of
   relying on a distant invariant maintained by `boot.c`.

## Key Findings

- **The original ASCII descriptor diagram was clever but
  fragile.** Trying to show ~12 cells with mixed widths in a
  monospace ASCII box, with the bit-position headers above the
  cells, almost always drifts. A markdown table is uglier in
  the source but renders cleanly in any markdown viewer and
  removes the maintenance hazard.
- **Self-review of chapter 6 turned up exactly one factual
  defect (the AP encoding count).** Everything else was
  cosmetic (broken ASCII alignment, plural / singular drift,
  wrong section title in a cross-link). The technical content
  — descriptor bit layout, walk arithmetic, MAIR/TCR/SCTLR
  decode, the fork-time TTBR0 juggling — held up. That's
  reassuring: the chapter was written carefully against the
  source the first time around.
- **Cross-link health benefits from "verify before merge".**
  The chapter 1 link looked correct because the file existed
  and the anchor-less form worked, but the section title in
  the link text was a name that doesn't exist anywhere. A
  future automated link checker (per the
  [BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md) "broken anchor
  audit" idea) would have caught this even though the link
  technically resolves.

## Decisions Made

- **Markdown table over ASCII box for the descriptor.**
  Consistent with chapter 1's existing tables (Sections, EL
  privileges); easier to maintain; renders cleanly without
  needing exact monospace column counting.
- **Keep the bit-by-bit bullet list under the table.** It's
  redundant if the table were the final word, but the bullets
  add prose context the table can't (e.g. "AP encoding is
  non-obvious — bit 6 selects EL0, bit 7 selects RO"). Lead-in
  changed to "In more detail:" to flag the relationship.
- **Connect the TTBR0 switch and PMM reservation explicitly.**
  Two separate sections explaining "we prevent the same bug"
  read as redundant unless the chapter says they're the same
  bug. One short side note resolves this without rewriting
  either section.

## Status at End of Session

- `book/06-virtual-memory-and-mmu.md` patched (descriptor
  diagram, AP wording, plural fix, chapter 1 link, fork
  side note). Still 1247-ish lines after the swap (table is
  shorter than the broken ASCII box; new side note is short).
- No source-code changes. Tests not re-run; test counts
  unchanged from session 127.
- Part III still complete (chapters 5–6).
- HANDOFF + this session log to be updated in the commit.

## Next Steps

- Same forward fork as the end of session 128: either
  continue the book with chapter 7 (Kernel threads and
  context switching, Part IV) or start Phase 9 (per-process
  MM rework). No new dependencies or blockers introduced
  this session.

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/06-virtual-memory-and-mmu.md` — five review fixes:
  ASCII descriptor diagram replaced with markdown table; AP
  bullet wording corrected ("three combinations" → "two");
  L3/offset pluralisation fixed; chapter 1 cross-link section
  title + anchor corrected; belt-and-suspenders side note
  added in §"Fork's user-window memcpy" connecting the TTBR0
  switch to the PMM reservation.

`proj_docs/nonux` (this commit):
- `HANDOFF.md` — Current Status, Latest session log line,
  Session Logs list, Next Actions updated.
- `logs/session-129-book-chapter-6-fixes.md` — new (this file).
