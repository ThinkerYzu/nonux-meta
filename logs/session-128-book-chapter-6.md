# Session 128: Book chapter 6 — Virtual memory and the MMU

**Date:** 2026-05-10
**Phase:** Phases 1–9b complete; book chapters 1–6 shipped (Part I + Part II + Part III); Phase 9 not yet started
**Branch:** master

---

## Goals

- Draft the next book chapter, picking up where chapter 5 left
  off (we have a page allocator; now we need to look at the
  MMU that makes the addresses it hands out *mean* something).
- Per [BOOK-OUTLINE.md](../BOOK-OUTLINE.md#chapter-list), that's
  **chapter 6: Virtual memory and the MMU** — the second (and
  final) chapter of Part III (Memory).
- Resolve the per-chapter "draft now vs after Phase 9 lands"
  call attached to chapter 6.

## What Was Done

### `sources/nonux/book/06-virtual-memory-and-mmu.md` (new, 1247 lines)

Walks the path from "what virtual memory even is" through the
descriptor format, MAIR/TCR/SCTLR setup, the MMU enable
sequence, the user window, per-process address spaces, and the
fork-time TTBR0 juggling — closing with the cross-link back to
chapter 5's `pmm_reserve_range` mystery.

Section structure:

1. **Intro** (4 paragraphs) — frames the gap (we've been
   loose about "address" so far), states the chapter's main
   claim (every memory access is translated; we're going to
   walk the table the CPU walks), promises the reader will
   be able to read `core/mmu/mmu.c` line by line.
2. **Files** — `core/mmu/{mmu.h,mmu.c}`, `core/boot/boot.c`,
   `core/cpu/el0_entry.S`, plus a forward-pointer to the
   ARM ARM at the end.
3. **Terms** — 17 entries: VA, PA, MMU, page table,
   descriptor, granule, translation regime, TTBR0/TTBR1,
   MAIR, TCR, SCTLR, identity map, AP, PXN, UXN, AF, TLB,
   ISB+DSB, eret. Each is the standard term + a one-paragraph
   explanation, per the style guide's terminology rules.
4. **Why turn on the MMU at all** — three reasons in
   increasing-importance order: caching/ordering,
   permissions, address-space isolation. Side note on
   *why turn it on so early* (avoid the cache-coherence
   transition pain that would come from running cache-off
   for any meaningful kernel work first).
5. **What we're going to build** — names the three knobs (4
   KiB granule, 39-bit VA, three levels) and walks the VA
   bit-layout diagram (L1 idx / L2 idx / L3 idx / offset).
   Forward-points the Phase 9 rework at the right level
   (concept stays; specifics will revise).
6. **Descriptors: the bit-level layout** — 64-bit descriptor
   diagram + bit-by-bit explanation of V, Tbl/Blk, AttrIdx,
   AP, SH, AF, PA, PXN, UXN. Then the three nonux helpers
   `device_block`, `normal_block`, `user_block` walked
   line-by-line, with a "three things to notice" callout
   (`device_block` is PXN+UXN, `normal_block` is UXN-only,
   `user_block` clears UXN).
7. **MAIR: indirection for memory types** — explains *why*
   the indirection exists (8-bit attributes are too big for
   every descriptor; 8 indices are enough), shows the two
   slots nonux populates (Device-nGnRnE at index 0,
   Normal WB-WA at index 1), notes the rest stay zero.
8. **TCR: telling the MMU what to expect** — bit-by-bit
   decode of `TCR_VALUE`: T0SZ=25, IRGN0/ORGN0/SH0,
   TG0=4 KiB, EPD1=1 (TTBR1 disabled — important for the
   chapter's later "no high-half kernel" callout), IPS=36-bit.
9. **The tables themselves** — `static uint64_t l1_table[512]`
   etc., the L2 population loop, the L1 wiring with two table
   descriptors. Side note on "we wrote VA into the L1 entry,
   not PA — only correct because identity map" (lays the
   groundwork for why the future high-half-kernel rework
   needs more care).
10. **The walk, by example** — concrete walk of VA
    `0x4009_0000` (a kernel-text address) and VA
    `0x0900_1000` (the PL011 `UARTDR`) through the table,
    showing how Normal-vs-Device classification falls out
    of the same machinery.
11. **Turning the MMU on** — the SCTLR sequence walked
    step by step: `dsb ish` → `msr mair_el1` → `msr tcr_el1`
    → `msr ttbr0_el1` → `isb` → `tlbi vmalle1` → `dsb ish`
    → `ic iallu` → `dsb ish` → `isb` → SCTLR write with
    M+C+I → final `isb`. Side note on *why the kernel
    survives its own MMU enable* (identity map; if we
    weren't identity-mapped, would need a trampoline).
12. **One more boot-time tweak: FP/SIMD** — `CPACR_EL1.FPEN`
    set so musl's FP-using `memset`/`memcpy` don't trap.
    Notes the kernel uses `-mgeneral-regs-only` so it
    doesn't itself touch FP.
13. **The user window** — slot 64 in `l2_ram_table` upgraded
    to a `user_block`. Walks the address arithmetic
    (`64 << 21` = `0x0800_0000`, so window VA =
    `RAM_BASE + 0x0800_0000` = `0x4800_0000`). Connects
    to `drop_to_el0` in `core/cpu/el0_entry.S` showing
    how the AP/UXN bits enforce the EL0 demotion.
14. **Per-process address spaces** — walks
    `mmu_create_address_space`'s three allocations (L1
    page, L2_ram page, 8 MiB user backing). The
    "shared kernel mappings, private user window"
    invariant is shown as a 2-table ASCII diagram.
    Side note flags the Phase 9 rework: the *concept*
    (shared kernel + private bottom 2 GiB) stays; the
    *implementation* (8 MiB block-descriptor backing)
    changes.
15. **Switching address spaces** —
    `mmu_switch_address_space` walked
    line-by-line: `msr ttbr0_el1` → `isb` → `tlbi vmalle1`
    → `dsb ish` → `isb`. Why it's safe to call from
    kernel mode (kernel mappings byte-identical across
    every L1). Side note on ASIDs being a future
    optimisation we don't have yet.
16. **Fork's user-window memcpy** —
    `mmu_copy_user_backing` walked, with the central
    insight: the caller's TTBR0 has the user window
    pointing at the *caller's* PA; we have to switch to
    the kernel TTBR0 for the duration of the memcpy
    so identity-map assumptions hold. The save/restore
    pattern is shown. Side note on COW being what
    eliminates the memcpy entirely (Phase 9).
17. **A boot-time consequence: PMM reservation** —
    closes the loop on chapter 5's
    `pmm_reserve_range(mmu_user_window_base(), …)`
    forward-reference. Shows why the bug would be
    intermittent and why the reservation makes it
    impossible.
18. **How it all fits together at runtime** — full ASCII
    diagram showing physical RAM, kernel-VA identity map
    L1, per-process L1 sharing kernel mappings + private
    user_backing.
19. **A few extra things to know** — seven side notes:
    no high-half kernel yet, no ASIDs yet, no demand
    paging yet, page tables come from kheap (and that's
    why the PMM reservation matters), one MMU per core,
    MMU also handles unaligned access, we never read
    descriptors back, MMU is also what makes
    `kprintf("%s", bad_ptr)` produce a clean abort
    instead of silent corruption.
20. **Where to read more** — source links + chapter 1 EL
    cross-ref, chapter 5 `pmm_reserve_range` cross-ref,
    ARM ARM, Linux's `arch/arm64/mm/proc.S` and
    `pgtable.h`, QEMU `hw/arm/virt.c`.

Length: 1247 lines — comfortably in the 600–1500 band.
Slightly longer than chapter 5 (1123) because the bit-level
descriptor decode + the bring-up sequence + the fork
juggling are all multi-paragraph stories.

### Cross-link patches in chapter 5

Two small patches to `book/05-physical-memory-and-pmm.md` that
turn its "next chapter on the MMU" forward-references into
real links now that chapter 6 is shipped:

- **Terms section MMU bullet** — "We'll explore the MMU
  properly in the next chapter" → "[Chapter 6](06-virtual-memory-and-mmu.md)
  walks the MMU properly".
- **§"Reserving a range"** — "The full reason will make
  more sense after the next chapter on the MMU, but the
  short version is…" → "The full reason is in
  [chapter 6 §"A boot-time consequence: PMM reservation"](06-virtual-memory-and-mmu.md#a-boot-time-consequence-pmm-reservation).
  Short version: …".

### Index / outline updates

- **`book/README.md`** — chapter 6 entry under Part III
  changed from plain text to
  `[Virtual memory and the MMU](06-virtual-memory-and-mmu.md)`.
- **`BOOK-OUTLINE.md`** — chapter 6 row updated:
  `planned → shipped`. The "see Phase 9 dependency below"
  parenthetical is replaced with "Drafted pre-Phase-9;
  revision pass scheduled when Phase 9 lands" — recording
  the call we made this session as a durable annotation
  on the outline.

## Key Findings

- **Phase 9 dependency choice: draft now (option 2 in the
  outline).** The architectural concepts of chapter 6 — the
  page-table walk, the descriptor format, MAIR/TCR/SCTLR,
  the identity map, the EL0 permission story — aren't
  changing in Phase 9. What changes is the *per-process*
  half: 8 MiB block-descriptor backing → L3 page-descriptor
  backing with VMAs and demand paging; byte-copy fork → COW
  fork. The chapter accurately describes the current state
  and flags both sections as places future-Phase-9 will
  revise. The alternative (defer the chapter until Phase 9
  lands) would have left Part III sitting half-written for
  an unknown amount of time, with chapter 5 no longer
  resolving its MMU forward-references.
- **The user-window aliasing rationale wants a real
  cross-link to land.** Chapter 5's
  `pmm_reserve_range(mmu_user_window_base(), …)` paragraph
  was deliberately kept short with a "next chapter will
  explain" forward-pointer; chapter 6's §"A boot-time
  consequence: PMM reservation" is exactly that
  explanation. Closing the loop with two patches in
  chapter 5 keeps the book usable as a sequence.
- **The fork TTBR0-switch trick is the most surprising
  detail.** Most readers will expect "fork copies bytes"
  and not realise that under the *parent's* TTBR0 the
  kernel's identity-map assumption is violated for VAs in
  the user window's PA range. Calling out the save/restore
  pattern explicitly is one of the chapter's most
  load-bearing pieces of writing — it's not in the ARM ARM
  and isn't immediately obvious from `mmu_copy_user_backing`'s
  comment alone.
- **The "kernel survives its own MMU enable" question is
  worth a side note.** A reader's first reaction to seeing
  the SCTLR.M flip is usually "wait, doesn't the next
  instruction fetch hit a now-translated address — what if
  it doesn't translate?" Identity map is the answer; making
  it explicit anchors the design choice in the same way
  chapter 5 anchored the "single-CPU still needs atomics"
  rationale.

## Decisions Made

- **Cover MMU enable, descriptors, *and* per-process spaces
  in one chapter.** The three are conceptually one story —
  the descriptor bits motivate the MAIR/TCR setup, the
  setup motivates the SCTLR flip, the SCTLR flip motivates
  the user window, the user window motivates per-process
  spaces. Splitting any of them off would have left a
  chapter ending mid-thought.
- **Defer process-table / `nx_process` framework.**
  Chapter 6 sticks to the *MMU* side of "address space" —
  the L1 root and what's allocated alongside it. Process
  state, scheduling integration, the EL0 transition path
  end-to-end live in the future Processes chapter
  (currently chapter 15 in the outline). Chapter 6 names
  `drop_to_el0` and treats it as "the one-way demotion";
  it doesn't show the full syscall return path.
- **Defer hooks / IPC / framework registry.** No mention
  of `nx_components`, `kernel.json`, the slot/component
  vocabulary. The MMU is core code, not a component, so
  the chapter can stay focused on architecture +
  bring-up + per-process layout.
- **No chapter-number references for unwritten future
  chapters.** Per
  [BOOK-STYLE-GUIDE.md §"What not to do"](../BOOK-STYLE-GUIDE.md#what-not-to-do):
  no "see chapter 15" or "as we'll cover in chapter 9" —
  just "the future chapter on processes" / "a future
  rework". Existing shipped chapters get real links
  (chapter 1's EL section, chapter 5's `pmm_reserve_range`).
- **Phase 9 dependency lives in *prose* and the *outline*,
  not in a banner at the top of the chapter.** A reader of
  the book doesn't need to track the project's roadmap.
  The two sections that will be revised mention "a future
  per-process MM rework" / "Phase 9" in passing as
  forward-pointers, but no big "this chapter is
  preliminary" disclaimer. The roadmap-flavoured language
  ("Phase 9", "block-descriptor backing → L3 with VMAs")
  is concentrated in the outline + this session log.
- **Hardcode the actual numbers.** Window VA `0x4800_0000`,
  4 × 2 MiB = 8 MiB window, `USER_WINDOW_INDEX = 64`,
  `T0SZ = 25`, `TCR.EPD1 = 1`, etc. The chapter calls
  these "the values we use today" rather than abstracting
  away — same approach chapter 5 took with `RAM_END =
  0x80000000`.

## Status at End of Session

- `book/06-virtual-memory-and-mmu.md` drafted, 1247 lines,
  staged in `sources/nonux`.
- `book/05-physical-memory-and-pmm.md` patched with two
  forward-reference cross-links to chapter 6, staged.
- `book/README.md` chapter-6 entry now a real link, staged.
- `BOOK-OUTLINE.md` chapter-6 row updated `planned → shipped`
  with the Phase-9-dependency annotation, staged in
  `proj_docs/nonux`.
- `HANDOFF.md` Current Status, Latest session log, Next
  Actions, Session Logs list — to be updated in this commit.
- `HANDOFF-ARCHIVE.md` — session 123 to be rolled into
  archive per the keep-last-5 convention.
- No code changes. Tests not re-run; test counts unchanged
  from session 127 (sessions 124–128 are all doc-only).

**Part III complete.** Six book chapters are now shipped
(Part I chapters 1–2, Part II chapters 3–4, Part III chapters
5–6). Chapter 7 (Kernel threads and context switching, first
chapter of Part IV — Tasks) is the natural next.

## Next Steps

- Either continue the book with chapter 7 (Kernel threads
  and context switching — first chapter of Part IV) or
  start Phase 9 (per-process MM rework — L3 4 KiB pages,
  VMAs, demand paging, COW fork; see
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework)).
- When Phase 9 *does* land, chapter 6 needs a revision
  pass: §"Per-process address spaces" updates to L3
  pages + VMA list, §"Fork's user-window memcpy" updates
  to COW-clone, §"A boot-time consequence: PMM
  reservation" likely shrinks or disappears (the user
  window may move to a high-half VA range that doesn't
  alias the identity map). The "draft now" decision
  taken this session is what makes that follow-up
  necessary.

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/06-virtual-memory-and-mmu.md` — new, 1247 lines.
- `book/05-physical-memory-and-pmm.md` — two forward-reference
  cross-links to chapter 6 wired up (Terms MMU bullet,
  §"Reserving a range").
- `book/README.md` — chapter-6 entry now a real link.

`proj_docs/nonux` (this commit):
- `BOOK-OUTLINE.md` — chapter 6 status `planned → shipped` +
  Phase-9-dependency annotation on the row.
- `HANDOFF.md` — Current Status, Latest session log line, Next
  Actions, Session Logs list updated.
- `HANDOFF-ARCHIVE.md` — session 123 rolled into archive.
- `logs/session-128-book-chapter-6.md` — new (this file).
