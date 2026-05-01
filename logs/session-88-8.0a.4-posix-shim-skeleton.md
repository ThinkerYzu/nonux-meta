# Session 88: slice 8.0a.4 — kernel `components/posix_shim/` skeleton

**Date:** 2026-04-30
**Phase:** Phase 8 — Group B (slice 8.0a, sub-slice 4 of 8)
**Branch:** master (source-side commit `d0d30f0` in `sources/nonux/`; this session's checkpoint is the proj_docs paperwork that records it)

---

## Goals

- Land slice **8.0a.4** — the kernel-side syscall-entry boundary
  component that DESIGN.md §"Every Component Occupies a Slot"
  mandates.  Every cross-component call originating from a syscall
  needs a graph-resident `src_slot` so cap-scan, pause/drain, and
  hooks have something to attach to; `posix_shim` is that component.
- Keep the slice a skeleton — manifest, init/enable/disable/destroy
  stubs, `g_posix_shim` singleton accessor, `handle_msg` returning
  `NX_EINVAL`.  Per-task `caller_slot` lifecycle lands in 8.0a.5;
  reply-routing body lands in 8.0a.6; STATE_LOST handler lands in
  8.0a.7.  No callers of the new slot yet — `framework/syscall.c`
  keeps its direct `vops->read(self, ...)` calls until slice 8.0c.
- Booted kernel composition advances from 6 slots / 6 components to
  **7 slots / 7 components** with `posix_shim` ACTIVE alongside the
  existing six.

## What Was Done

### Slice 8.0a.4 (commit `d0d30f0`, `sources/nonux/`)

Pure additive component landing.  Three new files in
`components/posix_shim/` + a new `kernel.json` slot + two
paving fixes in `tools/`.  ~246 insertions across 8 files.

**`components/posix_shim/manifest.json` (31 lines).**  Declares 4 deps
— `vfs`, `scheduler`, `memory.page_alloc`, `char_device.serial` —
all `mode: async, policy: queue`.  `vfs` is `stateful: true`; the
other three are stateless.  Async-only per DESIGN.md §"Sync-mode
caller must be on a dispatcher": syscall callers run on their own
task kstacks, not dispatcher threads, so sync-mode would let
`slot->active` be read off-dispatcher (R8 violation).

**`components/posix_shim/posix_shim.c` (121 lines).**  Five lifecycle
hooks plus a singleton accessor:

```c
struct posix_shim_state {
    struct posix_shim_deps deps;
    unsigned init_called, enable_called, disable_called, destroy_called;
    unsigned messages_handled;
};

struct posix_shim_state *g_posix_shim = NULL;
```

`init` sets `g_posix_shim = s` and bumps `init_called`; `destroy`
clears the singleton iff it still points at the dying instance.
`enable` / `disable` are bookkeeping-only — per-task slot wiring
happens at task_create-time (slice 8.0a.5), not at component
enable.  `handle_msg` returns `NX_EINVAL` until 8.0a.6 fills in the
reply-routing body.  Component registered via the standard
`NX_COMPONENT_REGISTER(name, state_struct, deps_field, ops_table,
deps_table)` macro.  No `pause_hook` (component doesn't spawn
threads).

**`components/posix_shim/README.md` (41 lines).**  Role overview
(boundary component for syscall-entry edges), dep table mirroring
the manifest, sub-slice plan rolling 8.0a.4 → 8.0a.7, cross-ref to
SLOT-CALL-API.md §"The posix_shim Component".

**`kernel.json` (+4 lines).**  New `posix_shim` slot in the `slots`
map:

```json
"posix_shim": { "impl": "posix_shim", "config": {} }
```

**`tools/gen-config.py` (+10 lines).**  Manifest deps may now carry
dotted slot names (`memory.page_alloc`, `char_device.serial`) for
direct kernel.json-slot-name parity.  `parse_dep` derives a
C-identifier `c_field` by replacing dots with underscores and
sanity-checks via `c_ident()`.  `render_deps_header` uses `c_field`
for the struct field name + the `offsetof()` arg, but keeps the
dotted `name` literal in the descriptor so runtime `nx_slot_lookup`
still finds the right slot.  Generated `gen/posix_shim_deps.h`
carries `struct nx_slot *memory_page_alloc;` with descriptor
`.name = "memory.page_alloc"`.  When `c_field == name` (no dot
flattening) the comment elides the slot-lookup-name note for
backward-compat noise reduction.

**`tools/verify-registry.py` (+19 lines).**  R2 (manifest↔fields
parity) now skips components that use the gen-config DI pattern.
Detection: `^\s+struct\s+\w+_deps\s+\w+\s*;` in the component's
`.c` file (i.e. an embedded `struct posix_shim_deps deps;`).
Without the skip, R2 produces false positives on dot-flattened
fields (manifest says `memory.page_alloc`, struct field says
`memory_page_alloc`).  With gen-config, the manifest↔fields
mapping is enforced by gen-config itself; re-checking here adds
nothing.

**`test/kernel/ktest_bootstrap.c` (+1 / -1).**  Snapshot-JSON buffer
bumped 2048 → 4096 bytes.  The rendered composition now exceeds 2
KiB once posix_shim's slot, component, and 4 connection edges
land.

**Build wiring (`Makefile`, +10 lines).**  New pattern rule
`gen/%_deps.h: components/%/manifest.json $(GENCONFIG)` — future
deps-bearing components drop a manifest in place with zero
Makefile churn.  Explicit
`components/posix_shim/posix_shim.o: gen/posix_shim_deps.h`
dependency added so the generated header is materialized before
the .c file compiles.

### Doc-side paperwork (this session's `proj_docs/nonux/` commit)

- This session log (`logs/session-88-8.0a.4-posix-shim-skeleton.md`).
- HANDOFF.md "Current Status" + Phase checklist + Next Actions —
  ☑ slice 8.0a.4, advance forward step to slice 8.0a.5.
- HANDOFF.md Session Logs — Session 88 entry prepended; Session 83
  pruned to keep the "last 5" rolling window.
- IMPLEMENTATION-GUIDE.md Group B table — "✓ Landed Session 88"
  stamp on the slice 8.0a.4 row.

## Decisions Made

### Dotted slot names in dep keys, flattened C field names

Two reasonable paths existed for letting `posix_shim` declare its
dep on `memory.page_alloc`:

1. **Aliased dep keys** — manifest carries `"page_alloc": { "slot":
   "memory.page_alloc", ... }`; aliased name used as both the C
   field and the descriptor's runtime lookup string.
2. **Dotted dep keys, flattened in C** — manifest carries
   `"memory.page_alloc": { ... }`; gen-config flattens dots →
   underscores for the C field but keeps the dotted name in the
   descriptor.

Took option 2.  Reasons: (a) `validate-config.py` already validates
manifest deps against `kernel.json` slot names; aliasing would
require an extra layer of name resolution and a new failure mode
when the alias clashes; (b) the dotted name is the canonical
identifier; the C-field name is a codegen artefact that exists
purely to satisfy C identifier syntax; (c) future components with
multi-namespace deps (e.g. `net.l4.tcp`) get the same treatment
without further schema work.

### `g_posix_shim` set in init, cleared in destroy

The per-task `caller_slot` plumbing in slice 8.0a.5 needs to bind
each new task slot to "the posix_shim instance" — there's exactly
one instance (the singleton-component-many-slots N→1 binding from
DESIGN.md §"Multi-slot binding (N:1)").  Setting the singleton
in `init` instead of `enable` means tests that touch the singleton
during component setup don't hit a NULL-check landmine.  Clearing
in `destroy` (with the `if (g_posix_shim == s)` guard so a re-init
during the same composition cycle isn't disturbed) keeps the
contract symmetric.

### R2 skip detected by `struct <name>_deps <field>;`, not by manifest opt-in

Considered adding an explicit `"_gen_config_di": true` opt-in field
to the manifest.  Rejected: the .c-file regex is direct evidence
that the component uses the pattern, with no opportunity for
manifest/code drift.  Existing components without a `*_deps`
struct (e.g. `uart_pl011`, `mm_buddy`, `ramfs`, `procfs`,
`vfs_simple`) keep the old R2 check — they declare deps as
individual `struct nx_slot *vfs;` fields, where the manifest key
*is* the C field name and R2's parity check stays meaningful.

### Snapshot buffer: bump to 4 KiB, not parameterize

`ktest_bootstrap.c` could have grown a `#define` for the snapshot
buffer size, or computed it from registry size.  Bumping the
literal to 4096 is the smaller change and matches the existing
"static buffer with comfortable headroom" pattern in the test
harness.  Next bump will likely be needed when 8.0a.5 lands ~N
more `task_caller_slot` registrations per running task — at that
point a `#define` or computed bound becomes worth it.

### No tests this slice

Same reasoning as 8.0a.3: skeleton with `handle_msg → NX_EINVAL`
and lifecycle-counter-only state doesn't justify a ktest entry.
The substantive ktests for posix_shim land in 8.0a.5 (per-task
slot lifecycle), 8.0a.6 (round-trip reply path), and 8.0a.8
(cross-cutting test infrastructure).  The existing
`generation_monotonically_increases` and `composition_snapshot`
host tests already exercise the bumped slot count without
modification.

## Findings

- The pattern rule `gen/%_deps.h: components/%/manifest.json` is
  the right granularity for future components with deps.  Today
  only `posix_shim` uses it; `mm_buddy`, `vfs_simple`, etc. still
  declare deps as individual `struct nx_slot *` fields without a
  `gen/` artefact (because their dep names are already valid C
  identifiers and they predate the gen-config flow).  Migrating
  the older components to the gen-config flow is a non-blocking
  follow-up; the slot-lookup-by-string runtime path is identical.

- The dotted-name → flattened-field codegen extension is small
  enough (~10 lines) that the cost-benefit clearly favors handling
  it in gen-config rather than at the manifest schema level.  If
  future components need to disambiguate between
  `memory.page_alloc` and (hypothetically) `memory_page_alloc` in
  the same manifest, an explicit alias schema can be added then;
  for now the flattening is unambiguous in every existing slot
  name.

- R2's gen-config skip is detected by source pattern
  (`struct <name>_deps <field>;`) rather than by manifest
  declaration.  This means a component author who copies the
  posix_shim shape but forgets to use gen-config still gets the
  R2 enforcement.  Future R2 documentation should note that the
  skip is conditional on the struct embedding pattern, not on a
  manifest flag.

- Slot-count bump (6 → 7) is observable in `make test`'s
  `composition_snapshot` output and in any boot-trace log that
  prints the composition line.  Next bump (8 → ?) lands with
  slice 8.0a.5's per-task slot registrations — at that point the
  snapshot text is dynamic per-task-count and the snapshot test
  will need to either filter task slots or assert against a
  pattern.

## Tests

- `make test` → **495/495 pass** (93 python + 283 host + 119
  kernel — same baseline as Session 87.  posix_shim.c is added to
  the kernel build, but with `handle_msg → NX_EINVAL` and zero
  callers of the new slot, behavior delta is zero.  The python
  test suite still passes against the gen-config dot-flattening
  extension — pre-existing tests use undotted dep names so the new
  code path isn't exercised, but the existing path is unchanged).
- `make test-interactive` → **7/7 pass** (echo_cat, echo_hello,
  echo_pipe, ls_root, mkdir_tmp, ps_smoke, visible_prompt).
- `make verify-iface-fresh` → 0 drift.
- `make verify-registry` → 0 findings (R2 correctly skips
  posix_shim per the new gen-config-DI detection).

## Next Steps

Slice **8.0a.5** — per-task `caller_slot`:
- `struct nx_task` grows embedded `caller_slot` + `reply_waitq` +
  `in_flight_reply_buf` + `in_flight_reply_buf_len` +
  `in_flight_reply_rc`.
- `nx_task_create` registers the embedded slot, binds
  `active = posix_shim_component` (via `g_posix_shim`), walks
  posix_shim's outgoing edges (`vfs`, `scheduler`,
  `memory.page_alloc`, `char_device.serial`) and registers
  parallel `(caller_slot → svc_slot)` edges with the same
  mode/stateful/policy fields (per DESIGN.md §"Edge inheritance").
- `nx_task_destroy` unregisters the slot + edges in reverse order.
- R3 cap-scan + pause hold-queue work uniformly for blocking-call
  senders without per-callsite special cases (Invariant 1 carries
  through to the embedded slot).
- Snapshot test extended to handle dynamic task-slot count (or
  asserts against the static-component-slot prefix).
- ktest coverage: per-task slot create/destroy pairing, edge
  inheritance against posix_shim's 4 deps, no leak on
  `task_destroy` after partial-init failure.
