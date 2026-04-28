# Session 15: boot-time composition bring-up (slice 3.9a)

**Date:** 2026-04-21
**Phase:** 3 — Component framework
**Branch:** master

---

## Goals

- Wire the registry + lifecycle + IPC framework (landed host-side
  in slices 3.1–3.8) into the real kernel boot path so a
  `kernel.json` composition actually runs under QEMU.
- Walk the `nx_components` linker section, register each
  descriptor's component, bind it to its slot, resolve deps,
  and run `nx_component_init` → `nx_component_enable` in topo
  order.
- Dump the initial composition via `nx_graph_snapshot_to_json`
  to serial.
- Land the first real nonux component (`uart_pl011`) as the
  proof that the boot walker actually drives something.

## What Was Done

### Linker section — `core/boot/linker.ld`

Added `__start_nx_components` / `__stop_nx_components` inside
`.rodata`, mirroring the existing `kernel_test_registry`
convention:

```
. = ALIGN(8);
__start_nx_components = .;
KEEP(*(nx_components))
__stop_nx_components = .;
```

`KEEP(*(...))` prevents `--gc-sections` from dropping descriptors
that nothing references by symbol (which is every descriptor until
the bootstrap walks them).

### Kernel heap shim — `core/lib/kheap.{h,c}`

Kernel-side `malloc` / `calloc` / `free` — needed because the
framework (registry + IPC) uses them unconditionally.
Implementation:

- Small allocations (≤ 256 B) bump out of a shared slab page
  carved from the PMM. When the page fills, grab another.
- Large allocations take contiguous runs via `pmm_alloc_pages`;
  size is tracked in a side list keyed by the returned pointer
  (using a header inline with the allocation would alias with
  slab offsets within the same page, so the side list is the
  cleanest distinguisher).
- `free(p)` releases large allocs back to the PMM; small allocs
  are a no-op (boot-time composition doesn't churn the heap).

Slice 3.9b will replace this with a real bucket allocator once
kthreads and runtime component swap exist. All framework
allocation call sites go through standard `malloc` /
`calloc` / `free`, so the swap is a single source file.

### Framework freestanding-readiness

`framework/registry.c` and `framework/ipc.c` used `<stdio.h>` +
`<time.h>` (for `snprintf` + `clock_gettime`) and `<stdlib.h>`
(for `malloc` / `calloc` / `free`). Wrapped every libc include
in a `#if __STDC_HOSTED__` guard; the freestanding branch pulls in
`core/lib/kheap.h` + `core/lib/lib.h`.

Replaced the two `snprintf` call sites in registry.c with
freestanding-friendly helpers (`fmt_hex4` for `\u00xx` JSON
escapes, `fmt_u64` for decimal). Replaced `clock_gettime` with a
host-only `CLOCK_MONOTONIC` read; kernel build returns 0 for now
(timestamps on events are informational — slice 3.9b will wire
`cntpct_el0`).

Added `strncmp` to `core/lib/string.{c,h}` — the bootstrap test
needed it and it was missing from the kernel's string set.

### Compiler flags — `Makefile`

Two additions needed to make the freestanding build actually run
on QEMU virt with MMU off:

- `-mgeneral-regs-only` — prevent GCC from auto-generating NEON
  code for struct copies; EL1 defaults have FPEN disabled, so
  any SIMD access traps.
- `-mstrict-align` — prohibit unaligned loads / stores. With MMU
  off, QEMU treats memory as Device-typed, which rejects
  unaligned `stp`/`ldp`. Without this flag, GCC emitted `stp`
  pairs at `sp + 44` (valid for normal memory, fatal for Device).

### Framework bootstrap — `framework/bootstrap.{h,c}`

New module, ~180 LOC. Single entry point `nx_framework_bootstrap()`:

1. Register every slot in `nx_boot_slots[]` (from
   `gen/slot_table.c`).
2. Walk `__start_nx_components … __stop_nx_components`, allocate
   one `struct nx_component` + state buffer per descriptor,
   register, and bind to its slot via `descriptor->name ==
   nx_boot_slots[i].impl_name`.
3. Repeated scan-and-emit topo sort: for each un-visited
   descriptor whose required deps are all visited, run
   `nx_resolve_deps` → `nx_component_init` → `nx_component_enable`
   and mark it visited. `NX_ELOOP` when a full scan makes no
   progress (cycle or missing required dep).

Declares weak fallbacks for `nx_boot_slots[]` / `nx_boot_slots_count`
so unit tests that don't generate a slot table see an empty table
and the walker degenerates cleanly.

### Lifecycle verbs — `framework/component.c`

Slice 3.8's table already documented this as 3.9a's job: wire
`nx_component_init` / `_enable` / `_disable` / `_destroy` to
invoke `ops->init` / `ops->enable` / `ops->disable` / `ops->destroy`
respectively. `init` / `enable` / `disable` bail early on non-zero
return so a failing op leaves the state machine untouched.
`destroy` is fire-and-forget (no return value on `ops->destroy`).
Slice 3.8 had already wired `pause` / `resume`; all four
lifecycle callbacks now fire.

### gen-config — `tools/gen-config.py`

`kernel` subcommand now also emits `gen/slot_table.c` with:

- One static `struct nx_slot` per binding — name, iface
  (derived from the part before the first `.`), mutability (HOT),
  concurrency (SHARED).
- A `nx_boot_slots[]` array pairing each slot with its `impl`
  name.
- `const unsigned nx_boot_slots_count` from
  `sizeof nx_boot_slots / sizeof nx_boot_slots[0]`.

`slot_name_to_ident()` helper derives the C variable name from
the slot name (`char_device.serial` → `char_device_serial`).
Output is deterministic — bindings sorted by slot name,
matching the existing sources.mk / config.h invariant.

Two new tests in `test_gen_config.py`:
- `test_slot_table_c_shape` — checks name/iface derivation and
  the `nx_boot_slots[]` layout.
- `test_slot_table_deterministic` — byte-identical output on
  repeat invocations.

### Makefile — source list + generated-target rules

- `CORE_C` gains `core/lib/kheap.c`.
- `FW_C` flips from empty to
  `registry.c component.c hook.c ipc.c bootstrap.c` — the full
  framework lands in every kernel build.
- `ALL_C` picks up `gen/slot_table.c` through a new `GEN_C` var
  (sources.mk is still `-include`d).
- `gen/sources.mk` / `gen/config.h` / `gen/slot_table.c` are now
  real targets cascading off `gen/sources.mk`'s rule so `make`
  auto-runs gen-config the first time any of them is needed.
  `make kernel-config` stays as a convenience alias.
- `KTEST_C` gains `test/kernel/ktest_bootstrap.c`.

### First real component — `components/uart_pl011/`

Thinnest possible component that proves the boot walker actually
runs something:

- `manifest.json` — name, version 0.1.0, iface `char_device`,
  no deps.
- `uart_pl011.c` — three counters (`init_called`,
  `enable_called`, `messages_handled`) in state, `ops->init` +
  `ops->enable` bump counters, `ops->handle_msg` writes
  `msg_type = UART_MSG_WRITE` payload bytes through the existing
  `uart_putc`.
- `NX_COMPONENT_REGISTER_NO_DEPS` macro simplified (see below).
- `README.md` documents behaviour, rationale, and the slice 3.9b
  evolution path.

### `NX_COMPONENT_REGISTER_NO_DEPS` — dropped `DEPS_FIELD`

The macro previously required a `DEPS_FIELD` name so
`offsetof(CONTAINER, FIELD)` could resolve — but the
`deps_offset` is only read when `n_deps > 0`, so no-deps
components had to invent a placeholder field they never used.
New signature: `NX_COMPONENT_REGISTER_NO_DEPS(NAME, CONTAINER,
OPS)`; `deps_offset` is always 0 (and dead) for no-deps
descriptors. One existing caller
(`test/host/component_deps_test.c::trivial_ops`) updated.

### Bootstrap wired into `boot_main`

New final phase in `core/boot/boot.c` (after PMM / GIC / timer):

```c
int fw_rc = nx_framework_bootstrap();
if (fw_rc == NX_OK) {
    static char snap_buf[4096];
    struct nx_graph_snapshot *snap = nx_graph_snapshot_take();
    if (snap) {
        int n = nx_graph_snapshot_to_json(snap, snap_buf, sizeof snap_buf);
        kprintf("[fw]   composition (gen=%lu, %lu slots, %lu components):\n",
                ...);
        if (n > 0) uart_puts(snap_buf);
        nx_graph_snapshot_put(snap);
    }
} else {
    kprintf("[fw]   bootstrap failed (rc=%d)\n", fw_rc);
}
```

On a successful bring-up the kernel now logs:

```
[fw]   composition (gen=5, 1 slots, 1 components):
{"generation":5,...,"slots":[{"name":"char_device.serial","iface":"char_device",
"mutability":"hot","concurrency":"shared","active":{"manifest":"uart_pl011",
"instance":"0"}}],"components":[{"manifest":"uart_pl011","instance":"0",
"state":"active"}],"connections":[]}
```

### Kernel test — `test/kernel/ktest_bootstrap.c`

Five new KTEST cases run after `boot_main`:

- `bootstrap_registers_expected_slot` — `char_device.serial` is
  looked up, iface `char_device`.
- `bootstrap_binds_uart_to_its_slot` — slot's `active` points at
  a component whose `manifest_id == "uart_pl011"` and state is
  `NX_LC_ACTIVE`.
- `bootstrap_invoked_component_init_and_enable` — peeks at the
  bound `impl`'s counters to confirm `ops->init` and
  `ops->enable` actually fired (1 each).
- `bootstrap_component_count_matches_descriptor_section` —
  `nx_graph_component_count() >= 1`.
- `bootstrap_snapshot_json_contains_bound_impl` — snapshot JSON
  contains `"name":"char_device.serial"` and
  `"manifest":"uart_pl011"`.

### kernel.json — trimmed

`kernel.json` previously declared four slots
(`scheduler` / `mem_manager` / `vfs` / `char_device.serial`) but
only one had a real component on disk. Generated `sources.mk`
would reference `components/mm_buddy/mm_buddy.c` etc. and the
kernel build would fail. Trimmed to just `char_device.serial ←
uart_pl011`; the other slots will be re-added as their components
land (sched_rr in Phase 4, mm_buddy in Phase 5, etc.).

### Test counts

- Python: **51** (49 → +2 for slot_table shape + determinism).
- Host:   **128** (unchanged — host tests don't exercise bootstrap).
- Kernel: **11** (6 → +5 bootstrap tests).
- Total:  **190 / 190 pass**, 0 leaks, 0 errors, exit 0.
- `make verify-registry` runs against the `components/uart_pl011/`
  tree and reports 0 findings (R2 + R4 machine-checked).
- `make` under QEMU prints the composition JSON and drops into
  the tick loop; `make test` also boots and runs all 11 ktests.

## Decisions

**Split 3.9 → 3.9a + 3.9b.** The original slice 3.9 plan
bundled the boot walker with a runtime dispatcher upgrade
(MPSC queue + per-CPU kthread). The dispatcher upgrade has a
hard dependency on Phase 4 — no kthreads exist yet. Session 15
kept only what works on the current synchronous kernel; the
async / pause-hook-deadline / rollback story moves to 3.9b.
See proj_docs/nonux/IMPLEMENTATION-GUIDE.md §Slice 3.9a / 3.9b.

**Heap is a dumb slab-on-PMM.** Real allocator work belongs in
Phase 5+; this slice just needs "you can call `malloc` / `free`
and not crash." 256-byte slab limit keeps component node structs
packed; large allocs (snapshot arrays) go whole-page. `free` is
precise for large allocs and a no-op for small ones — boot-time
composition doesn't churn, so the leak is bounded and observable.

**MMU stays off.** Turning the MMU on to relax Device-memory
alignment would be a much bigger slice (page tables, TTBR0/1,
MAIR, SCTLR tweaks, identity mapping). `-mstrict-align` plus
`-mgeneral-regs-only` is the two-flag workaround that keeps us
on a freestanding bare-metal build until a future phase owns the
MMU story.

**`NX_COMPONENT_REGISTER_NO_DEPS` signature simplification.**
Dropping `DEPS_FIELD` removes the need for placeholder fields in
no-deps component state structs. Only one in-tree caller had to
change; the migration is a plain edit.

**Descriptor-to-slot binding by name.** The walker binds
`descriptor->name == nx_boot_slots[i].impl_name`. An alternative
— binding by iface match — would allow multiple slots to share
one implementation without kernel.json knowing about it, but
slice 3.9a keeps things simple: one slot, one impl, explicit
in kernel.json. Multi-binding lands with multi-instance (Phase 6+).

**Weak `nx_boot_slots[]` fallback.** A weak fallback in
`bootstrap.c` lets unit tests that don't generate a slot table
see an empty table (not a linker error), which makes the
bootstrap code testable in isolation and keeps CI paths simple.
Production kernels always have the strong definition from
`gen/slot_table.c`.

**Freestanding timestamps return 0 for now.** Event
`timestamp_ns` on the host build is `CLOCK_MONOTONIC`; the
kernel build returns 0 and the JSON prints `"timestamp_ns":0`.
Timestamps are informational — neither correctness nor any
assertion depends on them. Slice 3.9b will read `cntpct_el0`
once we have the dispatcher thread to own monotonic-time
conventions.

## What's Next

**Slice 3.9b (blocked on Phase 4).** MPSC lock-free inbox
replacing the host FIFO; per-CPU dispatcher kthread that owns
`nx_ipc_dispatch`; `pause_hook` 1 ms wall-clock deadline;
pause-failure rollback (slot state reverts, hold queue replays);
runtime `NX_HOOK_SLOT_SWAPPED` dispatch from the registry's swap
path; `nx_ipc_enqueue_from_irq`.

**Phase 4 — scheduler.** `sched_rr` becomes the second real
component under the framework. Context switch, preemptive
multitasking, and a kthread-spawning API that 3.9b builds on.

**Opportunistic polish (anytime):**
- `-ffreestanding` Device-memory workaround is a compiler flag
  hack; Phase 5+ should set up the MMU with Normal memory
  attributes and drop `-mstrict-align`.
- Linker script should explicitly place `.rodata` ahead of
  `.data` / `.bss` and mark the LOAD segment non-RWX (the
  current `RWX LOAD segment` linker warning is real — we just
  tolerate it until the MMU arrives).
- Real AArch64 Linux Image header so `-kernel` self-describes
  the load offset.

## Files Touched

- `core/boot/linker.ld` — `nx_components` section markers.
- `core/boot/boot.c` — calls `nx_framework_bootstrap()` +
  snapshot dump.
- `core/lib/kheap.{h,c}` — new, slab-on-PMM allocator.
- `core/lib/string.{h,c}` — `strncmp` added.
- `framework/bootstrap.{h,c}` — new.
- `framework/component.{h,c}` — `ops->init/enable/disable/destroy`
  wired; `NX_COMPONENT_REGISTER_NO_DEPS` signature simplified.
- `framework/registry.c`, `framework/ipc.c` — freestanding guards
  (`#if __STDC_HOSTED__`) + replace `snprintf` / `clock_gettime`
  on the freestanding branch.
- `components/uart_pl011/{uart_pl011.c,manifest.json,README.md}` —
  new first component.
- `test/kernel/ktest_bootstrap.c` — new (5 tests).
- `test/host/component_deps_test.c` — `NO_DEPS` macro migration.
- `kernel.json` — trimmed to the one buildable slot.
- `Makefile` — `-mgeneral-regs-only -mstrict-align`; `FW_C` /
  `CORE_C` / `GEN_C` wiring; generated-target rules.
- `tools/gen-config.py` — `render_slot_table_c`,
  `slot_name_to_ident`, `slot_iface`; `cmd_kernel` writes the
  third output.
- `tools/tests/test_gen_config.py` — two new tests.
- `tools/README.md` — documents the new slot_table.c emit and
  the generated-targets rules.
- `docs/README.md` — bootstrap row added; thread-model line
  updated.
- `docs/framework-bootstrap.md` — new reference doc.
- `docs/framework-components.md` — ops call-site table updated
  (all four lifecycle ops now fire).
- `proj_docs/nonux/{HANDOFF,README}.md` — status + session-logs.
- `proj_docs/nonux/logs/session-15-bootstrap.md` — this log.
