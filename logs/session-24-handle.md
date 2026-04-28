# Session 24: slice 5.3 — handle framework

**Date:** 2026-04-22
**Phase:** 5 slice 5.3
**Branch:** master

---

## Goals

Ship the handle framework — `framework/handle.{h,c}` with a per-owner
table, typed handles, rights attenuation on duplicate, and a
generation counter on each slot so stale handle values can't accidentally
resolve after reuse. This is infrastructure for slice 5.4's syscall
entry (which will look syscall arguments up in the caller's handle
table before dispatching) and 5.6's channel handles.

## What Was Done

### `framework/handle.h` — new

Public API:

- `enum nx_handle_type` scaffolding: `HANDLE_INVALID`, `HANDLE_CHANNEL`,
  `HANDLE_VMO`, `HANDLE_PROCESS`, `HANDLE_THREAD`, `HANDLE_IRQ`,
  `HANDLE_FILE`. Only the enum members exist in 5.3 — real objects
  wire up in slices 5.5 (PROCESS/THREAD), 5.6 (CHANNEL), Phase 5+
  (VMO/IRQ/FILE).
- `uint32_t` rights bitmask: `NX_RIGHT_READ`, `_WRITE`, `_TRANSFER`,
  `_MAP`, `_WAIT`, `_SIGNAL`, `_ACK`, `_SEEK`, `_INFO`. Plus
  `NX_RIGHTS_ALL = 0xFFFFFFFFu` for in-kernel allocators that hand
  a fresh object's owner every right.
- `typedef uint32_t nx_handle_t` with `NX_HANDLE_INVALID = 0` reserved.
  Handle encoding: high 24 bits = generation, low 8 bits = index + 1.
- `struct nx_handle_table` — fixed 64-entry array (`NX_HANDLE_TABLE_CAPACITY`)
  + a count. No free-list — linear scan is one cache line and trivially
  correct at this capacity.
- Four ops: `nx_handle_alloc` / `_lookup` / `_close` / `_duplicate`
  (rights-attenuating).

### `framework/handle.c` — new

Implementation with three noteworthy details:

1. **Generation bumped on close, not alloc.** A fresh slot starts with
   generation 0; when closed, the slot's generation increments. Alloc
   inherits the current value so a stale pre-close handle value
   encoding the old generation fails subsequent lookups even if the
   slot is reused. Bumping on alloc instead would leak one generation
   value per close/alloc pair with no safety benefit.
2. **Close zeroes the slot THEN bumps generation.** A racing lookup
   either sees the cleared entry (returns NX_ENOENT on `type`) or the
   old generation (returns NX_ENOENT on the generation check) — never
   a half-closed slot. Not strictly needed under v1's single-CPU
   model but it pays forward for SMP.
3. **`nx_handle_duplicate` is attenuation-only.** `new_rights` must be
   a strict subset of the source's rights — any bit set in `new_rights`
   but not in `src_rights` returns `NX_EPERM`. This is the only
   userspace path to narrow a handle; no path broadens rights short of
   a fresh alloc by kernel-side code that owns the object.

### `framework/registry.h` — new error code

`NX_EPERM = -10` for rights-escalation attempts.

### Host tests (+14 in `test/host/handle_test.c`)

- Three alloc/lookup/close round-trip cases.
- Generation counter: `handle_value_after_reuse_differs_from_pre_close_value`
  proves stale values fail lookup after reuse.
- Four duplicate cases: same-rights, reduced-rights, expansion-is-EPERM,
  duplicate-of-closed-is-ENOENT.
- Capacity / edge cases: fill-to-cap then one more is ENOMEM (reopens
  after close); NX_HANDLE_INVALID rejected cleanly; NULL table on every
  entry point returns EINVAL; invalid type rejected; NULL object
  rejected.
- Two stress tests: 1000× alloc/close round-trip is leak-free;
  interleaved alloc/close sustains capacity.

### Kernel tests (+3 in `test/kernel/ktest_handle.c`)

- `handle_alloc_lookup_close_roundtrip_on_kernel` — basic smoke.
- `handle_duplicate_rights_attenuation_on_kernel` — duplicate with
  reduced rights succeeds, with escalation returns EPERM.
- `handle_stale_value_after_close_does_not_resolve_to_reused_slot` —
  generation-counter check in-kernel.

### `test/host/test_runner.h` — diagnostic buffer bump

Bumped `ASSERT_EQ_U` / `ASSERT_EQ_PTR` diagnostic buffer from 128 to
256 bytes. Handle-API expressions are verbose (`nx_handle_alloc(&t,
NX_HANDLE_CHANNEL, NX_RIGHT_READ, &dummy, &h)` is ~55 characters
stringified) and triggered `-Wformat-truncation` warnings under
`-Werror`. 256 is comfortable headroom for any future op surface.

### Build wiring

- Top-level `Makefile`: `FW_C += framework/handle.c`;
  `KTEST_C += test/kernel/ktest_handle.c`.
- `test/host/Makefile`: SRCS += `handle_test.c`,
  `../../framework/handle.c`.

Net: `make test` → **279/279** (51 python + 191 host + 37 kernel),
up from 262/262 — **+14 host + 3 kernel = +17 tests**.

## Key Decisions

- **Per-owner table, not per-task yet.** Slice 5.3 doesn't touch
  `struct nx_task`. The table is a standalone data structure that
  slice 5.5 will embed (or reference-by-pointer) from `struct nx_task`
  once EL0 processes exist. Keeps this slice's blast radius small.
- **Fixed 64-entry capacity.** Small, works for all near-term
  userspace patterns (busybox uses single-digit fd counts). Expanding
  to dynamic capacity is a slice 5.5-or-later refactor behind the
  same API; the 8-bit index field in the handle encoding could stretch
  to 255 entries without a handle-value format change.
- **Handle encoding: `(generation << 8) | (index + 1)`.** 24 bits of
  generation is plenty — at one close per microsecond, wraparound
  takes 16 seconds per slot. If wraparound ever matters we'll detect
  with a counter on the entry, not encode it in the handle value.
- **`init()` does NOT reset the generation counter to zero.** A
  table-init on a previously-used table still invalidates any lingering
  handle values from the prior incarnation. Zero-initialised fresh
  tables (from `calloc` or static storage) start at gen 0 naturally,
  which is the common case.
- **No ref-counting yet.** A handle is either alive or closed; there's
  no `nx_handle_retain`. Slice 5.6 may add `dup`-with-shared-object
  semantics where the channel endpoint itself refcounts, but that's
  an object-side concern, not a handle-table concern.

## Known Issues / Follow-ups

- Handle lookup is O(1) (direct index) but alloc is O(N) (linear scan
  for free slot). At N=64 it's one cache line; scaling-related when
  per-process tables grow beyond one page.
- The generation counter is per-entry `uint32_t` — no overflow
  detection. If a single slot cycles 2^24 times in a test session it
  wraps silently. Detection can be a debug-build assertion later.
- No test for "handle value from table A used against table B" — the
  handle encoding doesn't carry a table identity. Tables are siloed
  by their owner (task/process), so passing a handle across tables is
  a kernel-internal bug, not a userspace path.

## Next Actions

**Slice 5.4 — syscall entry.** `core/cpu/vectors.S` gets an SVC-from-EL1
handler path. `framework/syscall.c` implements a dispatch table keyed
by syscall number. First syscall: `nx_debug_write(buf, len)` — a smoke
test that writes bytes to UART and proves EL1→EL1 SVC round-trips
work before slice 5.5 introduces EL0. Second syscall:
`nx_handle_close(h)` — the minimal real user-facing op, exercising
the handle framework from a syscall context.

## Files Touched

- `framework/handle.h` — **new**
- `framework/handle.c` — **new**
- `framework/registry.h` — `NX_EPERM = -10`
- `test/host/handle_test.c` — **new**, 14 cases
- `test/kernel/ktest_handle.c` — **new**, 3 cases
- `test/host/test_runner.h` — diagnostic buffer 128 → 256 bytes
- `test/host/Makefile` — SRCS += handle_test.c + framework/handle.c
- `Makefile` — FW_C += framework/handle.c; KTEST_C += ktest_handle.c
