# Session 79: Phase 8 Plan — IPC Migration + Generator + Runtime Recomposition

**Date:** 2026-04-29
**Phase:** Phase 8 planning (no code changes)
**Branch:** master

---

## Goals

- Lay out a concrete plan for Phase 8 (runtime recomposition + config manager).
- Validate that the existing component substrate is hot-swap-safe — and if not, plan the prerequisite work to make it so.
- Decide what test coverage the migration needs.

## What Was Done

### Audit: how cross-component calls work today

Grepped `sources/nonux/` for `slot->active->descriptor->iface_ops` access and direct ops-table dereferences (`vops->`, `fops->`, etc.).  Found ~184 occurrences across 19 files, including production paths in `framework/syscall.c` (~30 vfs sites), `framework/dispatcher.c`, `framework/bootstrap.c`, `framework/component.c`, `framework/process.c`, and `components/vfs_simple/vfs_simple.c` (mount-table dispatch into ramfs/procfs).

None of these check `slot->pause_flag`, none route through `nx_ipc_send`, none participate in the slice-3.8 hold-queue / pause-policy / hook-chain / cap-scan machinery.  A hot-swap of vfs while a thread is mid-`vops->read` is a use-after-free.

Per-component check of `handle_msg`:

| Component | `handle_msg` | How driven today |
|---|---|---|
| `mm_buddy`     | NULL | Direct `mm_ops` calls |
| `sched_rr`     | NULL | Direct calls from `sched_tick` + framework |
| `vfs_simple`   | NULL | Direct `nx_vfs_ops` from `framework/syscall.c` |
| `ramfs`        | NULL | Direct `nx_fs_ops` from `vfs_simple` |
| `procfs`       | NULL | Direct `nx_fs_ops` from `vfs_simple` |
| `uart_pl011`   | smoke-test placeholder (counts invocations only) |
| `posix_shim`   | N/A — userspace library (crt0 + libnxlibc.a), not a kernel slot |

Source comments confirm this is by design today (e.g. `components/ramfs/README.md:60` — *"`handle_msg` is NULL: ramfs is driven via iface_ops, not IPC"*).  Slice-3.8's IPC router was wired into the kernel build, but production traffic never started flowing through it.

### Decision: route everything through IPC (Option A)

Considered two approaches:

- **Option A — IPC-route every cross-component call.** Matches DESIGN.md's async-first stance.  Pause flag, hold queue, hooks, cap-scan all just work.  Cost: every syscall pays at least a router dispatch.
- **Option B — pause-aware borrowed access (RCU-style).**  Per-slot in-flight counter; PAUSE waits for it to drain.  Cheap on the hot path, still pause-safe, but leaves DESIGN's "every cross-component call is a message" property aspirational.

Picked **Option A**.  Long-term direction wins over short-term cycle savings.  v1 ships sync-mode (caller blocks on its own stack); async-mode-per-edge is forward-compatible via DESIGN's existing `mode = IPC_SYNC|IPC_ASYNC` parameter and lands in a later slice.

### Decision: hand-shimming is a dead end — generator first

Naïvely migrating means writing a `handle_msg` shim per component (~30 ops × 5 interfaces = ~150 dispatch arms) plus per-op message structs plus typed sender wrappers.  That's hundreds of LOC of mechanical, error-prone code, and adding a 31st op or 6th interface later compounds the cost.

Instead: **JSON IDL per interface**, parsed by `tools/gen-iface.py`, emits four artifacts per interface:

1. `interfaces/<iface>_msg.h` — per-op tagged message structs + `enum nx_<iface>_op_id`.
2. `interfaces/<iface>.h` — the existing `struct nx_<iface>_ops` typedef, regenerated.
3. `framework/<iface>_call.h` — sender-side typed wrappers (`nx_vfs_open(slot, path, flags, &file)`).
4. `framework/<iface>_dispatch.h` — receiver-side `handle_msg` template; component supplies `NX_VFS_OP_OPEN_IMPL`/etc. macros pointing at internal static functions.  Zero hand-written switch statements.

JSON IDL was picked over parse-C because per-op metadata (pause-policy override, sync-only vs. async-allowed, trace tags) has no natural home in a `struct ops` declaration.  A future `verify-registry` rule can lint that the IDL matches the in-tree C declarations.

Generator detail discussion deferred to a future session (user request).  This session locked the *shape* of the generator's output, not its implementation.

### Decision: sync-during-pause = block on resume_waitq

DESIGN.md `pause_policy: queue` describes literal message buffering during pause.  Awkward for sync calls — caller is on the syscall thread, blocked.  Sync calls instead **park on `slot->resume_waitq`** (using slice 7.8a's wait-queue primitive) until RESUME.  Async edges keep DESIGN's literal-buffer semantics.  DESIGN.md needs a one-section update to reflect this split, landing with slice 8.0a.

### Test plan

Migration of 184 callsites and 5 components is high-risk surgery.  Coarse `make test` green is necessary but not sufficient.  Drafted ~270 new ktests across the 15-slice plan, with the bulk front-loaded:

- **Generator output (Group A):** ~50 ktests across schema validation, round-trip pack/unpack, snapshot diffing, cap-bearing ops, no-context ops, IRQ-entry shape.
- **Sync-call infrastructure (slice 8.0a):** ~70 ktests covering pause-state transitions, all three pause policies, in-flight counter, hook chain (incl. abort), cap-scan (incl. forgery), identity property under `PAUSE_NONE`, re-entrancy, lost-wakeup race, ISR-context guards.  This is the foundation everything else stands on.
- **Receiver shim activation (slice 8.0b):** ~50 ktests, dual-callable equivalence per op per component.
- **Migration regression coverage (slices 8.0c, 8.0d):** ~30 per-callsite golden tests pinning down behavior *before* the wrapper migration; `make test` 453/453 + interactive 7/7 stay green; hot-path perf gated to ≤10% regression.
- **Phase 8 proper (slices 8.1–8.6):** ~100 ktests across kernel-side pause/drain/resume, recompose orchestration, config handle API, sched_priority conformance, live-swap demo, mode switching.

Cross-cutting test infrastructure lands in slice 8.0a so every later slice can lean on it: configurable mock component, hook-chain inspector, recompose event logger, pause-injector ktest fixture, cap-forgery harness, per-callsite equivalence runner macro.

End-state target: ~700 tests when Phase 8 closes (today: 453).

### Slice plan landed in IMPLEMENTATION-GUIDE.md

15 slices in 3 groups:

- **Group A — Generator infrastructure** (4 slices): 8.0pre.1–4.
- **Group B — IPC migration** (5 slices): 8.0a–e.
- **Group C — Runtime recomposition** (6 slices): 8.1–8.6.

Estimated ~15-25 sessions.

## Key Findings

- `posix_shim` is *not* a kernel slot — it's the userspace library (crt0 + libnxlibc.a) replacing musl's runtime.  Manifest has just `name`+`version`.  Excluded from migration.
- `uart_pl011`'s existing `handle_msg` is a smoke-test placeholder ("Slice 3.9b will replace this component's `handle_msg` with a real RX-byte event" — `components/uart_pl011/README.md:48`).  The migration replaces it with the real generated shim.
- The slice-3.8 hold-queue keying (`(src, dst)` pairs) survives the migration unchanged — it's the right shape.  Sync calls add a parallel `block-on-resume-waitq` path; the existing buffered-message path is async-only.

## Decisions Made

- **Option A (full IPC routing) over Option B (pause-aware borrow).**  Why: long-term DESIGN alignment; the cost is real but bounded by the generator absorbing per-component shim authoring.
- **JSON IDL per interface, parsed by `tools/gen-iface.py`.**  Why: per-op metadata has no natural home in a C `struct ops`; project already runs Python tooling.
- **Sync-mode v1; async-mode-per-edge deferred to a later Phase 8 slice or Phase 10.**  Why: building real async (per-CPU dispatcher loops, reply waitqs, message wire-shape) on top of migrating 184 callsites is two phases of work.  Wrapper signatures are forward-compatible.
- **Sync-during-pause = block on `slot->resume_waitq`, not buffer.**  Why: caller has a stack; literal buffering would force async machinery for sync paths.  Async edges keep DESIGN's buffer semantics.
- **Per-op tagged message structs over generic argv.**  Why: easier to debug, easier for hooks to introspect, matches seL4 / L4 idl4 conventions.
- **Per-callsite equivalence tests land *before* migration in slice 8.0c.**  Why: the existing 453-test suite is too coarse a signal for behavior preservation across 30+ syscall sites.  Dual-callable window in slice 8.0b enables side-by-side assertion.
- **Hot-path perf gated to ≤10% regression on `read`/`write`/`open`.**  Why: the wrappers add atomic ops + hook walk; we need a numeric floor before continuing 8.0d → 8.1.
- **R3 is two-layer; AI verification is irreducible, not deferred.**  Earlier wording in DESIGN.md and AI-RULES.md framed the IDL machine check as a future "promotion" of R3 from AI to machine.  That overstated what IDL gives us: the meta-schema rejects payload fields *declared* as `slot_ref`, but C structurally permits `memcpy`'ing a slot pointer into a payload byte buffer regardless of any schema.  The IDL covers the *declarative* layer; AI review covers the *behavioral* layer (no hand-assembled payload bytes, no cast-through-integer smuggling, no hand-extension of generated message structs with trailing slot fields).  Both layers run concurrently; the AI procedure does not retire when the machine check lands.  AI-RULES.md §R3, DESIGN.md R1-R8 table, and AI-RULES.md §Evolution updated to reflect this — R3 is the canonical "two-layer" rule and the §Evolution section now explicitly distinguishes single-layer promotions from two-layer ones.  Why this matters for Phase 8: the IDL meta-schema is a deliverable of slice 8.0pre.1 (lands the schema validator alongside the generator); the existing R3 AI procedure stays in force throughout Group B's C-code migration (8.0a–e) and beyond.

## Status at End of Session

- No code changes.
- IMPLEMENTATION-GUIDE.md §Phase 8 expanded from a 5-step terse description to the full 15-slice plan with test plan + risk concentration.
- HANDOFF.md Current Status + Next Actions point at slice 8.0pre.1 as the next forward step.
- README.md Phase 8 row updated to reflect new scope (IPC migration prerequisite + live hot-swap + per-edge mode switching).
- Tests still 453/453; interactive still 7/7 (unchanged from Session 78).

## Next Steps

- **Session 80:** Detailed discussion of the generator IDL schema (per-op fields, cap declaration, return-type encoding, no-context vs. self-bearing ops, IRQ-entry shape).  Output: a draft schema spec + an example IDL file for `vfs`.  No code yet.
- **Session 81+:** Slice 8.0pre.1 kickoff — generator implementation + first interface (vfs).

---

**Files Changed:**

- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Phase 8 expanded with 3-group / 15-slice plan + test plan + risk concentration.
- `proj_docs/nonux/HANDOFF.md` — Current Status updated; Next Actions restructured around the new plan; Session 79 added to session list.
- `proj_docs/nonux/README.md` — Phase 8 row in Implementation Phases table updated.
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 74 entry rolled in from HANDOFF.md.
- `proj_docs/nonux/AI-RULES.md` — §R3 rewritten to two-layer framing (declarative machine check + behavioral AI procedure that never retires); §Evolution clarified to distinguish single-layer from two-layer rule promotions.
- `proj_docs/nonux/DESIGN.md` — R1-R8 table row for R3 updated to reflect two-layer enforcement.
- `proj_docs/nonux/logs/session-79-phase-8-plan.md` — this file.
