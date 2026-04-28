# Session 11: Config tooling (slice 3.5)

**Date:** 2026-04-21
**Phase:** 3 — Component framework
**Branch:** master

---

## Goals

- Extend the Python tool chain from slice 3.4 into a full config
  pipeline: read `kernel.json`, emit `gen/config.h` (kernel config
  defines) and `gen/sources.mk` (compile list); validate the whole
  tree against real JSON schemas; perform cross-manifest version
  checks and dep-graph cycle detection; wire the project venv into
  the Makefiles so `make validate-config` / `make deps` / `make
  deps-dot` work out of the box.
- Install the first Python dependency (`jsonschema`) — proves the
  venv machinery landed in slice 3.4 is actually usable.
- Add Python unit tests so the tooling has the same regression
  coverage as the C side.

## What Was Done

### `tools/schemas/` — JSON Schema Draft 2020-12 definitions

- `manifest.schema.json` — per-component manifest: requires `name`
  (valid C identifier) and `version` (`X.Y.Z`); optional `iface`
  (slot interface tag, validated harder in 3.7), `requires` and
  `optional` maps, each keyed by dep name. Dep spec: optional
  `version` (`>=X.Y.Z` only in v1), `mode` (`async`/`sync`), boolean
  `stateful`, `policy` (`queue`/`reject`/`redirect`). No extra
  properties — typos fail loudly.
- `kernel.schema.json` — top-level composition: requires `target`
  (string) and non-empty `components` map (slot → binding). Slot
  bindings require `impl` (valid C identifier) and allow optional
  `config` (object) and `iface`. Connection overrides are typed.
  `hooks` and `posix` are lenient (shape finalises in 3.8 / Phase 7).

### `tools/gen-config.py` — `manifest` + `kernel` subcommands

- Refactored to argparse subcommands:
  - `manifest <manifest.json> <outdir>` — unchanged deps.h emission
    from slice 3.4. Test/host/Makefile updated to pass the
    `manifest` subcommand.
  - `kernel <kernel.json> <components_dir> <outdir>` — new. Emits:
    - `gen/config.h` — `#define NX_TARGET`, per-slot
      `#define NX_SLOT_<SLOT>_IMPL "<impl>"`, per-component
      `#define NX_CONFIG_<COMP>_<KEY> <literal>` from each binding's
      `config` block. Dotted slot names like `char_device.serial`
      flatten to `CHAR_DEVICE_SERIAL`.
    - `gen/sources.mk` — `COMPONENT_SRCS := components/<impl>/<impl>.c \
      …` for every referenced impl. Deduplicated and sorted.
  - `components_dir` argument currently accepted but unused —
    reserved for slice 3.7 where `verify-registry.py` will need to
    cross-resolve manifests.
- `c_literal()` maps Python values to C tokens: ints verbatim,
  bools → `1`/`0`, strings quoted with JSON-style escaping, numeric-
  looking strings (hex `0x…` or decimal) passed through as literal
  integers — so `"base_addr": "0x09000000"` lands as
  `#define NX_CONFIG_UART_PL011_BASE_ADDR 0x09000000` instead of a
  quoted string.
- Deterministic: slot/component/config-key keys are sorted before
  emission in both subcommands.
- Still stdlib-only. Kernel mode does NOT invoke jsonschema — schema
  validation is `validate-config.py`'s job.

### `tools/validate-config.py` — first venv-dependent tool

- Runs four checks in order, accumulating errors so a single pass
  surfaces everything:
  1. **Schema** — kernel.json against `kernel.schema.json`; every
     resolved component's manifest.json against
     `manifest.schema.json`. Uses
     `jsonschema.Draft202012Validator.iter_errors` with sorted paths
     so error output is stable.
  2. **Existence** — every `impl` referenced in kernel.json has a
     `components/<impl>/manifest.json`, and the manifest's `name`
     matches the directory.
  3. **Versions** — `>=X.Y.Z` constraints on `requires.<dep>.version`
     / `optional.<dep>.version` are satisfied by the target
     component's top-level `version`. Only `>=` supported in v1;
     any other operator is a validation error, not a silent skip.
  4. **Cycles** — iterative DFS over the slot → dep-slot graph
     (both `requires` and `optional` edges are included, because a
     cycle through optionals is still a boot-order impossibility).
     Self-loops produce `[a, a]`; longer cycles reconstruct the
     full path.
- Same binary also serves `make deps` (`--deps`, one edge per line)
  and `make deps-dot` (`--deps-dot`, graphviz digraph).
- Exit code 2 on any check failure — Make surfaces as an error.
  Reports still print even when validation fails, so a user
  investigating a problem can see the partial graph.
- Missing `jsonschema` at import time prints a pointer to
  `make venv` / `pip install -r tools/requirements.txt` and exits 2.

### Makefile wiring

- New variables at the top: `VENV` / `VENV_PY` / `VENV_STAMP` /
  `PYTHON`. `VENV_STAMP = $(VENV)/.installed` is a plain file that
  `make venv` touches after `pip install` — avoids touching
  `$(VENV)/bin/python3`, which is a symlink to the system
  interpreter (previous attempt hit "Permission denied" because
  `touch` followed the symlink).
- `make venv` — one-shot: creates `.venv/` if missing, installs
  `tools/requirements.txt`, stamps the sentinel. Re-running is a
  no-op when requirements haven't changed.
- `make kernel-config` — **phony** target that emits `gen/config.h` +
  `gen/sources.mk`. Deliberately NOT a file-target, because an
  ordinary rule whose outputs include `gen/sources.mk` would fire
  from the top-level `-include gen/sources.mk` and try to compile
  the referenced-but-nonexistent `components/*.c` files. Phony
  keeps generation opt-in until real components land in Phase 4+.
- `make validate-config` / `make deps` / `make deps-dot` now
  depend on `$(VENV_STAMP)` so they install deps on first use.
- `make test-tools` — new test target that runs
  `python3 -m unittest discover -s tools/tests`. Wired into
  `make test` alongside `test-host` and `test-kernel`.
- `test/host/Makefile` — the existing `gen/%_deps.h` rule now
  invokes `gen-config.py manifest …` (the subcommand), per the
  refactored CLI.

### `tools/requirements.txt`

- Single line: `jsonschema>=4.0`. Pulled in only by
  `validate-config.py`; the generator stays stdlib-only so
  `make test-host` doesn't need the venv activated.

### `tools/tests/` — Python unit tests

- Stdlib `unittest` only — no pytest dep, no `conftest.py`, no
  fixtures plugin. Tests discoverable via `unittest discover`.
- `_helpers.load_tool()` loads `gen-config.py` and
  `validate-config.py` as importable modules despite the hyphenated
  filenames, via `importlib.util.spec_from_file_location`. The
  alternative — renaming the tools — would have bent the filename
  convention for the sake of the tests, which felt wrong.
- `test_gen_config.py` — 15 tests covering both subcommands:
  struct-field-per-dep, default propagation, explicit overrides,
  deterministic output across runs, alphabetical sorting within
  requires/optional, no-deps placeholder + no DEPS_TABLE, duplicate
  dep-name rejection, invalid-mode rejection, minimal kernel
  emission, dotted-slot-to-underscore, config-value literal shapes
  (int / numeric-string / plain-string / bool), impl dedup in
  sources.mk, determinism, missing-target, empty-components.
- `test_validate_config.py` — 16 tests: version matching
  (`satisfies`, `parse_semver`), cycle detection (acyclic / self-
  loop / 2-node / 3-node), and eight end-to-end fixture-driven
  cases: valid tree, missing manifest, name-directory mismatch,
  version constraint unsatisfied, required dep slot missing from
  kernel, optional dep slot missing (still OK), dependency cycle,
  invalid JSON, schema violation. All fixture trees are built in
  `tempfile.TemporaryDirectory()` so tests don't share state or
  leave disk residue.
- `jsonschema`-dependent tests skip gracefully if the module isn't
  installed — `@unittest.skipUnless(HAVE_JSONSCHEMA, …)`.

## Results

- `make test` now runs **three** suites:
  - `test-tools` → 31 Python tests, all pass.
  - `test-host` → 84 C host tests, 0 leaks, 0 errors.
  - `test-kernel` → 6 kernel tests under QEMU.
  - **Total: 121/121 pass.**
- `make venv` creates the venv + installs requirements (~4 s).
- `make kernel-config` emits `gen/config.h` + `gen/sources.mk` from
  the repo's aspirational `kernel.json` successfully.
- `make validate-config` / `make deps` correctly return exit 2 on
  the aspirational config (missing manifests) — expected state
  until real components land in Phase 4+.
- Production `kernel.bin` unchanged this slice.

## Key Findings

- **`-include gen/sources.mk` auto-rebuilds if a rule names
  `gen/sources.mk` as an output.** First draft of the Makefile had
  `gen/config.h gen/sources.mk: kernel.json $(GENCONFIG)` as an
  ordinary rule. `make test` → `-include` triggered the generator
  → sources.mk populated with nonexistent `components/*.c` → link
  failed. Fix: make `kernel-config` a phony target whose recipe
  writes the files as a *side-effect*; don't name them as outputs.
  `-include gen/sources.mk` then silently picks them up when
  they're there and silently skips when they're not.
- **`touch` follows symlinks by default.** Using `$(VENV)/bin/python3`
  as the venv sentinel failed with "Permission denied" because
  `python3` is a symlink to a system-owned `python3.13`. Switched
  to a plain stamp file `$(VENV)/.installed`, which we own.
- **`components_dir` unused in kernel mode today.** Kept the CLI
  argument even though `gen-config.py kernel` only reads
  `kernel.json` — the signature matches `validate-config.py`'s and
  gives slice 3.7 (`verify-registry.py`) a place to hang
  cross-manifest checks without a CLI break.

## Decisions

- **Only one venv dep for now** — `jsonschema`. Not pulling in
  `pyyaml`, `click`, `pydantic` etc. until there's a concrete need.
  Stdlib `argparse` and `json` carry the tool fine.
- **Stdlib `unittest`, not `pytest`.** No extra dep means
  `make test-tools` works out of the box with just the venv
  created by `make venv`. Loss of features vs. pytest is small at
  31 tests.
- **`>=X.Y.Z` is the only version operator.** Semver complexity
  (`^`, `~`, ranges, prerelease) isn't justified yet — a real case
  in components/ can drive it later.
- **Cycle detection includes optional edges.** A cycle through
  optionals means no topological boot order exists, even though
  any one optional edge could be dropped. Flagging it now beats
  surprising debug sessions later.
- **No auto-regeneration of `gen/<name>_deps.h` from kernel mode.**
  Each component's deps.h is a per-manifest artefact; production
  Makefile rules (and the test/host/Makefile's existing rule)
  generate them one-at-a-time. Keeps gen-config.py's two
  subcommands single-purpose.

## Open / Deferred

- **Real components in `components/`** — slice 3.5 proves the tool
  chain works against a valid fixture tree (in Python tests and
  tempdir-based cases). The repo's aspirational `kernel.json`
  references manifests that don't exist yet; `make validate-config`
  correctly refuses to validate until they do. Phase 4's first
  component (`sched_rr`) will lift the aspirational flag.
- **`iface` field matching.** Both schemas allow an optional `iface`
  string but don't yet check that a slot's declared `iface` matches
  the chosen `impl`'s declared `iface`. Slice 3.7
  (`verify-registry.py`) will pick this up.
- **Typed ops in the component descriptor.** Still `const void *`
  (carried over from 3.4). Slice 3.6 types it as
  `struct nx_component_ops *`.
- **R7 regenerate-and-diff gate.** Slice 3.7 will invoke
  `gen-config.py manifest` on every manifest under components/ and
  diff against the checked-in `gen/<name>_deps.h`. The
  deterministic-output property was a precondition; it held up
  under the new unit tests (see `test_deterministic_output` and
  `test_deterministic_output_across_runs`).

## Next

- **Slice 3.6 — IPC router.** Introduces `struct nx_component_ops`
  (types `nx_component_descriptor.ops`), the async-default /
  sync-shortcut message router, cap-array scanning
  (`ipc_scan_send_caps` / `ipc_scan_recv_caps`), and
  `slot_ref_retain` / `slot_ref_release` for cap-transferred slot
  refs that a receiver wants to hold past the handler.
