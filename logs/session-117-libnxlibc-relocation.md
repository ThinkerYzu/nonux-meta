# Session 117: `libnxlibc` Relocation to `lib/`

**Date:** 2026-05-07
**Phase:** Post-9b (housekeeping)
**Branch:** master

---

## Goals

- Move `libnxlibc` out of `components/` into a top-level `lib/` directory, since it is a userspace library (EL0 code), not a kernel slot implementation.

## What Was Done

### Move and path updates

Created `lib/` at the repo root and moved `components/libnxlibc/` → `lib/libnxlibc/`.

Updated all path references across 37 files:
- `Makefile` — all `components/libnxlibc` occurrences (build rules, `-L` flags, prerequisite lists)
- 34 C/header/markdown files in `test/kernel/` and `lib/libnxlibc/` — `#include "components/libnxlibc/..."` → `#include "lib/libnxlibc/..."`

## Key Findings

- `libnxlibc` has no `kernel.json` slot binding and is never compiled into `kernel.bin` — it is a purely EL0 static archive (`libnxlibc.a`) and runtime bootstrap (`crt0.S`), making `components/` the wrong home.
- `components/` should hold only kernel slot implementations (those with a manifest and a `kernel.json` binding).
- `lib/` is the natural top-level directory for userspace libraries; it can accommodate future EL0 libraries alongside `libnxlibc`.

## Decisions Made

- **`lib/libnxlibc/`** — top-level `lib/` preferred over `userspace/lib/` (flatter, matches common project conventions) and over `core/lib/` (which is kernel-only EL1 code).

## Status at End of Session

- Build: clean (`make` → 0 errors, `verify-registry` 0 findings, `verify-iface-fresh` 0 drift)
- `make test-tools` → **102/102 pass**
- `make test-host` → **485/485 pass**
- `make test-kernel` → **152/152 pass**

## Next Steps

Continue with Phase 9 (per-process MM rework — L3 4 KiB pages, VMAs, demand paging, COW fork).

---

**Files Changed:**
- `lib/libnxlibc/` — directory (moved from `components/libnxlibc/`)
- `Makefile` — all `components/libnxlibc` → `lib/libnxlibc`
- `test/kernel/*.c` (34 files) — include paths updated
- `lib/libnxlibc/nxlibc.c`, `lib/libnxlibc/README.md` — self-references updated
