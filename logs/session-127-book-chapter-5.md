# Session 127: Book chapter 5 — Physical memory and the page allocator

**Date:** 2026-05-09
**Phase:** Phases 1–9b complete; book chapters 1–5 shipped (Part I + Part II + first chapter of Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Draft the next book chapter, picking up where chapter 4 left
  off (we have a heartbeat; now we need real memory to give
  out).
- Per [BOOK-OUTLINE.md](../BOOK-OUTLINE.md#chapter-list), that's
  **chapter 5: Physical memory and the page allocator (PMM)** —
  the first chapter of Part III (Memory).

## What Was Done

### `sources/nonux/book/05-physical-memory-and-pmm.md` (new, 1123 lines)

Walks the path from "the kernel image ends at `__free_mem_start`,
RAM ends at `RAM_END`" all the way to the buddy component bound
to the `memory.page_alloc` slot, then ties the layers together
visually.

Section structure:

1. **Intro** (3 paragraphs) — frames the gap (so far the
   kernel has had only static storage), names what changes
   (PMM + buddy on top), promises the reader will be able to
   read the boot log line `[pmm] total=… free=… …` and know
   what every number means and where it came from.
2. **Files** — `core/pmm/{pmm.h,pmm.c}`, `core/boot/boot.c`,
   `core/boot/linker.ld`, `components/mm_buddy/{mm_buddy.c,
   README.md}`, `interfaces/mm.h`, `core/lib/kheap.h`.
3. **Terms** — 11 entries: physical memory, virtual memory,
   page, page frame, PMM, bitmap, first-fit, atomic op, order,
   buddy allocator, slot/component (latter cross-ref'd to the
   not-yet-shipped chapter 9).
4. **Two questions, two layers** — names the bookkeeping vs
   sizing split and pre-stages the PMM-then-buddy walk.
5. **Where memory comes from** — the three numbers
   (RAM_BASE = `0x40000000` on QEMU virt, `__free_mem_start`
   from the linker script, `RAM_END = 0x80000000` hardcoded);
   the `extern char __free_mem_start[]` linker-symbol idiom;
   forward-points the future DTB-parsing replacement.
6. **The bitmap** — one bit per page; size math (32 KiB for
   1 GiB pool, 0.003%); self-accounting layout (bitmap pages
   marked allocated at the start of init); `bit_byte` /
   `bit_mask` / `g_reserved` / `g_hint`.
7. **Two helpers for testing and claiming bits** —
   `bit_test` (relaxed prefilter), `try_claim` (atomic
   fetch_or, ACQUIRE), `release` (atomic fetch_and with
   inverted mask, RELEASE). Side note explaining *why
   single-CPU still needs atomics* (interrupts can preempt
   the RMW).
8. **Allocating one page** — `pmm_alloc_page` walked
   line-by-line: hint-based first-fit, wrap, skip
   reserved range, cheap test then expensive claim, hint
   bump on success.
9. **Returning a page** — `pmm_free_pages` short-and-sweet,
   plus the "set hint backwards on free" working-set /
   D-cache rationale.
10. **Multi-page contiguous allocation** — the two-pass
    pattern (cheap scan, then atomic claim with rollback)
    in `pmm_alloc_pages`, page-tables motivation, side note
    on what real kernels (Linux mm/page_alloc.c) do beyond
    this.
11. **Reserving a range** — `pmm_reserve_range`: motivation
    (user-window aliasing under per-process TTBR0), idempotence
    via `try_claim`, outward rounding, clipping to the pool.
12. **What `boot_main` actually prints** — annotates the
    real boot-log lines line by line so the reader can map
    between code and output.
13. **The buddy component** — three-reason motivation
    (power-of-two requests are common, coalescing, strategy
    is swappable). Then the walk:
    - `nx_mm_ops` interface contract (forward-points to
      chapter 10 for the IDL toolchain).
    - Pool geometry — 16 pages, why intentionally tiny.
    - State (`mm_buddy_state` + intrusive `free_node`).
    - **Allocation: split until you fit** — algorithm in
      English, then the C, then a worked example showing
      the free-list state through one order-2 alloc.
    - **Free: merge while the buddy is free** — algorithm,
      the address-XOR trick, then a worked example showing
      three allocs and three frees that round-trip back to
      the original order-4 block.
    - **Why an XOR works** — the offset-pattern explanation.
    - **Lifecycle** — `init`/`destroy` (carve / return the
      pool), `enable`/`disable` no-ops + why, `kernel.json`
      binding.
14. **How the layers stack at runtime** — full ASCII layered
    diagram from physical RAM up through PMM, kheap and
    mm_buddy, to the slot consumers.
15. **A few extra things to know** — six side notes: no
    per-CPU caches yet, PMM doesn't zero pages, why the
    pool is small, no alignment guarantee beyond page
    granularity, why `g_free` is `_Atomic` but the bitmap
    isn't, and the bookkeeping-not-scheduling property
    that makes memory simpler than CPU/IRQs/I/O.
16. **Where to read more** — source links + chapter 1
    cross-ref to the linker section, ARM ARM, Linux
    `mm/page_alloc.c`, QEMU `hw/arm/virt.c`.

Length: 1123 lines — comfortably in the 600–1500 band.
Longer than chapter 4 (760) because two systems are walked
(PMM and the buddy component) instead of one.

### Index / outline updates

- **`book/README.md`** — chapter 5 entry under Part III
  changed from plain text to
  `[Physical memory and the page allocator (PMM)](05-physical-memory-and-pmm.md)`.
- **`BOOK-OUTLINE.md`** — chapter 5 status `planned → shipped`.

## Key Findings

- **Two-layer story is the natural reading order.** The PMM
  alone is a complete story (it can hand out pages), but the
  buddy on top is what every kernel subsystem actually
  consumes. Walking the PMM first, then the buddy, mirrors
  how `boot_main` brings them up: PMM in `core/pmm/pmm.c`
  during `pmm_init`, buddy through the framework's
  bring-up of the `memory.page_alloc` slot. The chapter
  closes with a layered diagram tying them together.
- **The atomics-without-SMP rationale is worth a side note.**
  A reader fresh from chapter 4's "preempt during a
  not-yet-finished operation" lens will want to understand
  why the bitmap uses `__atomic_fetch_or` even though there's
  no second CPU. Calling out the interrupt-during-RMW failure
  mode explicitly anchors the design choice in the same
  preemption model the timer chapter set up.
- **Worked examples for the buddy pay off.** The split-on-alloc
  and merge-on-free paragraphs read fine in English, but the
  "free_lists state after each step" tables make the
  invariant — "an alloc and a matching free leave the pool
  bit-for-bit the way it started" — visible. Same pattern
  as chapter 4's drift arithmetic and chapter 3's vector-table
  byte budget.
- **The buddy chapter is mostly story, not API.** The
  `nx_mm_ops` table is four functions. The interesting parts
  are the algorithm, the in-place free-list trick, and the
  address-XOR insight. The chapter spends more time on
  *why* the buddy is shaped this way than on *what* the
  ops do — which is consistent with the book's overall
  "concept-not-reference" positioning.

## Decisions Made

- **Cover both PMM and the buddy in one chapter.** They sit
  at different layers but the buddy makes very little sense
  without the PMM directly under it. Splitting them would
  have left chapter 5 ending mid-thought ("…and the actual
  page allocator we use is in the next chapter").
- **Defer slot/component framework explanation.** The
  Terms section names slot and component in one bullet,
  with a forward pointer to chapter 9. The chapter avoids
  framework details (descriptors, the linker section, the
  bootstrap walk) entirely — those are for chapter 9. Here
  the buddy is just "the implementation behind
  `memory.page_alloc`".
- **Defer IDL toolchain explanation.** The `nx_mm_ops`
  contract is presented as "auto-generated from the IDL
  source", with a forward pointer to chapter 10 for *how*
  the generator works. The chapter doesn't show the JSON
  source or the python tool — that's chapter 10's job.
- **Defer MMU details.** Chapter 6 (next, planned) covers
  the MMU. Chapter 5 mentions identity-mapping in one
  sentence ("virtual and physical mean the same thing
  numerically") and treats the user-window reserve as a
  forward pointer to chapter 6's address-space split.
- **No chapter-number reference for the future MMU chapter.**
  Per [BOOK-STYLE-GUIDE.md §"What not to do"](../BOOK-STYLE-GUIDE.md#what-not-to-do):
  unwritten chapters are referenced by topic, not by
  number. The chapter says "the next chapter on the MMU"
  / "a future chapter" without claiming to know which
  number it'll be.
- **No content changes to earlier chapters.** Chapters 1–4
  don't have forward references that point at PMM
  topics, so no patches are needed.
- **Hardcode `RAM_END = 0x80000000`.** Match the actual
  source. The chapter calls it "a placeholder" and
  forward-points the DTB-parsing replacement, but doesn't
  paper over the current state.

## Status at End of Session

- `book/05-physical-memory-and-pmm.md` drafted, 1123 lines,
  staged in `sources/nonux`.
- `book/README.md` chapter-5 link wired up, staged.
- `BOOK-OUTLINE.md` chapter-5 row updated `planned → shipped`,
  staged in `proj_docs/nonux`.
- `HANDOFF.md` Current Status, Latest session log, Next
  Actions, Session Logs list — to be updated in this commit.
- `HANDOFF-ARCHIVE.md` — session 122 to be rolled into archive
  per the keep-last-5 convention.
- No code changes. Tests not re-run; test counts unchanged
  from session 126 (sessions 124–126 were also doc-only).

**Part III started.** Five book chapters are now shipped (Part I
chapters 1–2, Part II chapters 3–4, Part III chapter 5). Chapter
6 (MMU) is the natural next chapter; per
[BOOK-OUTLINE.md §"Phase 9 dependency"](../BOOK-OUTLINE.md#phase-9-dependency)
it's the chapter most affected by an eventual Phase 9 rework, so
the per-chapter call between "draft now and revise later" vs
"draft after Phase 9 lands" is open.

## Next Steps

- Either continue the book with chapter 6 (Virtual memory and
  the MMU — first chapter that has a Phase-9-dependency
  decision attached) or start Phase 9 (per-process MM rework —
  L3 4 KiB pages, VMAs, demand paging, COW fork; see
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework)).

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/05-physical-memory-and-pmm.md` — new, 1123 lines.
- `book/README.md` — chapter-5 entry now a real link.

`proj_docs/nonux` (this commit):
- `BOOK-OUTLINE.md` — chapter 5 status `planned → shipped`.
- `HANDOFF.md` — Current Status, Latest session log line, Next
  Actions, Session Logs list updated.
- `HANDOFF-ARCHIVE.md` — session 122 rolled into archive.
- `logs/session-127-book-chapter-5.md` — new (this file).
