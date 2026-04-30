# IDL Schema: nonux Component Interface Definition Language

**Project:** nonux
**Created:** 2026-04-29 (Session 80)
**Status:** Locked at Session 80; landed in slice 8.0pre.1 (Session 82).  The meta-schema gained a `forward_decls_doc` top-level field; the generator auto-detects forward-declaration *types* from op-param ctypes and uses this field for the optional block-comment prose.  Slice 8.0pre.2 (Session 83) extended the generator to also emit author-declared `includes:` in the message header, and to suppress auto-detected forward declarations when `includes:` is non-empty — interfaces that own data-shape types put them in a hand-written `<iface>_types.h` and reference it via `includes:`, keeping the IDL focused on operations.  Slice 8.0pre.3 (Session 84) added three orthogonal extensions: optional `ctype` on `opaque_self_handle` params (typed kernel-object pointers like `struct nx_task *task`); optional `ctype` on `void_ptr` returns (typed pointer returns like `pick_next` → `struct nx_task *`); new `u32` / `u64` return types (e.g. `mm.max_order` → `uint32_t`).  Same slice activated `framework/<iface>_isr.h` emission for IDLs with any `context: "irq"` op.

---

## Navigation

**Project Docs:** [README](README.md) | [SPEC](SPEC.md) | [DESIGN](DESIGN.md) | [IMPLEMENTATION-GUIDE](IMPLEMENTATION-GUIDE.md) | [HANDOFF](HANDOFF.md) | [IDL-SCHEMA](IDL-SCHEMA.md) *(you are here)* | [SLOT-CALL-API](SLOT-CALL-API.md)

**This Document:**
- [Purpose](#purpose)
- [File Layout](#file-layout)
- [Top-Level Schema](#top-level-schema)
- [Per-Op Record](#per-op-record)
- [Param Type System](#param-type-system)
- [Capabilities (`slot_ref`)](#capabilities-slot_ref)
- [Author-Declared Includes (`includes:`)](#author-declared-includes-includes)
- [Return Types](#return-types)
- [`self` Is Implicit](#self-is-implicit)
- [IRQ-Entry Ops](#irq-entry-ops)
- [Generated Artifacts](#generated-artifacts)
- [vfs Example](#vfs-example)
- [Open Items / Future Extensions](#open-items--future-extensions)

---

## Purpose

Phase 8 routes every cross-component call through `nx_slot_call_sync` (Option A, locked Session 79). Hand-shimming the migration would require ~150 dispatch arms of mechanical, error-prone code across five interfaces. Instead, **a JSON IDL per interface, parsed by `tools/gen-iface.py`, emits four artifacts per interface** (typedef header, message structs, sender wrappers, receiver dispatch template).

This document specifies the IDL schema. Slice 8.0pre.1 implements the generator and lands the first IDL file (`vfs`); slices 8.0pre.2–4 cover the remaining interfaces and cut over.

---

## File Layout

- **One JSON file per interface:** `interfaces/idl/<iface>.json`.
- **Meta-schema:** `tools/idl-meta-schema.json` (JSON Schema draft-07).
- **Generator:** `tools/gen-iface.py` reads `interfaces/idl/*.json`, writes the four artifacts per interface. Idempotent — re-running with no IDL change produces byte-identical output.
- **Build wiring:** `make gen-iface` regenerates everything; `make verify-iface-fresh` fails the build if any generated artifact is stale or any IDL file fails meta-schema validation. Both run as part of `make` and `make test`.

---

## Top-Level Schema

```json
{
  "interface": "vfs",        // identifier; must match filename stem
  "version": 1,              // bumped on breaking changes (op signature, removed op, type change)
  "iface_id": 4097,          // 16-bit stable id; unique across interfaces
  "doc": "...",              // optional; copied verbatim into header comment
  "includes": [ ... ],       // optional; author-declared #include lines (see Author-Declared Includes)
  "constants": [ ... ],      // optional; #define lines (see Constants section)
  "forward_decls_doc": "...", // optional; prose above auto-detected forward decls (only when no includes:)
  "ops": [ /* per-op records, see below */ ]
}
```

### Validation rules

- `interface` matches `^[a-z][a-z0-9_]*$` and matches the JSON filename stem.
- `iface_id` in `[1, 65535]`, globally unique. Future `verify-iface-fresh` extension will check uniqueness across all IDL files.
- `version` starts at 1; bump on any wire-incompatible change. The framework does not currently consume this (single-binary kernel; sender + receiver always agree), but future hot-attach scenarios may.
- Strict validation: unknown top-level keys are rejected. (Locked Session 80.)

---

## Per-Op Record

```json
{
  "name": "open",                // identifier; matches ^[a-z][a-z0-9_]*$
  "op_id": 1,                    // stable; never reused. Removed ops are gravestones.
  "params": [ ... ],             // ordered, named, typed
  "returns": { ... },            // see Return Types
  "context": "task",             // "task" (default) or "irq"
  "pause_policy": null,          // null = inherit slot default; v1 honours only inherit
  "trace_tag": "vfs_open",       // optional; default = "<iface>_<name>"
  "doc": "..."                   // optional; copied into generated comments
}
```

### Op-ID stability

Once an op has an id, **it is frozen**. Removing an op leaves the id as a gravestone — never reuse it. (Locked Session 80.) The meta-schema does not enforce this on its own; a `verify-iface-fresh` rule walks git history to flag id reuse.

### Per-op pause-policy override

The `pause_policy` field is reserved for future use. v1 ignores any value other than `null` and inherits the slot's pause policy. (Locked Session 80.) Documented now so IDL files written for v1 don't need rewriting when the field becomes live.

---

## Param Type System

Closed set, mapped to C and to wire shape. Unbounded count of any param type per op (locked Session 80) — including unbounded `slot_ref`.

| IDL type | C signature contribution | Wire encoding |
|---|---|---|
| `u8`, `u16`, `u32`, `u64` | unsigned scalar | by value in payload |
| `i8`, `i16`, `i32`, `i64` | signed scalar | by value in payload |
| `usize` | `size_t` | by value (8 bytes on aarch64) in payload |
| `bool` | `bool` | 1 byte in payload |
| `string_in` | `const char *` (NUL-terminated, borrowed) | 4-byte length + bytes (incl. NUL) in payload |
| `bytes_in { len: <name> }` | `const void *buf, size_t len` pair | 4-byte length + bytes in payload |
| `bytes_out { cap: <name> }` | `void *buf, size_t cap` pair | reserved slot in payload; receiver fills, wrapper copies back to caller buffer |
| `struct_in { ctype }` | by-value POD copy of named C struct | inline bytes in payload (sizeof struct) |
| `struct_out { ctype }` | caller pointer to POD struct, callee fills | reserved slot in payload; copy-back |
| `struct_inout { ctype }` | caller pointer to POD struct, callee may read+write | inline + copy-back |
| `slot_ref { ownership }` | `struct nx_slot *` | **lives in `msg->caps[]`, never payload** |
| `opaque_self_handle` | `void *` (opaque to sender) — or `<ctype> *` if optional `ctype` is present | `u64` in payload |

### Param record shape

```json
{
  "name": "buf",
  "type": "bytes_out",
  "cap": "buflen",          // for bytes_out; names the param holding the cap
  "direction": "out",       // optional; "in" (default), "out", or "inout"
  "doc": "..."              // optional
}
```

### Type-specific fields

| Type | Required extra fields |
|---|---|
| `bytes_in` | `len: <param-name>` — name of the param holding the length |
| `bytes_out` | `cap: <param-name>` — name of the param holding the buffer cap |
| `struct_in` / `struct_out` / `struct_inout` | `ctype: "struct nx_fs_dirent"` (verbatim C type name) |
| `slot_ref` | `ownership: "borrow"` (default) or `"transfer"` |
| `opaque_self_handle` | `ctype: "struct nx_task"` (optional; slice 8.0pre.3) — replaces the default `void` in the C signature.  Wire shape unchanged (still `u64` in payload).  Used for typed kernel-object pointers that cross the IPC boundary as borrowed handles. |

Any param may carry `"direction": "in" | "out" | "inout"` (default `"in"`). The generator uses this to decide copy-in/copy-out behavior in the wrapper and dispatch template.

### Why `string_in` is its own type (not `bytes_in` with implicit length)

`string_in` lets the IDL declare paths and similar without forcing the IDL author to introduce a separate length param. The generator measures length from the NUL terminator, caps at a per-iface max (configurable in the schema; default 4096), rejects non-NUL-terminated input. `bytes_in` is the lower-level form for binary data with explicit length.

---

## Capabilities (`slot_ref`)

A **cap** is a typed reference to a kernel object that travels in `msg->caps[]`, *not* in `msg->payload[]`. From `framework/ipc.h`:

```c
struct nx_ipc_cap {
    enum nx_cap_kind      kind;          // SLOT_REF (today); HANDLE, VMO_CHUNK later
    enum nx_cap_ownership ownership;     // BORROW or TRANSFER
    union { struct nx_slot *slot_ref; } u;
    bool                  claimed;
};
```

Today only `SLOT_REF` exists. Phase 5/7 will add `HANDLE` (per-process handle table entry) and `VMO_CHUNK` (page range for zero-copy). The IDL schema will grow `handle` and `vmo_chunk` types when those land.

### Why caps live outside payload

DESIGN.md rule **R3** (two-layer, per Session 79) requires every cross-component slot pointer to travel as a typed cap. Two reasons:

1. **The router scans them.** `nx_ipc_scan_send_caps()` rejects forged caps — sender must hold a registered connection edge to the slot it's referencing. `nx_ipc_scan_recv_caps()` complains if a `TRANSFER` cap was never claimed. None of this works if the slot pointer is hidden in raw payload bytes.
2. **The registry tracks them.** Retaining a received cap (`nx_slot_ref_retain`) emits a `CONNECTION_ADDED` event and registers the new edge in the graph — so `make verify-registry` can still see the live composition. A smuggled pointer creates an invisible edge.

### Borrow vs. Transfer

- **`BORROW`** (default): the cap is valid only while the handler is running. Receiver may inspect/dispatch through it but must not stash the pointer past handler return. The router silently drops borrowed caps after the handler.
- **`TRANSFER`**: receiver is *obligated* to either claim it (`nx_slot_ref_retain` — promotes it into a registered outgoing edge owned by the receiver) or treat it as an error. Unclaimed transfer caps trip a protocol-error count.

### Forward declarations are auto-detected

The generator walks every op param.  If a `struct_in` / `struct_out` / `struct_inout` param's `ctype` matches `^(struct|union|enum)\s+\S+$` (i.e. names a tagged type rather than a primitive like `uint32_t`), the type is added to a forward-declaration set in op-id-of-first-appearance order.  Forward declarations are emitted above the `struct nx_<iface>_ops` typedef.

The IDL author does **not** enumerate the set.  An optional top-level `forward_decls_doc` string supplies the leading block comment above the auto-detected `struct foo;` lines; if absent, the forward decls are emitted bare.  Locked Session 82.

### How the generator uses `slot_ref`

When a param is declared `slot_ref`:

- **Sender wrapper** takes `struct nx_slot *` arg, allocates a `struct nx_ipc_cap` on the kstack, threads it through `msg->caps[]`. The slot pointer never touches `msg->payload`.
- **Message struct** in `<iface>_msg.h` carries a `cap_id` integer (the cap's index in `caps[]`), not a `struct nx_slot *`.
- **Receiver dispatch template** extracts the cap by id, hands it to the impl as `struct nx_slot *`. For `ownership: transfer`, the template wraps a "did you claim?" check around the impl call.

Today's five interfaces (`vfs`, `fs`, `mm`, `scheduler`, `char_device`) have **zero** `slot_ref` params — v1 cap support is plumbing with no live users. The first real consumer is likely Phase 8.3's config-handle API.

---

## Author-Declared Includes (`includes:`)

The `includes:` top-level array names hand-written headers that provide types referenced by op-param `ctype` fields.  Locked Session 80; extended in slice 8.0pre.2 (Session 83) to also emit in the msg header and to suppress auto-detected forward decls.

```json
"includes": [
  { "path": "interfaces/fs_types.h",
    "doc": "Data shapes (struct nx_fs_dirent, struct nx_fs_stat) shared with vfs.h." }
]
```

### Fields

| Field | Required | Meaning |
|---|---|---|
| `path` | yes | Header path emitted verbatim in the `#include` line. `<...>` form if `system: true`, `"..."` otherwise. |
| `system` | no | `true` for system includes, default `false`. |
| `doc` | no | Trailing comment on the include line. |

### Where the includes are emitted

The generator emits the `includes:` list (after `<stddef.h>` / `<stdint.h>`) in **both** generated headers that may need the types:

- `interfaces/<iface>.h` — the typedef header.  Op signatures may take pointers to the included types.
- `interfaces/<iface>_msg.h` — per-op request/reply structs may **embed** the included types by value (e.g., `struct nx_fs_dirent out;` inside `struct nx_fs_msg_readdir`); the compiler needs the full definition for sizeof.

The wrapper header (`framework/<iface>_call.h`) and dispatch header (`framework/<iface>_dispatch.h`) include their iface headers transitively, so they don't need separate include emission.

### Forward-decl auto-detection is suppressed when `includes:` is non-empty

The generator walks every op param's `ctype` and (by default) emits forward declarations for tagged types (`struct foo`, `union bar`, `enum baz`).  When `includes:` is non-empty, this is skipped — the included header carries the full definition, and a redundant forward decl after a full definition is legal but visually noisy.  An IDL that needs explicit forward decls anyway should put them in its hand-written types header.

### Convention: hand-written `<iface>_types.h` for owned types

When an interface owns its data-shape types (defines them, vs. just referencing types defined by another interface), the canonical pattern is:

1. Hand-write `interfaces/<iface>_types.h` containing the struct definitions and any constants tightly bound to them (array sizes, enum-like discriminants).
2. Declare it in the IDL's `includes:` array so the generated `<iface>.h` and `<iface>_msg.h` `#include` it.
3. Operational constants (op flag bits, op enum-like inputs) stay in the IDL's `constants:` so they're emitted in the operations header alongside the ops table.

The naming convention `<iface>_types.h` is suggested but not enforced; the IDL author chooses the path.

### Why hand-written types over inline-in-IDL

The IDL's job is to describe *interface contracts* — op signatures, op-IDs, returns, capability flow.  Data-layout concerns (struct field types, alignment, sizes, macro-sized arrays, trailing per-field comments) are naturally C: the compiler reasons about them, hand-written code already supports any C-legal shape, and a JSON-encoded representation would be opaque to the generator's schema validation.  An earlier iteration of slice 8.0pre.2 prototyped a `structs:` schema field for inline struct definitions; reverted in favor of this includes-driven path because it kept the IDL focused on operations and avoided the verbatim-string `body` clutter (Session 83 decision).

### Slice 8.0pre.4 cutover scope

When all interfaces are IDL-driven, hand-written *ops* headers (`fs.h`, `vfs.h`, `mm.h`, `scheduler.h`, `char_device.h`) are deleted — the generator's output is canonical.  Hand-written *types* headers (`fs_types.h` and any future `<iface>_types.h`) survive the cutover.  They're declared as IDL dependencies via `includes:` and stay under hand maintenance.

---

## Return Types

```json
"returns": {
  "type": "int_status",       // see table below
  "out_param": "out_file",    // optional; names the param carrying principal output
  "ctype": "struct nx_task"   // optional; only for type: "void_ptr" (slice 8.0pre.3)
}
```

| `type` | C signature | Meaning |
|---|---|---|
| `void` | `void` | No return; e.g. `close`, `free_pages` |
| `int_status` | `int` | `NX_OK` or negative `NX_E*` |
| `i64_count_or_status` | `int64_t` | `>= 0` byte count; negative `NX_E*` on error |
| `usize` | `size_t` | Pure unsigned size (e.g. `mm.page_size`) |
| `u32` | `uint32_t` | Scalar unsigned-int return (slice 8.0pre.3); e.g. `mm.max_order` |
| `u64` | `uint64_t` | 64-bit unsigned return (slice 8.0pre.3); reserved for future use |
| `void_ptr` | `void *` — or `<ctype> *` if optional `ctype` is present | Raw or typed pointer return; no error code (e.g. `mm.alloc_pages` → `void *`; `scheduler.pick_next` → `struct nx_task *`) |

### Type mapping conventions

The IDL's signed-integer types map to existing C conventions in the hand-written headers, which are asymmetric with the unsigned forms:

| IDL | C | Notes |
|---|---|---|
| `u8` … `u64` | `uint8_t` … `uint64_t` | Always stdint forms. |
| `i8`, `i16` | `int8_t`, `int16_t` | Stdint forms (no native equivalent). |
| `i32` | `int` | Native, **not** `int32_t` — matches today's `int whence` etc.  C's `int` is 32-bit on aarch64. |
| `i64` | `int64_t` | Stdint form. |
| `usize` | `size_t` | |

Locked Session 82.

### `out_param`

Some ops mix a principal return (`int_status`) with an output-by-pointer param (`out_file`). Modelling that with a single `returns` block + an `out_param` cross-reference keeps the IDL tidy:

```json
{ "name": "open", "op_id": 1,
  "params": [
    {"name":"path","type":"string_in"},
    {"name":"flags","type":"u32"},
    {"name":"out_file","type":"opaque_self_handle","direction":"out"}
  ],
  "returns": {"type":"int_status","out_param":"out_file"} }
```

If a future op needs multiple output params, the schema will grow an `out_params: [...]` array. (Locked Session 80: combined for now; switch when an op forces it.)

### Why keep `void_ptr` for `mm.alloc_pages`

`mm.alloc_pages → void *` could be reshaped as `int alloc_pages(order, **out)` for cleaner error semantics, but that's a behavior change rippling through every caller. Keep the existing shape; `void_ptr` encodes as a `u64` field in the message. (Locked Session 80: revisit if a slice forces the cleaner shape.)

---

## `self` Is Implicit

Every C op in our existing interfaces takes `void *self` as its first arg. The IDL **does not** declare `self`. It's a structural artifact of the dispatch path:

- **Receiver template** extracts `impl = slot->active->impl_data` and passes it as `self` to the impl function.
- **Sender wrapper** takes `struct nx_slot *` (not `self`); the sender does not have access to the receiver's private state.

Generated `interfaces/<iface>.h` reproduces today's `(void *self, ...)` typedef so existing direct callers compile unchanged through 8.0pre.4 (the cutover). After 8.0c migrates callers to wrappers, the typedef-shaped header survives — components still implement `(void *self, ...)` ops; only the *callers* change.

---

## IRQ-Entry Ops

Some ops are sent from interrupt context (e.g. UART RX byte after slice 3.9b's deferred reshape). Declared with `"context": "irq"`. Implications:

- **Message must be POD** with no caller-stack pointers — the framework dispatcher kthread reads it after the ISR returns.
- **No caps allowed** — `nx_slot_ref_retain` requires handler context; ISR cannot allocate.
- **Generator emits a fixed-size pre-built message pool** sized at codegen time (per-interface knob; default 32 entries).
- **Generator emits `framework/<iface>_isr.h`** with `nx_<iface>_<op>_from_irq(slot, ...)` that grabs a pool slot, fills the message, calls `nx_ipc_enqueue_from_irq`. The ISR is bounded in instructions (matches `framework/ipc.h`'s contract).

Example (planned for char_device when slice 3.9b's deferred work lands):

```json
{ "name": "rx_byte", "op_id": 1,
  "params": [{"name":"byte","type":"u8"}],
  "returns": {"type":"void"}, "context": "irq" }
```

Slice 8.0pre.3 (Session 84) activated this end of the spec.  `interfaces/idl/char_device.json`'s `rx_byte` op declares `context: "irq"` and produces `framework/char_device_isr.h` containing:

- `#define NX_CHAR_DEVICE_ISR_POOL_SIZE 32` — the documented default pool size.  Future schema field will make it per-IDL tunable when a driver demands it.
- `void nx_char_device_rx_byte_from_irq(struct nx_slot *slot, uint8_t byte);` — declaration only; body lands with slice 8.0a's runtime infra (pool storage + dispatcher enqueue).

The ISR header is emitted **only** for IDLs that declare any IRQ-entry op.  IDLs with all-task-context ops (vfs, fs, mm, scheduler today) do not get an `_isr.h` artifact.

---

## Generated Artifacts

For each `interfaces/idl/<iface>.json`:

| Output | Purpose |
|---|---|
| `interfaces/<iface>.h` | `struct nx_<iface>_ops` typedef. Reproduces today's hand-written form so existing direct callers compile unchanged through 8.0pre.4. |
| `interfaces/<iface>_msg.h` | `enum nx_<iface>_op_id` + per-op message structs (`struct nx_<iface>_msg_open`, etc.). |
| `framework/<iface>_call.h` | Sender wrappers `nx_<iface>_<op>(struct nx_slot *slot, ...)`. Body builds the message, calls `nx_slot_call_sync(slot, op_id, msg)`. |
| `framework/<iface>_dispatch.h` | Receiver-side `handle_msg` template. Component supplies `NX_<IFACE>_OP_<OP>_IMPL(self, ...)` macros pointing at static impl functions; the template expands into a `switch (op_id)` that unpacks the message and calls the impls. |
| `framework/<iface>_isr.h` | (Only if any op has `context: "irq"`.) Pool-size define + per-IRQ-op `nx_<iface>_<op>_from_irq(struct nx_slot *slot, ...)` declarations.  Pool storage + bodies land with slice 8.0a (runtime infra). |

### Header guards & include ordering

- All generated headers include `framework/registry.h` (for slot type) and `framework/ipc.h` (for cap types).
- `interfaces/<iface>_msg.h` is the only generated header consumed by component sources directly; the others are consumed by `framework/syscall.c`, `framework/dispatcher.c`, and similar.
- A "GENERATED — DO NOT EDIT" banner heads every artifact; `verify-iface-fresh` checks the banner is present.

### Idempotence & freshness

- Generator output is byte-deterministic given the same IDL input + generator version.
- `make verify-iface-fresh`:
  1. Validate every IDL file against the meta-schema.
  2. Re-run the generator into a temp dir.
  3. `diff -r` against the in-tree headers.
  4. Fail the build if anything differs or if any IDL has been edited without regenerating.

---

## vfs Example

`interfaces/idl/vfs.json` (target shape):

```json
{
  "interface": "vfs",
  "version": 1,
  "iface_id": 4097,
  "doc": "Virtual filesystem interface — see DESIGN.md and the original interfaces/vfs.h for the contract.",
  "ops": [
    { "name": "open", "op_id": 1,
      "doc": "Open `path` (absolute, rooted at `/`).",
      "params": [
        {"name":"path","type":"string_in"},
        {"name":"flags","type":"u32"},
        {"name":"out_file","type":"opaque_self_handle","direction":"out"}
      ],
      "returns": {"type":"int_status","out_param":"out_file"} },

    { "name": "close", "op_id": 2,
      "doc": "Release per-open state. Idempotent against NULL.",
      "params": [{"name":"file","type":"opaque_self_handle"}],
      "returns": {"type":"void"} },

    { "name": "retain", "op_id": 3,
      "doc": "Bump per-open refcount (slice 7.6d.N.8).",
      "params": [{"name":"file","type":"opaque_self_handle"}],
      "returns": {"type":"void"} },

    { "name": "read", "op_id": 4,
      "doc": "Read up to `cap` bytes from `file` into `buf`.",
      "params": [
        {"name":"file","type":"opaque_self_handle"},
        {"name":"buf","type":"bytes_out","cap":"cap","direction":"out"},
        {"name":"cap","type":"usize"}
      ],
      "returns": {"type":"i64_count_or_status"} },

    { "name": "write", "op_id": 5,
      "doc": "Write `len` bytes from `buf` into `file`.",
      "params": [
        {"name":"file","type":"opaque_self_handle"},
        {"name":"buf","type":"bytes_in","len":"len"},
        {"name":"len","type":"usize"}
      ],
      "returns": {"type":"i64_count_or_status"} },

    { "name": "seek", "op_id": 6,
      "doc": "Reposition per-open cursor. Returns new absolute position.",
      "params": [
        {"name":"file","type":"opaque_self_handle"},
        {"name":"offset","type":"i64"},
        {"name":"whence","type":"i32"}
      ],
      "returns": {"type":"i64_count_or_status"} },

    { "name": "readdir", "op_id": 7,
      "doc": "Read next entry under `dir_path` (slice 7.7b.1).",
      "params": [
        {"name":"dir_path","type":"string_in"},
        {"name":"cookie","type":"struct_inout","ctype":"uint32_t","direction":"inout"},
        {"name":"out","type":"struct_out","ctype":"struct nx_fs_dirent","direction":"out"}
      ],
      "returns": {"type":"int_status"} },

    { "name": "mkdir", "op_id": 8,
      "doc": "Create a directory at `path` (slice 7.7b.1).",
      "params": [{"name":"path","type":"string_in"}],
      "returns": {"type":"int_status"} },

    { "name": "stat", "op_id": 9,
      "doc": "Report metadata for `path` (slice 7.7b.1).",
      "params": [
        {"name":"path","type":"string_in"},
        {"name":"out","type":"struct_out","ctype":"struct nx_fs_stat","direction":"out"}
      ],
      "returns": {"type":"int_status"} }
  ]
}
```

### Mapping to today's `interfaces/vfs.h`

| IDL op | Today's C signature |
|---|---|
| `open` | `int (*open)(void *self, const char *path, uint32_t flags, void **out_file)` |
| `close` | `void (*close)(void *self, void *file)` |
| `retain` | `void (*retain)(void *self, void *file)` |
| `read` | `int64_t (*read)(void *self, void *file, void *buf, size_t cap)` |
| `write` | `int64_t (*write)(void *self, void *file, const void *buf, size_t len)` |
| `seek` | `int64_t (*seek)(void *self, void *file, int64_t offset, int whence)` |
| `readdir` | `int (*readdir)(void *self, const char *dir_path, uint32_t *cookie, struct nx_fs_dirent *out)` |
| `mkdir` | `int (*mkdir)(void *self, const char *path)` |
| `stat` | `int (*stat)(void *self, const char *path, struct nx_fs_stat *out)` |

The compatibility test for slice 8.0pre.1: generated `interfaces/vfs.h` matches the existing hand-written file byte-for-byte (modulo the `GENERATED — DO NOT EDIT` banner).  Slice 8.0pre.1 (Session 82) achieved this by canonicalizing the hand-written `vfs.h`'s comment style: hand-written used a mix of compact and expanded multi-line comments (`retain` and `read/write` were compact; `seek` / `readdir` / `mkdir` / `stat` were expanded).  The generator picks one rule (single-line if the prose fits, expanded multi-line otherwise) and emits a uniform layout.  C-level shape (preprocessor macros, struct fields, forward declarations, signatures) is unchanged so callers compile identically.  Going forward the IDL is the source of truth; `make verify-iface-fresh` enforces byte-equivalence between in-tree generated files and a fresh regeneration.

---

## Open Items / Future Extensions

These are explicitly out of scope for slice 8.0pre.1 but the schema is designed to accommodate them without breaking changes.

1. **Per-op pause-policy override.** Field reserved (`pause_policy`); v1 ignores any value other than `null`. Becomes live when a real conflict appears.
2. **`handle` and `vmo_chunk` cap kinds.** Add new entries to the cap-type table when Phase 5/7 extensions land.
3. **`returns.out_params: [...]`.** Switch from singular `out_param` to plural array if an op needs multiple output-by-pointer params. None today.
4. **Generator-version tag.** Add a top-level `gen_version` so old IDL files trigger a regenerate-with-newer-generator path. v1 doesn't need it; pin to current generator version for now.
5. **Cross-IDL checks.** A future `verify-iface-fresh` extension walks all IDL files together to enforce: unique `iface_id` across files; op-ID gravestones (no reuse across history); ctype names that resolve to actual headers. Today's check is per-file only.
6. **Async-mode wire shape.** v1 ships sync-mode only (caller blocks on its own kstack). Async-per-edge (DESIGN's `mode = IPC_ASYNC`) is forward-compatible: same message struct; the wrapper signature gains a "fire-and-forget" variant when the slice lands.
7. **`mm.alloc_pages` reshape to `int alloc_pages(order, **out)`.** Keep `void_ptr` for now; reshape if a slice forces the cleaner shape.

---

## References

- [Session 80 log](logs/session-80-idl-schema.md) — the design discussion that produced this spec.
- [Session 79 log](logs/session-79-phase-8-plan.md) — the Phase 8 plan that motivated the IDL.
- [IMPLEMENTATION-GUIDE.md §Phase 8](IMPLEMENTATION-GUIDE.md#phase-8-runtime-recomposition-and-config-manager) — slice plan; 8.0pre.1 is the implementation entry point.
- [DESIGN.md](DESIGN.md) — R1–R8 rules; R3 (two-layer slot-ref discipline) is the rule the IDL enforces declaratively.
- [framework/ipc.h](../../sources/nonux/framework/ipc.h) — cap struct + router contract.
- [interfaces/vfs.h](../../sources/nonux/interfaces/vfs.h) — current hand-written interface; the byte-for-byte target for the generator's first output.
