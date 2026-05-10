# Session 122: Book outline (18 chapters in 9 parts)

**Date:** 2026-05-09
**Phase:** Phases 1–9b complete; post-9b fixes through Session 119; Phase 9 not yet started
**Branch:** master

---

## Goals

- Plan the full scope of the [nonux book](../../sources/nonux/book/README.md)
  before more chapters land.  Chapter 1 was written without a
  surrounding plan; subsequent chapters need a known structure so
  they can forward-reference earlier material and avoid covering the
  same ground twice.

## What Was Done

### Outline iteration

Three iteration rounds with the user before settling:

1. **First proposal** — 18 chapters in 8 parts.  IPC + storage
   (Part V) before the component framework (Part VII), userspace
   between (Part VI: Processes + Syscalls).
2. **First adjustment** — userspace moved later, just before
   userspace runtime/build.  IPC + storage and the framework
   ended up adjacent in the middle.
3. **Second adjustment** — the framework moved *before* IPC +
   storage (because IPC is slot calls and FS is component-routed,
   so they depend on the framework), then a *split* of the
   framework Part: basics (slots, components, registry, IDL)
   before IPC, advanced (hooks, recomposition) after.  Final
   structure 18 chapters in 9 parts.

### `proj_docs/nonux/BOOK-OUTLINE.md` added

Canonical reference for the book's scope.  ~210 lines covering:

- **Status legend** (`shipped` / `draft` / `planned`).
- **Chapter list** as a per-Part table with title, status, and a
  one-line "reader sees" column citing real source paths.
- **Reading dependencies.**  Explicit callouts for the
  framework-before-IPC decision, hooks-after-IPC, processes
  needing MMU + threads, syscalls needing exceptions + processes,
  POSIX shim needing syscalls.
- **Phase 9 dependency.**  Chapter 4 (MMU) and parts of chapter
  15 (Processes) describe behaviour that's slated to change in
  Phase 9 (per-process MM rework).  Two options noted (draft after
  vs draft pre-Phase-9 + revise) with the decision deferred per
  chapter.
- **Length and pace.**  Reaffirms 600–1500 lines per chapter from
  the style guide.  Chapter 1's 968 lines is the working data
  point.
- **Per-chapter workflow.**  Six steps from outline through
  session log.

### `sources/nonux/book/README.md` updated

Replaced the one-row chapter table with the full 18-chapter plan
grouped by 9 parts.  Convention: shipped = linked, planned =
plain text.  Appendices A–D listed at the end as planned.

The split between `BOOK-OUTLINE.md` and `book/README.md` is
deliberate: planning meta-info (dependencies, Phase 9 caveat,
workflow) lives in `proj_docs`, while readers of the book see
just scope and TOC.

## Final structure

```
Part I    — Coming up                       1. Boot/linker (shipped) · 2. Console
Part II   — Memory                          3. PMM · 4. MMU
Part III  — Time and interrupts             5. Exceptions/GIC/IRQs · 6. Timer
Part IV   — Tasks                           7. Threads · 8. Scheduler
Part V    — Framework basics                9. Slots/components/registry · 10. IDL
Part VI   — IPC and storage                 11. IPC · 12. FS
Part VII  — Framework, advanced             13. Hooks · 14. Recomposition + runtime config
Part VIII — Userspace                       15. Processes · 16. Syscalls
Part IX   — Userspace runtime and build     17. POSIX shim · 18. Build/test
```

Plus appendices A (ARM64 cheat sheet), B (Glossary), C (Source map),
D (Where to read more).

## Key Findings

- **Dependency direction matters more than user-facing order.**
  The original instinct was to put IPC + storage early (those are
  the most "kernel-y" topics readers expect).  But IPC in nonux
  literally *is* slot calls and FS is component-routed, so
  reversing to framework-first is the only way the IPC chapter
  can avoid hand-waving.  The reorder pays off across multiple
  chapters: hooks attach to slot-call boundaries (now defined),
  recomposition acts on running compositions (now exemplified by
  IPC), processes use slot-routed syscalls (now coherent).
- **Splitting the framework Part into basics + advanced gives
  every chapter concrete examples.**  Hooks before IPC = hooks
  with nothing to hook.  Hooks after IPC = hooks attached to
  visible slot calls.  Same for recomposition.

## Decisions Made

- **18 chapters in 9 parts.**  Big book; chapters land when they're
  written, no schedule.
- **`book/README.md` lists planned chapters without links;
  `BOOK-OUTLINE.md` carries the planning meta.**  Avoids dangling
  links in the source repo while keeping the canonical plan
  discoverable to authors.
- **Phase 9 chapters can be drafted either way.**  Per-chapter
  decision recorded in that chapter's session log when it lands.

## Status at End of Session

- `BOOK-OUTLINE.md` written; `book/README.md` updated.
- No code changes; tests not re-run.  Test counts unchanged from
  Session 118: 102/102 tools · 485/485 host · 152/152 kernel.

## Next Steps

- Phase 9 — per-process MM rework.  See
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework).
- Next book chapter (chapter 2 — The console: kprintf and the
  UART) when the user is ready.

---

**Files Changed:**

`proj_docs/nonux`:
- `BOOK-OUTLINE.md` — new (~210 lines).

`sources/nonux`:
- `book/README.md` — chapter table replaced with full 18-chapter
  plan grouped by 9 parts.
