# Session 13: verify-registry.py static checker (slice 3.7)

**Date:** 2026-04-21
**Phase:** 3 — Component framework
**Branch:** master

---

## Goals

- Land the static-checker build gate described in DESIGN.md §AI
  Verification. DESIGN enumerates eight rules (R1–R8) that every
  component must pass before being considered complete; this slice
  implements what can be checked honestly with regex-level analysis
  today and wires the result into `make` / `make test` so violations
  block the build.
- Do not pretend to enforce rules that need real C parsing or
  call-graph analysis — mark them `deferred` in the tool output
  rather than silently passing them.

## What Was Done

### `tools/verify-registry.py` (~230 LOC, stdlib only)

Walks `components/<name>/` directories with a `manifest.json` and
runs each enabled rule check; every finding becomes a line of output
on stderr with `<path>:<line> R<n>: <message>` when line info applies.

**Rule coverage (see `--list` for the live status):**

| Rule | Status        | Check                                     |
|------|---------------|-------------------------------------------|
| R1   | deferred      | fabricated slot pointers — needs clang / pycparser dataflow |
| **R2** | **implemented** | every `struct nx_slot *<field>;` in a component's `.c` files matches a `requires` / `optional` entry in the manifest; flags drift in both directions |
| R3   | deferred      | cap-only slot transfer — needs interface message schemas (not defined until 3.8+) |
| **R4** | **implemented** | `nx_slot_ref_retain(` and `nx_slot_ref_release(` call counts must match per component |
| R5   | deferred      | sender owns slot caps it passes — needs symbolic held-refs tracking |
| R6   | deferred      | handler doesn't stash borrowed caps — needs borrowed-cap → assignment-sink dataflow |
| R7   | deferred      | regenerate-and-diff presupposes gen/ is checked in; generator determinism is already covered by `tools/tests/test_gen_config.py::*determinism*` |
| R8   | deferred      | slot-resolve locality — needs ISR/kthread call-graph analysis |

**R2 details.** Regex `^\s+struct\s+nx_slot\s*\*\s*(IDENT)\s*;` matches
properly-indented struct-field declarations and skips column-0
file-scope declarations (which are usually forward-declared global
handles or helpers, not fields). One specific test guards against
the column-0 false-positive case.

**R4 details.** Counts `nx_slot_ref_retain(` and `nx_slot_ref_release(`
occurrences per component. Mismatched counts produce one finding per
excess call, pointing at the unpaired site. Doesn't prove the release
is reachable from `disable` / `destroy` (that needs control-flow
analysis, slice 3.8+), but catches the common "forgot to release at
all" bug.

**CLI:**

```
verify-registry.py [COMPONENTS_DIR]     default: components/
verify-registry.py --list               list rules + status
verify-registry.py --rule R2 [DIR]      run one rule only (repeatable)
```

Exit `0` on no findings (including empty `components/`), `2` on any
finding or unknown rule. Summary line on stdout so CI can grep it:
`verify-registry: N finding(s); ran R2,R4; deferred R1,...`.

### Makefile wiring

- `verify-registry` target added; runs `$(PYTHON)
  tools/verify-registry.py components/`.
- Wired as a **prerequisite** of both `all` (→ `kernel.bin`) and
  `test` (→ `test-tools test-host test-kernel`). Today this is a
  zero-cost no-op because `components/` is empty; when real
  components land in Phase 4+, violations will block the build
  without any further Makefile changes.
- `PYTHON` resolves to the venv interpreter but `verify-registry.py`
  itself is stdlib-only, so the gate doesn't force `make venv` on
  users who haven't touched the schemas. The venv is still required
  for `make test` because `test-tools` runs Python unit tests that
  import `jsonschema`-backed `validate-config.py`.

### Test coverage

16 new Python tests under
`tools/tests/test_verify_registry.py`:

- `TestListCommand` — `--list` prints every rule tag.
- `TestEmptyTree` — missing and empty `components/` both exit 0.
- `TestR2` — matching struct + manifest passes; field without
  manifest entry fails; manifest entry without field fails;
  optional treated like requires; column-0 `struct nx_slot *`
  declarations are not false-positives.
- `TestR4` — matched retain/release passes; retain without release
  fails; release without retain fails; zero-zero passes.
- `TestInvalidInputs` — unknown rule name exits 2; malformed
  manifest surfaces as a `meta` finding.
- `TestSummaryOutput` — summary shape + rule filtering behaviour.

All use `tempfile.TemporaryDirectory()` fixtures — no on-disk residue
between tests.

Additional tooling change: `tools/tests/_helpers.load_tool` now
registers the loaded module in `sys.modules` before `exec_module`.
Without this, `@dataclass` (which `verify-registry.py` uses) fails
with `AttributeError: 'NoneType' object has no attribute '__dict__'`
because stdlib `dataclasses._is_type` looks up
`sys.modules[cls.__module__]`.

### `tools/README.md`

Added a `verify-registry.py` section documenting the rule table, CLI,
exit codes, output format, and Makefile target. Test-count in the
"Unit tests" section bumped from 31 to 47.

## Results

- `make test` → **153/153 pass (47 python + 100 host + 6 kernel), 0
  leaks, 0 errors, exit 0**. The verify-registry prereq runs first
  and prints `verify-registry: 0 finding(s); ran R2,R4; deferred
  R1,R3,R5,R6,R7,R8` before any other test output.
- `make` (just kernel.bin) also runs verify-registry first.
- Production `kernel.bin` unchanged.

## Key Findings

- **Dataclass + dynamic module loading collides in stdlib.** First
  run of `test_verify_registry.py` exploded inside the `@dataclass`
  decorator (`dataclasses._is_type` walks `sys.modules[cls.__module__]`
  and crashes on NoneType when the module isn't registered). Fixed
  in `_helpers.load_tool` by adding `sys.modules[module_name] = mod`
  before `spec.loader.exec_module(mod)`. This is the pattern the
  Python docs recommend for `spec_from_file_location`-loaded
  modules but the gen-config / validate-config tests happened to
  not trigger it.
- **R7 can't be honestly implemented against the current layout.**
  DESIGN says "verify-registry.py regenerates the deps header and
  diffs against the checked-in copy (R7) — drift fails the build."
  But `.gitignore` excludes `gen/` — there's nothing to diff
  against. The *intent* (generator determinism + freedom from
  hand-edits) is covered by the `test_deterministic_output*` tests
  under `tools/tests/test_gen_config.py` landed in 3.5. Marked R7
  deferred with a pointer to those tests rather than inventing a
  check that doesn't apply.
- **Regex R2 false-positive guard.** First pass matched `struct
  nx_slot *foo;` anywhere in the file, including at column 0. That
  would have produced spurious findings for any file-scope handle
  variable (rare in component code but still possible). Constrained
  the regex to `^\s+` — at least one leading whitespace — and added
  `test_column_0_struct_nx_slot_ptr_is_not_flagged` as a guard.

## Decisions

- **Honest deferral over silent passing.** Every rule the checker
  can't enforce today is listed explicitly with a reason in
  `--list` and in the summary line. Users (and future AI agents
  reading the output) immediately see what's covered and what's
  not.
- **Regex for R2 / R4.** Real C parsing via pycparser would require
  preprocessing every component with the right include paths, which
  is complexity we don't need yet. Regex heuristics catch the
  common drift cases at zero dependency cost; when a component
  trips them falsely, the author can add a comment or rework the
  code to be regex-friendly.
- **Wire as a prereq even though components/ is empty.** The cost of
  running the gate today is one Python invocation; the benefit is
  that when Phase 4+ adds real components, the build already
  enforces the rules without a Makefile update. Much more robust
  than "add the prereq when components land" — easy to forget.
- **Findings to stderr, summary to stdout.** Matches the Unix
  conventions (`ls` error lines, `grep -c` count) and lets CI
  separately grep for findings vs. the summary.
- **`--rule R2` excludes, doesn't defer.** When the user explicitly
  filters to one rule, the others aren't "deferred" — they just
  weren't requested. Summary reflects that (`ran R2; deferred
  none`). A test exercises the distinction.

## Open / Deferred

- **pycparser-backed R1 / R6.** Real "where did this slot pointer
  come from?" analysis needs a proper AST. pycparser handles C99;
  would need a preprocessing step to resolve our headers. Candidate
  for a later slice — gate behind a flag (`--deep`) so the default
  `make test` remains fast.
- **Call-graph R8.** ISR / kthread entry points need an explicit
  tagging convention (function attribute or manifest field) before
  a call-graph check can draw the graph. Shape to be decided with
  slice 3.8 (hooks) + 3.9 (boot integration).
- **R3 enforcement needs interface schemas.** Slice 3.8 lands the
  hooking framework and the first real interface definitions;
  those are what R3 checks against.
- **R4 control-flow extension.** Today's count-based check misses
  "retain in enable, release only in enable's error path" — the
  release is paired but not reachable from `disable` / `destroy`.
  Revisit with pycparser or a lint-rule extension.

## Next

- **Slice 3.8 — Hooks + pause protocol.** Attach hooks to connection
  edges and slot mutation events; implement `pause_hook` per Session
  3; teach the IPC router to consult each connection's pause flag
  and apply the pause policy (`queue` / `reject` / `redirect` —
  currently dispatched regardless of component state). Also the
  slot-binding cross-check on `nx_component_destroy` (refuse if
  still bound to any slot's `active`).
