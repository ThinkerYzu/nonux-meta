# Session 97: Slice 8.0e — verify-registry R9 (ban iface_ops outside dispatcher)

**Date:** 2026-05-03
**Phase:** Phase 8 — Runtime Recomposition, Group B (IPC migration)
**Branch:** master
**Commit:** eb76163

---

## Goals

- Implement slice 8.0e: add R9 machine check to `verify-registry.py` that
  forbids `->iface_ops` reads outside `framework/dispatcher.c`.
- Rule should light up green immediately (all production paths already migrated).
- Add 9 unit tests for R9 in `tools/tests/test_verify_registry.py`.

---

## What Was Done

### 1. R9 rule added to `tools/verify-registry.py`

New machine-checked rule R9: "No iface_ops outside dispatcher."

**Three exemption categories:**

1. **By filename** — `framework/dispatcher.c` (intended home) and
   `framework/bootstrap.c` (one-time scheduler-init publication; kernel path).
2. **`#if __STDC_HOSTED__` guards** — host-build fast paths in `vfs_call.c`
   and `fs_call.c`.  Tracked via a simple preprocessor-directive stack in
   `_hosted_line_set()`.  `#else` / `#elif` flip the branch flag so the kernel
   side of `#if __STDC_HOSTED__` / `#else` is correctly not exempt.
3. **Comment lines** — `_iface_ops_in_comment()` skips lines where
   `->iface_ops` appears after `//` or where the content before the token
   starts with `*` / `/*` (block-comment line style).  Needed because
   `components/ramfs/ramfs.c:28` has `->iface_ops` in a doc-comment.

**Scope:** `framework/*.c` and `components/**/*.c` via `rglob`.  `test/`
is excluded — test code legitimately introspects slot/descriptor state.

### 2. New `REPO_CHECK_FNS` dispatch table + `run_checks` restructure

Per-component checks (R2, R4) stay in `CHECK_FNS` and run per-component.
R9 is a repo-level check that runs once; dispatched through new
`REPO_CHECK_FNS` dict.  `run_checks` now handles both loops, deriving
`repo_root = components_dir.parent` (so the Makefile's
`$(VERIFY) components/` invocation keeps working as-is).

### 3. `tools/tests/test_verify_registry.py` — 9 new R9 tests

`TestR9` class covers:
- Clean tree passes
- `dispatcher.c` and `bootstrap.c` exemptions pass
- `#if __STDC_HOSTED__` guard exemption passes
- `#if __STDC_HOSTED__` / `#else` correctly fails (kernel side of the else)
- Block-comment line passes
- Bare `->iface_ops` in framework file fails
- Bare `->iface_ops` in component file fails
- Missing `framework/` dir passes gracefully

### 4. Makefile comment updated

`R1-R8` → `R1-R9`; removed stale "no-op right now" text.

---

## Test Results

```
make verify-registry   →  0 finding(s); ran R2,R4,R9; ai-verified R1,R3,R5,R6,R7,R8
make test-tools        →  102/102 pass  (93 prior + 9 new R9 tests)
```

(Host, interactive, and kernel tests unchanged — no production code modified.)

---

## Key Decisions

- **Scope: framework + components only, not test/** — test files legitimately
  read `iface_ops` to assert composition state (e.g. `ktest_vfs.c`).  The
  rule's purpose is to prevent production dispatch code from bypassing the
  IPC wrappers, not to police test introspection.
- **`bootstrap.c` exemption by filename** — the scheduler-init publication
  (`sched_init(sops, ...)` in `framework/bootstrap.c`) is a one-time boot
  operation explicitly permitted by DESIGN.md §Scheduler.  Moving it into
  `dispatcher.c` would be artificial; a named exemption is clearer.
- **Simple `#else` flip vs full evaluator** — the only patterns in the codebase
  are `#if __STDC_HOSTED__` / `#endif` and `#if __STDC_HOSTED__` / `#else` /
  `#endif`.  A full `#elif` evaluator adds complexity for zero benefit.
- **`_iface_ops_in_comment` heuristic** — the only false positive in the real
  codebase is `ramfs.c:28` (a doc-comment backtick-quoted example).  Stripping
  comments properly would require a C parser; the leading-`*` heuristic covers
  the actual pattern without that complexity.

---

## Next Step

**Group C begins.  Slice 8.1** — lift pause/drain/resume from host test
infrastructure into the kernel build; roll back the slice-3.9 inline workaround.
