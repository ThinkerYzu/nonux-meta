# Session 85 — slice 8.0pre.4: Group A cutover paperwork

**Date:** 2026-04-30
**Phase:** Phase 8 (runtime recomposition + config manager) — Group A (generator infrastructure)
**Branch:** master
**Slice:** 8.0pre.4 — formalize "generator output is canonical" in the docs; audit and prune residual dual-maintenance language; declare Group A complete.
**Outcome:** Doc-only slice.  No code-shape changes; no in-tree generated header was modified.  `make verify-iface-fresh` clean throughout.  `make test` **495/495 pass**, `make test-interactive` **7/7 pass**, `make verify-registry` 0 findings — same baselines as Session 84.  Group A closed; Group B (IPC migration) starts at slice 8.0a.

---

## Goals

- Declare the IDL-driven generator's output canonical and document the enforcement in DESIGN.md (already wired in slice 8.0pre.1; the original plan called for a documented enforcement call-out).
- Audit the codebase + the Phase 8 spec docs for residual dual-maintenance language ("when the hand-written header is replaced", "post-cutover", "matches today's hand-written headers") and rewrite for the post-8.0pre.3 reality.
- Mark slice 8.0pre.4 ✓ in the implementation guide; bump status lines and footers; add a session log.

## What Was Done

### DESIGN.md — new §"Interface Definition Language" + Build Flow update

Added a new subsection between "Build Flow" and "Runtime Config Manager" formalizing the IDL pattern:

- Lists the five generator artefacts per IDL (`<iface>.h`, `<iface>_msg.h`, `<iface>_call.h`, `<iface>_dispatch.h`, optional `<iface>_isr.h`).
- States explicitly that generator output is **canonical** and **must not be hand-edited**.
- Documents `make verify-iface-fresh` as the machine check enforcing byte-equality, wired as a prerequisite of `make all` and `make test` alongside `make verify-registry` — same `-Werror`-grade gate.
- Documents the hand-written *types* header pattern (e.g. `interfaces/fs_types.h`) and the IDL's `includes:` field as the integration point.

The "Build Flow" diagram was extended to show the two parallel pipelines (kernel.json → validate-config → make → run, and interfaces/idl/*.json → gen-iface → verify-iface-fresh).  Verified with `make verify-iface-fresh` (clean) and `make test-tools` (93/93).

### DESIGN.md R7 row + AI-RULES.md R7 — parallel-rule note for IDL artefacts

R7's table row in DESIGN.md previously hedged with "Flips to machine when the workflow commits generated headers."  Updated to record that the parallel rule for IDL artefacts **is** machine-checked today, since `interfaces/<iface>.h` and `framework/<iface>_*.h` are committed and `make verify-iface-fresh` enforces byte-equality.  Manifest-derived `gen/<name>_deps.h` remains gitignored, so R7's manifest side keeps its existing AI-verified disposition.

AI-RULES.md's R7 section gained a parallel paragraph noting the same: IDL artefacts are committed; the rule is machine-checkable today via `verify-iface-fresh`; the manifest side remains as-is.

### Latent dual-maintenance language pruned

| File | Change |
|---|---|
| `tools/idl-meta-schema.json` | `constants` description rewritten — "preserves API surface … when the hand-written header is replaced by the generator output" → "carries the API surface that the IDL declares for the interface; the generator owns the value-column alignment and the per-group leading prose."  `includes` description expanded with the post-8.0pre.2 reality: emitted in both typedef and msg headers; suppresses auto-detected forward decls when non-empty. |
| `tools/README.md` | Gen-iface intro: "Per DESIGN.md R7, the IDL is the source of truth post-cutover" → "Per DESIGN.md §'Interface Definition Language', the IDL is the source of truth"; cross-reference points at the new section.  Type-system caveat: "asymmetric to match today's hand-written headers" → "asymmetric: u32 → uint32_t but i32 → int" (drops the historical-hand-written framing); added the optional-`ctype` mention to `opaque_self_handle`. |
| `tools/gen-iface.py` | Three source-comment touch-ups: "matches existing hand-written conventions" → "is the codified convention"; "matches today's hand-written conventions; an 80-char `open` signature wraps" → "the codified rule; an 80-char `open` signature wraps"; "Matches today's hand-written headers (vfs.h, fs.h both align across groups)" → "(vfs.h, fs.h both align across groups)" (dropped the "matches today's hand-written" framing).  No behavior change. |
| `Makefile` | `gen-iface` target comment: "(slice 8.0pre.1)" → "(slices 8.0pre.1 – 8.0pre.4)"; cross-reference points at the new DESIGN.md section in addition to IDL-SCHEMA.md.  `verify-iface-fresh` target comment: "Per DESIGN.md R7, the IDL is the source of truth" → "Per DESIGN.md §'Interface Definition Language', the IDL is the source of truth and this is the machine check (a parallel rule to R7's manifest-derived gen/<name>_deps.h)."  No recipe change. |

### IDL-SCHEMA.md and IMPLEMENTATION-GUIDE.md updated

- `IDL-SCHEMA.md` — Status line bumped to record slice 8.0pre.4's landing.  The "Slice 8.0pre.4 cutover scope" subsection rewrote in past tense ("landed Session 85"); explicitly notes the in-tree generated headers are *not* deleted (a fresh clone compiles without first running `make gen-iface`; reviewers can read the canonical contract directly).
- `IMPLEMENTATION-GUIDE.md` — slice 8.0pre.4 row in Group A table marked ✓ with a one-paragraph summary.  Phase 8 status line + "Last Updated" footer (both occurrences) updated to mark Group A complete and direct the next session to slice 8.0a.

### HANDOFF.md updated

(See HANDOFF.md update pass at end of session.)

### What was NOT done

- The in-tree generated headers (`interfaces/{vfs,fs,mm,scheduler,char_device}.h`, `interfaces/{vfs,fs,mm,scheduler,char_device}_msg.h`, `framework/{vfs,fs,mm,scheduler,char_device}_{call,dispatch}.h`, `framework/char_device_isr.h`) were **not** deleted.  Originally Session 83 framed cutover as "hand-written ops headers are deleted."  After 8.0pre.3 those files are already generator output (banner-tagged, byte-equal to a fresh regeneration).  Deleting them would force every fresh clone + every CI run to invoke `make gen-iface` before any useful build, with no compensating benefit — `make verify-iface-fresh` already proves they're not hand-edited.  Decision: keep them committed.  Recorded in IDL-SCHEMA.md §"Slice 8.0pre.4 cutover" so future readers don't trip on it.
- No new tests.  This is a documentation/spec slice; the tests that would prove the cutover landed (byte-equal regeneration, banner presence, `verify-iface-fresh` exit status) already exist in `tools/tests/test_gen_iface.py` (76 → 93 tests over 8.0pre.1–3).
- Manifest-derived `gen/<name>_deps.h` was **not** flipped from AI-verified to machine-checked.  R7 keeps its existing disposition because `gen/` is still gitignored (no committed copy to diff against).  Promoting that side is independent and not in scope for this slice.

## Key Findings

- The cutover work was almost entirely linguistic.  By end of slice 8.0pre.3 the generator infrastructure was already feature-complete for v1: every production interface IDL-driven, every artefact carrying `GENERATED — DO NOT EDIT`, `make verify-iface-fresh` already wired into `make all` and `make test` (since 8.0pre.1).  The "cutover paperwork" boiled down to (a) writing the §"Interface Definition Language" section that DESIGN.md was missing, (b) reflecting committed-IDL-artefacts in R7's table row + AI-RULES.md R7 prose, and (c) sweeping out the lingering "post-cutover" / "hand-written conventions" hedges that pre-dated the cutover landing.

- Latent dual-maintenance language tends to live in **descriptions**, not **code**.  The audit grep `(replaces? the hand-written|when the hand-written|hand-written header is replaced|dual.maintenance)` returned exactly one hit — meta-schema's `constants` description.  Broader greps (`hand-written|hand-edited`, `post-cutover|pre-cutover`, `until.*cutover|when.*cutover`) returned a handful of comment / README / tools-doc touch-ups, plus a few benign "matches today's hand-written headers" framing comments inside `gen-iface.py` itself.  No live code reaches around the generator; no scripts assume the in-tree files might be hand-edited.

- The "machine-check vs. AI-verified" disposition for R7 is now asymmetric: the IDL side is machine-checked (committed artefacts + `verify-iface-fresh`); the manifest side remains AI-verified (gitignored artefacts + generator-determinism tests).  Recording the asymmetry inline in the R7 row (rather than splitting into R7-IDL / R7-manifest as separate rules) keeps the rule landscape narrow and matches the existing R3 "two-layer" model (machine check on schema declarations + AI verification on C-code behavior).

## Decisions Made

- **Keep the in-tree generated headers committed (don't delete them as part of cutover).**
  - **Why:** Originally Session 83 framed the cutover as "hand-written ops headers are deleted."  At end of 8.0pre.3 those files are already generator output, banner-tagged, byte-equal to a fresh regeneration.  Deleting them would force every fresh clone + every CI invocation to run `make gen-iface` before any useful build — with no offsetting safety gain, since `make verify-iface-fresh` (already wired into `make all` / `make test`) already enforces the no-hand-edit invariant.
  - **How to apply:** A future slice that adds a *new* IDL gets a new committed generator output.  Authors regenerate and commit the artefacts together with the IDL change; `make verify-iface-fresh` blocks any drift.  No special-case directory exclusions, no `.gitignore` tweaks.

- **Document the IDL pattern as its own DESIGN.md subsection rather than amending Configuration and Manifests.**
  - **Why:** Manifests (kernel.json + components/<impl>/manifest.json) and IDL files (interfaces/idl/*.json) are conceptually parallel — both are JSON sources of truth that drive code generation — but they declare different things (composition vs. interface contracts) and have different generators (gen-config.py vs. gen-iface.py).  Folding the IDL into "Configuration and Manifests" would have muddied a section that's clearly about composition.
  - **How to apply:** Future cross-referenced subjects get their own subsection under "Configuration and Manifests" or wherever they fit topologically.  Avoid silently expanding existing sections to absorb tangentially-related new material.

- **Update R7's table row in place rather than splitting it into R7-IDL + R7-manifest.**
  - **Why:** The rule "generated artefacts must match a fresh generation from their declarative source; authors never hand-edit" is one rule applied to two artefact families.  The enforcement disposition (machine vs. AI) differs because of the gitignore arrangement, not because the rule differs.  Splitting would inflate the rule count without buying clarity; the existing R3 "two-layer" entry already establishes the precedent for noting layered enforcement inline.
  - **How to apply:** When adding parallel cases under an existing rule, prefer an inline note over a new rule unless the rules genuinely differ in what they require.

## Status at End of Session

- **Working:**
  - DESIGN.md §"Interface Definition Language" landed; "Build Flow" diagram updated; R7 row carries the parallel-rule note for IDL artefacts.
  - AI-RULES.md R7 carries the parallel-paragraph note for IDL artefacts.
  - IDL-SCHEMA.md status line + cutover-scope subsection reflect 8.0pre.4 landing.
  - IMPLEMENTATION-GUIDE.md slice 8.0pre.4 row marked ✓; Phase 8 status + Last Updated footers updated.
  - tools/idl-meta-schema.json `constants` and `includes` descriptions cleaned up.
  - tools/README.md, tools/gen-iface.py, Makefile comments aligned with post-cutover reality.
  - `make test` **495/495 pass**.
  - `make test-interactive` **7/7 pass**.
  - `make verify-iface-fresh` 0 drift.
  - `make verify-registry` 0 findings.
- **Known limitations (carried forward, unchanged from Session 84):**
  - `framework/<iface>_call.h` bodies are still forward declarations only; bodies land with slice 8.0a.
  - `framework/<iface>_dispatch.h` macro body is a skeleton; full unpack-and-call lands with slice 8.0b.
  - `framework/char_device_isr.h` carries declarations only; pool storage + `_from_irq` body land with slice 8.0a.
  - Generated `_call.h` / `_dispatch.h` / `_isr.h` artefacts are NOT yet included by production code — kernel build path unaffected; activation lands with 8.0a/8.0b.
  - `forward_decls_doc` schema field unused in production IDLs; cleanup deferred until a future revisit.
  - IRQ pool size hardcoded to 32; per-IDL knob deferred until a driver demands tuning.

## Next Steps

- **Slice 8.0a — kicks off Group B.**  Build `framework/slot_call.{h,c}` per [SLOT-CALL-API.md](../SLOT-CALL-API.md): `nx_slot_call_blocking(slot, msg)` API, per-task `caller_slot` lifecycle in `nx_task_create` / `_destroy`, kernel-side `posix_shim` component (manifest + impl + auto-generated `gen/posix_shim_deps.h`), rename today's userspace lib `components/posix_shim/` → `components/libnxlibc/`, per-CPU reply-message pool sized 256, `NX_EABORT = -10`, `posix_shim_on_dep_swapped` invalidating handle table on `STATE_LOST`, one-section update to DESIGN.md (sync-mode shortcut requires caller on a dispatcher; syscall edges therefore `mode: async`).  Cross-cutting test infrastructure (mock component, hook-chain inspector, recompose event logger, pause-injector fixture, cap-forgery harness, equivalence-runner macro) lands here too.  ~70 ktests.  This unblocks tightening every generated `_call.h` / `_dispatch.h` / `_isr.h` body so it's the highest-leverage Group B slice.

- **No deferrals introduced this slice.**  All deferral lists (signal-handler trampoline, fd-to-slot alignment, reap-on-wait, vector I/O follow-ons, Phase 5 follow-ups, test-harness follow-ups, verify-registry extensions, infra polish, GUI) carry forward unchanged from Session 84.

---

**Files Changed:**

- `proj_docs/nonux/DESIGN.md` — new §"Interface Definition Language" subsection (after "Build Flow", before "Runtime Config Manager"); "Build Flow" diagram extended with the IDL pipeline; R7 table row gained the parallel-rule note for IDL artefacts.
- `proj_docs/nonux/AI-RULES.md` — R7 section's "Machine check" gained a parallel-paragraph note for IDL artefacts (machine-checked today; manifest side remains AI-verified).
- `proj_docs/nonux/IDL-SCHEMA.md` — Status line bumped (records 8.0pre.4 landing); "Slice 8.0pre.4 cutover scope" subsection rewritten in past tense; explicit decision-record on keeping the in-tree generated headers committed.
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — slice 8.0pre.4 row in Group A marked ✓ with a one-paragraph summary; Phase 8 status line records Group A complete + directs next session to slice 8.0a; "Last Updated" footers (both copies) updated.
- `proj_docs/nonux/HANDOFF.md` — Current Status, Next Actions, Phase Checklist, Session Logs all bumped (see HANDOFF.md update pass).
- `proj_docs/nonux/logs/session-85-8.0pre.4-cutover.md` — NEW (this file).
- `sources/nonux/tools/idl-meta-schema.json` — `constants` description rewritten; `includes` description expanded with post-8.0pre.2 behavior.
- `sources/nonux/tools/README.md` — gen-iface intro cross-reference updated; type-system caveat cleaned up; `opaque_self_handle` ctype-option added to the type-system list.
- `sources/nonux/tools/gen-iface.py` — three comment touch-ups dropping "matches today's hand-written" framing in favor of "the codified rule" / direct fact.  No behavior change; `make verify-iface-fresh` clean.
- `sources/nonux/Makefile` — `gen-iface` and `verify-iface-fresh` target comments updated to reference DESIGN.md §"Interface Definition Language" and to record slices 8.0pre.1 – 8.0pre.4 as the relevant lineage.  No recipe change.
