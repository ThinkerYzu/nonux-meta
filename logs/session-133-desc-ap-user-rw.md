# Session 133: Book chapter 6 — `DESC_AP_USER_RW` added to descriptor macros block

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–6 shipped (Part I + Part II + Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Surface a missing helper macro (`DESC_AP_USER_RW`) in
  chapter 6's §"Descriptors: the bit-level layout" macros
  block, flagged by the user.

## What Was Done

### Background — what the user caught

The chapter listed nine descriptor helper macros from
`core/mmu/mmu.c` lines 38–46 in one block:

```c
#define DESC_VALID        (1UL << 0)
#define DESC_TABLE        (1UL << 1)
#define DESC_ATTR_IDX(n)  ((uint64_t)((n) & 0x7) << 2)
#define DESC_AP_EL1_RW    (0UL << 6)
#define DESC_SH_INNER     (3UL << 8)
#define DESC_SH_NONE      (0UL << 8)
#define DESC_AF           (1UL << 10)
#define DESC_PXN          (1UL << 53)
#define DESC_UXN          (1UL << 54)
```

But the AP bullet just above this block had told the reader
that "nonux uses two AP encodings: kernel R/W (`AP = 0b00`)
and user R/W (`AP = 0b01`)" — yet the macros block only
showed the kernel-RW one (`DESC_AP_EL1_RW`), not the user-RW
one.

The user-RW macro `DESC_AP_USER_RW` does exist in `mmu.c`,
but it lives at line 104 — outside the block listed above,
immediately above its only consumer `user_block`.  The
chapter's macros block had quietly mirrored the source
layout, leaving the user-RW macro out of sight.

### Two changes

1. **Macros block now lists `DESC_AP_USER_RW`** alongside the
   other descriptor helpers, with both AP encodings annotated:

   ```c
   #define DESC_AP_EL1_RW    (0UL << 6)   /* AP[2:1] = 0b00 — EL1-only RW */
   #define DESC_AP_USER_RW   (1UL << 6)   /* AP[2:1] = 0b01 — EL0+EL1 RW */
   ```

   A note added below the block explains why `mmu.c` keeps the
   user-RW macro near `user_block` instead of with the rest of
   the vocabulary (it's the macro's only consumer).

2. **`user_block` snippet no longer duplicates the `#define`.**
   The previous chapter showed `DESC_AP_USER_RW` defined inline
   with the `user_block` helper — accurate to the source, but
   now redundant with its appearance in the top block.  Removed
   the inline `#define` so the chapter doesn't appear to define
   the macro twice.  The reader still sees the same `user_block`
   function body using `DESC_AP_USER_RW`; only the duplicate
   definition is gone.

## Key Findings

- **Mirror-the-source structurally vs collect-for-reading.**  The
  source's "macro defined next to its consumer" pattern is good
  C-engineering practice (locality of reference, smaller search
  scope when reading the helper) but it's a bad reading
  experience for a chapter that's introducing the descriptor
  vocabulary.  The chapter now collects all the macros and
  notes the source's choice — best of both.
- **The bullet list above the macros block is what surfaced
  the gap.**  The bullet promised "two AP combinations",
  which created an expectation the reader could check against
  the macros block.  Without that bullet, the missing macro
  might have stayed invisible.

## Decisions Made

- **Add the missing macro to the chapter's block, don't remove
  it from `user_block`'s usage.**  The function body
  *consumes* `DESC_AP_USER_RW`; that consumption is the whole
  point of showing `user_block`.  Only the redundant `#define`
  line went away, not the use of the macro inside the
  function.
- **Note the source-side placement, don't argue for changing
  it.**  The chapter's job is to teach the reader to read
  `mmu.c`, not to prescribe how `mmu.c` should be organised.
  A one-paragraph note about the source's "macro near its
  only consumer" choice gives the reader the pointer they
  need without editorialising.

## Status at End of Session

- `book/06-virtual-memory-and-mmu.md` patched in
  §"Descriptors: the bit-level layout":
  - Macros block grew by one line (`DESC_AP_USER_RW`); two
    AP-encoding lines now have inline comments.
  - One-paragraph note added below the block explaining the
    source's "near its consumer" placement.
  - `user_block` snippet's redundant `#define` removed.
- No source-code changes; tests not re-run.

## Next Steps

- Same forward fork as sessions 128–132: continue the book
  with chapter 7 (Kernel threads and context switching, Part
  IV) or start Phase 9 (per-process MM rework).

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/06-virtual-memory-and-mmu.md` — `DESC_AP_USER_RW`
  added to the descriptor macros block alongside
  `DESC_AP_EL1_RW`, both AP-encoding lines annotated; note
  added explaining why `mmu.c` defines `DESC_AP_USER_RW`
  separately near its consumer; redundant `#define` removed
  from the `user_block` snippet.

`proj_docs/nonux` (this commit):
- `HANDOFF.md` — Latest session log line, Session Logs list,
  Next Actions updated.
- `logs/session-133-desc-ap-user-rw.md` — new (this file).
