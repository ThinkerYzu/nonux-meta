# Session 83 — slice 8.0pre.2: `fs` interface IDL + `fs_types.h` types-header split

**Date:** 2026-04-30
**Slice:** 8.0pre.2 — `interfaces/idl/fs.json` + hand-written `interfaces/fs_types.h` + generator extension to emit `includes:` in both typedef header and msg header.
**Outcome:** Second IDL lands.  Data shapes (`struct nx_fs_dirent`, `struct nx_fs_stat`, `NX_FS_DIRENT_NAME_MAX`, `NX_FS_KIND_*`) factored into a hand-written `interfaces/fs_types.h` shared between `fs.h` and `vfs.h` via the IDL's existing `includes:` array.  Generator emits author-declared includes in both `<iface>.h` and `<iface>_msg.h`; auto-detected forward declarations are suppressed when `includes:` is non-empty (the includes carry the full definitions).  IDL describes operations only; data layout stays in C.  Tests: `make test` 471 → **478/478 pass** (+7 new python tests covering `includes:` behavior in both header types and forward-decl suppression); `make test-interactive` **7/7 pass**; `make verify-iface-fresh`: 0 drift.

---

## Goal

Slice 8.0pre.1 (Session 82) landed the generator with `vfs` as the worked example.  vfs's IDL was a clean fit because it only references `struct nx_fs_dirent` / `struct nx_fs_stat` via opaque ctype names and forward-declares them.  The slice plan called out 8.0pre.2 as the de-risk for *richer ctype shapes* — and `fs.h` is the case: it owns the full definitions of those types plus their tightly-bound constants (`NX_FS_DIRENT_NAME_MAX`, `NX_FS_KIND_*`).

The question for the generator was *where* those definitions live: inline in the IDL (a new schema section), or in a hand-written types header that the IDL declares as a dependency.  Initial implementation took the inline path; user redirected mid-slice to the types-header path on the principle that **IDL describes operations, C describes data layout**.  Final landing reflects the redirect.

---

## What landed

### Hand-written `interfaces/fs_types.h`

New file, ~50 lines.  Contains:

- `struct nx_fs_dirent` (with `name_len` + `name[NX_FS_DIRENT_NAME_MAX]`)
- `struct nx_fs_stat` (with `kind` + `size`)
- `NX_FS_DIRENT_NAME_MAX` (the array-size macro `struct nx_fs_dirent` references)
- `NX_FS_KIND_FILE` / `NX_FS_KIND_DIR` (the kind discriminants `struct nx_fs_stat` carries)

Header guard `NONUX_INTERFACE_FS_TYPES_H`.  Includes only `<stddef.h>` + `<stdint.h>`.  Hand-written, manually maintained — `make verify-iface-fresh` does not regenerate it.

The split between this file and `interfaces/fs.h` (generated): operational constants (`NX_FS_OPEN_*`, `NX_FS_SEEK_*`) live in the IDL because they're part of op signatures; type-coupled constants and structs live here because they're part of data layout.

### Generator changes (`tools/gen-iface.py`)

Three changes — none of them require schema changes.

1. **Author-declared includes emitted in `_msg.h`.**  The existing `includes:` field (added in 8.0pre.1) only emitted into the typedef header.  Now also emits into `interfaces/<iface>_msg.h`, immediately after `<stddef.h>` / `<stdint.h>`.  Necessary because `_msg.h`'s per-op request/reply structs embed types like `struct nx_fs_dirent out;` by value — the compiler needs the full definition to compute sizeof.  The generator does not auto-detect when this is needed; it just emits the same `includes:` list verbatim in both headers.

2. **Forward-declaration auto-detection suppressed when `includes:` is non-empty.**  Old behavior: the generator walked op-param `ctype` fields and emitted `struct foo;` for any tagged type.  New behavior: if the IDL declares any `includes:`, the author has explicitly said where the types come from, so the generator skips the auto-detection.  Eliminates the redundant `struct foo; ... #include "fs_types.h"` (which has the full def).  Heuristic but reasonable — an IDL that includes a types header is taking responsibility for where its types come from.

3. **No new schema fields.**  Previous draft added a `structs:` top-level array for inline struct definitions; that was reverted per the redirect (see Decisions Made).  Only existing fields used.

### IDL changes (`interfaces/idl/`)

`fs.json` (new, ~110 lines):

- Top-level `doc` — verbatim prose from `fs.h`'s leading block (Path convention / Ownership / Concurrency / Error convention).
- `includes: [{path: "interfaces/fs_types.h", doc: "..."}]` — pulls in the hand-written types.
- `constants:` — 7 entries in 2 groups: `NX_FS_OPEN_*` (4) and `NX_FS_SEEK_*` (3).  `NX_FS_DIRENT_NAME_MAX` and `NX_FS_KIND_*` are NOT here — they live in `fs_types.h` because they're tightly bound to the struct definitions.
- `ops:` — 9 ops, op-IDs 1–9 mirroring vfs's per the deliberate shape parity.
- `iface_id`: 4098 (vfs is 4097).

`vfs.json` (modified):

- Added `includes: [{path: "interfaces/fs_types.h", doc: "..."}]` — same path as fs.json's, so both interfaces share the types via `#include`.
- Removed `forward_decls_doc` field (unused now that includes-driven path skips auto-detected forward decls).

### `interfaces/fs.h` canonicalization

Running the generator with the new IDL produces an `fs.h` that differs from the hand-written one along these axes:

| Difference | Resolution |
|---|---|
| Banner added (`GENERATED — DO NOT EDIT`). | Expected. |
| `#include "interfaces/fs_types.h"` added at top. | New: pulls in the data shapes (struct definitions + tightly-bound constants). |
| Inline `struct nx_fs_dirent` / `struct nx_fs_stat` definitions removed. | Moved to `fs_types.h`.  Operations header now describes operations only. |
| `NX_FS_DIRENT_NAME_MAX` + `NX_FS_KIND_*` constants removed from `fs.h`. | Moved to `fs_types.h` (they belong with the structs that consume them). |
| `/* ---------- Section name ----- */` divider comments removed. | Canonicalized away — each section's prose lives on its leading constant or struct. |
| Per-value `NX_FS_SEEK_*` trailing comments (`/* absolute offset */` etc.) folded into group leading doc. | Canonicalized away. |
| `int (*open)(...)` un-wrapped (fits 79 chars on single line). | Generator's greedy ≤79 wrap. |
| `int (*readdir)(...)` rewrapped after `cookie,` (was after `dir_path,`). | Generator's greedy ≤79 wrap. |
| Compact-vs-expanded multi-line comment styles mixed in source. | Single rule applied (single-line if fits, expanded otherwise). |

C-level shape unchanged (same `#define`s, same struct field types + names, same op signatures) so callers compile identically.  Per DESIGN.md R7, the IDL is source of truth post-cutover; `make verify-iface-fresh` enforces byte-equivalence.

`vfs.h` similarly regenerates with `#include "interfaces/fs_types.h"` and the bare `struct nx_fs_dirent;` / `struct nx_fs_stat;` forward-declaration block deleted (forward decls suppressed because `includes:` is non-empty).

### `_msg.h` headers (auto-fix)

A latent issue noted but deferred in the previous (reverted) draft: `interfaces/fs_msg.h` and `interfaces/vfs_msg.h` reference struct ctypes inline in msg structs (`struct nx_fs_msg_readdir { ...; struct nx_fs_dirent out; }`) but didn't `#include` the header that defines them.  Fixed in this slice as a side-effect of the generator change: both msg headers now emit `#include "interfaces/fs_types.h"` after the standard includes, matching their typedef-header siblings.  The msg headers are still not yet consumed by production code, but they now compile if anyone tried to build them.

### Tests (`tools/tests/test_gen_iface.py`)

7 new test methods across 2 new classes:

| Class | Count | Coverage |
|---|---|---|
| `TestFsByteForByte` | 2 | in-tree `interfaces/fs.h` byte-equal to a fresh regeneration; IDL declares `interfaces/fs_types.h` via `includes:`. |
| `TestIncludesAuthorDeclared` | 5 | `includes:` paths emitted in typedef header and msg header; forward-decl auto-detection suppressed when `includes:` is non-empty; legacy behavior preserved when `includes:` is absent; both headers carry the same include list. |

`make test-tools` 69 → **76**.

---

## Decisions Made

- **Hand-written types header over inline `structs:` schema extension.**  Initial draft added a `structs:` top-level array to the IDL with verbatim-string `body` fields, so an interface could fully describe its data shapes inline.  Worked end-to-end (47 lines of generator code, 1 schema definition, 5 tests, byte-equal canonicalization).  User asked: "Is inline struct definitions necessary?  Is it for types referenced by an interface?"  Reframed the problem as IDL-purpose: **the IDL describes interface contracts (operations); data layout is C concern.**  Verbatim `body` strings lacked schema validation, complicated the meta-schema, and didn't earn their keep — every benefit (single source of truth per interface) was matched by a cost (the IDL author edits a JSON-encoded C string instead of plain C).  Reverted the `structs:` extension; switched to author-declared `includes:` referencing a hand-written types header.

- **`_msg.h` emits the same `includes:` as `<iface>.h`.**  The msg header embeds struct values by sizeof in its per-op request/reply structs, so it needs the full definition just like the typedef header does.  Considered: (a) auto-detect when `_msg.h` references a non-primitive `ctype` and emit `#include "<iface>.h"` (transitive via the typedef header).  (b) Always emit the same author-declared `includes:` in both headers.  Took (b): simpler, no auto-detection, the IDL author's declared dependencies are honored uniformly.

- **Forward-decl auto-detection suppressed when `includes:` is non-empty.**  Without suppression, an IDL that declares `includes: [{path: "interfaces/fs_types.h"}]` AND has an op with `ctype: "struct nx_fs_dirent"` would emit *both* `#include "interfaces/fs_types.h"` (which has the full def) AND a bare `struct nx_fs_dirent;` (forward decl).  Legal C but visually noisy.  Suppression is a heuristic — it's conceivable an IDL has includes for some other purpose and still wants forward decls — but the typical case is "the includes carry my types," and authors who need explicit forward decls can put them in their hand-written header.

- **`NX_FS_DIRENT_NAME_MAX` and `NX_FS_KIND_*` go in `fs_types.h`, not `fs.json`.**  These constants are part of struct field shape (array size; kind discriminant), tightly bound to the struct definitions.  Splitting them across two files (constant in IDL, struct in types header) would require redundant include semantics.  Operational constants (`NX_FS_OPEN_*`, `NX_FS_SEEK_*`) stay in `fs.json` because they're part of op signatures.  The principle: constants live with the thing that defines their meaning.

- **`forward_decls_doc` field is now dead in vfs.json.**  Removed.  Schema field still exists (might be useful for an IDL that declares no `includes:` but consumes external types — none today).  Could be removed in a future cleanup.

---

## Key Findings

- **The IDL's job is operations.**  The generator's role is producing the IPC machinery (msg structs, sender wrappers, dispatch templates) and the typedef header that ties op signatures together.  Data layout is naturally C — the compiler reasons about it, hand-written code already supports any C-legal shape.  Forcing struct field declarations into JSON `body` strings is awkward; a hand-written `_types.h` is the natural home.

- **Typedef and msg headers share an include set.**  Anywhere the IDL references a non-primitive ctype, both headers need access to its definition (typedef for op signature compilation; msg header for embedded struct sizeof).  The cleanest path is: emit the same `includes:` in both.

- **Forward-decl auto-detection and explicit `includes:` are mutually exclusive in practice.**  An IDL declaring `includes:` for a header that defines its struct types doesn't want bare-line forward decls — they're noise after the include's full def lands.  Suppression on non-empty `includes:` is correct nine times out of ten.

- **The 8.0pre.4 cutover (delete all hand-written `interfaces/*.h`) is now narrower in scope.**  Originally the plan was to delete every hand-written ops header.  Now `interfaces/fs_types.h` (and any future `<iface>_types.h`) survives the cutover — the cutover only deletes the generator-replaceable ops headers (`fs.h`, `vfs.h`, eventually `mm.h` / `scheduler.h` / `char_device.h`).  Types headers are explicitly hand-written, declared as IDL dependencies, and stay.

---

## Status at End of Session

### Working
- `tools/gen-iface.py` emits author-declared `includes:` in both typedef header and msg header.
- `tools/idl-meta-schema.json` unchanged from 8.0pre.1 (no `structs:` field; existing `includes:` field carries the workload).
- `interfaces/fs_types.h` is hand-written and shared by `fs.h` and `vfs.h` via `includes:`.
- `interfaces/idl/fs.json` is the second worked IDL (after vfs.json).
- `interfaces/fs.h`, `interfaces/vfs.h` are canonical generator output.
- `interfaces/fs_msg.h`, `interfaces/vfs_msg.h` carry the same `#include "interfaces/fs_types.h"` so embedded struct fields have their definitions in scope.
- `framework/fs_call.h`, `framework/fs_dispatch.h` generated.
- `make test`: **478/478 pass** (76 python + 283 host + 119 kernel; +7 from 8.0pre.2 tests).
- `make test-interactive`: **7/7 pass**.
- `make verify-iface-fresh`: 0 drift.
- `make verify-registry`: 0 findings.

### Known limitations (carried forward)
- `_call.h` bodies are still forward declarations only; bodies land with slice 8.0a.
- `_dispatch.h` macro body is still a skeleton; full unpack-and-call logic lands with slice 8.0b.
- `forward_decls_doc` schema field is now unused in production IDLs; could be removed in a future cleanup.

---

## Next Steps

- **Slice 8.0pre.3** — apply the generator to `interfaces/sched.h`, `interfaces/mm.h`, `interfaces/char_device.h`.  `sched` and `mm` should be straightforward (similar shape to vfs/fs).  `char_device` introduces IRQ-entry ops (`context: "irq"`) which the schema already supports but the generator doesn't yet emit a `framework/<iface>_isr.h`.  That generator extension is part of 8.0pre.3.

- **Slice 8.0a** — kernel-side `framework/slot_call.{h,c}` infrastructure.  Independent of 8.0pre.2/3; can be done in parallel.  Once 8.0a lands the runtime, `_call.h` wrapper bodies and `_dispatch.h` macro bodies can be tightened to emit full pack/unpack + dispatch logic.

- **Slice 8.0pre.4 (cutover)** — when all four interfaces are IDL-driven, delete `interfaces/sched.h` / `mm.h` / `char_device.h` (the remaining hand-written ops headers — `fs.h` and `vfs.h` are already generator output post-canonicalization).  Hand-written *types* headers (`fs_types.h` and any future `<iface>_types.h`) survive the cutover.  `make gen-iface` runs as a Make prerequisite; `make verify-iface-fresh` is enforced.

---

**Files Changed:**
- `sources/nonux/tools/gen-iface.py` — `_msg.h` now emits author-declared includes; forward-decl auto-detection suppressed when `includes:` non-empty.
- `sources/nonux/tools/idl-meta-schema.json` — unchanged from 8.0pre.1 (the `structs:` extension prototyped earlier in this session was reverted).
- `sources/nonux/interfaces/fs_types.h` — NEW hand-written.  Contains `struct nx_fs_dirent`, `struct nx_fs_stat`, and the constants tightly bound to them.
- `sources/nonux/interfaces/idl/fs.json` — NEW.  Top-level doc + `includes` (for fs_types.h) + 7 constants in 2 groups + 9 ops.  No `structs:` field (the prototyped extension was reverted).
- `sources/nonux/interfaces/idl/vfs.json` — added `includes` for fs_types.h; removed `forward_decls_doc` (unused now that includes-driven path skips auto-detected forward decls).
- `sources/nonux/interfaces/fs.h` — regenerated to canonical generator output.
- `sources/nonux/interfaces/vfs.h` — regenerated.  No content drift in op signatures; forward-decl block replaced by the new `#include`.
- `sources/nonux/interfaces/fs_msg.h` — NEW (generated).
- `sources/nonux/interfaces/vfs_msg.h` — regenerated; gained `#include "interfaces/fs_types.h"`.
- `sources/nonux/framework/fs_call.h` — NEW (generated).
- `sources/nonux/framework/fs_dispatch.h` — NEW (generated).
- `sources/nonux/tools/tests/test_gen_iface.py` — +7 tests (TestFsByteForByte, TestIncludesAuthorDeclared); the previously-prototyped TestInlineStructs class was removed alongside the `structs:` extension revert.
