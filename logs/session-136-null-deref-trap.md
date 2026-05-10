# Session 136: `mmu_init` invalidates `l2_mmio_table[0]` to catch EL1 NULL-deref

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–6 shipped (Part I + Part II + Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Close a latent gap that the user surfaced while reading
  chapter 6: an EL1 NULL-pointer dereference silently
  succeeded (against QEMU virt's NOR flash region at PA 0)
  instead of producing a clean abort.

## What Was Done

### Background — what the user caught

While reading the chapter's §"The tables themselves" the user
asked: *"l2_mmio_table cover 0x0. what would happen if the
program access NULL?"*

The honest answer (working through the four cases):

| Access                          | Outcome                                                                                 |
|---------------------------------|-----------------------------------------------------------------------------------------|
| EL0 `*NULL` read/write          | Permission fault (`AP = 0b00` denies EL0) → `on_sync` kills the process.  ✓             |
| EL0 `((void(*)())0)()` call     | Instruction abort (`UXN = 1`) → `on_sync` kills the process.  ✓                         |
| EL1 `((void(*)())0)()` call     | Instruction abort (`PXN = 1`) → kernel panic via `on_sync`.  ✓                          |
| **EL1 `*NULL` read/write**      | **No fault.** AP=0b00 allows EL1 R/W; PA 0 is QEMU virt's NOR flash; silent corruption. ✗ |

That last row was the gap.  An EL1 NULL-deref didn't crash —
it silently hit flash, absorbed writes, returned `0xFF` for
reads, and the kernel continued.  Exactly the wrong outcome
for debuggability.

### Code fix — `core/mmu/mmu.c`

After the L2-population loop, added one line:

```c
/* Catch NULL-deref from EL1.  Without this, AP=0b00 in
 * l2_mmio_table[0] lets the kernel read/write PA 0..2 MiB ...
 */
l2_mmio_table[0] = 0;
```

`l2_mmio_table[0]` covered the 2 MiB block at PA
`0..0x00200000`.  Marking it invalid means every EL1 access
to a VA in that range now takes a clean synchronous abort
that routes through `on_sync`, which can then panic with the
faulting address in `FAR_EL1`.

**Why slot 0 specifically:** the loop assigned
`device_block(0)` to slot 0, which is the only descriptor in
the kernel mappings that gave a valid path to PA 0.  Slot 1
covers PA `0x200000..0x400000` (still flash on QEMU virt; a
NULL-deref with offset > 2 MiB would still get through), but
sharp NULL-deref bugs almost always land in the first 2 MiB
(typical struct sizes, array indices, etc.) so slot 0 is the
sweet spot.

**Why functionally safe:** no MMIO nonux talks to lives in PA
`0..0x200000`.  Per QEMU's `hw/arm/virt.c`, the first 128 MiB
is NOR flash (which nonux doesn't use), and the first real
device is the GIC distributor at `0x08000000` (slot 64 of
`l2_mmio_table`).  Slots 1..63 remain valid Device blocks
covering the (unused) flash range; slot 64+ covers the real
MMIO devices.

### Book update — `book/06-virtual-memory-and-mmu.md`

§"The tables themselves" updated to reflect the fix:

- The L2-population code snippet now shows the post-loop
  `l2_mmio_table[0] = 0` line.
- A new paragraph after the existing post-loop prose explains
  what slot 0 covers, why `device_block(0)` would have left
  EL1 NULL-deref wide open, what the invalidation changes,
  and why it's functionally safe.
- A side note clarifies that EL0 NULL-deref was already
  caught (AP=0b00 + UXN=1) and that EL1 was the only gap.

## Key Findings

- **The chapter's worked walk became the audit.**  The user
  was reading the chapter's L2-population code, traced
  `device_block(0)` to "valid 2 MiB Device block at PA 0",
  and immediately asked the right question: what does
  dereferencing NULL actually do?  The chapter's explicit
  bit-by-bit framing of the descriptor + `AP = 0b00` made
  the gap surface naturally.  Without the chapter, the bug
  would still be latent.
- **PXN/UXN already covered the *execution* side of the
  hazard.**  An EL1 jump to NULL panics cleanly.  Only the
  R/W side was wide open.  That's a subtle asymmetry — the
  same descriptor encoding can be safe for one access kind
  and unsafe for another.
- **One-line code change, one-paragraph chapter change.**
  Both are small, but the chapter change is what makes the
  one-line fix *visible* to future readers.  Without
  documenting it, a later read of `mmu_init` would see
  `l2_mmio_table[0] = 0` as an unexplained anomaly.

## Decisions Made

- **Invalidate only slot 0, not the whole flash range.**
  Slot 0 catches the canonical NULL-deref case (offsets <
  2 MiB into a struct).  Invalidating slots 1..63 would
  catch more pathological cases (NULL-deref with offset
  between 2 MiB and 128 MiB) but those are vanishingly
  rare, and the simpler change is easier to reason about
  and document.  If a real bug ever lands in slots 1..63,
  we can widen the invalidation then.
- **Comment in `mmu.c` carries the full rationale, the
  chapter carries the *teaching* version.**  The source
  comment is for someone reading `mmu.c` who needs to
  understand why slot 0 is special.  The chapter passage
  is for someone learning what page tables are *for* and
  how nonux uses them.  Different audiences; both need
  the same fact phrased differently.
- **Side note on EL0-was-already-caught.**  Without this,
  a reader could conclude "the chapter is admitting nonux
  had a NULL-deref bug across the board" — when in fact
  the bug was only on the EL1 side.  The side note is a
  small symmetry-restoring touch.

## Status at End of Session

- `core/mmu/mmu.c` — one new line + comment after the L2
  population loop.
- `book/06-virtual-memory-and-mmu.md` — §"The tables
  themselves" augmented with the new snippet + paragraph +
  side note.
- No test impact expected — no nonux code or test depends on
  reading/writing PA `0..0x200000`.  Tests not re-run in this
  session (kernel ktests take ~3600 s); test counts unchanged
  from session 123 until the next time the full suite is run.

## Next Steps

- Eventually re-run the kernel test suite to confirm the
  slot-0 invalidation doesn't break anything (expected: no
  effect).  Bundling that re-run with the next code change
  is fine.
- Same forward fork as sessions 128–135: continue the book
  with chapter 7 (Kernel threads and context switching, Part
  IV) or start Phase 9 (per-process MM rework).

---

**Files Changed:**

`sources/nonux` (two commits in this session):
- `core/mmu/mmu.c` — invalidate `l2_mmio_table[0]` after the
  L2-population loop to catch EL1 NULL-deref; comment
  explains the rationale and the safety argument.
- `book/06-virtual-memory-and-mmu.md` — §"The tables
  themselves" gained the post-loop `l2_mmio_table[0] = 0`
  line in the code snippet, a paragraph explaining the
  invalidation and its safety argument, and a side note
  clarifying that EL0 NULL-deref was already caught (AP=0b00
  + UXN=1) and EL1 was the only gap.

`proj_docs/nonux` (this commit):
- `HANDOFF.md` — Latest session log line, Session Logs list,
  Next Actions updated.
- `logs/session-136-null-deref-trap.md` — new (this file).
