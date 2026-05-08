# Session 118: Subdirectory README files

**Date:** 2026-05-08
**Phase:** Phases 1–9b complete; post-9b fixes through Session 117; Phase 9 not yet started
**Branch:** master

---

## Goals

- Add a README.md to every top-level subdirectory in `sources/nonux/` so each
  directory is self-documenting.
- Add a "Directory Structure" table to the top-level `sources/nonux/README.md`
  linking to each subdirectory README.

## What Was Done

### Subdirectory README files created

Eight new README files, one per subdirectory that lacked one:

| File | Content |
|------|---------|
| `core/README.md` | Describes each of the 8 `core/` subdirectories (`boot`, `cpu`, `irq`, `lib`, `mmu`, `pmm`, `sched`, `timer`) with per-file role descriptions. |
| `framework/README.md` | Describes all ~30 `.c/.h` files grouped by role: composition core (bootstrap, registry, component, recompose, config), IPC (ipc, dispatcher, slot_call, channel), IDL-generated shims (char_device/fs/mm/scheduler/vfs call and dispatch headers), and OS services (syscall, process, handle, hook, elf, console). |
| `components/README.md` | Describes all 8 components (`mm_buddy`, `posix_shim`, `procfs`, `ramfs`, `sched_priority`, `sched_rr`, `uart_pl011`, `vfs_simple`) — slot filled, purpose, and design notes. |
| `interfaces/README.md` | Explains the generated/hand-authored split, lists interface headers and message struct files, documents `fs_types.h`, and describes the `idl/` JSON source files. |
| `lib/README.md` | Explains `libnxlibc/` contents (`posix.h`, `crt0.S`, `nxlibc.c/.h`, `manifest.json`) and the Session 117 relocation history. |
| `gen/README.md` | Documents the four generated files (`config.h`, `sources.mk`, `slot_table.c`, `posix_shim_deps.h`) and their generators. Note: `gen/` is gitignored so this README is not tracked in the source repo. |
| `third_party/README.md` | Describes `busybox/` (ARM64 static binary) and `musl/` (libc headers + library) and how tests consume them. |
| `test/README.md` | Describes all four tiers (`host/`, `kernel/`, `interactive/`, `bench/`) with per-file/per-group descriptions, including conformance suites, EL0 program blobs, and interactive script pairs. |

### Top-level README updated

Added a "Directory Structure" table between "What is this?" and
"Documentation" sections, linking to every subdirectory README.

### HTML conversion

All new/modified markdown files converted to HTML by `scripts/convert_kb.sh`.

## Key Findings

- `gen/` is gitignored in `sources/nonux/.gitignore` — `gen/README.md` is
  created on disk but not committed to the source repo. This is correct
  behaviour (the gen/ directory is intentionally untracked).
- `tools/README.md` and `docs/README.md` already existed and were not
  modified.
- Component READMEs already existed for most components under
  `components/*/README.md`; the new `components/README.md` is an index one
  level up.

## Status at End of Session

- All 8 subdirectory READMEs created and HTML-converted.
- Top-level README updated with directory structure table.
- `make test` not re-run (no code changes — documentation only).
- Test counts unchanged from Session 117: 102/102 tools · 485/485 host · 152/152 kernel.

## Next Steps

- Phase 9 — per-process MM rework (L3 4 KiB pages, VMAs, demand paging, COW fork).
  See [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework).

---

**Files Changed:**
- `sources/nonux/README.md` — added Directory Structure table
- `sources/nonux/README.html` — regenerated
- `sources/nonux/core/README.md` — new
- `sources/nonux/core/README.html` — new
- `sources/nonux/framework/README.md` — new
- `sources/nonux/framework/README.html` — new
- `sources/nonux/components/README.md` — new
- `sources/nonux/components/README.html` — new
- `sources/nonux/interfaces/README.md` — new
- `sources/nonux/interfaces/README.html` — new
- `sources/nonux/lib/README.md` — new
- `sources/nonux/lib/README.html` — new
- `sources/nonux/third_party/README.md` — new
- `sources/nonux/third_party/README.html` — new
- `sources/nonux/test/README.md` — new
- `sources/nonux/test/README.html` — new
