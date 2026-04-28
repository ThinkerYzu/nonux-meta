# Session 10: Dependency injection (slice 3.4)

**Date:** 2026-04-21
**Phase:** 3 — Component framework
**Branch:** master

---

## Goals

- Land the C-side dependency-injection mechanism described in
  DESIGN.md §Dependency Injection Mechanism: component descriptor type,
  `NX_COMPONENT_REGISTER` macro expanding into a `.components` linker
  section, and a framework-side `nx_resolve_deps()` that writes slot
  pointers into a component's state and registers connection edges.
- Ship the minimum Python generator (`tools/gen-config.py`) that reads
  a single component `manifest.json` and emits `gen/<name>_deps.h` (the
  typed deps struct + the `_DEPS_TABLE(CONTAINER, FIELD)` macro). Full
  `kernel.json` handling + validation is deferred to slice 3.5.
- Set up a Python venv at `sources/nonux/.venv/` (gitignored) for the
  tool chain. v1 generator is stdlib-only; the venv exists so 3.5 can
  add `jsonschema` etc. without polluting the system interpreter.

## What Was Done

### Virtualenv + .gitignore

- `python3 -m venv sources/nonux/.venv` — CPython 3.13.7, system pip.
- `.gitignore` gains `.venv/`, `__pycache__/`, `*.pyc`. Verified
  `git status --ignored` correctly excludes the venv.

### `framework/component.h` — descriptor types + macro

- `struct nx_dep_descriptor` — `{ name, offset, required, version_req,
  mode, stateful, policy }`. `offset` is from the container root (not
  from the deps sub-struct), so `nx_resolve_deps` only needs one
  pointer+offset arithmetic per field.
- `struct nx_component_descriptor` — `{ name, state_size, deps_offset,
  deps, n_deps, ops }`. `ops` is deliberately `const void *` for now;
  slice 3.6 (IPC router) types it as `struct nx_component_ops`.
- `NX_COMPONENT_REGISTER(NAME, CONTAINER, DEPS_FIELD, OPS, DEPS_TABLE)`
  expands to a static `nx_dep_descriptor[]` (from the generator's
  `_DEPS_TABLE` macro) plus a `const nx_component_descriptor
  NAME##_descriptor` placed in the `nx_components` linker section.
  Section name is a C identifier (not `.components`) so GNU ld
  auto-generates `__start_/__stop_nx_components` markers without
  needing special linker-script entries — same trick the host test
  runner uses for `test_registry`.
- `NX_COMPONENT_REGISTER_NO_DEPS(NAME, CONTAINER, DEPS_FIELD, OPS)` —
  separate macro for components with zero deps. Rationale: C forbids
  zero-length arrays; rather than emit a sentinel and rely on
  `n_deps = 0` to skip it at runtime, a parallel macro keeps the
  descriptor clean (`.deps = NULL, .n_deps = 0`) and the generator's
  no-deps header explicitly points authors at it.

### `framework/component.c` — `nx_resolve_deps()`

```c
int nx_resolve_deps(const struct nx_component_descriptor *d,
                    struct nx_slot *self_slot,
                    void *state);
```

- Walks `d->deps[0..n_deps-1]`. Per dep: `nx_slot_lookup(name)`; if
  missing and required → `NX_ENOENT`; if missing and optional → skip
  (leave pointer as-is, no edge). If found: write the pointer at
  `state + dep->offset` *then* call `nx_connection_register(self_slot,
  target, mode, stateful, policy, &err)`. Pointer-first-then-edge order
  means an ENOMEM from the registry won't leave a registered edge
  without the component field populated.
- `self_slot` may be NULL: the registry already accepts `from_slot =
  NULL` as the boot/external-entry edge marker, so passing NULL through
  `nx_resolve_deps` emits boot edges for each resolved dep. Useful for
  the first component wired at boot time (it has no "owning" slot yet).

### `tools/gen-config.py` — minimum generator

- ~180 LOC, pure stdlib (`argparse`, `json`, `pathlib`, `sys`). Reads
  one `<name>_deps.json`-style manifest, emits `<outdir>/<name>_deps.h`
  with the deps struct and the `<NAME>_DEPS_TABLE` macro.
- Manifest schema for slice 3.4 (deliberately narrow): top-level
  `name` (required), `version` (optional, ignored), `requires` (map of
  deps), `optional` (map of deps). Each dep accepts `version` (string,
  optional), `mode` (`"async"`|`"sync"`, default `async`), `stateful`
  (bool, default `false`), `policy` (`"queue"`|`"reject"`|`"redirect"`,
  default `queue`).
- Defaults match DESIGN: async / stateless / queue. Only explicitly
  declared fields diverge from those defaults.
- **Deterministic output.** Deps are emitted in sorted order
  (requires by name first, then optionals by name). Two consecutive
  runs produce byte-identical headers — a precondition for the R7
  diff-check planned in slice 3.7 (`verify-registry.py`).
- Maps `mode`/`policy` string to the `NX_CONN_*` / `NX_PAUSE_*` enum
  names declared in `framework/registry.h`. The generator only needs
  to emit those identifiers; the C compiler resolves them.
- Input validation raises `ManifestError` with a descriptive message
  for: malformed JSON, missing `name`, non-boolean `stateful`,
  unknown `mode` / `policy`, duplicate dep name across
  requires/optional. Exit code 2 on error.
- No-deps manifest: emits a `struct <name>_deps` with a single
  `char _nx_no_deps` placeholder (C forbids empty structs) and omits
  the `_DEPS_TABLE` macro entirely; a comment directs authors to
  `NX_COMPONENT_REGISTER_NO_DEPS`.

### Test fixtures + host tests

- `test/host/fixtures/sample_deps.json` — one required dep (`timer`,
  async, stateless, queue) and one optional dep (`stats`, sync,
  stateful, queue). Exercises all manifest-field combinations that
  matter at this slice: defaults, explicit overrides, a dep with a
  version string and one without.
- `test/host/fixtures/trivial_deps.json` — name only; no deps.
- `test/host/component_deps_test.c` — 9 tests covering:
  - **Descriptor shape** — `descriptor_fields_match_manifest`,
    `trivial_descriptor_has_zero_deps`. Asserts `name`, `state_size`,
    `deps_offset`, `n_deps`, per-dep `{name, offset, required,
    version_req, mode, stateful, policy}`.
  - **Resolution happy/edge paths** —
    `resolve_deps_writes_pointers_and_registers_edges` (both deps
    present), `resolve_deps_with_optional_dep_missing_still_succeeds`,
    `resolve_deps_with_missing_required_returns_enoent` (verifies the
    state struct stays clean too), `resolve_deps_with_null_self_slot_emits_boot_edges`,
    `resolve_deps_on_trivial_descriptor_is_a_noop`.
  - **Argument validation** — `resolve_deps_rejects_null_args`.
  - **Wiring correctness** —
    `resolve_deps_preserves_mode_and_stateful_from_manifest` walks the
    connection list with a filter and asserts the timer edge is
    async+stateless, the stats edge is sync+stateful (sanity for the
    mode/policy/stateful pass-through).

### Test Makefile wiring

- Added `gen/%_deps.h: fixtures/%_deps.json $(GENCONF)` rule. `mkdir -p
  gen` first so the first clean build doesn't fail.
- `component_deps_test.o` depends on both generated headers, so any
  manifest edit triggers regeneration before recompile.
- `-I.` added to CFLAGS so `#include "gen/sample_deps.h"` from a
  different translation unit would also work if other tests adopted
  the pattern; today `component_deps_test.c`'s quote-form search finds
  the headers next to itself.
- `clean` now `rm -rf gen/` in addition to `*.o` and `test_host`.
- `PYTHON ?= python3` — defaults to system interpreter. When slice 3.5
  adds jsonschema, this becomes `PYTHON ?= ../../.venv/bin/python3`
  (or the Make rule `source`s the venv's activate). Deferring until
  there's an actual dep.

## Results

- `make test` → **90/90 pass (84 host + 6 kernel), 0 leaks, 0 errors,
  exit 0**.
- `make` (production kernel) unchanged — framework is host-side only
  until slice 3.9 wires it into boot.
- `.venv/` created and ignored. Generator runs on the system Python;
  venv is ready for 3.5.

## Key Findings

- **GCC quote-form search is enough.** `#include "gen/sample_deps.h"`
  from `test/host/component_deps_test.c` finds
  `test/host/gen/sample_deps.h` without any extra `-I` flag because
  the `"..."` form searches the including file's directory first. This
  keeps the test build isolated from the production `gen/` directory
  (which lives at `sources/nonux/gen/`).
- **Section name must be a C identifier, not `.components`.** DESIGN.md
  uses `.components`; GCC accepts both but GNU ld only auto-generates
  the `__start_/__stop_` markers for sections whose names are valid C
  identifiers. Switched to `nx_components` so the boot-time walker in
  slice 3.9 can reference `__start_nx_components` directly — same
  pattern as the existing `test_registry` (host) and
  `kernel_test_registry` (ktest) sections.
- **Descriptor `ops` left opaque.** DESIGN.md shows `ops` as
  `struct component_ops *` (init/enable/pause/...). Slice 3.3's
  lifecycle verbs operate on `nx_component`, not on the descriptor
  yet. The descriptor's `ops` pointer will type-check against
  `nx_component_ops` once slice 3.6 introduces it. For now, tests
  stash a sentinel `const int *` and the only assertion is pointer
  equality.
- **Deterministic sort matters.** First draft of the generator
  iterated manifest dict keys in insertion order (Python 3.7+
  behaviour). Swapped to `sorted()` so regenerating the same manifest
  twice produces byte-identical output — required for the
  regenerate-and-diff build gate in slice 3.7.

## Decisions

- **Two macros instead of one** — `NX_COMPONENT_REGISTER` for the
  has-deps path and `NX_COMPONENT_REGISTER_NO_DEPS` for the zero-deps
  path. Merging them would either require a zero-length array (C
  UB) or a sentinel dep that the resolver has to skip. Separate
  macros keep the descriptor schema honest (`deps = NULL, n_deps = 0`
  is a clean encoding of "no deps").
- **`version_req` carried through but unused.** Slice 3.4 ships the
  version string in the dep descriptor; version matching against the
  target component's manifest is a slice 3.5 concern. This keeps the
  descriptor type stable so 3.5 doesn't have to change any generator
  output.
- **Pointer-first, edge-second in `resolve_deps`.** See §component.c
  above — means a late `NX_ENOMEM` from the registry leaves the
  pointer written without a registered edge, which is the lesser
  evil; a registered edge without the pointer would make the pointer
  audit (planned for slice 3.7) fail confusingly.
- **Fixture manifests under `test/host/fixtures/`.** Production
  components ship manifests under `components/<name>/manifest.json`
  per IMPLEMENTATION-GUIDE. The fixtures live apart from that tree so
  the test suite doesn't accidentally imply that `sample` is a real
  component.

## Open / Deferred

- **Version matching + full schema.** Slice 3.5 (`validate-config.py`)
  owns:
  - Real JSON schema for `manifest.json` (types / required fields /
    enum values).
  - Cross-manifest version checks (`requires.timer.version` against
    the target component's `"version": "0.1.0"`).
  - `kernel.json` composition → `gen/config.h` + `gen/sources.mk`.
  - Cycle detection in the dep graph.
- **R7 regenerate-and-diff.** Slice 3.7 (`verify-registry.py`) will
  invoke the generator on every manifest, diff the output against the
  checked-in `gen/<name>_deps.h`, and fail the build on drift. The
  deterministic-output property landed this slice is the precondition.
- **Typed ops.** Slice 3.6 lands `struct nx_component_ops` and
  re-types `nx_component_descriptor.ops`. Test descriptors using the
  sentinel pointer will need a trivial update then.
- **Boot-time walker.** Slice 3.9 walks `__start_nx_components
  … __stop_nx_components`, registers each descriptor's component +
  slot, and calls `nx_resolve_deps` → `nx_component_init` →
  `nx_component_enable` in dependency order. The machinery for all
  three steps now exists; the walker itself is a thin glue layer.

## Next

- **Slice 3.5 — Config tooling.** Extend `gen-config.py` to also read
  `kernel.json`, emit `gen/config.h` and `gen/sources.mk`, and add
  real schema validation (first use of the venv — `jsonschema`). Add
  `validate-config.py` as a separate entry point for the
  `make validate-config` target that already exists in the top-level
  Makefile.
