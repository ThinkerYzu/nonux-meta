# Session 84 — slice 8.0pre.3: `sched`, `mm`, `char_device` IDLs + IRQ-entry codegen

**Date:** 2026-04-30
**Slice:** 8.0pre.3 — three remaining IDLs (`scheduler`, `mm`, `char_device`) + generator extension to emit `framework/<iface>_isr.h` for ops with `context: "irq"`.  Also schema extension: optional `ctype` on `opaque_self_handle` params and on `void_ptr` returns; new `u32` / `u64` return types.
**Outcome:** All five production interfaces are now IDL-driven.  `scheduler.h` and `mm.h` canonicalized to byte-equal generator output; `char_device.h` is a new file authored fresh from IDL (no hand-written predecessor).  IRQ-entry op support landed end-to-end: `char_device.json`'s `rx_byte` op exercises `context: "irq"` and produces `framework/char_device_isr.h` with the `_from_irq` wrapper declaration + `NX_CHAR_DEVICE_ISR_POOL_SIZE` define.  Tests: `make test` 478 → **495/495 pass** (+17 new tests across 6 new classes — three byte-equivalence checks, one IRQ-artifact suite, two for the typed-pointer schema extensions, one for u32 return); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.

---

## Goal

Slice 8.0pre.2 (Session 83) landed the second IDL (`fs`) and refined the includes-driven types-header pattern.  The slice plan called out 8.0pre.3 as the slice that:

1. Brings the remaining three interfaces (`scheduler`, `mm`, `char_device`) under the IDL roof.
2. Exercises **no-message-context ops** (mm.alloc_pages → `void *`).
3. Exercises **IRQ-entry ops** (`context: "irq"`) — the schema already supports them but slice 8.0pre.1 left the generator's `framework/<iface>_isr.h` template unimplemented.

The de-risk question for `scheduler` was how the IDL describes `struct nx_task *` — a typed kernel-object pointer that crosses the IPC boundary as a borrowed handle.  Today's hand-written `interfaces/scheduler.h` uses `struct nx_task *task` extensively (5 of 6 ops); none of the existing IDL types fit cleanly.  Resolved by extending two existing types with an optional `ctype` field, rather than introducing a new type.

---

## What landed

### Generator extensions (`tools/gen-iface.py`, ~80 lines net)

Three orthogonal changes:

1. **Optional `ctype` on `opaque_self_handle` params.**  Without ctype: emits `void *<name>` (current behavior).  With ctype: emits `<ctype> *<name>` (e.g. `struct nx_task *task`).  `direction: "out"` produces double-pointer (`<ctype> **<name>`).  Wire shape unchanged — still encoded as a `u64` in `msg->payload`; only the C-level signature in the typedef gains type information.

2. **Optional `ctype` on `void_ptr` returns.**  Symmetric to the param-side change.  Without ctype: emits `void *` (current).  With ctype: emits `<ctype> *` (e.g. `struct nx_task *`).  Wire shape unchanged; reply struct's `rc` field is still `uint64_t`.  Used by `scheduler.pick_next` to return `struct nx_task *`.

3. **New return types: `u32` and `u64`.**  Asymmetric mapping per the convention codified in 8.0pre.1: `u32 → uint32_t` (stdint), `u64 → uint64_t`.  Used by `mm.max_order` (returns `unsigned`, canonicalized to `uint32_t`).  Reply struct's `rc` field gets the matching stdint type.

4. **`framework/<iface>_isr.h` emission.**  `write_artifacts` now adds a fifth output for IDLs where any op has `context: "irq"`.  Header contents (slice 8.0pre.3 emits declarations only; bodies land with slice 8.0a's runtime infra):

   - Banner + guard `NONUX_FRAMEWORK_<IFACE>_ISR_H`
   - Includes: `<stddef.h>`, `<stdint.h>`, the typedef + msg headers, `framework/registry.h`, `framework/ipc.h` (for `nx_ipc_enqueue_from_irq` which already exists in slice 3.9b)
   - Pool-size define `NX_<IFACE>_ISR_POOL_SIZE 32` (per-interface knob; v1 hardcoded default per IDL-SCHEMA §IRQ-Entry Ops; future schema field will make it tunable)
   - One `nx_<iface>_<op>_from_irq(struct nx_slot *slot, ...)` declaration per IRQ-entry op

5. **Forward-decl auto-detection extended.**  The walker now also collects ctypes from `opaque_self_handle.ctype` (param-side) and `void_ptr.ctype` (return-side), in op-id-of-first-appearance order with op-params before op-returns.  Suppression-when-`includes:`-non-empty rule unchanged from 8.0pre.2.

6. **`return_c_type` helper introduced.**  Replaces direct `RETURN_C_TYPES[op["returns"]["type"]]` lookups in `render_op_member`, the second-member rendering loop, and `wrapper_return_type`.  Centralizes the `void_ptr + ctype` substitution.

### Meta-schema extensions (`tools/idl-meta-schema.json`)

- `return_spec.type` enum gains `u32`, `u64`.
- `return_spec` gains optional `ctype` field (string).
- `param_spec.ctype` description updated to mention the new `opaque_self_handle` use.

No new top-level fields; no schema-shape changes that would break existing IDLs.  vfs.json and fs.json validate identically before/after.

### IDL files

#### `interfaces/idl/mm.json` (new, ~50 lines)

- `iface_id`: 4099 (next sequential after fs=4098).
- 4 ops: `alloc_pages` (returns `void_ptr`, takes `u32 order`), `free_pages` (takes `opaque_self_handle ptr` + `u32 order`, returns `void`), `page_size` (returns `usize`), `max_order` (returns `u32`).
- Top-level doc: 4 prose blocks (Allocation granularity / Ownership / Concurrency / Error convention) preserved verbatim from the hand-written header.
- No `includes:` (mm interface has no shared types).
- No `constants:` (mm interface defines no public constants).

`interfaces/mm.h` regenerates byte-equal to canonical generator output.  Diff against the hand-written predecessor:

| Difference | Resolution |
|---|---|
| `GENERATED — DO NOT EDIT` banner added. | Expected. |
| `#include <stdint.h>` added (always emitted). | Expected. |
| `unsigned order` → `uint32_t order` (3 sites). | Canonicalization per the asymmetric `u32 → uint32_t` convention (slice 8.0pre.1).  On aarch64-linux-gnu `uint32_t` is `typedef unsigned int`, so function-pointer assignment from the existing `unsigned`-using `mm_buddy_alloc_pages` impl remains type-compatible — host tests + kernel tests pass without touching `components/mm_buddy/mm_buddy.c`. |
| `unsigned (*max_order)` → `uint32_t (*max_order)`. | Same canonicalization. |
| `void  (*free_pages)` (two spaces) → `void (*free_pages)` (one space). | Generator's uniform `<ret> (*<name>)(...)` shape. |

C-level shape unchanged at the linker level; existing callers (mm_buddy impl, conformance_mm tests, kheap, pmm) compile identically.

#### `interfaces/idl/scheduler.json` (new, ~50 lines)

- `iface_id`: 4100.
- 6 ops: `pick_next` (returns typed `void_ptr` with `ctype: "struct nx_task"`), `enqueue` / `dequeue` / `set_priority` (each takes `opaque_self_handle task` with `ctype: "struct nx_task"`), `yield` / `tick` (no params, void return).
- `set_priority`'s `priority` param uses `i32` → `int` per the existing asymmetric convention.
- `includes: [{path: "core/sched/task.h"}]` brings in the full `struct nx_task` definition.  Forward-decl auto-detection is suppressed; the include carries the definition.
- Top-level doc: 4 prose blocks (Ownership / Concurrency / Error convention + interface intro).

`interfaces/scheduler.h` regenerates byte-equal.  Diff:

| Difference | Resolution |
|---|---|
| Banner added. | Expected. |
| `#include <stddef.h>`, `#include <stdint.h>` added (always). | Expected. |
| `#include "core/sched/task.h"` gained a trailing comment from the IDL's `doc` field. | Expected — the `includes:` array's `doc` field becomes the trailing comment on the include line (slice 8.0pre.2 behavior). |

C-level shape, struct-field types, all op signatures unchanged.

#### `interfaces/idl/char_device.json` (new, ~50 lines)

Authored fresh — there was no hand-written `interfaces/char_device.h`.  The existing `components/uart_pl011/` component receives a `UART_MSG_WRITE` raw-payload message via `handle_msg` (slice 3.9a smoke-test path); it does not yet implement a `struct nx_char_device_ops` table.  Slice 8.0b will activate the generated `handle_msg` shim that calls into a properly-shaped `char_device.write` op.  This slice introduces the contract.

- `iface_id`: 4101.
- 2 ops:
  - `write(buf, len) → i64_count_or_status` — task-context output op; shape parity with `nx_fs_ops.write` for forward compatibility with byte-count partial-write semantics (uart_pl011 today is fully synchronous so the count is always `len` on success).
  - `rx_byte(byte) → void` with `context: "irq"` — IRQ-entry input op delivered from the UART RX-irq handler.
- Top-level doc: 4 prose blocks (intro / Ownership / Concurrency / Error convention).
- No `includes:`, no `constants:`.

`interfaces/char_device.h` is new.  `framework/char_device_isr.h` is also new — first emission of the IRQ-entry artifact:

```c
#define NX_CHAR_DEVICE_ISR_POOL_SIZE 32

void nx_char_device_rx_byte_from_irq(struct nx_slot *slot, uint8_t byte);
```

Pool sizing follows IDL-SCHEMA's documented default (32 entries).  The `_from_irq` wrapper body lands in slice 8.0a alongside `nx_slot_call_blocking`; today only the declaration ships.

### Tests (`tools/tests/test_gen_iface.py`)

17 new test methods across 6 new classes:

| Class | Count | Coverage |
|---|---|---|
| `TestSchedulerByteForByte` | 2 | in-tree `interfaces/scheduler.h` byte-equal to fresh regeneration; IDL declares `struct nx_task` via typed `opaque_self_handle.ctype` (params) and typed `void_ptr.ctype` (return). |
| `TestMmByteForByte` | 1 | in-tree `interfaces/mm.h` byte-equal to fresh regeneration. |
| `TestCharDeviceByteForByte` | 2 | in-tree `interfaces/char_device.h` byte-equal; in-tree `framework/char_device_isr.h` byte-equal (the new ISR-header artifact). |
| `TestIrqEntryArtifact` | 3 | ISR header **not** emitted when no op has `context: "irq"`; ISR header **is** emitted when at least one op declares irq context (only the IRQ ops get `_from_irq` wrappers; non-IRQ ops do not); pool-size default is 32. |
| `TestTypedOpaqueSelfHandle` | 4 | no-ctype emits `void *`; with-ctype emits `<ctype> *`; with-ctype + direction:out emits `<ctype> **`; ctype that names a struct/union/enum participates in the auto-detected forward-decl set when no `includes:` is declared. |
| `TestTypedVoidPtrReturn` | 3 | no-ctype emits `void *`; with-ctype emits `<ctype> *`; struct-named return ctype participates in forward-decl set. |
| `TestU32Return` | 2 | u32 return emits `uint32_t (*op)(void *self)`; reply struct's `rc` field is `uint32_t`. |

`make test-tools` 76 → **93**.

---

## Decisions Made

- **Optional `ctype` over a new IDL type for typed kernel-object pointers.**  The candidates for representing `struct nx_task *task` in the IDL were:
  - (a) Add a new param type `kobject_ref` with required `ctype`.
  - (b) Extend the existing `opaque_self_handle` with an optional `ctype`.
  - (c) Use `struct_in` / `struct_inout` and rely on the wire-format being u64-equivalent.
  
  Took (b).  Semantically `opaque_self_handle` is already "typed kernel-object pointer carried as u64 in payload, opaque to the sender's introspection" — adding a ctype to put a name on the C-level signature fits without inventing new wire semantics.  Symmetric extension to `void_ptr` returns falls out naturally.  Rejecting (a) because it would have created two near-duplicate types.  Rejecting (c) because `struct_*` semantics are "by-value POD copy in payload" — fundamentally different wire shape from "u64 raw pointer encoding."

- **`u32` return type added; mm.max_order canonicalized to `uint32_t`.**  Today's `unsigned (*max_order)(void *self)` had no IDL-side mapping.  Two options: (a) add `unsigned` as a new type (would conflict with the existing `u32 → uint32_t` convention by introducing a third C-level shape), or (b) add `u32` return type matching the param-side mapping.  Took (b); accepts that the regenerated typedef has `uint32_t` instead of `unsigned`.  On aarch64-linux-gnu these are the same type (`uint32_t` is typedef'd as `unsigned int`), so the existing `mm_buddy_alloc_pages` impl assignment to the function-pointer slot remains valid without touching `components/mm_buddy/mm_buddy.c`.  Verified: `make test` passes 495/495 with no component-side changes.

- **Forward-decl auto-detection walks both param ctypes and return ctypes.**  The previous behavior (ctype on `struct_*` params only) didn't anticipate ctypes on `opaque_self_handle` or `void_ptr`.  Extension is mechanical — the walker now visits each op's params (in declared order) and the op's return ctype (if present).  Suppression-when-`includes:`-non-empty rule unchanged.  scheduler.json declares `includes:` so no forward decls are emitted there; the typed-pointer tests in `TestTypedOpaqueSelfHandle` and `TestTypedVoidPtrReturn` exercise the no-`includes:` path.

- **char_device.write returns `i64_count_or_status`, not `int_status`.**  Today's uart_pl011 returns `int 0` from handle_msg regardless of bytes processed — fundamentally a fire-and-forget shape that `int_status` would model accurately.  But the interface is being authored for a hot-swap-capable future; a virtio-console or a hardware-flow-controlled UART would have partial-write semantics, and shape parity with `nx_fs_ops.write` (which uses `i64_count_or_status`) lets the syscall layer treat all byte-stream sinks uniformly.  Took the more general shape; uart_pl011's slice-8.0b shim will return `len` on success.

- **`rx_byte` chosen as the IRQ-entry op example.**  IDL-SCHEMA.md's worked example since Session 80 has been `rx_byte(byte: u8) → void` with `context: "irq"`.  Used the same shape unchanged so the canonical IDL example is now the canonical implementation.  Pool size 32 is the IDL-SCHEMA documented default; not exposed as an IDL field yet (a future schema field can make it tunable when a real driver demands it).

- **ISR-header pool size is a generated `#define`, not a configurable IDL field.**  Two paths: (a) hardcode in the generator (simpler), (b) IDL `irq_pool_size` field defaulting to 32 (more flexible).  Took (a) for v1 — no driver in-tree needs a different size, and adding a schema field for a single in-tree user violates "the schema is locked-down strictly" (Session 80 decision).  When a future driver demands tuning, the field is added then.

- **8.0pre.4 cutover scope (carried from Session 83): hand-written *types* headers survive.**  This slice introduces no new types headers (mm/scheduler/char_device don't own shared types — scheduler uses `core/sched/task.h` which is a pre-existing kernel-internal header, not a new IDL artifact).  `interfaces/fs_types.h` (introduced in 8.0pre.2) remains the only hand-written types header in the IDL ecosystem.  When 8.0pre.4 deletes the hand-written ops headers, only `fs_types.h` (and any future `<iface>_types.h`) survives.

---

## Key Findings

- **The `opaque_self_handle` type is the natural home for any "typed kernel-object pointer that crosses the IPC boundary as a u64."**  The original semantic (slice 8.0pre.1) was specifically the per-iface opaque file/state handle — but the wire shape (u64 in payload, no copy semantics) is identical to the borrowed kernel pointer case.  Adding optional `ctype` made the type general without changing its wire role.

- **Schema extensions land cleanly when they're optional + backward-compatible.**  Existing IDLs (`vfs.json`, `fs.json`) validate identically before/after the meta-schema diff; nothing depends on the new fields being absent.  The strict-validation rule (locked Session 80) doesn't fight a permissive extension because the new fields are added to the *enum* / *property list*, not removed from anywhere.

- **The IRQ-entry artifact is generator-side only; it has no production consumers yet.**  `framework/char_device_isr.h` carries a function declaration (`nx_char_device_rx_byte_from_irq`) whose body lives in an unwritten future runtime.  This is identical to how `framework/<iface>_call.h` shipped in 8.0pre.1 (declarations only; bodies in slice 8.0a).  The pattern keeps the generator landings focused on contract definition, defers runtime obligations to slice 8.0a.

- **All five production interfaces now sit on the IDL.**  Group A is one slice (8.0pre.4) away from completion: deletion of any remaining hand-written headers + verification that the build proves no consumer reaches around the generator.  At end of slice 8.0pre.3, every `interfaces/*.h` file (vfs.h, fs.h, mm.h, scheduler.h, char_device.h) is either a generator output or a hand-written types header (`fs_types.h`); every IDL source (`interfaces/idl/*.json`) is meta-schema-validated.

---

## Status at End of Session

### Working
- `tools/gen-iface.py` emits five artifacts per interface when an op has `context: "irq"`; four otherwise.
- `tools/idl-meta-schema.json` accepts `u32` / `u64` return types + optional `ctype` on `void_ptr` returns + (existing field) optional `ctype` on `opaque_self_handle` params.
- `interfaces/idl/scheduler.json`, `interfaces/idl/mm.json`, `interfaces/idl/char_device.json` are all checked-in and meta-schema-validated.
- `interfaces/scheduler.h`, `interfaces/mm.h` are canonical generator output (regenerated; previous hand-written content discarded).
- `interfaces/char_device.h` is new (no hand-written predecessor).
- `interfaces/{scheduler,mm,char_device}_msg.h` generated.
- `framework/{scheduler,mm,char_device}_call.h`, `framework/{scheduler,mm,char_device}_dispatch.h` generated.
- `framework/char_device_isr.h` generated — first IRQ-entry artifact.
- `make test`: **495/495 pass** (93 python + 283 host + 119 kernel; +17 from this slice's tests).
- `make test-interactive`: **7/7 pass**.
- `make verify-iface-fresh`: 0 drift.
- `make verify-registry`: 0 findings.

### Known limitations (carried forward)
- `framework/<iface>_call.h` bodies are still forward declarations only; bodies land with slice 8.0a (runtime infrastructure for `nx_slot_call_blocking`).
- `framework/<iface>_dispatch.h` macro body is still a skeleton; full unpack-and-call logic lands with slice 8.0b (component shim activation).
- `framework/<iface>_isr.h` similarly carries declarations only; pool storage + `_from_irq` body land with slice 8.0a.
- `framework/scheduler_call.h`, `mm_call.h`, `char_device_call.h`, all corresponding `_dispatch.h` and the new `_isr.h` are NOT yet included by production code (verified by grep) so they don't affect the kernel build path.
- `forward_decls_doc` schema field is now unused in production IDLs (suppression rule active for vfs / fs / scheduler — all three declare `includes:`).  Cleanup deferred.
- IRQ pool size is hardcoded to 32; per-IDL knob deferred until a driver demands tuning.

---

## Next Steps

- **Slice 8.0pre.4** — cutover.  At end of 8.0pre.3, every `interfaces/*.h` is either a generator output (vfs / fs / mm / scheduler / char_device) or a hand-written types header (`fs_types.h`).  The cutover slice formally ends the dual-maintenance window: declare the generator output canonical, lock `make verify-iface-fresh` as a `make all` prerequisite (already wired in slice 8.0pre.1; just needs a documented enforcement call-out in DESIGN.md), and remove any latent references that would surprise a fresh checkout.  Likely a small slice — most of the structure is already in place.

- **Slice 8.0a** — runtime infra.  Independent of 8.0pre.4 (and of 8.0pre.3 in retrospect — could have run in parallel).  Build `framework/slot_call.{h,c}` per [SLOT-CALL-API.md](../SLOT-CALL-API.md).  This unblocks tightening the `_call.h` wrapper bodies, the `_dispatch.h` macro bodies, and the `_isr.h` `_from_irq` bodies — all of which currently end at the declaration.  Cross-cutting test infrastructure (mock component, hook-chain inspector, recompose event logger, pause-injector fixture, cap-forgery harness, equivalence-runner macro) lands here.

- **Possibility for parallel work:** 8.0pre.4 (cutover paperwork) and 8.0a (runtime) are independent.  If a session has time pressure, 8.0a is the higher-leverage choice — it unblocks every Group B slice — but 8.0pre.4 is shorter and cleaner-feeling.

---

**Files Changed:**
- `sources/nonux/tools/gen-iface.py` — new `return_c_type` helper; `param_c_signature` honors `opaque_self_handle.ctype`; `collect_forward_decls` walks param ctypes + return ctype; new `has_irq_ops` + `render_isr_header` + `DEFAULT_IRQ_POOL_SIZE`; `write_artifacts` emits a 5th artifact for IRQ-bearing IDLs; `RETURN_C_TYPES` gains `u32` / `u64`; reply-struct rendering handles the new return types.
- `sources/nonux/tools/idl-meta-schema.json` — `return_spec.type` enum gains `u32` / `u64`; `return_spec` gains optional `ctype`; `param_spec.ctype` description updated.
- `sources/nonux/interfaces/idl/scheduler.json` — NEW. 6 ops; uses typed `opaque_self_handle.ctype` (3 sites) + typed `void_ptr.ctype` (1 site).  `includes: [{path: "core/sched/task.h"}]` brings in `struct nx_task` full def.
- `sources/nonux/interfaces/idl/mm.json` — NEW. 4 ops; uses `u32` return for max_order; `void_ptr` return for alloc_pages.
- `sources/nonux/interfaces/idl/char_device.json` — NEW. 2 ops including the first `context: "irq"` op (rx_byte).  No hand-written predecessor.
- `sources/nonux/interfaces/scheduler.h` — regenerated (canonicalized to byte-equal generator output).
- `sources/nonux/interfaces/mm.h` — regenerated (`unsigned` → `uint32_t` canonicalization in 3 sites; banner + stdint added; `void  (*free_pages)` collapsed to single space).
- `sources/nonux/interfaces/char_device.h` — NEW (generated).
- `sources/nonux/interfaces/scheduler_msg.h`, `mm_msg.h`, `char_device_msg.h` — NEW (generated).
- `sources/nonux/framework/scheduler_call.h`, `scheduler_dispatch.h` — NEW (generated).
- `sources/nonux/framework/mm_call.h`, `mm_dispatch.h` — NEW (generated).
- `sources/nonux/framework/char_device_call.h`, `char_device_dispatch.h` — NEW (generated).
- `sources/nonux/framework/char_device_isr.h` — NEW (generated; first IRQ-entry artifact).
- `sources/nonux/tools/tests/test_gen_iface.py` — +17 tests across 6 new classes (`TestSchedulerByteForByte`, `TestMmByteForByte`, `TestCharDeviceByteForByte`, `TestIrqEntryArtifact`, `TestTypedOpaqueSelfHandle`, `TestTypedVoidPtrReturn`, `TestU32Return`).
