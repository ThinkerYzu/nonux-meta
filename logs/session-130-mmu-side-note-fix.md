# Session 130: Book chapter 6 — MMU "why not later" side note correction

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–6 shipped (Part I + Part II + Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Address a technical inaccuracy in the chapter-6 side note
  *§"Why turn on the MMU at all" → "why not later?"* spotted
  by the user in review.

## What Was Done

### Background — what the user caught

The original side note read:

> The trouble is the *transition*: if data has been written
> without the cache (uncached stores) and then the cache
> turns on, those stores aren't visible to cached loads
> until you do an awkward dance of `dc cisw` invalidations.

The user pointed out: with MMU off, all memory is Device-nGnRnE,
so uncached stores commit to memory directly — they don't enter
the cache.  After invalidation, a cached load to that physical
address misses and pulls fresh from memory.  There's no
"uncached stores invisible to cached loads" hazard the way the
note described.

The user is right.  The note conflated two things:

- **Mandatory invalidate-before-enable dance** — required because
  cache and TLB state at reset is *architecturally undefined*
  (per ARM ARM B2.4.7).  The cache may hold valid lines with
  garbage from firmware, speculative prefetches, or simulator
  state.  Required regardless of whether the kernel made any
  pre-MMU stores.
- **A "uncached stores get hidden by cache" hazard** — this isn't
  a thing.  The stores went to memory; the cache wasn't involved.

The actual *interaction* between the two (when it does occur) is:
if reset left a stale cache line for PA X, *and* the kernel did
an uncached store to PA X pre-MMU, *then* enabling the cache
without invalidating first would shadow the kernel's write with
the stale line.  But the fix (invalidate before enable) is needed
for the reset state on its own, so this isn't really an argument
about *what the kernel did first*, it's an argument about *what
reset left behind*.

### Side-note rewrite

Replaced the original side note with three numbered reasons,
each anchored in a real architectural concern:

1. **Cache and TLB state at reset is architecturally undefined.**
   Invalidating both is mandatory before relying on either; doing
   it early (before the kernel has memory state worth preserving)
   keeps the sequence simple.  Parenthetical clarifies the
   non-issue: pre-MMU Device-nGnRnE stores don't enter the cache,
   so they aren't the source of the "stale cache" hazard — the
   reset state is.
2. **Pre-MMU memory is all Device-nGnRnE.** Slow, strictly
   ordered, and LDXR/STXR atomics may not behave as expected on
   Device memory.
3. **Bad-pointer faults are clean once the MMU is on.** Without
   the MMU, a bad pointer at EL1 yields a bus error at best and
   silent corruption at worst.

The previous final sentence ("the only thing the kernel has
touched is the UART") is preserved as the closing note.

## Key Findings

- **The original side note was a victim of "explain by analogy
  to the wrong mechanism."** The intuition that "transitioning
  from uncached to cached operation is fraught" is broadly
  correct; the *mechanism* offered was wrong.  Replacing it with
  the actual mechanism (undefined reset state) preserves the
  conclusion (turn the MMU on early) while making the reasoning
  defensible.
- **User-spotted technical defects are highest-value review
  signal.** This fix wouldn't have come from running tests or
  rereading the chapter — it took a careful reader pushing back
  on the reasoning.  The fix is a few lines but the chapter is
  measurably more correct now.
- **Three-reason framing is more durable than one-reason.** The
  original "why not later" was load-bearing on the (broken)
  cache transition argument.  The replacement spreads the
  weight across three independent concerns; even if one is
  later questioned, the conclusion still holds.

## Decisions Made

- **Fix in place rather than a "see corrections" footnote.** This
  is a self-published book chapter; we can edit history freely.
  No PR / review trail to preserve.
- **Keep the parenthetical about pre-MMU stores not entering the
  cache.** The user's question was *exactly* "but uncached
  stores don't go through the cache, so where's the bug?" —
  having the answer to that question explicit in the side note
  pre-empts other readers from getting stuck on the same
  reasoning.
- **Don't expand into a separate "Cache state at reset" section.**
  The chapter has already covered MMU bring-up in detail; this is
  a sub-point of "why early."  A standalone section would over-
  weight the topic relative to the rest of the chapter.

## Status at End of Session

- `book/06-virtual-memory-and-mmu.md` patched in §"Why turn on
  the MMU at all": one side note rewritten (~12 lines → ~22
  lines — slightly longer because the corrected version is
  three-pronged).
- No other content changed.
- No source-code changes; tests not re-run.

## Next Steps

- Same forward fork as sessions 128–129: continue the book with
  chapter 7 (Kernel threads and context switching, Part IV) or
  start Phase 9 (per-process MM rework).

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/06-virtual-memory-and-mmu.md` — §"Why turn on the MMU
  at all" side note rewritten.  Removed false claim that
  pre-MMU uncached stores aren't visible to cached loads;
  replaced with three correct reasons (undefined reset state,
  Device-typed slow path + atomic semantics, clean
  synchronous-abort behaviour).

`proj_docs/nonux` (this commit):
- `HANDOFF.md` — Current Status, Latest session log line,
  Session Logs list, Next Actions updated.
- `logs/session-130-mmu-side-note-fix.md` — new (this file).
