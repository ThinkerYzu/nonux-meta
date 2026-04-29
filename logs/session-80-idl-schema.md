# Session 80: Generator IDL Schema — Spec + vfs Example

**Date:** 2026-04-29
**Phase:** Phase 8 prep (no code changes; design only)
**Branch:** master

---

## Goals

- Lock the JSON IDL schema for `tools/gen-iface.py` ahead of slice 8.0pre.1 implementation.
- Produce a draft schema spec doc + a worked vfs IDL example.
- Pre-resolve eight design questions (file format, opaque-handle wire shape, op-ID stability, cap count, return-type encoding, `mm.alloc_pages` shape, per-op pause policy, meta-schema strictness).

## What Was Done

### Schema design discussion

Walked the user through a draft IDL proposal organised around the five HANDOFF questions Session 79 deferred (per-op fields, cap declaration, return-type encoding, no-context vs. self-bearing ops, IRQ-entry shape) plus eight smaller open questions. User accepted all eight defaults; followed up with a request to explain caps in detail.

### Cap explainer

Walked the user through `framework/ipc.h`'s cap machinery: caps live in `msg->caps[]` (not payload); `SLOT_REF` is the only kind today, with `HANDLE` and `VMO_CHUNK` reserved for Phase 5/7; `BORROW` is default and dropped after handler return; `TRANSFER` requires the receiver to claim via `nx_slot_ref_retain` or trip a protocol-error count. Tied this back to DESIGN.md R3 (two-layer slot-ref discipline) and to how the generator emits `slot_ref` params: sender wrappers thread the pointer through `msg->caps[]`, message structs carry only the cap_id integer, receiver dispatch templates re-extract the slot pointer from the cap. Today's five interfaces have zero `slot_ref` params, so v1 cap support is plumbing without live users; first real consumer is Phase 8.3's config-handle API.

### Spec document landed

Wrote `proj_docs/nonux/IDL-SCHEMA.md`. Sections:
- File layout (`interfaces/idl/<iface>.json`, meta-schema, `make gen-iface` + `make verify-iface-fresh`).
- Top-level schema (interface, version, iface_id, ops).
- Per-op record (name, op_id, params, returns, context, pause_policy, trace_tag).
- Closed param-type system: 12 types covering scalars, strings, byte buffers, structs, opaque handles, slot_refs.
- Cap section: BORROW vs TRANSFER, cap-scan, R3 rationale.
- Return-type encoding: `void`, `int_status`, `i64_count_or_status`, `usize`, `void_ptr` + optional `out_param` cross-reference.
- `self` is implicit (resolved by dispatch template from `slot->active->impl_data`).
- IRQ-entry ops: `context: "irq"` implies POD-only message, no caps, fixed-size pre-built pool, generated `<iface>_isr.h`.
- Generated artifacts table (5 outputs per IDL; the 5th lands only if any op declares irq context).
- Worked vfs example with all 9 ops mapped to today's `interfaces/vfs.h` signatures.
- Open items (per-op pause-policy field reserved; future cap kinds; multi-out_param; generator-version tag; cross-IDL checks; async-mode wire shape; `mm.alloc_pages` reshape).

### Doc-web wiring

- Added IDL-SCHEMA.md to project navigation (HANDOFF Project Docs row, IMPLEMENTATION-GUIDE row).
- Added explicit reference from IMPLEMENTATION-GUIDE.md §Phase 8 Group A to the spec doc.
- Updated HANDOFF.md Current Status + Next Actions: slice 8.0pre.1 is now "implement gen-iface.py against IDL-SCHEMA.md", no remaining design discussion needed.

## Key Findings

- The compatibility test for slice 8.0pre.1 is "generated `interfaces/vfs.h` matches the existing hand-written file byte-for-byte (modulo the GENERATED banner)". Locking this constraint up front means callers don't change at all when 8.0pre.1 lands — the migration risk is concentrated in the generator template, not in 30+ syscall.c sites.
- `opaque_self_handle` is the v1 punt for `void *file` in the fs/vfs interface. Sender never inspects it; receiver minted it on `open` and gets it back on `read`/`write`/`close`. **Not a cap** (it never crosses to a third party). Encoded as a plain `u64` on the wire — no producer-slot tag (locked Session 80) since the higher-level handle table already enforces routing.
- Per-op pause-policy override is reserved as a field but ignored in v1. Future-proofing the IDL means v1 IDL files don't need rewriting when the field becomes live (locked Session 80).
- IRQ-entry ops (`context: "irq"`) won't land until slice 8.0pre.3 — `char_device` is the first user, after slice 3.9b's deferred reshape. The schema accommodates it now so the design isn't backloaded.

## Decisions Made

All eight open questions resolved:

- **JSON over YAML over custom DSL** — zero pip deps; project already runs Python tooling.
- **`opaque_self_handle` = plain `u64`** — handle table enforces routing; producer-slot tag is overkill.
- **Op-ID stability frozen; gravestones for removed ops** — `verify-iface-fresh` walks git history to flag reuse.
- **Unbounded `slot_ref` params per op** — dispatch template handles a small `caps[]` walk.
- **Combined `returns` block with optional `out_param`** — split into `out_params: [...]` array if/when a real op needs multi-out.
- **`mm.alloc_pages` keeps `void_ptr` returns** — reshape to `int alloc_pages(order, **out)` is a behavior change; defer until a slice forces it.
- **Per-op pause-policy reserved, not honoured in v1** — declarative-only; revisit when a real conflict appears.
- **Strict meta-schema validation** — unknown keys rejected; the IDL is the source of truth.

Plus three design moves not in the original eight questions:

- **Type-specific cross-references via `len: <name>` / `cap: <name>`** for `bytes_in` / `bytes_out` — keeps the schema flat (no nested type unions) while still binding paired params.
- **`direction: "in" | "out" | "inout"` per param** — generator uses it to decide copy-in/copy-out; defaults to `"in"`.
- **`string_in` as a distinct type from `bytes_in`** — IDL author doesn't have to introduce a length param for paths and similar; generator measures length from NUL terminator and caps at a per-iface max.

## Status at End of Session

- `proj_docs/nonux/IDL-SCHEMA.md` written (~250 lines + vfs example).
- `proj_docs/nonux/logs/session-80-idl-schema.md` (this file).
- HANDOFF.md + IMPLEMENTATION-GUIDE.md updated with cross-references.
- No source code changes.
- Tests unchanged: `make test` 453/453; `make test-interactive` 7/7.

## Next Steps

- **Session 81+:** Slice 8.0pre.1 implementation kickoff.
  1. Write `tools/idl-meta-schema.json` (JSON Schema draft-07 form of IDL-SCHEMA.md's rules).
  2. Write `tools/gen-iface.py` skeleton: meta-schema validation + emit each of the 4 (or 5 if irq) artifacts.
  3. Write `interfaces/idl/vfs.json` (the body is essentially copy-paste from IDL-SCHEMA.md's vfs example).
  4. Wire `make gen-iface` + `make verify-iface-fresh` into the top-level Makefile.
  5. Compatibility test: generated `interfaces/vfs.h` matches the existing hand-written file byte-for-byte modulo banner.
  6. Add the slice's ~12 ktests on schema validation + round-trip pack/unpack against the worked vfs IDL.

- Slice 8.0a's API contract for `nx_slot_call_sync` is a prerequisite for the wrapper template — that's the next risk-concentrating item to settle before 8.0pre.1 starts. Worth a brief spike (1 session) on the sync-call signature once 8.0pre.1's generator skeleton is up enough to produce sender-wrapper output.

---

**Files Changed:**

- `proj_docs/nonux/IDL-SCHEMA.md` — new doc; the IDL schema spec + worked vfs example.
- `proj_docs/nonux/HANDOFF.md` — Current Status + Next Actions updated; Session 80 added to session list; doc-web row gains IDL-SCHEMA link.
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Phase 8 Group A row for slice 8.0pre.1 cross-references IDL-SCHEMA.md.
- `proj_docs/nonux/logs/session-80-idl-schema.md` — this file.
