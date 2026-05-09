# Session 120: Book chapter 1 — boot and linker

**Date:** 2026-05-09
**Phase:** Phases 1–9b complete; post-9b fixes through Session 119; Phase 9 not yet started
**Branch:** master

---

## Goals

- Write a tutorial-style document explaining bootloaders, linker
  scripts, ARM64 boot addresses, ELF vs raw binary, and the
  compiler/linker/loader pipeline — aimed at first-time kernel
  readers.
- Restructure the doc as the start of a kernel-construction book
  rather than a one-off reference.

## What Was Done

### Initial draft

Created `docs/boot-and-linker.md` covering:

- Why a kernel needs a bootloader (no OS underneath, no loader,
  uninitialised RAM, etc.).
- QEMU's `-kernel` shortcut: 1 GiB RAM at `0x40000000`, kernel
  loaded at `0x40080000`, `x0 = DTB`, mini-bootloader stub.
- The ARM64 entry contract (EL2 → EL1 drop, MMU off, caches off,
  IRQs masked, SP undefined).
- What `start.S` has to do: detect EL, drop to EL1, set SP, zero
  BSS, branch to `boot_main`.
- The linker script (`core/boot/linker.ld`): `ENTRY`, `MEMORY`,
  `SECTIONS`, location counter, linker-defined symbols
  (`__stack_top`, `__bss_start/end`, `__free_mem_start`,
  `__start_nx_components` / `__stop_nx_components`), the
  linker-as-plugin-registry pattern.
- ELF vs raw binary: why we strip with `objcopy -O binary`.
- Full boot timeline diagram from `0x40080000` to scheduler
  startup.

### Revisions for readability

Three rounds of revision based on feedback:

1. **Beginner-friendly rewrite** — short sentences, a "Terms you'll
   see" section, restaurant-opening analogy as the doc's intro.
2. **Restaurant analogy fix** — "freezer hasn't been turned on" was
   wrong (freezers run 24/7); replaced with "coffee machine is
   unplugged" + "chairs stacked on tables".
3. **Standard terminology pass** — replaced invented metaphors with
   standard terms that are still plain English. Affected: ELF
   (dropped "moving box / packing list", uses "ELF header"); raw
   binary (dropped "dumped on the floor", says "no header, no
   metadata"); boot ROM (dropped "sticky note inside the front
   door"); interrupt (dropped "tap on the shoulder"); CPU (dropped
   "brain of the computer"); bootloader (dropped "morning shift
   program"). Restaurant intro analogy retained as the doc's
   framing.

### New section: Compiler, linker, loader, and sections

Inserted between "Terms you'll see" and "Why a kernel can't just
'run'" as foundational background. Four subsections:

- **The compiler — source code to object files.** What's in a `.o`,
  why it's incomplete on its own, the placeholder model.
- **The linker — object files to one executable.** `ld`'s four
  jobs: match uses to definitions, pick addresses, fill in
  placeholders, group by section. "undefined reference to ..."
  named.
- **The loader — executable to running program.** What the loader
  does (8 steps), then the punchline that bare-metal kernels have
  no loader, so `start.S` does the equivalent work.
- **Sections.** Permission table for `.text` / `.rodata` / `.data`
  / `.bss`; why-split explanation grounded in page permissions and
  security; three follow-on facts about how sections work in the
  toolchain (compiler picks the section,
  `__attribute__((section))` overrides, linker collects same-named
  sections).

### Restructured into a book

Moved `docs/boot-and-linker.md` to `book/01-boot-and-linker.md` and
added supporting structure:

- **`book/`** — new top-level directory under `sources/nonux/`.
- **`book/README.md`** — book index. Who it's for, how to read it,
  chapter table (currently just chapter 1), conventions used,
  cross-links to project README and `docs/`.
- **`book/01-boot-and-linker.md`** — the moved chapter. Sibling
  link to `framework-bootstrap.md` updated to point at
  `../docs/framework-bootstrap.md`; other relative links (`../core/...`)
  unchanged because `book/` and `docs/` are at the same depth.
- **`docs/README.md`** — dropped the "Background" mini-table that
  pointed at the moved file; added a one-paragraph pointer to
  `book/` for tutorial-style material.
- **`README.md`** (top-level) — added a `book/` row to the
  Directory Structure table.

`git mv` preserves history; the move shows up as a rename with
content changes (since the readability/section revisions landed
in the same logical change set).

## Key Findings

- **The "8th-grade readable" target is enforced by removing
  invented metaphors in favour of standard terms.** The first
  rewrite leaned heavily on metaphors (ELF as a "moving box with a
  packing list"); user feedback was that standard terms like "ELF
  header" and "ELF's structure" are clearer *and* searchable, while
  staying in plain English. Standard terminology + short
  sentences > clever metaphors.
- **Restaurant analogy passes the freezer test.** A fact-check
  revealed the original analogy mentioned turning the freezer on,
  which doesn't happen daily. Lesson: even framing analogies need
  to match real-world experience.
- **`book/` belongs at the top level, not under `docs/`.** Reasoning:
  the book is a major artifact for learners and deserves visibility
  from the project root, peer to source code rather than nested in
  reference docs. `docs/` is the manual; `book/` is the guided
  tour.

## Decisions Made

- **Top-level `book/` directory** rather than `docs/book/`. Makes
  the book a first-class part of the repo.
- **Numeric chapter prefixes** (`01-boot-and-linker.md`). Reading
  order is visible from the file listing without needing to consult
  a TOC.
- **Restaurant analogy retained as intro.** It serves as
  motivation, not as a technical claim, so a single fact-check fix
  was enough; we didn't need to remove it entirely.

## Status at End of Session

- Chapter 1 written, revised three times, restructured into a book
  layout (`book/01-boot-and-linker.md` + `book/README.md`).
- No code changes; tests not re-run. Test counts unchanged from
  Session 118: 102/102 tools · 485/485 host · 152/152 kernel.
- Source repo has two commits this session:
  - `48e8b34` — initial draft as `docs/boot-and-linker.md`.
  - (this session's rollup) — revisions + move into `book/` + book
    index + README updates.

## Next Steps

- Phase 9 — per-process MM rework (L3 4 KiB pages, VMAs, demand
  paging, COW fork). See
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework).
- Future book chapters as more nonux subsystems are explained.
  Natural next chapters: process/scheduler/MMU/PMM/IPC/components.

---

**Files Changed (sources/nonux):**

This session's two source-repo commits (initial draft + rollup):

- `docs/boot-and-linker.md` — created in commit `48e8b34`, then
  moved to `book/01-boot-and-linker.md` in the rollup.
- `book/01-boot-and-linker.md` — final location of chapter 1
  (~860 lines).
- `book/README.md` — new; book index / TOC.
- `docs/README.md` — Background section removed; pointer to
  `book/` added.
- `README.md` — Directory Structure table got a `book/` row.
