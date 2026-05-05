# Implementation Guide: nonux

**Project:** nonux
**Created:** 2026-04-17
**Last Updated:** 2026-05-03 (Phases 1–7 complete.  Phase 8 in progress.  **Group A complete** (8.0pre.1 → 8.0pre.4, Sessions 82–85).  **Group B — slice 8.0d CLOSED Session 96:** 8.0a (Sessions 86–92) + 8.0b (Session 93) + 8.0c (Sessions 94–95) + 8.0d (Session 96) landed.  Slice 8.0d: `framework/fs_call.c` — 9 `nx_fs_*` sync-dispatcher wrappers; `vfs_simple.c` migrated to use them (removed `resolve_fs`/`resolve_for_path`); 24 new host tests.  Key finding: GCC -O2 strict-aliasing heisenbug required `__asm__ volatile(""::: "memory")` barrier after every `hmsg()` call.  Tests: `make test-tools` **93/93 pass**; `make test-host` **421/421 pass**; `make test-interactive` **7/7 pass**; `make test-kernel` **123/123 pass**; `make verify-iface-fresh` clean; `make verify-registry` clean.  Next forward step: slice 8.0e (`verify-registry` rule banning `iface_ops` access outside `framework/dispatcher.c`).)
**Status:** Phase 3 — Component framework (Phases 1–2 done)

---

## Navigation

**Project Docs:** [README](README.md) | [SPEC](SPEC.md) | [DESIGN](DESIGN.md) | [IMPLEMENTATION-GUIDE](IMPLEMENTATION-GUIDE.md) *(you are here)* | [HANDOFF](HANDOFF.md) | [IDL-SCHEMA](IDL-SCHEMA.md) | [SLOT-CALL-API](SLOT-CALL-API.md)

**This Document:**
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [File Structure](#file-structure)
- [Implementation Details](#implementation-details)
- [Implementation Phases](#implementation-phases)
- [Build & Test](#build--test)

---

## Overview

### What We're Building

A composable microkernel for ARM64 that boots on QEMU, runs busybox, and lets you swap components like Lego bricks — driven by JSON manifests and a declarative config.

### Key Principle

**Vertical slices.** Each phase delivers a bootable, testable system. We don't build all the infrastructure first and hope it works — we boot on day one (even if it only prints "hello") and add layers incrementally.

---

## Prerequisites

### Dependencies

| Tool | Version | Purpose |
|---|---|---|
| `aarch64-linux-gnu-gcc` | 10+ | Cross-compiler for ARM64 kernel and userspace |
| `aarch64-linux-gnu-ld` | (from binutils) | Linker |
| `aarch64-linux-gnu-objcopy` | (from binutils) | Binary image extraction |
| `qemu-system-aarch64` | 6.0+ | Emulation target (virt machine) |
| `make` | 4.0+ | Build system |
| `python3` | 3.8+ | Config validator, manifest tools |
| `jq` | 1.6+ | JSON processing in scripts (optional but handy) |

### Build Environment Setup

```bash
# Ubuntu/Debian
sudo apt install gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu \
                 qemu-system-arm make python3 jq

# Verify
aarch64-linux-gnu-gcc --version
qemu-system-aarch64 --version
```

---

## File Structure

```
sources/nonux/
├── Makefile                    # Top-level build, reads kernel.json
├── kernel.json                 # Kernel configuration (which components to use)
├── tools/
│   ├── validate-config.py      # Config validator
│   ├── gen-config.py           # Generates gen/config.h from kernel.json
│   └── verify-registry.py      # Static checker for registry rules R1-R7 (see DESIGN.md)
│
├── core/                       # Kernel core (frozen, not swappable)
│   ├── boot/
│   │   ├── start.S             # ARM64 entry point (EL1 setup, MMU, stack)
│   │   ├── boot.c              # C entry: DTB parse, core init, launch components
│   │   └── linker.ld           # Linker script (memory layout)
│   ├── cpu/
│   │   ├── context.S           # Context switch (save/restore regs)
│   │   ├── vectors.S           # Exception vector table
│   │   ├── cpu.c               # Timer setup, EL0/EL1 transitions
│   │   └── cpu.h
│   ├── pmm/
│   │   ├── pmm.c               # Physical page allocator (bitmap)
│   │   └── pmm.h
│   ├── irq/
│   │   ├── gic.c               # GIC driver for QEMU virt
│   │   └── irq.h
│   └── lib/
│       ├── string.c            # memcpy, memset, strlen, etc.
│       ├── printf.c            # Minimal kernel printf
│       └── lib.h
│
├── framework/                  # Component framework (the wiring)
│   ├── component.c             # Lifecycle manager (init/enable/disable/destroy)
│   ├── component.h
│   ├── registry.c              # Component Graph Registry (slots, comps, connections)
│   ├── registry.h              # Registry API + graph_event subscribe/snapshot
│   ├── ipc.c                   # IPC router (async + sync shortcut, cap scanning)
│   ├── ipc.h
│   ├── handle.c                # Handle table (per-process, typed, rights)
│   ├── handle.h
│   ├── hook.c                  # Hook dispatch chains
│   ├── hook.h
│   ├── config.c                # Runtime config manager
│   ├── config.h
│   └── syscall.c               # Syscall dispatch (handle-based)
│
├── interfaces/                 # Interface definitions (C headers)
│   ├── scheduler.h             # struct scheduler_ops
│   ├── mem_manager.h           # struct mem_manager_ops
│   ├── vfs.h                   # struct vfs_ops
│   ├── block_device.h          # struct block_device_ops
│   └── char_device.h           # struct char_device_ops
│
├── components/                 # Swappable component implementations
│   ├── sched_rr/
│   │   ├── manifest.json
│   │   ├── sched_rr.c
│   │   ├── README.md
│   │   └── test/
│   │       └── test_sched_rr.c
│   ├── sched_priority/
│   │   ├── manifest.json
│   │   ├── sched_priority.c
│   │   ├── README.md
│   │   └── test/
│   │       └── test_sched_priority.c
│   ├── mm_buddy/
│   │   ├── manifest.json
│   │   ├── mm_buddy.c
│   │   ├── README.md
│   │   └── test/
│   │       └── test_mm_buddy.c
│   ├── vfs_simple/
│   │   ├── manifest.json
│   │   ├── vfs_simple.c
│   │   ├── README.md
│   │   └── test/
│   │       └── test_vfs_simple.c
│   ├── ramfs/
│   │   ├── manifest.json
│   │   ├── ramfs.c
│   │   └── README.md
│   ├── uart_pl011/
│   │   ├── manifest.json
│   │   ├── uart_pl011.c
│   │   └── README.md
│   ├── block_virtio/
│   │   ├── manifest.json
│   │   ├── block_virtio.c
│   │   └── README.md
│   └── posix_shim/
│       ├── manifest.json
│       ├── posix_shim.c
│       └── README.md
│
├── test/                       # Test infrastructure
│   ├── host/                   # Host-side test framework (run on build machine)
│   │   ├── Makefile
│   │   ├── test_runner.c       # Test harness main
│   │   ├── test_runner.h       # ASSERT macros, test registration
│   │   ├── mem_track.c         # Memory tracking layer (leak/UAF/double-free/overflow)
│   │   ├── mem_track.h
│   │   ├── registry_assert.c   # Registry invariant assertions (pointer audit, event trace)
│   │   ├── registry_assert.h
│   │   └── conformance/        # Interface conformance test suites
│   │       ├── conformance_scheduler.c
│   │       ├── conformance_vfs.c
│   │       ├── conformance_mem_manager.c
│   │       └── conformance_block_device.c
│   ├── kernel/                 # In-kernel integration tests
│   │   ├── test_ipc.c
│   │   ├── test_lifecycle.c
│   │   └── test_hooks.c
│   └── bench/                  # Benchmarks
│       ├── bench_ipc.c
│       └── bench_context_switch.c
│
├── gen/                        # Generated files (not checked in)
│   ├── config.h                # Generated from kernel.json
│   ├── sources.mk              # Compile list (COMPONENT_SRCS)
│   └── <component>_deps.h      # Per-component: deps struct + _DEPS_TABLE macro
│
└── docs/                       # Architecture & component docs
    ├── ARCHITECTURE.md         # Living architecture document
    ├── COMPOSITION-GUIDE.md    # How to wire components together
    └── COMPONENT-TEMPLATE.md   # Template for component README.md
```

---

## Implementation Details

### Boot Sequence (ARM64 QEMU virt)

QEMU loads the kernel image at a fixed address and jumps to it in EL1 (or EL2; we drop to EL1). The boot sequence:

```
start.S:
  1. Check current EL, drop to EL1 if needed
  2. Set up EL1 stack pointer
  3. Zero BSS section
  4. Set up initial identity-mapped page table (1:1 VA=PA)
  5. Enable MMU
  6. Jump to boot_main() in C

boot.c boot_main():
  1. Initialize UART (early console — hardcoded PL011 at 0x09000000)
  2. Print boot banner
  3. Parse DTB for memory size, device addresses
  4. Initialize physical memory allocator (PMM)
  5. Initialize GIC (interrupt controller)
  6. Initialize component framework
  7. Read compiled-in config, initialize component slots
  8. Enable components in dependency order
  9. Create init process
  10. Jump to scheduler — system running
```

### UART Bootstrap

The UART driver has a special role: we need serial output before the component framework is up. Solution:

- `core/lib/printf.c` has a hardcoded `uart_putc()` that writes directly to PL011 MMIO at `0x09000000`
- This is used only during early boot (steps 1-6 above)
- Once the component framework is up, the `uart_pl011` component takes over
- The early printf remains available as a panic/debug fallback

### Memory Map (QEMU virt)

```
0x00000000 - 0x08000000    Flash (unused, we boot from RAM)
0x08000000 - 0x08FFFFFF    GIC distributor + redistributor
0x09000000 - 0x09000FFF    PL011 UART
0x0a000000 - 0x0a003FFF    virtio MMIO devices
0x40000000 - 0x7FFFFFFF    RAM (1GB default, configurable with -m)

Kernel layout in RAM:
0x40000000 - 0x40080000    Kernel image (.text, .rodata, .data, .bss)
0x40080000 - 0x400C0000    Kernel stack (256KB)
0x400C0000 - 0x40100000    Initial page tables
0x40100000 - ...           Free physical memory (managed by PMM)
```

### Makefile Design

The top-level Makefile reads `kernel.json` via a Python helper to determine which component sources to compile:

```makefile
# Generate config and source list from kernel.json
gen/config.h gen/sources.mk: kernel.json tools/gen-config.py
	python3 tools/gen-config.py kernel.json gen/

include gen/sources.mk   # sets COMPONENT_SRCS

# Core sources (always compiled)
CORE_SRCS := $(wildcard core/**/*.c core/**/*.S)
FRAMEWORK_SRCS := $(wildcard framework/*.c)

# All sources
SRCS := $(CORE_SRCS) $(FRAMEWORK_SRCS) $(COMPONENT_SRCS)

# Cross-compiler
CROSS := aarch64-linux-gnu-
CC := $(CROSS)gcc
LD := $(CROSS)ld
OBJCOPY := $(CROSS)objcopy

CFLAGS := -ffreestanding -nostdlib -Wall -Wextra -O2 \
          -Icore -Iframework -Iinterfaces -Igen

kernel.elf: $(OBJS) core/boot/linker.ld
	$(LD) -T core/boot/linker.ld -o $@ $(OBJS)

kernel.bin: kernel.elf
	$(OBJCOPY) -O binary $< $@

run: kernel.bin
	qemu-system-aarch64 -M virt -cpu cortex-a53 -nographic \
	    -kernel kernel.bin -m 1G

validate-config: kernel.json
	python3 tools/validate-config.py kernel.json components/

test: test-host test-kernel

test-host:
	$(MAKE) -C test/host

test-kernel: kernel.bin
	# Boot kernel with test flag, capture serial output, check results
	timeout 30 qemu-system-aarch64 -M virt -cpu cortex-a53 -nographic \
	    -kernel kernel.bin -append "test" | tee test/output.log
	python3 test/check_results.py test/output.log

bench: kernel.bin
	timeout 60 qemu-system-aarch64 -M virt -cpu cortex-a53 -nographic \
	    -kernel kernel.bin -append "bench" | tee bench/output.log
	python3 bench/report.py bench/output.log
```

---

## Implementation Phases

### Phase 1: Boot to Serial Output

**Goal:** ARM64 kernel boots in QEMU and prints to serial console.
**Status:** DONE (2026-04-20, Session 4)

Steps:
1. Set up source tree and Makefile skeleton
2. Write `start.S` — EL2→EL1 drop, stack setup, zero BSS, jump to C
3. Write `linker.ld` — place kernel at 0x40000000, define sections
4. Write minimal `uart_putc()` in `core/lib/printf.c` (direct PL011 MMIO)
5. Write `boot.c boot_main()` — just print "nonux booting..." and halt
6. Build and test: `make run` shows output in QEMU terminal

**Validation:** `make run` (interactive) or `tools/run-qemu.sh -t 3` (scripted) prints the boot banner with memory layout and halts in `wfe`. Confirmed in Session 4 — `.text` 0x870 B at 0x40000000, `kernel.bin` 4720 B, free memory starts at `0x40042000` (kernel end + 256 KiB stack).

### Phase 2: Test Harness and Physical Memory

**Goal:** Host-side test infrastructure working. PMM and GIC implemented with tests proving memory correctness.
**Status:** DONE (2026-04-20, Session 5)

Steps:
1. Write `test/host/test_runner.c` — test harness with ASSERT macros, test registration, pass/fail reporting
2. Write `test/host/mem_track.c` — memory tracking layer (leak detection, use-after-free, double-free, red-zone overflow detection)
3. Write `core/pmm/pmm.c` — bitmap allocator (DTB-parsed memory info deferred; Phase 2 hardcodes RAM_END from QEMU `-m` flag)
4. Write PMM host-side tests — alloc/free correctness, no leaks after full alloc/free cycle, contiguous allocation, fragmentation recovery
5. Write `core/irq/gic.c` — **GICv2** (not GIC-400) on QEMU virt; QEMU CLI forces `-M virt,gic-version=2` since QEMU defaults to v3 in TCG mode
6. Write `core/cpu/vectors.S` — EL1 exception vector table (16 entries, 2 KiB aligned, 272-byte trap frame)
7. Set up ARM Generic Timer — EL1 physical timer (PPI 30) periodic tick interrupt at 10 Hz

**Validation:** `make test-host` → `PASS: 11/11 tests passed, 0 leaks, 0 errors`. `tools/run-qemu.sh -t 4` boots, prints `[pmm] total=261939 free=261939 pages`, `[timer] cntfrq=62500000 Hz, interval=6250000 ticks, rate=10 Hz`, then `[tick] 10 / 20 / 30` at 1-second intervals.

**Gotchas documented in Session 5:**
- QEMU `-kernel` loads the image at `RAM_BASE + 0x80000` — linker base is `0x40080000` (not `0x40000000`).
- GCC 15 freestanding builds need `-mno-outline-atomics`.
- `CNTHCTL_EL2.EL1PCEN` must be 1 before dropping to EL1 for `CNTP_*` access.
- `start.S` clears `CNTVOFF_EL2` so the virtual counter tracks the physical one.

### Phase 3: Component Framework

**Goal:** The framework that makes composition work — components can be initialized, enabled, disabled. Lifecycle, IPC, and the Component Graph Registry are proven correct by tests. Registry rules are enforceable by AI via a static checker.
**Status:** IN PROGRESS — slices 3.1 + 3.2 + 3.3 + 3.4 + 3.5 + 3.6 + 3.7 + 3.8 + 3.9a done. Framework lands in every kernel build (`FW_C` is no longer empty); `core/lib/kheap.{h,c}` provides `malloc`/`calloc`/`free` on top of the PMM. `framework/registry.{h,c}` (~800 LOC) provides: slot/component/connection register/lookup/unregister/swap/retune + traversal + generation counter (3.1); 9 typed change events with subscribe/unsubscribe + 256-entry ring-buffer change log with C11 atomics + refcounted mutation-stable snapshot + JSON serialisation (3.2); slot `_Atomic(pause_state)` + `fallback` + `nx_component_is_bound` / `_foreach_bound_slot` (3.8); freestanding-friendly (host-vs-kernel guarded, 3.9a). `framework/component.{h,c}` adds the lifecycle state machine (six verbs + legal transition matrix, 3.3) — all four lifecycle ops (`init`/`enable`/`disable`/`destroy`) plus the pause / resume protocol now fire (3.8 + 3.9a); the dep-injection mechanism (`NX_COMPONENT_REGISTER*` macros → `nx_components` linker section, `nx_resolve_deps`, 3.4; `NO_DEPS` variant simplified to drop the placeholder DEPS_FIELD parameter, 3.9a); `struct nx_component_ops` (3.6 — including `pause_hook` since 3.8); lifecycle hook dispatch (3.8); bound-slot destroy guard (3.8). `framework/hook.{h,c}` (3.8, ~250 LOC) provides 7 hook points with priority-sorted chains and mark-then-sweep unregister. `framework/ipc.{h,c}` (3.6) provides `nx_ipc_message` + `nx_ipc_cap` types, per-slot inbox, `nx_ipc_send` (sync direct / async enqueue), `nx_ipc_dispatch`, send/recv cap scanning, and `nx_slot_ref_retain/_release`; 3.8 adds IPC_SEND / IPC_RECV hook dispatch, per-edge `NX_PAUSE_{QUEUE,REJECT,REDIRECT}` routing, a per-`(src,dst)` hold queue, `nx_ipc_flush_hold_queue`, and a REDIRECT loop guard (`NX_ELOOP` after depth 4). `framework/bootstrap.{h,c}` (3.9a, ~180 LOC) walks the `nx_components` linker section, binds components to slots via `descriptor->name == nx_boot_slots[i].impl_name`, and runs the `init → enable` topo sort. Python tool chain: `tools/gen-config.py` (`manifest` + `kernel` subcommands; `kernel` now also emits `gen/slot_table.c`, 3.9a); `tools/validate-config.py`; `tools/verify-registry.py`. `make venv` / `make test-tools` / `make kernel-config` / `make verify-registry` wired into top-level Makefile; gen/sources.mk + gen/slot_table.c are real Makefile targets (3.9a). First real component: `components/uart_pl011/` (3.9a). 51 Python + 128 host + 11 kernel tests passing (190 total). Next: Phase 4 (scheduler) and slice 3.9b (runtime dispatcher upgrade). See [HANDOFF.md](HANDOFF.md#current-status) for the slice checklist.

Steps:
1. Define interface headers in `interfaces/` (scheduler.h, mem_manager.h, etc.)
2. Write `framework/registry.c` — Component Graph Registry (see DESIGN.md §Component Graph Registry):
   - `struct graph_registry`, `slot_node`, `component_node`, `connection`
   - `slot_register` / `component_register` / `connection_register` and their unregister / state-change variants
   - Traversal API (`graph_foreach_*`, `slot_foreach_dependent/dependency`, `graph_walk_subgraph`)
   - Snapshot + JSON serialization (`graph_snapshot_take`, `graph_snapshot_to_json`)
   - Change event dispatch (`graph_subscribe`, `graph_unsubscribe`)
   - Bounded append-only change log (ring buffer, C11 atomics)
3. Write `framework/component.c` — lifecycle state machine, dependency resolver (topo sort), dependency injection. Every lifecycle transition and every slot wiring calls into `framework/registry.c` — the registry is the sole source of truth.
4. Write `framework/ipc.c` — message queues, async send/recv, sync shortcut, **cap-array scanning** (`ipc_scan_send_caps`, `ipc_scan_recv_caps`). Cap-slot transfer support: `slot_ref_retain` / `slot_ref_release`.
5. ~~Write `framework/hook.c` — hook chain registration and dispatch. Hooks attach to registry connection edges and slot events.~~ **Done (slice 3.8.)** `framework/hook.{h,c}` provides 7 hook points (`IPC_SEND`, `IPC_RECV`, `COMPONENT_{ENABLE,DISABLE,PAUSE,RESUME}`, `SLOT_SWAPPED`), priority-sorted chains, mark-then-sweep unregister, typed per-point context union. IPC_SEND/RECV fire from `nx_ipc_send` / `nx_ipc_dispatch`; lifecycle hooks fire from `nx_component_{enable,disable,pause,resume}`. Runtime dispatch on `SLOT_SWAPPED` lands with slice 3.9.
6. Write `tools/gen-config.py` — read kernel.json and every selected component's `manifest.json`, emit:
   - `gen/config.h` — kernel config defines
   - `gen/sources.mk` — compile list
   - `gen/<component>_deps.h` per component — the `struct <component>_deps` type and the `_DEPS_TABLE(CONTAINER, FIELD)` macro used by `NX_COMPONENT_REGISTER` (see DESIGN.md §Dependency Injection Mechanism). Headers are regenerated on every build; the generator is deterministic (covered by `tools/tests/test_gen_config.py::*determinism*`), so R7's "manifest is source of truth" property holds. When the workflow commits `gen/<name>_deps.h` alongside the source, `verify-registry.py` gains a regenerate-and-diff check.
7. Write `tools/validate-config.py` — check config completeness, dependency graph validation, cycle detection
8. Write `tools/verify-registry.py` — Layer-1 machine checker for rules R1–R8 from DESIGN.md §AI Verification. Slice 3.7 implements R2 (struct field / manifest drift, regex) and R4 (retain/release call counts per component); R1 / R3 / R5 / R6 / R7 / R8 are marked `ai-verified` in `--list` output with the technical reason the machine check isn't in scope (clang/pycparser dataflow, interface schemas not yet defined, call-graph analysis, gen/ gitignored). Layer-2 enforcement — the per-rule rubric authoring and reviewing agents consult — lives in [AI-RULES.md](AI-RULES.md). Output: file:line citations for each violation. Exit code nonzero on any finding. Wired as a prerequisite of `make` and `make test`.
9. Write `test/host/registry_assert.c` — runtime invariant assertions used by every component test:
   - Pointer audit (every `struct slot *` field in component state maps to a registered connection)
   - Generation fence after swaps
   - Event trace matching against the change log
10. Create first `kernel.json` with minimal config
11. Wire boot sequence to initialize framework + registry, resolve dependencies via registry, load components in dependency order. Boot log uses the registry's JSON snapshot to print the initial composition.
12. Host-side tests:
    - Registry: slot_register + lookup, component_register + lifecycle state, connection_register + traversal, subgraph walk, snapshot consistency under concurrent reads + simulated swap, change log correctness, event subscriber delivery
    - Lifecycle state machine: init→enable→disable→destroy, verify memory clean at each boundary AND registry clean (zero residual nodes after destroy)
    - IPC: async send/recv, sync shortcut, message ordering, **cap scanning** (forged slot ref rejection, borrowed cap drop, unclaimed transfer flagged), retain/release correctness
    - Hook dispatch: register/unregister (on registry connection edges), observe/modify/abort, subscribe to graph events
    - Dependency resolver: valid ordering, cycle detection, missing dep errors
    - Lifecycle cycling: init→enable→disable→destroy repeated 100 times, zero leaks AND zero stale registry entries
    - `make verify-registry`: runs on framework + any example components, zero findings

**Validation:** `make test-host` — all framework tests pass with 0 leaks and 0 registry violations. `make validate-config` and `make verify-registry` pass. Kernel boots with framework, dumps initial composition via `graph_snapshot_to_json()`, prints component init log in dependency order.

#### Slice 3.9a — boot-time composition bring-up (done, Session 15)

Plan preserved below for reference; the narrative + actual deltas live
in [logs/session-15-bootstrap.md](logs/session-15-bootstrap.md).  Keep
an eye on the "Risks / open questions" block — the `-mstrict-align`
workaround and the linker-section MMU story both remain open against
the MMU bring-up milestone.

*Plan as of 2026-04-21 (pre-implementation):*

**Why split 3.9 into 3.9a / 3.9b.** The original 3.9 plan bundled
boot-time component bring-up with a runtime-dispatcher upgrade
(MPSC queue + per-CPU kthread). The dispatcher upgrade has a hard
dependency on Phase 4 (no kthreads exist yet). 3.9a keeps only the
parts that run with the kernel's current synchronous model; 3.9b
picks up the dispatcher / `pause_hook`-deadline / rollback story
once Phase 4 has landed kthread spawning.

**Five deliverables:**

1. **`framework/bootstrap.{h,c}`** — `nx_framework_bootstrap()` is
   the single entry point called from `boot_main`. Walks
   `__start_nx_components … __stop_nx_components`, registers every
   descriptor's component (instance_id = `"0"` for the v1 one-
   instance-per-manifest case), and matches components to slots by
   `descriptor->name` against the generated slot table.
2. **`tools/gen-config.py kernel` extension** — emits
   `gen/slot_table.c` containing:
   - A static `struct nx_slot` for each slot in `kernel.json`'s
     `components` map, named with a deterministic C identifier
     (e.g. the slot name with `.` → `_`).
   - A `struct nx_boot_slot nx_boot_slots[]` array mapping each
     slot to the implementation name (the `impl` field from
     `kernel.json`), so the bootstrap walker can bind
     `<iface, impl>` pairs.
   - `const size_t nx_boot_slots_count`.
   Adds two tests to `tools/tests/test_gen_config.py`: output
   shape + determinism.
3. **Topo sort + bring-up loop.** Bootstrap resolves deps (calling
   `nx_resolve_deps` to wire `struct <name>_deps` and emit
   connection edges) in dependency order — Kahn's algorithm over
   the descriptor section, with components that have no
   unsatisfied required deps scheduled first. For each scheduled
   descriptor: `nx_component_init` (UNINIT → READY), then
   `nx_component_enable` (READY → ACTIVE). Missing-required-dep is
   a boot panic with a clear file:line-style diagnostic pointing
   at the manifest.
4. **First real component — `components/uart_pl011/`.** Thinnest
   possible component that proves the boot walker actually runs
   something: a `handle_msg` that writes its payload to the already-
   initialised UART. No deps (`NX_COMPONENT_REGISTER_NO_DEPS`),
   no state, no pause semantics. `manifest.json` declares
   `iface: "char_device"`, matches `char_device.serial` in
   `kernel.json`.
5. **Kernel test — `test/kernel/ktest_bootstrap.c`.** Asserts after
   `nx_framework_bootstrap()`:
   - `nx_graph_component_count() >= 1`
   - `nx_slot_lookup("char_device.serial")->active != NULL`
   - The bound component's manifest_id equals `"uart_pl011"` and
     state is `NX_LC_ACTIVE`.
   - `nx_graph_snapshot_to_json` emits a non-empty, parseable JSON
     shape (contains the expected manifest name).

**In scope, opportunistic (not blocking):**

- **Runtime `NX_HOOK_SLOT_SWAPPED` dispatch.** Easy to add now —
  fan the registry's existing `NX_EV_SLOT_SWAPPED` emit out to
  hook handlers alongside subscribers. Logically pairs with 3.9b
  but has no dependency on kthreads.
- **Static hook wiring from `kernel.json`.** `kernel.json` already
  has an empty `hooks: []` array. gen-config emits `gen/hooks.c`
  with an `nx_boot_hooks[]` array; bootstrap registers them
  before the first `nx_component_enable`.

Either / both of these can be folded into 3.9a if the core story
lands quickly; otherwise they move to 3.9b.

**Out of scope — 3.9b territory (blocked on Phase 4 kthreads):**

- MPSC lock-free inbox replacing the host FIFO — dispatcher side
  needs a real thread to own `nx_ipc_dispatch`.
- `pause_hook` 1 ms wall-clock deadline — needs a monotonic-clock
  API and the dispatcher thread to enforce against.
- Pause-failure rollback — slot `pause_state` reverts to `NONE`,
  inbox / hold-queue messages resume. Couples with dispatcher
  ownership.
- Preemption + IRQ-context enqueue — `nx_ipc_enqueue_from_irq` as
  a separate, lock-free entry point. Also depends on Phase 4.

**Design decisions (3.9a):**

- **Instance IDs default to `"0"`.** `kernel.json` doesn't yet
  carry multi-instance bindings; gen-config + bootstrap hardcode
  `"0"` for every component. Multi-instance composition is
  Phase 6+ territory.
- **Bootstrap panics on missing required dep.** The alternative —
  skip-and-continue — defers the failure to an opaque runtime
  null deref. Panic-early gives a direct pointer to the manifest.
- **Framework sources enter the kernel build via `FW_C`.** Today
  `FW_C :=` is empty; 3.9a flips it to
  `registry.c component.c hook.c ipc.c bootstrap.c`. The host-
  side Makefile already includes these; the kernel build just
  hadn't been wired yet because there was nothing to call.
- **Bootstrap runs at the very end of `boot_main`.** PMM, GIC,
  timer, and UART are all up before components bring themselves
  up — so a component's `init` / `enable` callbacks can safely
  allocate memory and talk to the UART. Park in `wfi` after the
  bootstrap log.
- **Snapshot JSON goes to the UART via the existing `printf`
  path, chunked.** The snapshot's buffer can be larger than a
  reasonable stack allocation; bootstrap uses a static 4 KiB
  buffer in `.bss`.

**File-by-file changes:**

New files — `framework/bootstrap.h`, `framework/bootstrap.c`,
`components/uart_pl011/uart_pl011.c`,
`components/uart_pl011/manifest.json`,
`components/uart_pl011/README.md`,
`test/kernel/ktest_bootstrap.c`,
`proj_docs/nonux/logs/session-15-bootstrap.md`.

Modified — `core/boot/boot.c` (call `nx_framework_bootstrap()`
after timer init), `Makefile` (set `FW_C`; add
`test/kernel/ktest_bootstrap.c` to `KTEST_C`; add `gen/slot_table.c`
to generated sources), `tools/gen-config.py` (render_slot_table),
`tools/tests/test_gen_config.py` (two tests), `tools/schemas/kernel.schema.json`
(probably — verify no schema change needed), [DESIGN.md](DESIGN.md)
(§Execution Model v1 addendum pointing at the real bootstrap entry
point), [HANDOFF.md](HANDOFF.md) (checklist flip), this file
(steps 10 + 11 + 12 marked partially done; plan section removed
or archived).

**Test targets:** 49 py + 128 host + **7** kernel = **184** after
3.9a (+1 kernel test: `bootstrap_brings_up_expected_composition`).
`make verify-registry` starts catching real findings once
`components/uart_pl011/` lands (and a R2 struct field drift is the
first check that fires in anger).

**Risks / open questions:**

1. **Linker section on `-kernel` load.** `nx_components` descriptors
   land in `.rodata` by default; verify `__start_nx_components` /
   `__stop_nx_components` resolve as expected and aren't GC'd. Fix
   if needed with an explicit section output in `core/boot/linker.ld`.
2. **Component sources compiled with `-ffreestanding`.** The host
   build uses the system `cc`; the kernel build uses the cross-
   compiler with no libc. Double-check `uart_pl011.c` doesn't pull
   in anything it shouldn't.
3. **`gen/slot_table.c` chicken-and-egg.** `-include gen/sources.mk`
   at the top of Makefile silently succeeds on a fresh tree; the
   new `gen/slot_table.c` needs the same treatment — `make
   kernel-config` must emit it before `kernel.bin` depends on it.

**Estimate:** ~300 LOC C (bootstrap 120, uart_pl011 30, kernel test
60, linker / Makefile wiring 30, misc 60), ~80 LOC Python. Single
session, two commits (code + doc pass).

#### Slice 3.9b plan — runtime dispatcher upgrade

**Status:** unblocked now that Phase 4 has landed `sched_spawn_kthread`.
Cut into two reviewable sub-slices so the mechanical refactor (who
drives `nx_ipc_dispatch`) is separable from the correctness work
(rollback + deadlines + hook fan-out).

##### 3.9b.1 — Dispatcher mechanics

**Goal:** Move `nx_ipc_dispatch` off the caller's thread onto a
framework-owned dispatcher kthread. The existing component / IPC
test coverage stays green; externally-visible behaviour is
unchanged; the machinery underneath matches DESIGN.md §Execution
Model.

New:

- `framework/dispatcher.{h,c}` — `nx_dispatcher_init()` + the loop
  body. One dispatcher thread in v1 (single-CPU); the shape leaves
  room for the per-CPU dispatchers that SMP will bring.
- **MPSC lock-free inbox** — Vyukov-style intrusive queue.
  Producers (any thread, including ISRs) push via a single atomic
  `xchg` on the tail; the dispatcher pops in order. Added to
  `framework/ipc.c` alongside the existing per-slot hold-queue
  side-table, which stays as-is (paused-slot buffering has
  different requirements from the hot IPC path).
- `nx_ipc_enqueue_from_irq(slot, msg)` — ISR-safe entry point that
  *only* appends to the MPSC inbox. No `slot->active` deref, no
  allocation, no blocking. R8 remains intact.
- `components/uart_pl011` may evolve to consume an IRQ-driven
  message path later, but slice 3.9b.1 doesn't force that change.

Modified:

- `framework/ipc.c` — `nx_ipc_send` (async path) enqueues to the
  dispatcher inbox instead of invoking `slot_dispatch` inline.
  Sync-mode send shortcut stays synchronous (spec-level promise).
- `framework/bootstrap.c` — after the scheduler is handed off,
  spawn the dispatcher kthread via `sched_spawn_kthread`.
- **Host / kernel split.** Host tests cannot spawn kthreads; they
  keep the existing "enqueue then manually drain via
  `nx_ipc_dispatch`" shape. The kernel build replaces the manual
  pump with the dispatcher loop. Same `nx_ipc_send` contract
  either way, same queue type; the difference is who turns the
  crank. This mirrors the precedent set by `kheap.c`, timestamp
  handling in `registry.c`, and `cpu_switch_to`'s host stub.

Tests (target counts):

- Host: +3–5 MPSC queue unit tests
  (concurrent-producer-single-consumer ordering, wraparound,
  empty-pop returns NULL). No behavioural test changes for the
  existing 128 IPC cases — host path stays manual-pump.
- Kernel: +2 tests. `dispatcher_brought_up_by_bootstrap` (after
  bootstrap, the dispatcher kthread is enqueued and running). One
  ISR-enqueue end-to-end test using `nx_ipc_enqueue_from_irq`
  from the timer handler, confirming the dispatcher consumes it.

**Risks:**

- Dispatcher-thread vs ktest_main interleaving. Same concerns as
  slice 4.4's two-tasks test; the `reset_current_to_idle` between
  ktests covers the preempt path. Dispatcher lifetime is
  "from bootstrap until shutdown" — tests that spawn it don't need
  to tear it down.
- `cntpct_el0` reads from a handler aren't needed for 3.9b.1 (that's
  3.9b.2 territory), so no new asm accessors here.

##### 3.9b.2 — Correctness polish

**Goal:** Close the known gaps in slice 3.8's pause protocol and
wire the runtime hook fan-out.

New / modified:

- **Pause-failure rollback.** When `pause_hook` or `ops->pause`
  returns non-zero, `framework/component.c` reverts the slot
  `pause_state` back to `NONE` and replays the hold-queue entries
  through `nx_ipc_send`. Today the slot is left in `DRAINING`
  with a half-built hold queue — a latent bug.
- **`pause_hook` 1 ms deadline.** Add `core/cpu/monotonic.h`
  (reads `cntpct_el0` and `cntfrq_el0`). `nx_component_pause`
  samples before the hook call, reschedules if the deadline
  elapses — rollback path reuses the one from the bullet above.
  Host build: `CLOCK_MONOTONIC`.
- **Runtime `NX_HOOK_SLOT_SWAPPED` dispatch.** `framework/registry.c`'s
  existing `NX_EV_SLOT_SWAPPED` emit also fans out to
  `nx_hook_dispatch(NX_HOOK_SLOT_SWAPPED, &ctx)`. Test coverage
  includes a hook that observes the swap, another that `ABORT`s
  (the swap still completes — `SLOT_SWAPPED` is post-fact
  notification).

Tests (target counts):

- Host: +4–6. Rollback semantics (pause_hook fails, slot returns
  to NONE, replayed messages observed); deadline-elapsed rollback
  (mock monotonic in tests); runtime hook dispatch on swap.
- Kernel: +1 sanity (monotonic source reads back sane values).

**Out of scope for 3.9b entirely:**

- Multi-CPU dispatchers (SMP) — shape is compatible but the
  second+ CPU comes online in a dedicated SMP slice later.
- Runtime swap UX (the `nx_config_swap` handle API) — Phase 8's
  job; 3.9b only closes the swap-path *mechanics* such that Phase
  8 has a working substrate.

### Phase 4: Context Switch and Scheduler

**Goal:** Preemptive multitasking driven by the timer tick, with the scheduler split into a frozen core driver (`core/sched/`) and a swappable policy component (`components/sched_rr/`). See [DESIGN.md §Scheduler: Core Driver + Component](DESIGN.md#scheduler-core-driver--component) for the architectural split.
**Status:** COMPLETE — slices 4.1 / 4.2 / 4.3 / 4.4 all landed (Sessions 16–19).

**Slice map** — Phase 4 is cut into four reviewable slices, each of which lands `make test` green:

| Slice | Deliverable | Blocks |
|---|---|---|
| **4.1** | `struct task`, `core/cpu/context.S`, preempt counters | 4.4 |
| **4.2** | `interfaces/scheduler.h` + host conformance suite (runs against `sched_null` fixture) | 4.3 |
| **4.3** | `components/sched_rr/` manifest + impl + `NX_HOOK_CONTEXT_SWITCH` enum addition | 4.4 |
| **4.4** | Timer→`sched_check_resched`→`cpu_switch_to`, idle task, kthread primitive, interleaved-task demo | 3.9b |

Critical path: 4.1 → 4.3 → 4.4. Slices 4.1 and 4.2 are independent and can land in either order. Runtime scheduler swap and a second policy implementation (`sched_priority`) are deferred to Phase 8 alongside the real recomposition API — Phase 4's swappability demo is the interface-conformance-passes property plus manifest-driven binding, not a live runtime swap.

#### Slice 4.1 — task + context switch (core primitive)

**Goal:** A host-testable task representation and an ARM64 `cpu_switch_to` that demonstrably swaps register state. No scheduler policy, no timer hookup, no preemption.

New files:

- `core/cpu/context.S` — `cpu_switch_to(prev, next)`. Saves callee-saved regs (x19–x28, x29, x30), SP, and `TPIDR_EL1` into `prev->cpu_ctx`; loads the same set from `next->cpu_ctx` and returns. Classic ARM64 coroutine switch: no FP state (consistent with `-mgeneral-regs-only`), save pairs use `stp`/`ldp` at 16-byte-aligned offsets so `-mstrict-align` stays happy.
- `core/sched/task.h`, `core/sched/task.c` — `struct task` (id, name, `enum task_state` { `NX_TASK_READY`, `NX_TASK_RUNNING`, `NX_TASK_BLOCKED`, `NX_TASK_ZOMBIE` }), `struct nx_cpu_ctx` (12 × u64 callee-saved + sp + tpidr), kernel stack base+size, preempt_count, a `struct nx_list_node sched_node` for the future runqueue. `nx_task_create(name, entry, arg, kstack_pages)` / `nx_task_destroy` / `nx_task_current()`.
- `core/sched/sched.h`, `core/sched/sched.c` — declarations for `cpu_switch_to`, `nx_task_current()` (reads `TPIDR_EL1`), `preempt_disable()`/`preempt_enable()` (nested counter on `current->preempt_count`). No policy, no stashed `g_sched` yet (slice 4.4 adds those).
- `core/lib/list.h` — intrusive doubly-linked-list helpers if not already present (they aren't); slice 4.3's runqueue will use them.
- `test/host/task_test.c` — `nx_task_create` populates `cpu_ctx` so the first switch lands at `entry(arg)` (x0 = arg, pc = entry, sp inside the allocated kstack). `nx_task_destroy` is leak-free. `preempt_disable`/`enable` nest correctly.
- `test/kernel/ktest_context.c` — two tests: `context_switch_round_trip` (create task B, switch to it, B calls `cpu_switch_to(B, A)`, assert sentinel the B entry wrote is observable from A); `context_switch_preserves_callee_saved` (load known pattern into x19..x28 before switch out, assert intact after switch in).

Modified:

- `Makefile` — add `core/cpu/context.S` to `CORE_S`; add `core/sched/task.c core/sched/sched.c` to `CORE_C`; add `test/kernel/ktest_context.c` to `KTEST_C`.
- `test/host/Makefile` — add `task_test.c` to `SRCS` and the task / sched sources to the cross-compiled-source vpath.

**Out of scope:** No scheduler policy, no timer integration, no idle task, no preemption yet — this slice is the ARM64 save/restore primitive + `struct task`. Everything else slots in on top in slices 4.3 / 4.4.

**Test delta target:** +1–2 host tests, +2 kernel tests. `make test` expected at ~**192–193 / 192–193**.

**Risks:**
- `TPIDR_EL1` convention. Phase 4 claims `TPIDR_EL1` as the "current task pointer" slot. Phase 5's per-process address spaces will still need a place to stash current, and `TPIDR_EL1` is the conventional choice; this commits to it.
- `-mstrict-align`. `struct nx_cpu_ctx` must be 16-byte aligned so `stp`/`ldp` pairs land on 16-byte offsets. Enforce with `__attribute__((aligned(16)))`.
- First-switch initial state. The initial `cpu_ctx` hand-crafted in `nx_task_create` needs an entry thunk in `context.S` that jumps through `x19 (entry)` / `x20 (arg)` to the task function — the classic ARM64 trick where the first "return" of `cpu_switch_to` lands on the thunk rather than a real call-site. Document this clearly in the assembly.

#### Slice 4.2 — conformance suite + scheduler interface header

**Goal:** Define `struct scheduler_ops` in a real interface header, ship the host-side conformance suite every scheduler impl must pass, and exercise the suite itself against a `sched_null` host fixture. No real scheduler yet.

New files:

- `interfaces/scheduler.h` — `struct scheduler_ops` with `pick_next(self) -> struct task *`, `enqueue(self, task *)`, `dequeue(self, task *)`, `yield(self)`, `set_priority(self, task *, int)`, `tick(self)`. Ownership annotations match the manifest-interface convention: every task pointer is `borrow`.
- `test/host/conformance/conformance_scheduler.h/.c` — `conformance_scheduler_run(const struct scheduler_ops *ops, void *self, const struct conformance_factory *factory)`. Cases: `pick_on_empty_returns_null`, `enqueue_then_pick_returns_task`, `enqueue_fifo_order`, `dequeue_removes`, `dequeue_nonexistent_returns_error`, `priority_api_consistent` (either implements or uniformly returns `NX_EINVAL`), `enqueue_100_pick_100_residue_free`.
- `test/host/fixtures/sched_null.c` — deliberately trivial "valid but empty" impl: always returns NULL from `pick_next`, always OK from enqueue/dequeue. Proves the conformance harness itself works.
- `test/host/conformance_scheduler_test.c` — runs the harness against `sched_null` (expected passes) and a deliberately-broken variant (expected failures) to show the harness distinguishes.

Modified:

- `test/host/Makefile` — adds conformance_scheduler + sched_null + the new test file.

**Test delta target:** +~10 host tests. `make test` at ~**202 / 202**.

#### Slice 4.3 — `sched_rr` component + `NX_HOOK_CONTEXT_SWITCH` enum

**Goal:** Ship `components/sched_rr/` as a real component. Passes conformance. Wires into the framework the same way `uart_pl011` does. **No preemption yet** — `sched_rr` is called cooperatively.

New files:

- `components/sched_rr/manifest.json` — name `sched_rr`, version `0.1.0`, iface `scheduler`, `spawns_threads: false`, `pause_hook: false`, `config: { time_quantum_ms: { default: 10, ... } }`. No `requires` (MVP). Binding to the `scheduler` slot comes later via `kernel.json`.
- `components/sched_rr/sched_rr.c` — `struct sched_rr_state { struct nx_list_head runqueue; struct task *current; unsigned quantum_ticks; unsigned remaining; };`. `ops->init/enable/disable/destroy`, plus a separate `sched_rr_ops` table (`struct scheduler_ops`) that the core driver looks up. `handle_msg` dispatches by `msg_type` to the scheduler verbs so tests that drive via IPC work too.
- `components/sched_rr/README.md` — per the uart_pl011 template. Documents: no deps, no worker threads, runqueue is internal state so `ops->pause` just stops accepting enqueues while `current` drains.

Modified:

- `framework/hook.h` — extend `enum nx_hook_point` with `NX_HOOK_CONTEXT_SWITCH` (before `NX_HOOK_POINT_COUNT` sentinel). Extend `struct nx_hook_context.u` with the `csw { struct task *prev, *next; }` arm (already documented in DESIGN.md). Slice 4.3 adds the enum and the union arm; slice 4.4 is where the hook is actually dispatched from the reschedule shim.
- `kernel.json` — add `scheduler` slot binding to `sched_rr`.
- `tools/gen-config.py` — no change expected; the existing `cmd_kernel` picks up the new slot automatically.

New tests:

- `test/host/component_sched_rr_test.c` — lifecycle cycling (init→enable→disable→destroy × 100, zero leaks), invokes `conformance_scheduler_run(&sched_rr_ops, ...)`, confirms ownership contract (after enqueue, task pointer is still the caller's).
- `test/kernel/ktest_sched_bootstrap.c` — after `nx_framework_bootstrap()` the `scheduler` slot is bound to `sched_rr` in `NX_LC_ACTIVE` state.

**Test delta target:** +~10 host, +1 kernel. `make test` at ~**213 / 213**.

**Phase 3 invariants:**
- **R2**: no `struct nx_slot *` fields in state (no deps) → trivially passes.
- **R4**: no retain/release → trivially passes.
- **R6**: `handle_msg` does not stash inbound caps (scheduler messages carry `struct task *`, not `struct nx_ipc_cap`).
- **R8**: the component itself is fine; the timer-ISR hookup (slice 4.4) is where R8 gets pressure-tested.

#### Slice 4.4 — preemption + kthread primitive + interleaved-task demo

**Goal:** Timer tick triggers a context switch through `sched_rr`. Two kernel test tasks run concurrently and their output interleaves. Idle task backs the system. This slice is where Phase 4's headline visibly works under QEMU.

New files:

- `core/sched/idle.c` — idle task entry: `for (;;) wfi;`. One slot on the runqueue (special-cased as fallback when `pick_next` returns NULL).
- `core/sched/kthread.c` — `sched_spawn_kthread(name, entry, arg) -> struct task *`. Wraps `nx_task_create`, enqueues on the active policy via `g_sched->enqueue(g_sched_self, task)`. **Not** a public `nx_*` framework API — it's a core/sched/ symbol; slice 3.9b promotes it when the dispatcher needs it.
- `test/kernel/ktest_sched.c` — three tests: `sched_two_tasks_interleave` (cooperative yield), `sched_tick_drives_preemption` (CPU-busy tasks get time via the tick), `sched_spawn_kthread_runs_entry` (sanity-check the primitive).

Modified:

- `core/sched/sched.{h,c}` — add stashed `g_sched` / `g_sched_self`, `sched_init(ops, self)`, `sched_start()` (promotes boot context into the idle task, runs first `cpu_switch_to` to the picked task), `sched_tick()` (called from the timer ISR, decrements `current->quantum_remaining`, sets `need_resched`), `sched_check_resched()` (called at IRQ-return — preempt-disable, `pick_next`, fire `NX_HOOK_CONTEXT_SWITCH`, `cpu_switch_to`, preempt-enable), `nx_task_yield()` (explicit cooperative yield).
- `core/timer/timer.{h,c}` — `on_tick` calls `sched_tick()`. Add `timer_pause()` / `timer_resume()` (nested counter, masks/unmasks the timer PPI at the GIC). These are the bookends the recomposition protocol will call in Phase 8 — **introduced here** so the recomposition machinery doesn't have to reach into the timer's guts later.
- `core/cpu/vectors.S` — after the existing `RESTORE_TRAPFRAME` in `_irq_stub`, branch into `_check_resched` (a thin C helper that checks `current->need_resched` and calls `sched_check_resched()` on the interrupted task's kernel stack). EL0-return paths (Phase 7) will reuse the same shim.
- `framework/bootstrap.c` — after the topo-sorted init/enable loop, look up the `scheduler` slot; if bound, fetch `sched_rr_ops`/self and call `sched_init`. If unbound the boot path skips `sched_start()` and falls back to the existing `wfi` — useful for unit tests that don't compose a scheduler.
- `core/boot/boot.c` — replace the final `for (;;) wfi;` with `sched_start()` (idle task takes over the same CPU context).

**Test delta target:** +3 kernel. `make test` at ~**216 / 216**.

**Phase 3 invariants pressure-tested here:**
- **R8 (slot-resolve locality).** The timer ISR *must not* read `slot->active`. The design is: ISR path sets `current->need_resched`; IRQ-return shim reads `g_sched` (a direct pointer stashed at init time, not a slot pointer held by ISR-era code), calls `pick_next` on a dispatcher-equivalent context (preempt disabled). R8's AI rubric treats `g_sched` as a named framework escape hatch — verifies it's written only by `sched_init` and read only from within `core/sched/`.
- **Timer quiescence.** `timer_pause()`/`timer_resume()` bookends give future recomposition a clean way to freeze ticks during the PAUSE → REWIRE → RESUME window, closing the race where a tick mid-swap would read a torn `g_sched`.
- **`NX_HOOK_CONTEXT_SWITCH` dispatch.** Fires from `sched_check_resched` *after* `pick_next` and *before* `cpu_switch_to`. Preempt-disabled, so hook handlers can assume the selected `next` is the one that'll actually run.

**Risks:**
- IRQ-return reschedule is ARM64-fiddly. The conventional shape: in `_irq_stub`, after `RESTORE_TRAPFRAME` restored the user state on the stack but *before* `eret`, branch to a tiny assembly trampoline that checks a per-task flag and, if set, *re-saves* the still-on-stack state into the outgoing task's `cpu_ctx`, calls `sched_check_resched()`, and `eret`s to the incoming task's frame. Budget a full session for this if it turns out nasty.
- `uart_putc` interleave. Two tasks printing concurrently under preemption will byte-interleave. That's actually what demonstrates preemption is working — keep it.

#### Deferred from Phase 4

- **`sched_priority` as a second policy** — now a Phase 8 deliverable. A second impl only demonstrates swappability if there's a live swap path to exercise it; Phase 4 already proves swappability through the conformance suite.
- **Runtime scheduler hot-swap** — Phase 8.
- **Sleep / wait** — needs a timer as a real component (currently core-only), so pairs with Phase 5's mm bring-up at the earliest.

**Validation (end of Phase 4):** `make test` all green. Two kernel test tasks visibly interleave their output when booted under `tools/run-qemu.sh`. Swapping scheduler implementations is possible at *build time* by editing `kernel.json` — proof lives in the conformance suite + manifest-driven bind, not a runtime swap.

### Phase 5: Virtual Memory and Handle System

**Goal:** Per-process address spaces. Handle-based syscall API.
**Status:** NOT STARTED. Planned as six session-sized slices (5.1–5.6).

Phase 5 folds the MMU, a buddy allocator, the handle table, the syscall path,
EL0 bring-up, and channels into one phase. Shipped as slices rather than a
single big push so each session leaves the tree buildable and tested:

#### Slice 5.1 — MMU + identity map

Turn the MMU on at EL1 with a simple kernel address space.

1. Build an identity-mapped page table in BSS:
   - L1 table + two L2 tables (4 KiB each).
   - RAM region (0x40000000–0x80000000) as Normal memory, cacheable, RWX for
     now — split perms land in 5.2 once mm_buddy owns the page table.
   - MMIO region (0x00000000–0x40000000) as Device-nGnRnE (covers GIC and
     UART0 on QEMU virt).
2. Configure MAIR_EL1 (attrs 0=Device-nGnRnE, 1=Normal WB-WA), TCR_EL1
   (T0SZ=25 → 39-bit VA, 4 KiB granule, IPS=36-bit), TTBR0_EL1.
3. Enable SCTLR_EL1.M / .C / .I with the obligatory `isb` + TLB flush + icache
   invalidate. Keep `.WXN = 0` for now.
4. Drop `-mstrict-align` from Makefile CFLAGS — Normal-memory unaligned loads
   are legal once the MMU is on.
5. Kernel tests: deliberate unaligned u32 read/write (must no longer fault),
   MMU-enabled bit check via `SCTLR_EL1`, MMIO still routable (UART output
   still works end-to-end).

**Exit criteria:** `make test` green with `-mstrict-align` removed; unaligned
access test passes in QEMU; boot log unchanged.

#### Slice 5.2 — `components/mm_buddy/`

Buddy allocator shipped as a swappable component.

1. `components/mm_buddy/{manifest.json,mm_buddy.c,README.md}` — conformant
   component following the slice-4.3 `sched_rr` pattern.
2. `interfaces/mm.h` — interface for `alloc_pages(order)` / `free_pages(p,
   order)` / `page_size()`.
3. Conformance suite (`test/host/conformance/conformance_mm.{h,c}`) — empty
   allocator state, alloc/free round-trip, 100× lifecycle cycling.
4. `kernel.json` binds `memory.page_alloc ← mm_buddy`; bootstrap brings it
   up to `NX_LC_ACTIVE`.
5. Refactor the boot-time kernel page table to use mm_buddy for L2/L3
   dynamic growth (page-table allocation is the first real consumer).

**Exit criteria:** Conformance suite green; buddy component drives 100
lifecycle cycles with no residue; kernel still boots.

#### Slice 5.3 — Handle framework

`framework/handle.{h,c}` with typed handles and rights attenuation.

1. `struct nx_handle_table` — per-task (later per-process) array with a
   free-list. Fixed capacity for v1 (64 handles per task).
2. `enum nx_handle_type`: `HANDLE_CHANNEL`, `HANDLE_VMO`, `HANDLE_PROCESS`,
   `HANDLE_IRQ` — only the scaffolding; actual objects wire up in later
   slices.
3. `uint32_t rights` bitmask: `RIGHT_READ`, `RIGHT_WRITE`, `RIGHT_TRANSFER`,
   `RIGHT_MAP`, `RIGHT_WAIT`.
4. `nx_handle_alloc / _lookup / _close / _duplicate` — duplicate enforces
   rights attenuation (never grants more than the source has).
5. Host + kernel tests: alloc/lookup/close, duplicate with reduced rights,
   duplicate with expanded rights fails, closed handles stay closed.

**Exit criteria:** Handle table under `make test`; handle lifecycle runs
cleanly for 1000 iterations with no leak.

#### Slice 5.4 — Syscall entry (SVC from EL1 for now)

`framework/syscall.c` + SVC plumbing in the vector table.

1. `core/cpu/vectors.S` gets an SVC-from-EL1 path (synchronous-from-same-EL
   vector) that saves the trap frame and jumps to `syscall_entry`.
2. `framework/syscall.c` dispatch table keyed by syscall number.
3. First syscall: `nx_debug_write(buf, len)` — writes `len` bytes to UART.
   Smoke test from a kernel test: issue SVC, verify bytes show up on UART.
4. Second syscall: `nx_handle_close(h)` — minimal real syscall that touches
   the handle table.
5. `nx_status_t` typedef + status code table.

**Exit criteria:** A kernel test issues SVC and comes back with the right
return value; `nx_debug_write` output appears on UART.

#### Slice 5.5 — First EL0 process

Spawn a trivial EL0 program that makes syscalls.

1. `core/cpu/el0_entry.S` — drop-to-EL0 path using `eret` with SPSR_EL1 set
   to EL0t.
2. Per-task TTBR0 page table (each EL0 task gets its own address space;
   kernel TTBR1 shared).
3. Baked-in EL0 test program — linked into the kernel image at a fixed VA,
   calls `nx_debug_write("hello from el0\n")` then `nx_debug_write` again
   and exits via `nx_process_exit`.
4. `sched_spawn_el0_task` helper — extension of 4.4's `sched_spawn_kthread`
   that sets up EL0 state.
5. Kernel test: spawn the EL0 program, wait for it to exit, assert UART log
   contains the expected output.

**Exit criteria:** EL0 program runs, makes two syscalls, exits cleanly.
Kernel survives and continues to idle.

#### Slice 5.6 — Channels

`HANDLE_CHANNEL` + `nx_channel_create / _send / _recv` on top of existing IPC.

1. `struct nx_channel_endpoint` wraps a slot + a direction tag. Create returns
   two handles (one per endpoint) with `RIGHT_READ | RIGHT_WRITE | RIGHT_TRANSFER`.
2. `nx_channel_send` maps to `nx_ipc_send`; `nx_channel_recv` blocks until
   the dispatcher runs the endpoint's handler (v1 uses a simple wait queue).
3. First channel-based test: kernel service (an in-kernel echo component)
   exposes a channel; EL0 test program sends a message, receives the echo.

**Exit criteria:** End-to-end EL0 → channel → kernel handler → channel →
EL0 round trip green under QEMU.

**Phase 5 validation (end of 5.6):** Userspace process runs in EL0, creates
a channel, sends/receives a message via handle syscalls.

### Phase 6: VFS and Ramfs

**Goal:** Filesystem layer. Processes can open/read/write files.
**Status:** COMPLETE. Sliced into four sessions (6.1–6.4); all four shipped.

Phase 6 lands a pluggable VFS layer, one real filesystem driver (ramfs),
file-handle syscalls, and an EL0 round-trip through all three. Shipped as
slices so each session leaves the tree buildable and tested:

#### Slice 6.1 — FS-driver interface + conformance harness

Interface-first plumbing before either real component lands.

1. `interfaces/fs.h` — `struct nx_fs_ops { open, close, read, write }` (the
   lower API that every filesystem driver implements). `NX_FS_OPEN_READ /
   _WRITE / _CREATE` flags. Per-open state owned by the driver between
   `open` and `close`; `read` / `write` advance a per-open cursor.
2. `test/host/conformance/conformance_fs.{h,c}` — universal conformance
   cases exercising the `nx_fs_ops` contract (create-on-open, ENOENT
   without create, write/reopen/read round-trip, read-past-EOF, two-open
   cursor independence, EPERM on write without WRITE-right).
3. `test/host/conformance_fs_test.c` — TEST()s against an in-file
   `fs_stub` fixture driver (a minimal in-memory dictionary of named
   byte buffers). Proves the harness is exercisable; ramfs plugs into
   it unchanged in 6.2.
4. No components, no bootstrap changes, no kernel.json change — matches
   the 4.2 pattern (interface + conformance before first real component).

**Exit criteria:** `make test` green with the seven universal + two
fixture-internal cases all passing; `fs_stub` is test-only code not
linked into kernel.bin.

#### Slice 6.2 — `components/vfs_simple/` + `components/ramfs/`

First real VFS layer plus the first real filesystem driver.

1. `interfaces/vfs.h` — `struct nx_vfs_ops { open, close, read, write }`
   (the upper API the syscall layer will dispatch through in 6.3).
   Bit-compatible `NX_VFS_OPEN_*` flags with `NX_FS_OPEN_*` so
   vfs_simple can forward them verbatim.
2. `components/ramfs/` — first real filesystem driver; bound to a new
   `filesystem.root` slot. Implements `nx_fs_ops` (published through
   the descriptor's `iface_ops` field). Static fixed-capacity inode
   table (8 files × 256-byte buffers, 32-slot per-open pool — same
   geometry as slice 6.1's `fs_stub`); no `memory.page_alloc`
   dependency in v1 (deferred until a consumer needs variable-size
   storage). Passes the universal `conformance_fs` cases + 100×
   lifecycle cycling + two ramfs-specific exhaustion smoke tests.
3. `components/vfs_simple/` — bound to the `vfs` slot. Implements
   `nx_vfs_ops` as a thin pass-through: every op calls
   `nx_slot_lookup("filesystem.root")` and dispatches through
   `slot->active->descriptor->iface_ops` — no cached pointer, so
   future hot-swap of the root filesystem lands as a single
   `nx_slot_swap` with no stale references inside vfs_simple. No
   manifest dep on `filesystem.root` — the slot itself is the
   late-binding primitive (DESIGN §Slot-Based Indirection), and
   in-tree components don't have per-component deps.h emission yet.
   Relative paths rejected with `NX_EINVAL` before reaching the
   driver.
4. `kernel.json` gains two slots: `filesystem.root ← ramfs` and
   `vfs ← vfs_simple`; bootstrap brings both up to `NX_LC_ACTIVE`
   (live boot log reports `5 slots, 5 components`).
5. Kernel tests: `ktest_vfs` proves both slots are bound in
   `NX_LC_ACTIVE`, and drives a create/write/close/reopen/read round-
   trip through vfs_simple → ramfs on the booted instance. Host
   tests cover ramfs conformance + exhaustion paths and vfs_simple's
   dispatch + late-binding behaviour via an in-test fake fs driver.

**Exit criteria:** Two new components in-tree, both conformant; kernel
boots with vfs+ramfs active; kernel tests cover the end-to-end path.

#### Slice 6.3 — File syscalls + `HANDLE_FILE` destructor

Wire the VFS to the existing syscall and handle machinery; first EL0 file
I/O.

1. Three new syscalls in `framework/syscall.c` following the slice-5.6
   channel shape:
   - `NX_SYS_OPEN(path, flags)` — `copy_path_from_user` reads the
     NUL-terminated path one byte at a time through the user-window
     bounds check (so a path that ends before the window boundary
     succeeds even when copying `NX_PATH_MAX = 128` bytes in one go
     would overflow).  Resolves the `vfs` slot, calls
     `vfs_ops.open`, allocates a `HANDLE_FILE` with rights derived
     from flags (`NX_RIGHT_READ` / `_WRITE` from the matching open
     flags), and returns the handle (or rolls back the driver open
     on handle-alloc failure).
   - `NX_SYS_READ(h, buf, cap)` / `_WRITE(h, buf, len)` — look up
     the handle, validate `type == HANDLE_FILE` and required
     rights, `copy_from_user` / `copy_to_user` against a fixed
     `NX_FILE_IO_MAX = 256` staging buffer (caller loops for more;
     matches the channel shape), dispatch through
     `vfs_ops.read` / `.write`.
2. `HANDLE_FILE` branch in the type-aware `sys_handle_close`: if the
   lookup yields a HANDLE_FILE with a non-NULL object, resolve the
   `vfs` slot and call `vfs_ops.close` before releasing the handle
   slot.  If the slot has been unmounted mid-flight the driver-side
   close is skipped (documented edge case — handle slot still
   reclaimed).
3. EL0 test program `test/kernel/user_prog_file.S` runs the full
   round-trip: open `/hello` with `READ|WRITE|CREATE`, write
   `"[el0-file-ok]"`, close (runs the destructor), reopen READ-only,
   read 13 bytes, close, `debug_write` the buffer.  The kernel test
   (`el0_file_open_write_close_reopen_read_roundtrip`) asserts the
   `debug_write` counter rises and the ktest log shows the marker.
4. Host tests in `test/host/file_syscall_test.c` use the same in-test
   `fake_fs` injection pattern as `component_vfs_simple_test.c`,
   extended to stand up the full `vfs` slot + vfs_simple component so
   the syscall layer has a complete composition to dispatch through.
   Covers: round-trip + close-destructor counter, rights derivation,
   wrong-type handles, stale handles, no-VFS → NX_ENOENT, NULL path,
   no-NUL-in-cap, and length-capping at NX_FILE_IO_MAX.

**Exit criteria:** EL0 userspace reads/writes a file via syscalls;
kernel-test log shows `[el0-file-ok]`.

#### Slice 6.4 — Readdir + seek + directory iteration

Close the phase with the last two common file ops.

1. Interfaces grow two methods each:
   - `nx_fs_ops.seek(self, file, offset, whence)` — per-open cursor
     reposition.  `NX_FS_SEEK_SET / _CUR / _END` whence values;
     resulting position clamped to `[0, size]` (past-EOF → NX_EINVAL;
     no hole-filling in v1).
   - `nx_fs_ops.readdir(self, uint32_t *cookie, struct nx_fs_dirent
     *out)` — filesystem-level enumeration, not tied to a dir handle.
     Caller-owned monotonic cookie (driver-defined semantics: v1
     ramfs uses the slot index directly).  NX_OK / NX_ENOENT (done)
     / NX_EINVAL.  No dir handles in v1 because ramfs is flat-
     namespaced; tree-FS drivers will grow a second `readdir_at(dir,
     cookie, out)` flavour when they land.
   - `struct nx_fs_dirent` carries `{uint32_t name_len; char
     name[NX_FS_DIRENT_NAME_MAX=64]}`.  Always NUL-terminated.
   - `nx_vfs_ops` gets matching methods; v1 VFS forwards unchanged.
2. Conformance suite adds 5 cases: readdir-empty-returns-enoent,
   readdir-yields-created-then-enoent, seek-set-zero-restarts,
   seek-end-returns-size, seek-past-size-einval (also covers
   unknown-whence).  fs_stub and ramfs both implement the new ops.
3. Two new syscalls:
   - `NX_SYS_SEEK(h, offset, whence)` — look up HANDLE_FILE with
     `NX_RIGHT_SEEK`, forward to `vops->seek`.  Returns new absolute
     position (≥ 0) or negative NX_E*.
   - `NX_SYS_READDIR(user_cookie, user_out)` — `copy_from_user` the
     cookie, dispatch through `vops->readdir`, `copy_to_user` the
     dirent + updated cookie.  NX_OK / NX_ENOENT / NX_EINVAL.
4. `sys_open` now grants `NX_RIGHT_SEEK` implicitly whenever it
   grants READ or WRITE — any seekable open gets seek.  A future
   non-seekable stream type can drop the bit or attenuate via
   `nx_handle_duplicate`.
5. EL0 `user_prog_readdir.S` seeds `/rd`, loops
   `readdir(&cookie, &ent)` until NX_ENOENT (emitting each
   `ent.name` + `\n` to UART via `debug_write`), then emits the
   `[el0-rdr-ok]` tail marker.  The kernel ktest
   `el0_readdir_walks_root_and_emits_names_then_marker` asserts
   the debug_write counter ≥ 3 (the minimum: 1 name + newline + tail
   marker; actual log varies by ktest execution order).

**Exit criteria:** EL0 program lists filenames in `/`; ktest log
shows `[el0-rdr-ok]`.

**Phase 6 validation (end of 6.4):** Userspace can create, write, read,
seek, and list files on ramfs — the full basic-file-I/O contract.

### Phase 7: Process Model and POSIX Shim

**Goal:** fork/exec, signals, pipes. Busybox runs.
**Status:** IN PROGRESS. Sliced into 7.1–7.7 (the first four are process-
model plumbing; the last three are busybox integration).  Slices 7.1 + 7.2 + 7.3 + 7.4 + 7.5 + 7.6a + 7.6b + 7.6c.0 + 7.6c.1 + 7.6c.2 + 7.6c.3 + 7.6c.4 + 7.6d.1 + 7.6d.2 (= 7.6d.2a + 7.6d.2b + 7.6d.2c) + 7.6d.3a + 7.6d.3b + 7.6d.3c + 7.6d.N.0 + 7.6d.N.1 + 7.6d.N.2 + 7.6d.N.3 + 7.6d.N.4 + 7.6d.N.5 + 7.6d.N.6a + 7.6d.N.6b + 7.6d.N.7 + 7.6d.N.8 + 7.6d.N.9 + 7.6d.N.10 + 7.6d.N.11 complete; 7.6d.N.12 (sigaction + handler dispatch) + 7.6d.N.13 (tolerable stubs) + 7.6d.N.14 (3-stage pipe, skippable) + 7.6d.N.15 (fork-FILE-inheritance, skippable) + 7.6d.N.16 (drop in-handle generation, skippable) + 7.6d.N.final (UART RX + interactive prompt) + 7.7 pending.

Phase 7 builds the process abstraction on top of the EL0-task work from
Phase 5 and the file-I/O stack from Phase 6, then layers a POSIX shim
and a cross-compiled busybox on top.  Shipped as sliced sessions so
each one leaves the tree buildable and tested:

#### Slice 7.1 — `struct nx_process` + per-process handle tables

First process abstraction.  No address-space isolation yet (shared
TTBR0 — that's 7.2's job).

1. `framework/process.{h,c}` — `struct nx_process` with `pid`,
   `name`, `handles` (a `struct nx_handle_table`), `state`
   (ACTIVE / EXITED), and `exit_code`.  `nx_process_create(name)`,
   `_destroy(p)`, `_current()` (reads `nx_task_current()->process`),
   `_lookup_by_pid(pid)`.  PIDs monotonic from 1; pid 0 is the
   kernel process (a static `g_kernel_process` so host tests that
   have no scheduler state still get a valid process).
2. `struct nx_task` gains a `struct nx_process *process` field.
   `nx_task_create` inherits from the caller's current process
   (falling back to `&g_kernel_process`).  Bootstrap sets
   `g_idle_task.process = &g_kernel_process` in `sched_start`.
3. `framework/syscall.c`:
   - `nx_syscall_current_table()` returns `&nx_process_current()
     ->handles` (was: a single static `g_kernel_handles`).
   - `nx_syscall_reset_for_test()` resets the current process's
     handles instead of the retired global.
   - `NX_SYS_EXIT = 11` — sets current process's state/exit_code
     and parks in `wfe`; ktest still dequeues externally.
4. Tests: host `process_test.c` covers create/destroy/lookup + two
   processes having independent handle tables; kernel
   `ktest_process.c` covers the bootstrap-provisioned kernel
   process at pid 0 + EL0 program calling `NX_SYS_EXIT`.

**Exit criteria:** Every task has a `process`; two processes'
handle tables are independent; an EL0 program can call `sys_exit`
and the kernel records the code.

#### Slice 7.2 — Per-process address space (TTBR0 per process)

Each process gets its own TTBR0 root; context switches flip TTBR0.
The full "kernel moves to TTBR1 high half" DESIGN goal is deferred
— keeping every per-process TTBR0 a full copy of the kernel
identity map is simpler for v1 and still delivers the core
isolation: two processes can hold different physical bytes at the
same VA, and cross-process context switches transparently flip the
mapping.

1. `core/mmu/mmu.{h,c}` grows:
   - `mmu_create_address_space()` — allocates a fresh L1 + L2_ram
     page (via `malloc`/kheap→PMM so both are 4 KiB-aligned) plus a
     2 MiB user-window backing chunk (malloced with a 2 MiB
     alignment buffer).  L1 is a copy-of-kernel: `[0]` shares the
     kernel's static `l2_mmio_table` (read-only reference), `[1]`
     points at the new L2_ram.  L2_ram is a byte-copy of
     `l2_ram_table` except slot 64 (the user window) is rewritten
     to `user_block(user_backing_pa)`.  Returns the L1 physical
     address.
   - `mmu_destroy_address_space(root)` — looks up the bookkeeping
     entry and frees the L1, L2_ram, and user-window backing.
   - `mmu_switch_address_space(root)` — `msr ttbr0_el1; isb; tlbi
     vmalle1; dsb ish; isb`.
   - `mmu_kernel_address_space()` — returns `&l1_table` (the
     static kernel root).
   - Host builds stub the MMU calls: `mmu_create_address_space`
     returns 0 (no MMU), `mmu_switch_address_space` just records
     the last-passed value for host tests to observe via
     `mmu_current_address_space_for_test`.
2. `struct nx_process` gains a `uint64_t ttbr0_root`.
   `nx_process_create` calls `mmu_create_address_space` on kernel,
   rolls back (remove-from-table + free) on failure; host keeps
   `ttbr0_root = 0`.  `nx_process_destroy` calls the free
   counterpart.
3. Bootstrap sets `g_kernel_process.ttbr0_root =
   mmu_kernel_address_space()` in `nx_framework_bootstrap` (runs
   after `mmu_init`).  Host leaves it 0.
4. `sched_check_resched` in `core/sched/sched.c` reads
   `curr->process` and `next->process`; if their `ttbr0_root`
   values differ (and neither is 0), calls
   `mmu_switch_address_space(next->process->ttbr0_root)` BEFORE
   `cpu_switch_to`.  Done in C (not asm in `context.S`) because
   there's only one call site and the flip must happen with
   `next`'s identity-map already live when the switch_to's own
   kernel instructions fetch.
5. Kernel tests (`ktest_process.c`):
   - `process_create_allocates_fresh_ttbr0_root` — two processes
     get distinct roots, both ≠ the kernel root.
   - `process_two_address_spaces_hold_different_bytes_at_same_va`
     — write 'A' into process A's window, 'B' into B's, flip
     TTBR0 back and forth, both bytes persist.
   - `process_context_switch_flips_ttbr0` — two kthreads in
     different processes each read `user_va[0]`; the scheduler-
     flip path hands each its own byte.

**Out of scope** (deferred): kernel moves to TTBR1 high half
(substantial linker + boot rework — the v1 per-TTBR0 design with
kernel copies in each root already gives the isolation guarantee
without it); migrating the existing EL0 ktests (channel / file /
readdir) to their own processes (7.3 handles this when the ELF
loader replaces the baked-in EL0 programs).

**Exit criteria:** Two kthreads running in different processes
observe different bytes at the same VA; kernel code stays
reachable across TTBR0 flips; all existing tests continue to
pass.

#### Slice 7.3 — ELF loader (in-memory blob)

Introduce the ELF64 parser + PT_LOAD loader so EL0 programs can
live as their own build-system artefacts instead of being baked
raw into kernel-test.bin's `.rodata`.  Slice 7.3 operates on an
in-memory blob; reading from ramfs / vfs is deferred to 7.3.5 or
7.4 (where fork/exec needs a path-based loader anyway).

1. `framework/elf.{h,c}` — ELF64 validator + PT_LOAD walker.
   - `nx_elf_parse(blob, len, *out)` validates magic / class /
     encoding / type (ET_EXEC) / machine (EM_AARCH64), bounds-
     checks the program-header table, and reports entry point +
     PT_LOAD count.
   - `nx_elf_segment(blob, len, idx, *out)` returns one PT_LOAD's
     shape (file_offset, vaddr, file_size, mem_size, flags).
     Indexing is dense — non-LOAD phdrs are skipped, so `idx` runs
     `0..segment_count`.
   - `nx_elf_load_into_process(target, blob, len, *out_entry)`
     copies every PT_LOAD into the target's user-window backing,
     zeros `mem_size - file_size` (BSS), emits `dsb ish ; ic
     iallu ; dsb ish ; isb` to flush I-cache so the new code page
     is fetched fresh.  No TTBR0 flip during the copy: writes go
     via the kernel-visible alias returned by
     `mmu_address_space_user_backing(target->ttbr0_root)`.  Host
     stub validates the header + reflects the entry point.
2. `core/mmu/mmu.{h,c}` gains `mmu_address_space_user_backing
   (root)` — returns the kernel-accessible pointer to the
   process's 2 MiB user backing (identity-mapped, VA==PA).  NULL
   for the kernel root or an unknown root.
3. `test/kernel/init_prog.S` — tiny standalone EL0 program:
   `NX_SYS_DEBUG_WRITE("[el0-elf-ok]", 13)` + `wfe` park.
4. `test/kernel/init_prog.ld` — linker script placing `.text` +
   `.rodata` at `0x48000000` (the user-window base).
5. `test/kernel/init_prog_blob.S` — `.incbin`s `init_prog.elf`
   into `.rodata` flanked by `__init_prog_blob_{start,end}`.
6. Top Makefile: rules for `test/kernel/init_prog.o → .elf`
   (linked standalone with `init_prog.ld`) and an explicit prereq
   on `init_prog_blob.o` so the blob gets re-embedded when the
   source ELF changes.  `KTEST_S += init_prog_blob.S` pulls the
   blob into `kernel-test.bin`.
7. `test/kernel/ktest_elf.c` — three tests:
   `elf_parse_header_reports_entry_and_one_pt_load` (parser
   summary), `elf_parse_rejects_malformed_magic` (negative
   case), `elf_load_and_drop_to_el0_runs_marker_syscall`
   (full round-trip: create process, load ELF, spawn kthread,
   reassign its process, yield until `debug_write_calls > 0`,
   dequeue + restore kernel TTBR0).  Live ktest log shows
   `[el0-elf-ok]` alongside the slice-5.5/5.6/6.x/7.2 markers.
8. `test/host/elf_test.c` — 7 parser tests against hand-built
   synthetic blobs (a small builder constructs header + phdrs
   + PT_LOAD bytes in a static buffer).  Covers the happy path,
   non-LOAD phdr filtering, dense-index iteration, bad magic,
   wrong class/encoding, wrong type/machine, and truncated
   blobs.

**Out of scope** (deferred): reading the ELF from a ramfs file
through vfs_simple (ramfs's current 256-byte file cap is too
small to hold a minimal static ELF — raising it lands with the
first consumer); `nx_process_create_from_elf(path)`; migrating
the baked-in EL0 programs (channel/file/readdir) to the loader —
those tests still work as-is, and the migration pays for itself
when fork/exec arrive in 7.4.

**Exit criteria:** A standalone AArch64 ELF binary, built as its
own artefact, loads into a fresh process and runs to
`NX_SYS_DEBUG_WRITE` under EL0.  `[el0-elf-ok]` appears in the
ktest log.

#### Slice 7.4 — `fork` + `exec` + `wait` + posix_shim

Map POSIX onto nonux.  Further sub-sliced because fork's trap-
frame replay deserves isolated verification before exec's
address-space reset layers on top.

**7.4a — `NX_SYS_FORK`** (this sub-slice, complete):

1. `core/mmu/mmu.{h,c}` grows `mmu_copy_user_backing(src, dst)`
   — byte-copies the 2 MiB user window between two address
   spaces with the same I-cache-flush discipline the ELF loader
   uses (`dsb ish ; ic iallu ; dsb ish ; isb`).
2. `nx_process_fork(parent)` in `framework/process.{h,c}` —
   `nx_process_create` allocates a fresh address space; then
   copies the parent's user backing in.  Handle table left
   empty in v1 (handle duplication lands with 7.4c).
3. `core/cpu/vectors.S` gains `nx_task_fork_child_entry`: a
   new label that does RESTORE_TRAPFRAME + eret, reusing the
   same macro as `_sync_stub`'s tail.  When the scheduler
   first picks a fork-child task, `cpu_switch_to` `ret`s to
   this label — it unpacks the child's pre-populated trap
   frame and erets straight to EL0.
4. `core/sched/task.{h,c}` grows `nx_task_create_forked(name,
   parent_tf)` — allocates a new kstack, byte-copies the
   parent's trap frame to its top, rewrites the copy's `x[0]`
   to 0, sets `cpu_ctx.sp = frame_addr` and `cpu_ctx.x30 =
   &nx_task_fork_child_entry`.  Caller sets `task->process`
   before enqueueing.
5. `framework/syscall.c`:
   - Per-CPU stash `g_current_tf` published via
     `nx_syscall_current_tf()`, set by `nx_syscall_dispatch`
     around each syscall body.  Needed because `sys_fork`
     needs the parent's trap frame but per-syscall handlers
     don't receive it as an arg.
   - `sys_fork()` calls `nx_process_fork`, builds the child
     task via `nx_task_create_forked`, enqueues it, returns
     the child's pid for the parent's `x[0]`.
6. `NX_SYS_FORK = 12` added to the enum.
7. `test/kernel/user_prog_fork.S` — EL0 program that forks,
   branches on `x0`, emits `[fork-parent]` or `[fork-child]`
   to UART.  `test/kernel/ktest_fork.c` asserts the
   debug_write counter reaches 2 and both markers appear in
   the live log — proves the child's trap-frame replay lands
   at the same EL0 instruction with `x0 = 0`.

**7.4b — `NX_SYS_WAIT`** (complete).

1. `NX_SYS_WAIT = 13` added to the enum.
2. `sys_wait(pid, user_status)` in `framework/syscall.c`:
   - `nx_process_lookup_by_pid(pid)` — NX_ENOENT on unknown /
     kernel process.
   - NX_EINVAL if the target is the caller's own process.
   - Loop `while (target->state != EXITED) nx_task_yield();`.
   - `copy_to_user(user_status, &target->exit_code, 4)` if the
     user pointer is non-NULL (bad pointer is non-fatal — we
     still return the pid so the caller learns the target
     exited).
   - Return pid.
3. `test/kernel/user_prog_wait.S` — fork + child exit(42) +
   parent wait + compare to 42.  Emits `[wait-parent]`,
   `[wait-child]`, and (when wait returns 42 via `copy_to_user`)
   `[wait-ok]`.
4. `test/kernel/ktest_wait.c` — polls for three debug_writes +
   confirms the child's process struct is EXITED with
   exit_code == 42.  Deliberately skips `nx_process_reset_for_
   test` because earlier ktests (e.g., `ktest_fork`) strand
   tasks in `wfe`; wiping the process table would leave
   dangling `task->process` pointers and crash
   `sched_check_resched`'s TTBR0 flip on next pick.
5. **IRQ-enable fix in `nx_process_exit`.**  Discovered during
   this slice: hardware masks DAIF.I on exception entry, so
   `sys_exit`'s `wfe` park can't wake on timer ticks — the CPU
   resumes from wfe but the masked interrupt isn't delivered,
   leaving the outer `b 1b` to re-enter wfe forever.  Unmasked
   by `msr daifclr, #2` before the park loop.  Without this,
   a waiting parent's yield loop never got its chance to see
   EXITED.

**Exit criteria:** EL0 program's parent observes the child's
exit code after `NX_SYS_WAIT`; `[wait-ok]` appears in the ktest
log.  Closes the fork/wait pair sufficient for many shell-style
use cases even without exec.

**7.4c — `NX_SYS_EXEC`** (complete).

1. `NX_SYS_EXEC = 14` added to the enum.
2. `sys_exec(path)` in `framework/syscall.c`:
   - `copy_path_from_user` (128-byte cap via `NX_PATH_MAX`).
   - Resolve the `vfs` slot; `vops->open(path, READ)`.
   - Loop `vops->read` up to `SYS_EXEC_MAX_FILE = 8192` bytes
     into a freshly-`malloc`'d kernel buffer.  Close file.
   - `nx_elf_parse` for validation (error returns the parser's
     `NX_E*` after freeing the buffer).
   - `mmu_create_address_space()` → new_root.  Swap
     `current->ttbr0_root = new_root` so
     `nx_elf_load_into_process`'s calls to
     `mmu_address_space_user_backing` resolve the new backing.
     Rollback path restores old_root on load failure.
   - `nx_elf_load_into_process(current, buf, len, &entry)`.
   - Rewrite the current trap frame via
     `nx_syscall_current_tf()`: zero every x-reg, set `pc =
     entry`, `sp_el0 = top-of-user-window`, `pstate = 0`.
   - `mmu_switch_address_space(new_root)` +
     `mmu_destroy_address_space(old_root)`.
   - Return NX_OK.  Dispatcher writes `tf->x[0] = 0`; then
     `RESTORE_TRAPFRAME + eret` delivers EL0 at the ELF's
     entry with zeroed registers + fresh user stack.
3. `nx_syscall_current_tf()` return type is now `struct
   trap_frame *` (was const) — sys_exec needs to write the
   saved frame.
4. Absorbed deferred **slice 7.3.5**:
   - `components/ramfs/ramfs.c`: `RAMFS_FILE_CAP` 256 → 4096.
   - `Makefile`: `init_prog.elf` link rule adds `-n`
     (`--nmagic`) so the phdr table + LOAD segment pack
     directly after the header — blob shrinks ~66 KiB → ~800
     bytes.
   - `test/kernel/init_prog.S`: appends `NX_SYS_EXIT(17)`
     after the `[el0-elf-ok]` write so the exec'd child
     signals a distinguishable exit code.
5. Scheduler quantum lowered: `SCHED_RR_DEFAULT_QUANTUM_TICKS`
   10 → 2 (1 s → 200 ms per slice).  Needed because each
   stranded EL0-`wfe` task from a prior slice-7.4 ktest burns
   a full quantum before the scheduler rotates past it; at
   1 s × 4 stranded tasks, the exec test blew past the
   15-second QEMU test timeout.  200 ms still leaves the
   slice-4.4 preemption demo visibly interleaving two kthreads.
6. New test-only helper
   `sched_rr_purge_user_tasks(self, keep)` in
   `components/sched_rr/sched_rr.c`: walks the runqueue and
   dequeues every task whose `task->process != &g_kernel_
   process` and whose task != `keep`.  Storage is not freed —
   only unlinked from the runqueue.  `ktest_exec` calls it
   first thing so the fork-child + wait-child leftovers can't
   starve its own kthread.  Declared directly in the test
   file (not in a public sched_rr header) because it's a
   policy-specific test escape hatch; the proper fix (reap
   child when `wait()` collects status) lands naturally when
   slice 7.5/7.6 need more concurrent processes.
7. `test/kernel/user_prog_exec.S` — fork + child
   `exec("/init")` + parent wait + compare status to 17 +
   emit `[exec-ok]`.
8. `test/kernel/ktest_exec.c` — purges stranded tasks, seeds
   `/init` in ramfs by writing the `init_prog.elf` blob (via
   `init_prog_blob.S`'s `.incbin`) through vfs_simple's ops
   directly (no syscalls — those need a live process context
   we don't have at the test body), spawns the exec parent
   kthread, polls for three debug_writes, independently
   verifies a process with exit_code == 17 exists in the
   process table.

**Exit criteria:** EL0 program's child `exec`s the init_prog
ELF, init_prog emits `[el0-elf-ok]` + exits 17, parent's wait
sees exit_code == 17 and emits `[exec-ok]`.  All three markers
(`[exec-parent]`, `[el0-elf-ok]`, `[exec-ok]`) appear in the
ktest log.  Closes the fork/exec/wait triad — a shell written
on top of our `NX_SYS_*` surface can now spawn a child from an
ELF path, wait for it, and read its exit code.

**7.4d — `components/posix_shim/`** (complete).

1. `components/posix_shim/posix.h` — header-only.  Every entry
   is a `static inline` function that emits `svc #0` with the
   right `NX_SYS_*` number in `x8` and arguments in `x0..x5`,
   matching the calling convention in `framework/syscall.h`.
   Exposed: `nx_posix_debug_write / exit / fork / execve /
   waitpid / open / close / read / write / lseek`.  No `.c`
   file; the header is the whole implementation.  `#include`d
   directly by EL0 C programs — posix_shim stays off
   `kernel.bin`'s link line (it's a userspace library, not a
   slot implementation).
2. `components/posix_shim/manifest.json` — minimal (name +
   version; no iface, no deps, no worker threads).  Metadata
   only; gen-config doesn't emit any bindings because posix_
   shim isn't in `kernel.json`'s components map.
3. `components/posix_shim/README.md` — documents the API
   surface, POSIX mapping gaps (no errno, no WIFEXITED, no
   signals), and why it's header-only (different link domain
   from kernel.bin).
4. Demo: `test/kernel/posix_prog.c` — C-compiled EL0 ELF that
   forks + child exit(23) + parent waitpid + compare status +
   emit three markers.  Uses `_start` as the ELF entry (no
   crt0 needed since the demo doesn't use argc/argv/envp;
   drop_to_el0 sets sp_el0 before eret).  Built with
   `-ffreestanding -nostdlib -Wall -Wextra -Werror -O2
   -mno-outline-atomics -mgeneral-regs-only
   -fno-stack-protector -fno-pic -I.`  against the shared
   `test/kernel/init_prog.ld` linker script (VA 0x48000000,
   the user-window base).  The resulting `.elf` is ~912 bytes
   and embedded into `kernel-test.bin` via
   `posix_prog_blob.S`'s `.incbin`.
5. `test/kernel/ktest_posix.c` — loads the embedded blob via
   the slice 7.3 ELF loader (`nx_elf_load_into_process`),
   spawns a kthread pinned to the target process, drops to
   EL0 at the ELF entry, yields until the debug_write counter
   reaches 3 (three markers: `[posix-parent]`,
   `[posix-child]`, `[posix-ok]`), independently verifies a
   process with `exit_code == 23` exists in the process
   table.  Same `sched_rr_purge_user_tasks` drain at start
   as `ktest_exec` to keep the 15 s QEMU budget.

**Exit criteria:** EL0 C program compiles, links at the
user-window VA, and executes fork + waitpid end-to-end.  All
three markers (`[posix-parent]`, `[posix-child]`,
`[posix-ok]`) appear in the ktest log + an EXITED process
with exit_code == 23 exists afterwards.  Proves the C-level
ABI surface produces the right SVCs (wrong syscall number or
register mangling breaks the markers or the exit code,
failing the test loudly).  Closes slice 7.4 — a shell written
on top of posix_shim could now spawn / exec / wait on nonux
without hand-rolling assembly.

**Exit criteria (7.4a, complete):** EL0 program calls fork;
both parent and child emit distinct UART markers; the child's
trap frame correctly returns `x0 = 0` at the instruction after
the fork SVC.

#### Slice 7.5 — Pipes + polled signals

Two more POSIX primitives built from existing framework pieces.
**Complete.**

1. `NX_SYS_PIPE = 15` — `sys_pipe(int fds[2])`:
   - Calls the slice-5.6 `nx_channel_create` to allocate an
     endpoint pair.
   - Allocates two `HANDLE_CHANNEL` handles with asymmetric
     rights — `NX_RIGHT_READ` on `fds[0]` (read side), `NX_RIGHT_
     WRITE` on `fds[1]` (write side).  No `NX_RIGHT_TRANSFER`
     in v1 (pipes don't yet cross process boundaries via IPC
     cap transfer).
   - `copy_to_user`s both handles into the user-owned
     `fds[2]` array.  Handle allocation failure cleanly unwinds
     the partial state (`nx_handle_close` + `nx_channel_endpoint_
     close` mirror calls) — same pattern as `sys_channel_create`.
2. `sys_read` / `sys_write` are now handle-type-polymorphic —
   `HANDLE_CHANNEL` dispatches through `nx_channel_recv` /
   `nx_channel_send` (256-byte message cap), `HANDLE_FILE` keeps
   the vfs path.  The lookup path is factored so rights + type
   are checked once at the top; the CHANNEL/FILE branches differ
   only in the object-dispatch tail.
3. `NX_SYS_SIGNAL = 16` — `sys_signal(pid, signo)`:
   - `nx_process_lookup_by_pid(pid)` → `NX_ENOENT` if unknown
     or the kernel process.
   - Unsupported signo → `NX_EINVAL`.  v1 supports only
     SIGTERM (15) and SIGKILL (9).
   - `__atomic_fetch_or` sets the matching bit in the target's
     new `pending_signals` field (`_Atomic uint32_t` on
     `struct nx_process`).
   - Does NOT wake the target — delivery is polled.
4. `struct nx_process` grows an `_Atomic uint32_t
   pending_signals` field.  Static init zeroes it (like every
   other field); fork inherits via `nx_process_create` which
   `memset`s the whole struct to 0 first.
5. Signal poll in `sched_check_resched`:
   ```c
   if (curr->process &&
       curr->process->state == NX_PROCESS_STATE_ACTIVE) {
       uint32_t pending = __atomic_load_n(
           &curr->process->pending_signals, __ATOMIC_ACQUIRE);
       if (pending & (1u << NX_SIGKILL))
           nx_process_exit(128 + NX_SIGKILL);
       if (pending & (1u << NX_SIGTERM))
           nx_process_exit(128 + NX_SIGTERM);
   }
   ```
   Runs at every reschedule point — timer preemption, voluntary
   yield, syscall-return.  The ACTIVE guard is load-bearing: if
   a process is already EXITED, its task is stuck in
   `nx_process_exit`'s `wfe` loop and re-entering
   `nx_process_exit` would starve any waiting parent because
   we'd never return to the `cpu_switch_to` below.  Matches
   POSIX's `128 + signo` exit-status convention — a parent's
   `waitpid` sees the signal death through the same channel
   as a voluntary `exit(n)`.
6. posix_shim gains `nx_posix_pipe(int fds[2])` + `nx_posix_kill
   (pid, signo)` + `NX_POSIX_SIGKILL / SIGTERM` constants
   matching POSIX's numeric values.
7. Demos: `test/kernel/posix_pipe_prog.c` (single-process
   write→read roundtrip through the shim's pipe + polymorphic
   read/write; proves the dispatch plumbing), and `test/kernel/
   posix_signal_prog.c` (parent fork + child busy-loops + parent
   kill(SIGTERM) + waitpid + asserts status == `128 + 15 =
   143`; `[signal-parent][signal-child][signal-ok]`).  The
   child's busy-loop without syscalls relies on timer-tick
   preemption eventually routing through `sched_check_resched`,
   at which point my signal poll catches the pending bit.
8. Cross-process pipe inheritance is **deferred**.  Slice 7.4a
   left fork's child handle table empty; the "pipe + fork +
   exec" POSIX pattern is blocked on handle duplication,
   which naturally lands early inside slice 7.6 when busybox
   exercises it.  The pipe demo is therefore single-process —
   it still exercises the full dispatch path, but doesn't
   prove cross-process semantics.
9. Drive-by fix: `nx_ipc_reset` used to walk each per-slot
   inbox's messages and clear their `_next` fields for
   tidiness.  Messages are caller-owned stack storage, and
   once a host test's stack frame popped, subsequent reuse
   could overwrite `_next` with garbage — the next reset
   segfaulted.  Now we just free the queue-entry bookkeeping
   and leave message storage alone (caller owns its own
   lifetime).

**Exit criteria:** `[pipe-ok]` appears after the pipe roundtrip;
`[signal-ok]` appears after a SIGTERM delivery + waitpid
observation of `128 + SIGTERM`; both round-trips are
independently verified by `ktest_posix_pipe.c` /
`ktest_posix_signal.c` asserting the target processes' exit
codes directly.  `echo hello | cat` end-to-end is deferred to
slice 7.6 (needs handle inheritance + busybox).

#### Slice 7.6 — Busybox cross-compile + initramfs

External toolchain integration.  Sub-sliced 7.6a–d because the
work spans multiple sessions in practice.

**7.6a — Fork handle-table inheritance for HANDLE_CHANNEL**
(complete).  The pipe-based shell pattern (`pipe → fork → child
reads, parent writes`) was blocked because slice 7.4a left the
forked child's handle table empty.  This sub-slice unblocks it
for channel handles (file handles defer to a later slice — see
below).

1. **Per-endpoint refcount in `framework/channel.{h,c}`.**
   Replaced `struct nx_channel`'s shared `_Atomic int refcount`
   with `_Atomic int handle_refs` *per endpoint* on
   `struct nx_channel_endpoint`.  `nx_channel_create`
   initialises each to 1 (one handle per endpoint at create
   time).  Lifecycle:
   - `nx_channel_endpoint_retain(e)` bumps `e->handle_refs`.
   - `nx_channel_endpoint_close(e)` decrements; only when refs
     hit 0 does the endpoint set `closed = true`.  When BOTH
     endpoints are closed (refs = 0 on each), the whole
     `struct nx_channel` allocation is freed.
   - The free path re-loads the peer's `handle_refs` rather
     than caching anything because two final closers on
     opposite endpoints can race; only one sees both refs at
     zero, and that one frees.
2. **Handle-table inheritance in `sys_fork`.**  Walks the
   parent's handle table (slot 0..NX_HANDLE_TABLE_CAPACITY-1).
   For each entry of type HANDLE_CHANNEL with a non-NULL
   object: call `nx_channel_endpoint_retain(object)` then
   `nx_handle_alloc(child_table, type, rights, object,
   &dup_h)`.  Same rights as the parent; same object pointer
   (a fresh handle slot in the child's table aliases the same
   endpoint).  On allocation failure (child table exhausted)
   the just-issued retain is released and `sys_fork` returns
   `NX_ENOMEM`; `nx_process_destroy` cleans up partial child
   state.
3. **HANDLE_FILE inheritance is intentionally NOT done.**  The
   per-handle file object pointer in vfs_simple owns the
   per-open cursor (current offset, etc.); a naive alias
   would make `lseek` from one process visible to the other.
   A real dup needs vfs_simple to grow a `dup` op (clone the
   per-open with its own cursor) or for the framework to
   track cursors outside the driver — both bigger than this
   sub-slice.  Inherit-on-fork for files lands when busybox
   actually exercises the pattern, likely inside slice 7.6d.
4. **Test coverage.**  `test/kernel/posix_pipe_xproc_prog.c` —
   parent creates pipe + forks; child closes write side,
   reads "hello\n" from fds[0], emits `[xpipe-child]`,
   exits 41; parent closes read side, emits
   `[xpipe-parent]`, writes "hello\n" to fds[1], closes
   write side, waits, asserts status == 41, emits
   `[xpipe-ok]`.  `ktest_posix_pipe_xproc.c` loads the demo
   via the slice 7.3 ELF loader, verifies all three markers
   appear + an EXITED process with exit_code == 41.  +1
   kernel test.

**Exit criteria (7.6a):** parent + child do the canonical
close-then-read/write pipe pattern end-to-end; the
per-endpoint refcount keeps each side live until the matching
party voluntarily closes.

**7.6b — Initramfs format + ramfs slurp at bootstrap**
(complete).  cpio-newc on disk; embedded into `kernel-test.bin`
via `.incbin`; ramfs walks it at `init()` and seeds the file
table.

1. **`tools/pack-initramfs.py`.**  ~120 lines of stdlib-only
   Python.  Takes a `path:archive-name` entry list on argv and
   emits a binary cpio-newc archive — 110-byte ASCII-hex header
   per entry, NUL-terminated name, 4-byte-aligned data, plus
   a `TRAILER!!!` sentinel record.  System `cpio -t` round-trips
   the output, which is a useful sanity check for format
   correctness.  c_mtime fixed to 0 for reproducibility (the
   build is deterministic — same inputs, byte-identical cpio).
2. **`test/kernel/initramfs_blob.S`.**  `.incbin
   "test/kernel/initramfs.cpio"` between `__ramfs_initramfs_
   blob_start` / `_end` labels.  Same shape as
   `init_prog_blob.S` — `.balign 16` to keep alignment
   consistent with adjacent rodata.
3. **Makefile rules.**  `test/kernel/initramfs.cpio` depends on
   the packer + every entry source (so editing
   `banner.txt` re-runs the packer; updating
   `init_prog.elf` triggers a fresh archive).  Manifest
   currently:
   - `/init` ← `init_prog.elf` (lets `ktest_exec` and
     friends rely on a pre-seeded ELF instead of hand-
     creating one via vfs syscalls).
   - `/banner` ← `banner.txt` ("hello from initramfs\n").
     Renamed from `/hello` after a registration-order
     race with `ktest_vfs` (which creates its own
     `/hello`) — keep the namespaces disjoint.
4. **cpio-newc parser in `components/ramfs/ramfs.c`.**  The
   slurp function `ramfs_slurp_initramfs(s, blob, total)` is
   bound by `RAMFS_FILE_CAP` (4096) and `RAMFS_MAX_FILES` (8)
   in v1; entries that exceed either cap are silently skipped.
   Regular files only — mode bits `0x8000`; dirs (`0x4000`),
   symlinks (`0x2000`), and special files are dropped because
   ramfs's flat namespace has no representation for them.
   Stop conditions: blob exhausted, magic mismatch, malformed
   field, or the `TRAILER!!!` sentinel name.
5. **Weak symbols for production safety.**  `extern char
   __ramfs_initramfs_blob_start[] __attribute__((weak));` lets
   `kernel.bin` link cleanly without `initramfs_blob.S` —
   the weak fallbacks resolve to 0, the `init` body's
   `blob_end > blob_start` guard short-circuits, and the slurp
   does nothing.  Test build provides the real symbols via the
   embedded blob.  Footprint on shipping kernels: zero (the
   slurp function itself remains in the binary, but it never
   runs without the blob).
6. **Drive-by bug.**  gcc 12+'s `-Werror=array-compare` rejects
   the natural `__sym_end > __sym_start` comparison when both
   sides are array-typed externs.  Fixed with the
   `&__sym[0] > &__other_sym[0]` form, which the compiler
   accepts because the operands are explicit pointers, not
   arrays decaying to pointers.  Same address comparison the
   linker resolves; same behaviour at runtime.
7. **`ktest_exec` slimming.**  Dropped `seed_init_file` — its
   slice-7.4c hand-rolled write loop is now redundant since
   `/init` is seeded at boot.  Test stays otherwise unchanged.
8. **New `ktest_initramfs.c`.**  Two checks: `/banner` reads
   back as the expected 21-byte ASCII string; `/init`'s first
   four bytes are the ELF magic.  Both go through vfs_simple
   (the same path real syscalls take), proving the seeded
   files are reachable through the live filesystem.

**Exit criteria:** `make test-kernel` runs `ktest_initramfs`
green; `ktest_exec` continues to pass without its bespoke
seeding code.  Production `make kernel.bin` doesn't pull in
the cpio (only `kernel-test.bin` links `initramfs_blob.S`).

**7.6c — musl → NX_SYS_* adapter** (sub-sliced 7.6c.0 done; 7.6c.1
pending).

**7.6c.0 — EL0 C-runtime bootstrap** (complete).  Lands the crt0,
linker conventions, and tiny libc surface that musl will replace
piece by piece in 7.6c.1.  Splitting them lets the runtime
plumbing stabilise on its own — once musl arrives, the diff is
"swap our crt0 + libc-stubs for musl's, change linker flags".

1. **`components/posix_shim/crt0.S`.**  `_start` is the ELF entry
   symbol; linked first so it lands at the user-window base.
   - Round-trips `sp` through `x9` to apply a defensive 16-byte
     align (`mov x9, sp; and x9, x9, #~15; mov sp, x9`).  ARM64
     has no direct `bic sp, sp, #imm` form — caught at assembly
     time, fixed with the canonical idiom.
   - Synthesises `argc=1, argv = { "nonux", NULL }, envp = NULL`
     in `.rodata`, loads them into x0/x1/x2 via `adrp` +
     `:lo12:` PIC arithmetic.  sys_exec doesn't push real
     argv/envp on the user stack today; that lands when slice
     7.6d's busybox actually parses them.  Programs that don't
     care `(void)argc; (void)argv;` in their main.
   - `bl main` — main is provided by the user object; missing
     symbol fails the link cleanly.
   - main's return value (in x0) is dispatched to
     `NX_SYS_EXIT = 11` directly via `mov x8, #11; svc #0` —
     avoids a C↔asm hop right at program shutdown.  The
     wfe-loop fall-through is unreachable in practice but
     keeps the assembler from elide the trailing
     instructions.
2. **posix_shim header gains four libc helpers.**
   `nx_strlen / strcmp / memcpy / memset` as `static inline`.
   Just enough that EL0 C programs aren't rolling the basics by
   hand; small enough that musl's eventual replacements are a
   `static inline → extern` swap with no API churn.
3. **Demo `posix_main_prog.c`** — `int main(int argc, char
   **argv, char **envp)` form.  Exercises every libc helper
   (memcpy + strlen + strcmp), debug_writes a `[main-ok]`
   marker, asserts `argv[0] == "nonux"`, returns `argc + 46 =
   47`.  The crt0 forwards 47 to nx_posix_exit, and the ktest
   pins both the marker AND the exit code.
4. **Build wiring.**  `components/posix_shim/crt0.o` is its
   own translation unit (no `.c` companion).  `posix_main_prog.elf`
   links `crt0.o` first so `_start` is at offset 0 of `.text`,
   then `posix_main_prog.o` whose `main` symbol satisfies
   crt0's `bl main` reference.  Reuses `init_prog.ld` for the
   user-window VA — no new linker script.
5. **`ktest_posix_main.c`** — same shape as the other slice-7.x
   ELF-loader-driven tests.  Validates: marker fires (proves
   crt0 → main path), host process exits with code 47 (proves
   crt0 → main return → nx_posix_exit path).

**Exit criteria (7.6c.0):** EL0 C programs can use `int main()`
form via posix_shim/crt0.  Demo asserts argc/argv setup +
return-value flow + libc-helper compilation.

**7.6c.1 — `libnxlibc.a`: minimal POSIX-named C surface as a
static archive** (complete).  Offline-clean placeholder for
musl; same link-line shape (`-lnxlibc` vs `-lc`) so slice 7.6c.2
can swap the archive without touching EL0 C source.

1. **`components/posix_shim/nxlibc.h`.**  POSIX-named surface:
   `write / read / _exit / fork / execve / waitpid / kill /
   open / close / lseek / pipe` + four libc helpers (`strlen /
   strcmp / memcpy / memset`).  Uses nxlibc-namespaced function
   names (`nxlibc_write`, etc.) + macro constants
   (`NXLIBC_STDOUT_FILENO`, etc.) — slice 7.6c.2 will switch the
   demo to pure `write` / `STDOUT_FILENO` when musl lands the
   real POSIX symbol names.
2. **`components/posix_shim/nxlibc.c`.**  Every function is a
   thin shim over the matching `nx_posix_*` / `nx_*` static
   inline from `posix.h`.  Two stdio-shaped specialisations:
   - `nxlibc_write(fd, buf, n)` routes fd 1 or 2 through
     `NX_SYS_DEBUG_WRITE` (UART); fd > 2 goes through
     `NX_SYS_WRITE` (vfs_simple or CHANNEL via the slice-7.5
     type-polymorphic handler).
   - `nxlibc_read(STDIN_FILENO, ...)` returns 0 (EOF) — no
     console input in v1.
   - `nxlibc_close(fd)` is a no-op for fds 0..2 (magic
     stdio; no handle-table entry to release).
3. **`libnxlibc.a` packaging.**  The top-level Makefile adds
   `AR := $(CROSS)ar` and a rule that runs `ar rcs
   components/posix_shim/libnxlibc.a crt0.o nxlibc.o`.  The
   archive carries exactly two object files — crt0 (ELF entry)
   and nxlibc (POSIX symbols).
4. **Demo `posix_libc_prog.c`** — uses POSIX names only
   (`nxlibc_write`, `NXLIBC_STDOUT_FILENO`, `nxlibc_exit`).
   Linker line: `ld -T init_prog.ld -o posix_libc_prog.elf
   posix_libc_prog.o -Lcomponents/posix_shim -lnxlibc`.  The
   archive pulls in crt0.o (for `_start`) + nxlibc.o
   (for the POSIX symbols) — standard GNU ld archive-search
   behaviour.
5. **Build sizes.**  `libnxlibc.a` is 4.5 KB; the demo
   `posix_libc_prog.elf` is 2.3 KB.  The archive grows
   linearly with every libc function we add; when musl
   replaces it, expect `libc.a` to balloon to ~80 KB.

**Exit criteria (7.6c.1):** POSIX-named `write(1, ...)` +
`_exit(n)` work end-to-end through `libnxlibc.a`; ktest
confirms marker fires + exit code matches.

**7.6c.2 — printf + stdio in libnxlibc** (complete).  Inserted
ahead of the musl pin because printf is a single-session
deliverable that's immediately useful, and a more capable
libnxlibc gives slice 7.6c.3's musl pin a richer baseline to
diff against (or fall back to if musl integration runs into
ABI surprises).

1. **`nxlibc.h` additions:** `nxlibc_putchar / puts / printf /
   vsnprintf / snprintf / atoi`.  printf and friends are
   marked `__attribute__((format(printf, ...)))` so gcc
   typechecks the format strings at every call site.
2. **`nxlibc.c` printf core.**  `nxlibc_vsnprintf` walks the
   format string a char at a time; `%X` lookup table dispatches
   to one of `emit_char` / `emit_str` / `emit_uint` /
   `emit_int`.  Buffer is stack-local in `nxlibc_printf` —
   `NXLIBC_PRINTF_BUF = 256` bytes, same size class as
   `NX_FILE_IO_MAX` / `NX_CHANNEL_MSG_MAX`.  Returns the
   would-have-been length on truncation (matching glibc's
   `snprintf` convention) so callers can detect overflow.
3. **Conversions supported:** `%d / %i / %u / %x / %X / %s
   / %c / %%`.  No width, precision, padding, or floating
   point — those land with the eventual musl pin in slice
   7.6c.3.
4. **`nxlibc_putchar` / `puts`.**  Each routes through
   `nxlibc_write(STDOUT_FILENO, ...)` — same magic-fd path
   as the slice-7.6c.1 `write`.  `puts` writes the body
   followed by a single `'\n'` byte (POSIX semantics).
5. **`nxlibc_atoi`.**  Skip whitespace + optional sign +
   decimal digits.  No errno, no overflow detection — sized
   for argv parsing where the caller has validated input
   shape.
6. **Demo `posix_printf_prog.c`** exercises every supported
   conversion in one program: `[printf-int=42]`,
   `[printf-uint=3735928559]` (= 0xDEADBEEF), `[printf-hex=ff]`,
   `[printf-HEX=CAFE]`, `[printf-str=nonux]`, `[printf-char=Q]`,
   `[printf-pct=%]`, `[printf-multi=1/nonux/ab]`, `[atoi=-23]`,
   `[printf-puts]`.  A regression in any single formatter
   breaks exactly one marker.
7. **`ktest_posix_printf.c`** confirms 10+ debug_writes fired
   (10 printf bodies + the puts trailing newline) + the
   host process EXITED with `exit_code == 37`.

**Exit criteria (7.6c.2):** every supported conversion
formats correctly + writes its output via the stdout-routing
path; demo exits 37; `libnxlibc.a` grows from 4.5 KB to 7.4 KB.

**7.6c.4 — sys_exec argv push** (complete).  Inserted ahead of
the musl pin because the original slice 7.6c.0 crt0 had to
synthesize a fake `argv = { "nonux", NULL }` while sys_exec
ignored the user-supplied argv.  Real argv push closes that
gap — busybox shells in slice 7.6d would have failed
mysteriously without it.

1. **NX_SYS_EXEC ABI extension.**  Was `(const char *path)`;
   now `(const char *path, char *const argv[], char *const
   envp[])`.  envp is reserved (a2) — currently always
   treated as the empty environment.  posix_shim's
   `nx_posix_execve(path, argv, envp)` forwards argv via the
   svc3 register-passing convention.
2. **`sys_exec` argv copy (before the TTBR0 flip).**  Walks
   the user-supplied argv array (in OLD address space): up to
   `SYS_EXEC_ARGV_MAX = 16` entries, total
   `SYS_EXEC_ARGV_BYTES = 1024` bytes of string data.  Each
   slot is a copy_from_user for the pointer + a
   copy_path_from_user for the string body.  Argv strings get
   packed into a stack-local kernel buffer; per-string offsets
   tracked separately.
3. **Synthesised argv = { path, NULL } when user passes NULL.**
   Matches Linux's convention.  Lets older callers that don't
   care about argv keep working.
4. **`sys_exec` argv push (after the AS flip — but still
   accessing the new backing through the kernel-visible
   alias).**  System V layout, low-to-high in the user
   window:
       sp+0:                    argc
       sp+8 .. sp+8*argc:       argv[0 .. argc-1]
       sp+8*(argc+1):           NULL (argv terminator)
       sp+8*(argc+2):           NULL (envp[0] = end-of-envp)
       (high)                   strings: "arg0\0arg1\0..."
   `tf->sp_el0` lands at the argc slot.  16-byte-aligned per
   AAPCS.  Alignment math caught one bug — `fixed_size = 16
   + 8*argc` was off-by-one (should be `8*(3+argc)`); fixed
   when the first run showed argv[0] reading as zeros.
5. **`crt0.S`.**  Reads argc from `[sp]`.  argc > 0 → use
   stack-pushed layout (real argv from sys_exec).  argc == 0
   → fall back to the synthesised default (drop_to_el0
   entries, where the user backing is calloc'd zeroes).
   Both paths converge at `bl main` with x0/x1/x2 set per
   AAPCS.
6. **v1 caps bumped.**  Cumulative test usage of stranded
   processes hit the previous caps:
   - `NX_PROCESS_TABLE_CAPACITY` 16 → 32 in
     `framework/process.c`.
   - `MMU_MAX_ADDRESS_SPACES` 16 → 32 in `core/mmu/mmu.c`.
   - `RAMFS_FILE_CAP` 4096 → 8192 in
     `components/ramfs/ramfs.c` so libnxlibc-linked C
     programs (~4.5 KB) fit at a ramfs path.
7. **Stack-boundary fix.**  Old drop_to_el0 callers passed
   `sp_el0 = (base + size) & ~0xf` — top-of-window
   (exclusive).  crt0's new `ldr x0, [sp]` faulted on that
   boundary.  Bulk-updated 15 ktest_*.c callers via sed to
   `sp_el0 = (base + size - 16) & ~0xf` — leaves 16 bytes of
   in-bounds zeroed memory at the top so `ldr [sp]` reads
   argc as 0 and crt0 falls through to the synthesised
   path.
8. **Demo.**  Two new C-compiled EL0 binaries:
   - `argv_child_prog.c` — exec'd target.  Uses `int main(
     int argc, char **argv, char **envp)`, prints argc + each
     argv slot via printf, validates each slot byte-for-byte
     against `{ "/argv_child", "hello", "world" }`, exits
     `argc + 60 = 63`.  Embedded in initramfs as
     `/argv_child` (alongside `/init` and `/banner`).
   - `argv_parent_prog.c` — drop-to-EL0 entry point.
     fork()s; child execve's `/argv_child` with the explicit
     argv; parent waits + asserts status == 63 + emits
     `[argv-ok]`.
9. **Ktest** — pins three observations: child reaches its
   `[argv-child-ok]` marker (every slot matched), parent
   emits `[argv-ok]` (status == 63 observed), an EXITED
   process with exit_code == 63 exists in the table.  Six
   distinct child sentinel exit codes (81..85) pinpoint
   which slot regressed if anything breaks.

**Exit criteria (7.6c.4):** `nx_posix_execve(path, argv,
envp)` forwards the argv strings end-to-end; the exec'd
child sees real argc/argv via standard `int main(int argc,
char **argv, ...)` form; ktest confirms via byte-for-byte
slot validation + exit-code propagation.

**7.6c.3 — Vendor + cross-build musl** (in progress; sub-sliced
7.6c.3a–c).  Snapshot of musl under `third_party/musl/`,
patched at the syscall site so it emits `NX_SYS_*` numbers
via `svc 0`, then linked into successive demos as kernel-side
scaffolding (AUXV, brk) lands.  Three sub-slices, one session
each — keeps each landing single-shaped and testable.

**7.6c.3a — Vendor + patch syscall layer** (complete).  Bring
the source tree into the repo and make musl's `svc 0` site
land on our numbers.  No EL0 program links against the patched
libc.a yet — the next two sub-slices wire that in as kernel-
side scaffolding (AUXV, brk) lands.

1. **Vendor.**  Extract `musl-1.2.5.tar.gz` (sha256
   `a9a118bbe84d8764da0ea0d28b3ab3fae8477fc7e4085d90102b8596fc7c75e4`,
   official 2024-03-01 release) to `third_party/musl/`.  Stock
   upstream + the two patches below, no submodule (so `make
   test` stays offline-clean).
2. **Patch the C-level dispatcher.**
   `third_party/musl/arch/aarch64/syscall_arch.h`.  Stock musl
   funnels every C-level syscall through `__syscallN(n, args)`
   which puts `n` in `x8` and emits `svc 0`.  Add a
   `static inline long __nx_translate(long n)` switch above
   the macros, mapping the nine "args already match" cases
   below to our NX_SYS_* numbers; default returns -1.  Each
   `__syscallN` checks the translation result and returns
   `-38` (-ENOSYS) on miss, otherwise programs `x8` with the
   nonux number and falls through to the original
   `__asm_syscall` macro.  Compile-time-constant `n` (every
   `__syscall(__NR_foo, ...)` site) const-folds the switch
   under `-O2` to a single `mov`, no runtime overhead.
3. **Patch the asm cancellation-point dispatcher.**
   `third_party/musl/src/thread/aarch64/syscall_cp.s`.  The
   stock asm takes `n` in `x1`, moves into `x8`, and emits
   `svc 0` — bypassing the C-level dispatcher entirely.
   Patch interposes the same translation table inline:
   `cmp x1, #57; b.eq .Lnx_close; ...; movn x0, #37; ret`
   (-ENOSYS), each label jumps into a tail block that
   programs `x8` and falls through to the same `mov x0, x2;
   ...; svc 0`.  `__cp_begin / __cp_end / __cp_cancel` symbols
   are still emitted — `pthread_cancel.c` references them.
4. **Translation table.**  Nine syscalls map cleanly:

   | Linux                    | nonux                     |
   |--------------------------|---------------------------|
   | `__NR_close      (57)`   | `NX_SYS_HANDLE_CLOSE (2)` |
   | `__NR_lseek      (62)`   | `NX_SYS_SEEK         (9)` |
   | `__NR_read       (63)`   | `NX_SYS_READ         (7)` |
   | `__NR_write      (64)`   | `NX_SYS_WRITE        (8)` |
   | `__NR_exit       (93)`   | `NX_SYS_EXIT        (11)` |
   | `__NR_exit_group (94)`   | `NX_SYS_EXIT        (11)` |
   | `__NR_kill      (129)`   | `NX_SYS_SIGNAL      (16)` |
   | `__NR_execve    (221)`   | `NX_SYS_EXEC        (14)` |
   | `__NR_wait4     (260)`   | `NX_SYS_WAIT        (13)` |

   `wait4(pid, status, options, rusage)` and our
   `NX_SYS_WAIT(pid, status)` both put pid in x0 + status in
   x1; the extra args in x2/x3 are ignored.  `execve(path,
   argv, envp)` matches our slice-7.6c.4 ABI exactly.  Other
   syscalls (openat needs dirfd stripping, pipe2 drops flags,
   clone has the SIGCHLD-only fork shape) need argument
   translation and are left unmapped (i.e. -ENOSYS) for now.
5. **Wire the build.**  Top-level Makefile:

   ```make
   MUSL_DIR    := third_party/musl
   MUSL_LIBC   := $(MUSL_DIR)/lib/libc.a
   MUSL_STAMP  := $(MUSL_DIR)/.configured

   $(MUSL_STAMP):
       cd $(MUSL_DIR) && ./configure \
           --target=aarch64-linux-gnu \
           --prefix=/usr/local/musl \
           --disable-shared \
           CC=$(CC) CROSS_COMPILE=$(CROSS) >/dev/null
       @touch $@

   $(MUSL_LIBC): $(MUSL_STAMP) \
                 $(MUSL_DIR)/arch/aarch64/syscall_arch.h \
                 $(MUSL_DIR)/src/thread/aarch64/syscall_cp.s
       $(MAKE) -C $(MUSL_DIR) lib/libc.a

   musl-libc: $(MUSL_LIBC)
   ```

   `make test` deps `musl-libc` so any future patch breakage
   is caught; subsequent runs are no-ops (~1 s stat check).
   `musl-clean` removes the stamp + `obj/` + `lib/` so the
   next `make musl-libc` reconfigures from scratch.
6. **Verify.**  `objdump -d
   third_party/musl/obj/src/exit/_Exit.lo` should show
   `mov x8, #11; svc 0` (Linux exit_group/exit both fold to
   NX_SYS_EXIT).  `objdump -d
   third_party/musl/obj/src/unistd/getpid.lo` should show
   `mov w0, #-38; ret` (Linux 172 not in our table, the
   unreachable svc is dead-code-eliminated).  `make test` →
   403/403, no regressions.

**Gotcha:** musl's incremental rebuild only tracks `.c → .lo`
deps, not arch-header inclusion.  If you patch
`syscall_arch.h` and run an incremental `make lib/libc.a`,
the `.lo` files are stale (mtime predates the header edit)
and the patched translation isn't visible.  Workaround: run
`make musl-clean` for any patch to `syscall_arch.h`.

**Exit criteria (7.6c.3a):** `make musl-libc` produces a
clean `libc.a`; spot-checked codegen confirms the translation
landed; `make test` runs 403/403 with no regressions; no
demos linked against musl yet (that's 7.6c.3b's job).

**7.6c.3b — AUXV push + first demo against musl** (complete).

1. **AUXV push in `sys_exec`.**  Three pairs between envp's
   NULL terminator and the argv strings:
   - `AT_PAGESZ = 4096` (musl's malloc rounds to pagesize).
   - `AT_RANDOM = window_base + at_random_offset` (pointer
     to a 16-byte pad sitting just below the strings;
     `__init_ssp` derefs it for the stack canary).
   - `AT_NULL = 0` terminator.

   The 16-byte pad gets `(seed >> (i*4)) ^ (i*0x37)` per
   byte where `seed = nx_monotonic_raw()` (CNTPCT_EL0).
   Pseudo-random — see Phase-9 follow-up for real entropy.
   `fixed_size` grows from `8 * (3 + argc)` to `8 * (9 +
   argc)` to hold the +6 longs.  `sp_offset` is computed
   from the pad's offset, not the strings', so sp lands
   below pad lands below strings.
2. **`CPACR_EL1.FPEN = 0b11` enabled in `mmu_init`.**
   musl's `__init_libc` calls `memset(local_aux, 0, 304)`
   which compiles to `dup v0.16b, w1` (NEON).  Without
   FPEN, the very first libc call traps with EC=7
   ("trap from FP/SIMD without permission").  Kernel
   itself stays `-mgeneral-regs-only` so no FP-save-on-
   context-switch is needed in v1; cross-task FP
   persistence is a future slice when busybox's longer-
   running multi-process workloads force it.
3. **Build musl's crt1.o + crti.o + crtn.o** alongside
   libc.a.  `MUSL_CRT1 / MUSL_CRTI / MUSL_CRTN` Makefile
   vars; all four targets share a single `make -C
   $(MUSL_DIR) lib/libc.a lib/crt1.o lib/crti.o lib/crtn.o`
   rule (musl produces them in one pass).
4. **New demo `posix_musl_prog.c`** (NOT a re-link of
   `posix_libc_prog.c`).  Reason: posix_libc_prog uses
   `nxlibc_*` names defined only in libnxlibc.  Re-linking
   would force a source change to use POSIX-standard names
   — cleaner to add a side-by-side demo with distinct exit
   code (57 vs libnxlibc's 53).  Source is 8 lines:
   ```c
   typedef long          ssize_t;
   typedef unsigned long size_t;
   extern ssize_t write(int fd, const void *buf, size_t n);
   extern void    _exit(int status) __attribute__((noreturn));
   int main(int argc, char **argv, char **envp)
   {
       (void)argc; (void)argv; (void)envp;
       write(1, "[musl-ok]", 9);
       _exit(57);
   }
   ```
   Inline forward declarations match musl's signatures —
   pulling in `<unistd.h>` would drag the rest of musl's
   header tree.  Link line is the standard musl
   incantation: `LD -n -T init_prog.ld $(MUSL_CRT1)
   $(MUSL_CRTI) demo.o --start-group $(MUSL_LIBC)
   --end-group $(MUSL_CRTN)`.  Final ELF: 7.3 KB (vs
   libnxlibc-linked at 2.3 KB) — musl pulls in TLS
   bootstrap + locale init + signal mask init + stack
   canary even for a minimal demo.
5. **`ktest_posix_musl.c`** — drop_to_el0 path with
   `sp_el0 = top - 64` (vs the stock `top - 16`).  musl's
   `__init_libc` walks "argc / argv-NULL / envp-NULL /
   auxv-NULL" dereferencing each; at `top - 16`, envp[0]
   (read at sp+16) lands exactly at the user-window
   boundary -> permission fault.  64 bytes of slack
   give crt1 enough zero-padded headroom.  No marker
   check — `write(1, ...)` returns NX_EBADF (no stdio
   plumbing yet); slice 7.6c.3c wires the magic-fd-handle
   so the marker shows up in the live log.
6. **The four other svc-direct asm files stayed stock.**
   `clone.s / vfork.s / __unmapself.s / restore.s` aren't
   pulled in by the linker — the demo doesn't call
   fork/vfork/pthread_create/sigaction.  They'll need
   patching once busybox enters the picture.

**Exit criteria (7.6c.3b):** musl-linked EL0 program runs
through `__libc_start_main` to `_exit`, exit_code observable
in the process table.  +1 kernel test
(`posix_musl_prog_runs_main_through_init_libc_and_exits_57`).
`make test` 404/404.

**7.6c.3c — Stdio + brk + writev + sys_exec → musl-child
test + musl-linked printf** (complete).  Closes slice 7.6c.3.

Three new kernel pieces wire musl-linked demos into a working
end-to-end stack:

1. **Stdio magic-fd-handle in `sys_write` / `sys_read`.**  When
   `nx_handle_lookup` returns NX_ENOENT and the handle index is
   1 or 2, `sys_write` falls back to `sys_debug_write` (UART);
   index 0 in `sys_read` returns 0 (EOF).  Lookup-then-fallback
   ordering preserves the regular dispatch for legitimate
   handles (e.g. `posix_pipe_prog` allocates fds 1+2 to a
   channel — the pipe-roundtrip test still passes).
2. **`NX_SYS_BRK = 17`** + per-process `brk_addr` field
   in `struct nx_process`.  Heap region:
   `[base + NX_PROCESS_HEAP_OFFSET = 1 MiB ..
     base + NX_PROCESS_HEAP_LIMIT = 1.5 MiB)` = 512 KiB,
   inside the existing 2 MiB user-window backing.  No extra
   kernel allocation; just track the high-water mark per
   process.  `brk_addr` initialized in `nx_process_create`,
   reset in `sys_exec` (fresh image gets a fresh heap),
   inherited byte-for-byte across `fork` (the user-window
   backing is already byte-copied).
3. **`NX_SYS_WRITEV = 18`** + iovec-walking `sys_writev` that
   `copy_from_user`'s the iovec array (cap 16 entries) and
   dispatches each non-empty entry through `sys_write`.  Stops
   on the first short write per Linux's writev convention.
   Required because musl's `__stdio_write` flushes via writev
   exclusively (no SYS_write fallback) — without it, `printf`
   silently sets F_ERR and discards the output.

Translation table additions (both `syscall_arch.h` and
`syscall_cp.s`):
- `__NR_brk    (214)` → `NX_SYS_BRK    (17)`
- `__NR_writev  (66)` → `NX_SYS_WRITEV (18)`

Three new EL0 demos validate the stack:

1. **`posix_musl_prog.c`** (already from slice 7.6c.3b).  ktest
   gets a marker assertion now that the magic-fd path makes
   `[musl-ok]` reach UART via the route: `write(1, ...)` →
   `__NR_write (64)` → `NX_SYS_WRITE (8)` → magic-fd fallback
   → `sys_debug_write`.
2. **`musl_exec_parent_prog.c`** (libnxlibc-linked).  Forks
   + `nxlibc_execve("/musl_prog", argv, NULL)` against the
   initramfs-seeded musl child.  Validates AUXV consumption
   end-to-end — slice 7.6c.3b's AUXV push only fires through
   sys_exec, and only this test actually drives a sys_exec →
   musl-linked image.  Live-log invariant:
   `[musl-exec-parent][musl-ok][musl-exec-ok]`.
3. **`posix_musl_printf_prog.c`** (musl-linked).  Single
   `printf` with `%d %u %x %s %c \n`, then `exit(67)` (NOT
   `_exit` — `__stdio_exit` only runs from `exit`, otherwise
   the buffered output is discarded).  Pulls in vfprintf +
   locale init + libgcc softfloat helpers (`__netf2`,
   `__addtf3`, etc. — used by the `%f` code path which is
   reachable from format-string-controlled dispatch even when
   no `%f` is present).  `LIBGCC := $(shell $(CC)
   -print-libgcc-file-name)` resolves the cross-toolchain's
   libgcc.a; --start-group / --end-group bundles libc.a +
   libgcc.a so the cycle resolves.  Final ELF: 37 KB.  Live
   log: `[musl-printf d=-1 u=4294967295 x=deadbeef s=ok c=Q]`.

**Drive-by gotchas captured during this slice:**

- `exit()` not `_exit()` for stdio-using programs.
- musl's `__stdio_write` uses SYS_writev exclusively — caught
  when posix_musl_printf reached `exit(67)` cleanly but no
  output reached UART.
- `vfprintf` pulls in libgcc TF-arith helpers via the `%f`
  code path even when no float conversion is in the format
  string.
- Host `sys_write_on_stale_handle_returns_enoent` test moved
  to h=3 (h=1/2 are now magic-routed); new test
  `sys_write_to_unopened_stdout_routes_to_debug_write`
  documents the magic-fd contract.
- `MUSL_LIBC` Makefile rule wipes `$(MUSL_DIR)/obj` +
  `$(MUSL_DIR)/lib` on dep change — fixes the slice-7.6c.3a
  gotcha that musl's `.lo` cache doesn't track arch-header
  inclusion.

**Exit criteria (7.6c.3c):** `make test` → 407/407 pass.
musl-linked programs run through `__libc_start_main` →
`main` → printf → exit cleanly; output reaches UART; AUXV
consumption verified end-to-end via sys_exec.  +3 kernel tests
+ 1 host test.  **Slice 7.6c.3 closed.**

---

**(Original sub-slice plan, retained for archeology):**
The original 7.6c.3c plan also included re-linking the existing
libnxlibc demos against musl.  Skipped in favour of new
side-by-side demos:
posix_libc_prog.c uses `nxlibc_*` names defined only in libnxlibc;
re-linking would force a source change to use POSIX-standard names.
The new side-by-side demos (posix_musl_prog, posix_musl_printf_prog)
+ the side-by-side libnxlibc demos give us strict regression
detection for both libcs simultaneously.

---

**(Below is the original 7.6c.3c pending plan — kept for
historical reference; superseded by what actually landed.)**

1. **Add `brk` syscall.**  musl's `mallocng` extends the
   heap via `brk` first, falling back to `mmap` if brk
   returns -1.  brk is the simpler shape: one new syscall
   (`NX_SYS_BRK = 17`) + a per-process `brk_addr` field
   tracked alongside `ttbr0_root`, sized within the existing
   2 MiB user window (musl's malloc is incremental and won't
   exhaust 2 MiB in our v1 demos).  Updates `__nx_translate`
   to map `__NR_brk = 214` to the new number.
2. **sys_exec → musl-linked-child test.**  Validates AUXV
   consumption end-to-end.  Slice 7.6c.3b implemented the
   AUXV push but only exercised it through drop_to_el0
   (which doesn't use the AUXV layout — sp lands in
   zero-init memory).  A small libnxlibc parent program
   (similar shape to `argv_parent_prog`) does
   `nx_posix_execve("/musl_prog", argv, NULL)` against a
   ramfs-seeded `posix_musl_prog.elf`; child reads argv +
   auxv via crt1's stack walk; exit_code propagates through
   `wait()`.  Without this test, AT_RANDOM / AT_PAGESZ are
   "implemented but never read".
3. **Stdio plumbing** so `write(1, ...)` reaches UART.
   Two options:
   - (a) Pre-allocate fd 1/2 in every process to a
     debug-write handle (each new process gets two
     pre-populated handle slots).
   - (b) Magic-fd-handle inside `NX_SYS_WRITE`: if the
     handle index is 1 or 2 and the slot is free, route
     to `nx_posix_debug_write`.  Mirrors what libnxlibc
     does at the libc layer; closer to how Linux works
     (kernel knows about stdio via the open file table).

   (b) is the closer-to-long-term approach.  Once stdio
   works, slice 7.6c.3b's `[musl-ok]` marker shows up in
   the live ktest log, ktest gains a marker check.
4. **Re-link `posix_printf_prog.elf`** against musl —
   pulls in malloc the moment musl's printf formats `%d`
   (locale init allocates).  Live ktest log gains
   musl-formatted printf markers (different from
   libnxlibc-formatted ones, useful for spotting
   format-string regressions).
5. **Optionally re-link `argv_parent_prog.elf`.**
   Currently uses libnxlibc only for `_exit + write`;
   should "just work" against musl as long as the
   kernel-side ENOSYS-y syscalls aren't hit during init.

**Exit criteria (7.6c.3c):** musl-linked program runs
end-to-end via sys_exec, malloc works, stdout reaches
UART.  Closes slice 7.6c.3.

**7.6d — Busybox cross-compile + boot to shell** (pending).
Sub-sliced 7.6d.1 / 7.6d.2 / 7.6d.3+ / 7.6d.N to mirror how
7.6c.3 was decomposed: vendor first, then incremental
discovery of what kernel-side gaps need closing before the
shell prompt comes up.

**7.6d.1 — Cross-compile busybox** (pending; this slice).

Build infrastructure only.  No boot integration; success means
a static aarch64 ELF lands on disk and survives a sanity check
(file/readelf/`-h`).

1. **Vendor busybox under `third_party/busybox/`** (mirrors
   the musl pattern from 7.6c.3a — full source tree checked in,
   patches live in tree).  Use a stable upstream tarball
   (busybox 1.36.1, the last release before the 1.37 series'
   shell rewrite, picked because it's the most-deployed
   version across embedded distros and so most exercised
   against musl).
2. **Commit a minimal `.config`** (tracked at
   `third_party/busybox/configs/nonux_defconfig`).  Built from
   `make defconfig` then turned off:
   - `CONFIG_PAM` / `CONFIG_LOGIN` / `CONFIG_GETTY` / `CONFIG_PASSWD`
     / `CONFIG_SU` (we have no PAM and no /etc/passwd)
   - All `CONFIG_FEATURE_*_LONG_OPTIONS` (smaller binary)
   - `CONFIG_FEATURE_USE_INITTAB` / inittab-driven init
     (we run `/init = busybox sh` directly)
   - All networking applets (`NETWORKING / WGET / TFTP / FTP /
     PING / IFCONFIG / IP / NETSTAT / ARP / ROUTE` etc.)
   - All modutils (`INSMOD / RMMOD / MODPROBE` — we have no
     module loader)
   - All printutils, mailutils, runit, klogd
   - On to keep: `SH / ASH / LS / ECHO / CAT / PS / MKDIR /
     RM / MV / CP / PWD / TRUE / FALSE` (the slice-7.7
     smoke-test set + obvious dependencies).
3. **`tools/build-busybox.sh`** drives the build:
   - Copies `nonux_defconfig` into busybox's `.config` slot.
   - Runs `make ARCH=arm64 CROSS_COMPILE=$CROSS oldconfig`
     to merge any new upstream config keys.
   - Runs the actual build with `CC` / `LD` / `AR` /
     `CFLAGS` / `LDFLAGS` pointing at our patched musl's
     crt+libc set (the same `--start-group $(MUSL_LIBC)
     --end-group` shape from `posix_musl_prog.elf`).  Output:
     `third_party/busybox/busybox` (a static aarch64 ELF).
4. **Top-level Makefile gains `BUSYBOX_DIR / BUSYBOX_BIN /
   BUSYBOX_STAMP` + `busybox / busybox-clean` targets** (same
   shape as the musl block that lives a few lines above
   them).  `make test` does NOT yet depend on `busybox` — the
   build is opt-in until 7.6d.4 wires it into the boot image.
5. **No new EL0 demos, no new ktests**.  Sanity check is
   manual: `file third_party/busybox/busybox` should report
   `ELF 64-bit LSB executable, ARM aarch64, statically linked`,
   and `aarch64-linux-gnu-readelf -h` should show the entry
   point in a sane range.  This is intentional — actually
   *running* busybox against the kernel is 7.6d.2's job, and
   it's where the discovery-driven follow-ups (mmap, the
   four svc-direct asm files, process-table caps) get
   triaged.

**Exit criteria (7.6d.1):** `make busybox` produces a static
aarch64 ELF; spot-checks pass; `make test` still 407/407 (we
deliberately don't hook busybox into the test pipeline yet —
that lands in 7.6d.4).

**7.6d.2 — Try to boot busybox `--help` from a libnxlibc
parent** (pending; discovery-driven).  Further sliced
7.6d.2a / 7.6d.2b / 7.6d.2c — discovered during 7.6d.2's
plan-out that the kernel-side preconditions are large enough
to deserve their own per-session scope.

The original 7.6d.2 plan was "ramfs-seed busybox, fork+exec it
from a libnxlibc parent, capture whichever trap fires first."
That ran into two preconditions before any kernel-side
discovery could happen:

1. **Address-space mismatch.**  busybox is statically linked
   at VA `0x400000` (kbuild default for `-static -no-pie`).
   Our user window sits at `0x48000000`.  The ELF loader
   honours `PT_LOAD.p_vaddr` verbatim (same as Linux for
   static-no-pie binaries), so loading busybox-as-built
   would write to `0x400000`, which lives in MMIO space.
   Fix: pass `-Wl,-Ttext-segment=0x48000000` at link time so
   the ELF carries the right VA for our user window.
2. **Window too small.**  Even with the re-link, busybox's
   text+data segments span `~1.91 MB` (LOAD #1 at `0x400000`
   sized `0x1c6434` + LOAD #2 at `0x5d9770` sized `0xf1c0`,
   end-to-end `0x1e8930`).  Our user window is exactly
   `2 MB`, which leaves ~84 KiB for stack + heap — and the
   current brk region is hardcoded at `[base+1 MiB ..
   base+1.5 MiB)`, smack inside the segment-2 zone busybox
   wants for its `.bss`.  Fix: grow the user window (e.g. 4
   ×2 MiB blocks → 8 MiB) and relocate the brk region so it
   doesn't overlap busybox's own VA span.

These two have to land *before* we can usefully exec busybox
to discover what's missing at runtime.  Keeping them in
separate sub-slices means each is independently reviewable
and bisectable, and the address-space change (which touches
every existing EL0 test's invariants) doesn't get bundled
with discovery work.

**7.6d.2a — Re-link busybox at `0x48000000`** (pending).

Build-script-only change.  `tools/build-busybox.sh`'s real-build
step gains `-Wl,-Ttext-segment=$(USER_WINDOW_BASE)` in
`EXTRA_LDFLAGS`; the value is hardcoded as `0x48000000` in the
script (matches `mmu_user_window_base()` and `init_prog.ld`'s
`. = 0x48000000;`).  `readelf -l busybox` should show the new
VA.  No kernel changes; no ktest changes.  busybox still
doesn't *fit* the 2 MiB window — that's 7.6d.2b — but the ELF
now describes itself correctly relative to where it'll
eventually land.

**Exit criteria (7.6d.2a):** `readelf -l third_party/busybox/busybox`
shows segment 1 at `VirtAddr 0x48000000`; binary still
statically-linked aarch64 ELF; `make test` still 407/407.

**7.6d.2b — Grow the user window to 8 MiB + relocate brk
region** (pending).

`USER_WINDOW_SIZE` bumps from `2 MiB → 8 MiB` (4 contiguous L2
ram blocks at indexes 64..67 instead of just 64).  L2 blocks
are 2 MiB granular, so 8 MiB is exactly four entries — the
window doesn't need new alignment math, just four entries
overwritten in `mmu_create_address_space` and four entries
upgraded to `user_block` in `mmu_init`'s kernel L2_ram (the
kernel's identity map keeps the user-window VA EL0-accessible
in the kernel's own address space too, matching the existing
slot-64 pattern).

Per-process backing allocation in `mmu_create_address_space`
malloc footprint scales: previously `USER_WINDOW_SIZE * 2 =
4 MiB` per process, now `USER_WINDOW_SIZE + 2 MiB = 10 MiB`
per process.  Two changes here: (1) keep alignment to 2 MiB
(the L2 block alignment) rather than the entire window size
— the four blocks just need to start on a 2 MiB boundary,
they don't need to start on an 8 MiB boundary; (2) trim the
alignment slack from `USER_WINDOW_SIZE` (overkill — would
waste 8 MiB) to a fixed 2 MiB.  With 32 process slots that's
worst-case `320 MiB` peak overhead, vs `512 MiB` if we kept
the existing `*2` formula.

brk region moves from `[base+1 MiB .. base+1.5 MiB)` to
`[base+6 MiB .. base+7.5 MiB)` (well past busybox's data/bss
segment which ends near `base+1.91 MiB`); `sp_el0` for fresh
processes stays `top - 16` (formula already references
`mmu_user_window_base() + mmu_user_window_size()` so it
auto-adjusts to the new 8 MiB window).

Every existing EL0 ktest — drop_to_el0 sites, exec
chains, fork demos — has to stay green.  The risk surface is
real: 15+ ktests pass `top - 16` as `sp_el0` and the new top
is at a different VA.  `ktest_el0.c` has a hard
`KASSERT(size == 2 MiB)` that has to bump to `8 MiB`.
Verification path: run the full `make test` after the bump;
if anything regresses, narrow to the offending test and
inspect.

`NX_PROCESS_HEAP_OFFSET` and `NX_PROCESS_HEAP_LIMIT` constants
shift in `framework/process.h`; ramfs / vfs / channel paths are
untouched (they don't care about the user window's size).

**Exit criteria (7.6d.2b):** `mmu_user_window_size() == 8 MiB`;
brk region relocated and not overlapping the busybox VA span;
`make test` 407/407 unchanged.

**7.6d.2c — Bump caps + ramfs-seed busybox + try to exec**
(complete — closes 7.6d.2).

The actually-discovery-driven sub-slice.  By the time we get
here, busybox is linked at the right VA and the user window
is big enough to hold it.  Three cap bumps:

- `RAMFS_FILE_CAP`: `8192 → 4 MiB` (busybox is 2.29 MB; round
  up to the next power-of-two-ish for headroom).  Watch the
  static `data[RAMFS_FILE_CAP]` array per file slot — bumping
  to 4 MiB × `RAMFS_MAX_FILES = 8` is `32 MiB` of static .bss
  in the ramfs component, which the kernel image absorbs.
  v1 hack documented in the comment; long-term direction is
  per-file dynamic allocation as part of the Phase-8+ memory
  rework.
- `SYS_EXEC_MAX_FILE`: `8192 → 4 MiB` to match.  This is the
  kheap buffer `sys_exec` slurps the ELF into; per-call
  malloc, no static cost.
- `tools/pack-initramfs.py`'s entry list gains
  `third_party/busybox/busybox:/bin/busybox`.

A new EL0 demo `posix_busybox_help_prog.c` (libnxlibc-linked
parent) forks + `nxlibc_execve("/bin/busybox", argv, NULL)`
with `argv = { "/bin/busybox", "--help", NULL }`.  Matching
`ktest_posix_busybox.c` drives the parent kthread.

**Captured failure** (Session 52 log has the full decode):

```
posix_busybox_help_parent_forks_and_execs_busybox [busybox-parent]
[EXC] sync ESR=0x9200004e FAR=0x20 ELR=0x48002460 SPSR=0x20000000
```

`ESR=0x9200004e` decodes to EC=0x24 (Data Abort, lower EL),
WnR=1 (write), DFSC=0x0e (Permission fault, level 2).  The
faulting EL0 PC `0x48002460` is at `__memset_generic+0xa0`
(`str q0, [x0]`); `x0 = 0x20` matches the offset of `errno`
within musl's `struct pthread`.  Classic symptom of musl's
`TPIDR_EL0` (TLS pointer) being uninitialized at EL0 entry —
`errno = &__pthread_self()->errno` resolves to a small NULL-
derived offset, and busybox touches errno on every syscall
return.

The new test stays OUT of `KTEST_C` until 7.6d.3a (signal-on-
fault) lands — without it, the EL0 fault propagates as
`halt_forever` and breaks `make test` 407/407.  Source files
live in the tree as the discovery deliverable; the `KTEST_C`
comment documents the hold-out.

**Exit criteria (7.6d.2c → closes 7.6d.2):** Kernel test
source exists; failure mode is captured in the live ktest log
(via manual enable); 7.6d.3+ sub-slices enumerated in
HANDOFF.md based on actual observation rather than the
crystal-ball.  ✓ All met.

**7.6d.3+ — Add scaffolding** (pending; concrete shape now
grounded in 7.6d.2c's captured failure rather than the
HANDOFF crystal-ball).

**7.6d.3a — Signal-on-fault** (complete).  `on_sync` in
`core/cpu/exception.c` now converts non-SVC + lower-EL fault
EC values into `nx_process_exit(128 + signo)` so the parent's
`wait()` collects an EL0 process's death cleanly:

- EC `0x20` (Inst Abort, lower EL) → SIGSEGV (signo 11) →
  exit 139
- EC `0x21` (Inst Abort, current EL) → kernel panic (unchanged)
- EC `0x24` (Data Abort, lower EL) → SIGSEGV → exit 139
- EC `0x25` (Data Abort, current EL) → kernel panic
- EC `0x00` (Unknown reason) from EL0 → SIGILL (signo 4) →
  exit 132 (origin distinguished by checking `tf->pstate`'s
  M[3:0] bits)
- EC `0x00` from EL1 → kernel panic
- Other lower-EL fault EC values (FP/SIMD trap, watchpoint,
  HVC/SMC, etc.): default branch panics — they don't fall
  through to SIGSEGV because masking unexpected EC values
  would hide kernel bugs.  Convert when a real demo trips
  them.

New helper `deliver_el0_fault_signal(signo, reason, esr, far,
pc)` is `__attribute__((noreturn))` (`-Werror=implicit-
fallthrough` caught the missing attribute in the first compile
attempt; without it gcc complains about the switch cases).
Helper prints a one-line marker (`[EXC] EL0 <reason>
ESR=... -> exit N`) before calling `nx_process_exit`.

Hardcoded signo constants `NX_FAULT_SIGSEGV = 11`,
`NX_FAULT_SIGILL = 4` — distinct from slice-7.5's polled-
signal `NX_SIGKILL`/`NX_SIGTERM` constants.  Explicitly so a
future renumbering in either path doesn't silently change
exit codes in the other.

Two new ktests demonstrate:

- `posix_segfault_prog` — child writes `*(volatile int *)0`,
  parent observes 128+SIGSEGV (= 139), emits `[segv-ok]`.
- `posix_undef_prog` — child executes `asm volatile (".word
  0")` (UDF #0), parent observes 128+SIGILL (= 132), emits
  `[undef-ok]`.

Slice 7.6d.2c's `ktest_posix_busybox` (held out of `KTEST_C`
because its unconverted fault would `halt_forever`) is
re-enabled in this slice — busybox's TLS-gap fault now
cleanly exits 139 and the parent prints
`[busybox-status=8b][busybox-help-failed]`.

**Exit criteria (7.6d.3a):** EL0 faults convert to a
parent-observable process exit; +2 ktests demonstrate the
conversion; `make test` 410/410 (was 407/407).  ✓ All met.

**7.6d.3b — Push richer AUXV.**  At minimum:
- `AT_HWCAP` (16) — zero is acceptable; busybox/musl just
  won't take feature-gated fast paths
- `AT_HWCAP2` (26) — zero
- `AT_PLATFORM` (15) — pointer to a static `"aarch64\0"` in
  user memory

Per-architecture musl needs vary; check
`third_party/musl/src/env/__init_tp.c` and
`__init_libc` paths for which AT_* it dereferences early.
The push happens in `sys_exec` alongside the existing
AT_PAGESZ + AT_RANDOM + AT_NULL pairs.

**Exit criteria (7.6d.3b):** the existing `posix_musl_prog`
+ `musl_exec_parent_prog` ktests still pass; manually-enabled
`ktest_posix_busybox` reaches further into busybox init
before any new fault.

**7.6d.3c — Pre-initialize `TPIDR_EL0`.**  Allocate a small
zeroed thread-control area per process (or per task — TBD;
per-process is simpler since tasks within a process share the
same EL0 view today) at process-create time.  Before `eret`
to EL0 in `drop_to_el0` and the post-fork / post-exec paths,
`msr tpidr_el0, x0` it.  Buffer size: 64 bytes is enough for
musl's `errno` + a few early-touched struct fields; musl's
`__init_libc` later runs `__set_thread_area(td)` which
overwrites TPIDR_EL0 with its own properly-allocated
`struct pthread`.  The kernel's pre-init buffer just has to
keep us alive long enough for `__init_libc` to construct the
real one.

Per-process buffer lives at the start of the user backing
(e.g. `[base, base+64)`) — falls inside the user-window VA
range so EL0 can read it.  Or in a separate kheap allocation
(EL0 reads via the kernel-identity-mapped PA).  Latter is
cleaner; former saves an alloc.

**Exit criteria (7.6d.3c):** with 3a + 3b in place, the
busybox test's child either reaches further into musl init
(no more TLS-related fault) OR fails on a different gap
(mmap, clone, ENOSYS svc) — that next-fault becomes
7.6d.3.x.

**7.6d.3.x (later) — `mmap` / `clone.s` / `vfork.s` /
`__unmapself.s` / `restore.s`** — only if busybox's `--help`
path actually reaches them after the TLS gap is closed.
Re-test, observe what fails next.  Predicted in 7.6c.3a but
no demo has pulled them in until busybox.

For mmap specifically: `NX_SYS_MMAP = 19?` as anonymous
private mapping inside the existing user-window backing
(same high-water-mark shape as brk; carve from
`[base+7.5 MiB .. base+8 MiB)` downward, leaving stack
space).  musl's mallocng falls back to mmap once brk runs
out.

For the four svc-direct asm files: each ends with `svc 0`
after putting a Linux syscall nr in `x8`; patch the
syscall-translation switch in
`third_party/musl/arch/aarch64/syscall_arch.h` (and the asm
counterparts) to map them to NX_SYS_* values or to ENOSYS
where we don't intend to support them.

For `NX_PROCESS_TABLE_CAPACITY`: bump again or land proper
cross-test reap on `wait()` (deferred since slice 7.4).
busybox under heavy smoke-testing creates dozens of short-
lived processes; the current 32-slot table runs dry without
reap-on-wait.

Each 7.6d.3.x sub-slice ships with a kernel test that the
addition unblocks (not just an "added the syscall" change),
matching the same per-slice pattern that closed 7.6c.3.

**7.6d.N — `/init = busybox sh` boots to a shell prompt over
serial** (in progress — further sliced 7.6d.N.0 / 7.6d.N.1 /
7.6d.N.2 / 7.6d.N.3 / 7.6d.N.4 / 7.6d.N.5 (all done) /
7.6d.N.6+ (discovery-driven, just like 7.6d.2/7.6d.3 was)).
Closes slice 7.6d.

Final form (after all sub-slices land):

1. `tools/pack-initramfs.py` gains a busybox-aware mode that
   emits `/bin/busybox` + the conventional applet symlinks
   (`/bin/sh -> busybox`, `/bin/ls -> busybox`, etc.).  Slice
   7.6b's ramfs-slurp already consumes the same cpio-newc
   format, so no kernel changes needed if symlinks are
   represented as duplicate file entries (or the lookup-by-
   name table treats them as such — TBD when we get there).
2. `/init` becomes busybox; no other PID 1 candidate.
3. `tools/run-qemu.sh` runs the production `kernel.bin`; the
   shell prompt should land on the serial console within
   500 ms of MMU bring-up.

**Exit criteria (7.6d.N → full 7.6):** `make && make run`
boots to a busybox shell prompt over serial.

**7.6d.N.0 — First attempt at exec'ing busybox AS A SHELL +
capture failure** (DONE — Session 55).  New libnxlibc-linked
`posix_busybox_sh_prog.c` forks + `nxlibc_execve("/bin/busybox",
{ "sh", "-c", "exit 42", NULL }, NULL)` against the
initramfs-seeded busybox binary.  Three details that fall out
of how busybox dispatches applets:

- `argv[0] = "sh"` (NOT `"/bin/busybox"`).  busybox dispatches
  via `basename(argv[0])`; the applet table is keyed on the
  basename only.  No `/bin/sh -> busybox` symlink needed in
  the ramfs for in-process applet dispatch — the symlink
  layout is only needed when the *shell* tries to exec
  external `sh` (which it doesn't for -c).
- `-c "exit 42"` makes it one-shot, non-interactive — no TTY
  setup, no fork-into-applet (since `exit` is a shell
  builtin), no execvp lookups.
- Discovery-only: no kernel changes, just observation.

Captured failure (live ktest log):
```
[bbsh-parent]sh: out of memory
[bbsh-status=01][bbsh-failed]
```
ash got far enough into startup to print its OOM diagnostic
(busybox's `bb_msg_memory_exhausted` from `libbb/messages.c:17`)
and exit cleanly with status 1.  No kernel fault, no SIGSEGV,
no SIGILL — process exit was POSIX-clean.

Root cause: musl 1.2.5 uses mallocng exclusively (`config.mak`:
`MALLOC_DIR = mallocng`); mallocng calls `mmap(MAP_PRIVATE |
MAP_ANON)` for every slab allocation
(`src/malloc/mallocng/malloc.c:249`); there's no brk-only
fallback inside mallocng.  Our `arch/aarch64/syscall_arch.h`
translation table doesn't map `__NR_mmap (222)`, so mmap
returns -ENOSYS, slab allocation returns NULL, malloc returns
NULL, ash's xmalloc trips its OOM exit.  brk IS implemented
(slice 7.6c.3c) but mallocng doesn't use it — the
`[+6 MiB, +7.5 MiB)` brk slot in our 8 MiB user window is
sitting unused by the actual allocator.

**7.6d.N.1 — Minimal anonymous `mmap` + no-op `munmap`** (DONE
— Session 56).  Only the mallocng shape:

- `mmap(addr=0, length, prot, MAP_PRIVATE|MAP_ANON, -1, 0)`.
- Page-aligned address chosen by kernel from a per-process
  bump arena.  No `MAP_FIXED`, no file-backed mappings, no
  per-page protection (the user window is uniformly
  EL0-RW), no overcommit.
- Layout: carved the unused `[+2 MiB, +5 MiB)` window region
  (3 MiB between busybox text+data+bss end at +1.91 MiB and
  the TLS area at +5 MiB) as the per-process mmap arena.
  `nx_process.mmap_bump` tracks the high-water mark — bump
  pointer; OOM (return -ENOMEM, which musl's `__syscall_ret`
  turns into MAP_FAILED + errno) when bump exceeds arena.
- Returned region zeroed to honour MAP_ANONYMOUS's zero-init
  contract — the underlying user_window backing is plain
  `malloc()` memory and may carry stale bytes from earlier
  exec cycles.
- `munmap` returns NX_OK unconditionally (PMM reclaim
  happens at process exit; mallocng treats `munmap` failure
  as fatal so we can't return -ENOSYS).
- `nx_process.mmap_bump` initialized in `nx_process_create`
  / `sys_exec` to `mmu_user_window_base() +
  NX_PROCESS_MMAP_OFFSET`; inherited across fork (matches
  `brk_addr`).

Translation-table additions (slice 7.6c.3a's pattern):
- `__NR_mmap (222) -> NX_SYS_MMAP = 19` in
  `arch/aarch64/syscall_arch.h` and the asm-level switch in
  `src/thread/aarch64/syscall_cp.s`.
- `__NR_munmap (215) -> NX_SYS_MUNMAP = 20` likewise.

**Result:** `posix_busybox_sh` ktest now captures
`[bbsh-status=2a][bbsh-ok]` — ash startup ran without OOM,
parsed `-c "exit 42"`, dispatched the `exit` builtin, exited
cleanly with status 42.  Substantially better than predicted
— the original 7.6d.N.0 plan expected mmap to unblock malloc
but ash to then hit a sigaction / getuid / ioctl gap; for
the simple `exit 42` builtin path, none of those were
needed.

**7.6d.N.2 — `busybox sh -c "echo hello"`** (DONE — Session
57).  Escalation past `exit 42`.  ash parses multi-token -c
string + dispatches `echo` builtin; `echo` writes `hello\n`
via musl stdio (`__stdio_write` → SYS_writev →
magic-fd-handle → UART); ash runs end-of-input shutdown
(flush, atexit, teardown) + exits 0.  First-run captured
exec failure (NX_ENOMEM = -2 from `mmu_create_address_space`,
diagnostic dump showed byte 0xfe → -2): cumulative test
sweep had filled the 32-slot MMU-spaces table after now
exec'ing busybox 4 times (7.6d.2c `--help`, 7.6d.N.0 +
7.6d.N.1 sharing `sh exit 42`, 7.6d.N.2 `sh echo hello`).
Fix mirrors slice 7.6c.4's earlier 16 → 32 bump: bump
`MMU_MAX_ADDRESS_SPACES` 32 → 64 + `NX_PROCESS_TABLE_CAPACITY`
32 → 64.  Memory cost: 64 × 8 MiB = 512 MiB peak — workable
inside QEMU's 1 GiB guest RAM.  Real fix is reap-on-wait
(deferred since slice 7.4).

**7.6d.N.3 — `busybox sh -c "echo a; echo b"`** (DONE —
Session 58).  Multi-statement script — escalation past
single-statement `echo hello`.  ash's parser builds a
sequence node with two child commands (exercises the `;`
separator path); the eval loop iterates dispatching each
`echo` builtin in turn; two consecutive stdio writes confirm
buffering across calls.  Clean first try — no kernel
changes, no cap exhaustion (slice 7.6d.N.2's 64-slot bump
still has headroom), no new syscalls reached, no faults.
ash's sequence machinery is inside the working set
established by 7.6d.N.2.

**7.6d.N.4 — `busybox sh -c "ls /"` reaches the `ls`
applet** (DONE — Session 59).  First non-builtin
escalation.  Lowest-friction approach: cpio-duplicate
`/bin/busybox` as `/bin/ls` in initramfs so ash's PATH walk
finds it (no symlink machinery yet).  Diagnostic
instrumentation (temporary musl pass-through of unmapped
Linux numbers + kernel `kprintf("[svc?=N]")`, both reverted
before commit) confirmed `[svc?=79]` (newfstatat) was the
blocking syscall during ash's PATH walk.  Fix: new
`NX_SYS_FSTATAT = 21` (~70 lines) — opens path through vfs
to check existence, fills 128-byte aarch64 struct stat with
`st_mode = S_IFREG | 0755`, returns Linux errno values
directly (`-2` for ENOENT, not the NX_E* convention)
because ash's PATH walk distinguishes ENOENT-as-"next-
candidate" from other errnos as fatal.  musl translation:
`__NR_newfstatat (79) → NX_SYS_FSTATAT (21)` in both
`syscall_arch.h` and `syscall_cp.s`.  Drive-bys:
`RAMFS_MAX_FILES` 8 → 16 (cpio-duplicate `/bin/ls` + 5
prior + 3 test-side creates = 9 > 8; static cost 32 → 64
MiB); host-test ATTEMPTS bumps in
`component_ramfs_test.c` (12 → 24, 64 → 96); QEMU
`test-kernel` timeout 15s → 90s.  Live ktest log:
`[bbsh-ls-parent]ls: /: No such file or directory[bbsh-ls-status=01][bbsh-ls-failed]`
— ash forked + execve'd /bin/ls successfully, ls reached
its own `stat("/")` and bailed because vfs has no `/`
directory entry.  Test parent always exits 0 (kernel-test
KASSERT condition); the ls workload fails but the test
passes structurally.

**7.6d.N.5 — `busybox sh -c "ls /"` runs end-to-end** (DONE
— Session 60).  Closed three gaps in one slice: (a)
`sys_fstatat` special-cases path `/` → `S_IFDIR | 0755`
(no vfs round-trip; vfs_simple has no directory inode for
`/`); (b) new `NX_HANDLE_DIR` handle type + `sys_open` `/`
branch (allocates a `struct nx_dir_cursor` with the readdir
cookie under a HANDLE_DIR; gated `!__STDC_HOSTED__`) +
`sys_handle_close` HANDLE_DIR branch (frees the cursor on
close); (c) new `NX_SYS_GETDENTS64 = 22` (~75 lines) walks
`vops->readdir` against the per-handle cursor + packs Linux
`struct linux_dirent64` records (8-byte aligned,
`d_type = DT_REG` for v1's flat namespace) into a 4 KiB
kernel staging buffer + `copy_to_user`'s out.  Plus new
`NX_SYS_OPENAT = 23` (~25 lines) Linux-shape wrapper that
converts Linux O_* flag bits to our `NX_VFS_OPEN_*` and
forwards to `sys_open` — musl's `open()` becomes
`openat(AT_FDCWD, ...)` on aarch64 (no plain open syscall),
and `__NR_openat (56)` was completely unmapped before this
slice.  musl `__NR_openat (56) → 23` and
`__NR_getdents64 (61) → 22` in both `syscall_arch.h` and
`syscall_cp.s`.  ash forks, execs `/bin/ls`, busybox
dispatches to ls applet, ls calls `opendir("/")` →
`getdents64` → sorts alphabetically + prints all 9 ramfs
entries one-per-line.  `bin/busybox` and `bin/ls` render as
basenames (`busybox`, `ls`) because busybox ls's display
layer strips path prefixes — fine for `ls /` but real fix
is multi-level directory inodes (deferred).

**7.6d.N.6a — First-pipe scaffolding for `echo hello |
cat`** (DONE — Session 61).  Mapped four syscalls and
implemented two new ones; pipe ktest written but held out
of `KTEST_C` until 7.6d.N.6b reserves slots 0/1/2.

Translations: `__NR_pipe2 (59) → NX_SYS_PIPE` (one-line —
musl's `pipe(int[2])` is `pipe2(fds, 0)` on aarch64; the
kernel ignores the unused flags arg via the standard arg-
register slot semantics).  `__NR_clone (220) → NX_SYS_FORK`
(closes session 59's fork-mystery: musl's `fork()` is
`clone(SIGCHLD, NULL, NULL, NULL, NULL)` — we drop all
five args; previous tests' single-command shortcut had
masked the unmapped clone because ash was exec'ing in-
place rather than actually forking).  `__NR_dup3 (24) →
NX_SYS_DUP3` (new).  `__NR_readv (65) → NX_SYS_READV`
(new).

`NX_SYS_DUP3` (~95 lines in `framework/syscall.c`):
close-then-replace at the destination slot.  Looks up
oldfd via standard `nx_handle_lookup`; closes whatever's
at newfd's slot (decref endpoint for HANDLE_CHANNEL, vfs
close for HANDLE_FILE, free cursor for HANDLE_DIR);
retains the source's underlying object; installs the
source's `(type, rights, object)` at the destination
slot, *forcing the slot's generation to match newfd's
encoded gen* so the encoded handle for the slot equals
newfd exactly (callers like ash assume the literal value
they passed in keeps working).  Special-cases
`newfd == 0` (POSIX STDIN_FILENO) since encoded value 0
is reserved for `NX_HANDLE_INVALID` — install at slot 0
with gen=0.

`NX_SYS_READV` (~35 lines): mirrors `sys_writev` one-to-
one — copy_from_user the iovec array (cap 16 entries),
dispatch each entry through `sys_read`, stop on first
short read per Linux semantic, return running total.

`sys_read` magic-fd path extended: the existing
`if (rc == NX_ENOENT && h == 0) return 0` guard never
fired because `h == 0` makes `nx_handle_lookup` return
`NX_EINVAL` (not `NX_ENOENT`), so cat saw `-NX_EINVAL =
-1` = Linux EPERM, printed `cat: read error: Operation
not permitted`, and busy-looped.  New shape: when
`h == 0`, look up slot 0 directly (bypass the encoded-
value generation check); if slot 0 has a real handle,
dispatch normally; if empty, return 0 = EOF.

Initramfs adds `/bin/cat` + `/bin/echo` cpio duplicates
of busybox.  RAMFS_MAX_FILES = 16 has slack (now 11/16
used).

**Captured failure modes** (held out of `KTEST_C` for
7.6d.N.6b to fix):

1. *Slot collision.*  `pipe()` allocates handles at
   slots 0/1, encoded as 1/2 — colliding with POSIX
   `STDOUT_FILENO=1` / `STDERR_FILENO=2`.  Stage 1's
   `dup2(p[1]=2, 1)` overwrites the pipe READ end with
   the pipe write end; subsequent close(p[0])+close(p[1])
   drops both pipe-write refs in stage 1 without ever
   wiring the pipe read into stage 2's stdin.  echo's
   stdout flushes via slot 0 → magic-fd → debug_write →
   UART, fully bypassing the pipe.  Fix in 7.6d.N.6b:
   reserve slots 0/1/2 in `nx_handle_alloc` (skip them;
   allocations start at slot 3 = encoded handle 4) so
   pipe returns Linux-shape fds 4/5.

2. *cat hang.*  busybox cat's `bb_copyfd_size_eof` takes
   the sendfile fast-path on regular files; our
   `sys_fstatat` reports every file as `S_IFREG | 0755`
   (chosen 7.6d.N.4) so cat thinks stdin is a regular
   file.  `__NR_sendfile (71)` is unmapped → -EPERM
   (NX_EINVAL collision); cat falls back to read-loop;
   the diagnostic trace shows `[svc?=71][svc?=135]
   [svc?=135]` then silence — exact loop point not
   pinned.

**Side discovery** (opportunistic cleanup): `NX_EINVAL =
-1` (`framework/registry.h:33`) collides with Linux's
`EPERM = -1`.  The kernel returns `NX_EINVAL` for unknown
syscall numbers; musl interprets that as Linux EPERM,
so unmapped syscalls show up in userspace as "Operation
not permitted" instead of "Function not implemented".
Hygiene fix: change the kernel's "unknown syscall" return
to `-ENOSYS = -38`.

**7.6d.N.6b (NEXT) — Console device + remove magic-fd
hacks + slot reservation + slots-survive-exec.**  Closes
the structural gap surfaced by 7.6d.N.6a: stdin/stdout/
stderr today are a magic-fd hack (no handle in the table,
falls through to debug_write/UART or EOF), which forces
every syscall to special-case fd 0/1/2 *and* makes
pipe()'s natural slot-0/1 allocation collide with POSIX
STDOUT_FILENO/STDERR_FILENO.  Real fix is a console
primitive that occupies those slots legitimately.

Sub-pieces:

1. **`NX_HANDLE_CONSOLE` handle type +
   `framework/console.{h,c}`.**  Add the enum entry
   between `NX_HANDLE_DIR` and `NX_HANDLE_TYPE_COUNT`.
   `framework/console.{h,c}` exposes:
   - `int nx_console_read(void *buf, size_t cap)` —
     returns 0 (EOF) until UART RX is wired (slice
     7.6d.N.final).
   - `int nx_console_write(const void *buf, size_t len)`
     — per-byte `uart_putc` loop, mirrors the loop body
     of `sys_debug_write`.

   No singleton struct needed — the console object pointer
   in the handle entry can be a fixed sentinel address
   (e.g. `&g_nx_console_token`) since handle dispatch
   only checks the type tag.  Host build provides a
   no-op shim (host has no UART; host-side dispatch
   short-circuits to "wrote N bytes" / "read 0").

2. **Per-process console fds.**  `nx_process_create`
   allocates three HANDLE_CONSOLE entries via
   `nx_handle_alloc`:
   - Slot 0, rights `NX_RIGHT_WRITE` → STDOUT.  Encoded
     handle for slot 0 with gen 0 is `1` = POSIX
     `STDOUT_FILENO` ✓.
   - Slot 1, rights `NX_RIGHT_WRITE` → STDERR.  Encoded
     handle is `2` = POSIX `STDERR_FILENO` ✓.
   - Slot 2, rights `NX_RIGHT_READ` → STDIN.  Slot 2's
     encoded form is `3` but POSIX `STDIN_FILENO` = 0
     has no natural encoding (encoded value 0 is reserved
     for `NX_HANDLE_INVALID`); routed via `h == 0` special
     case (next sub-piece).

   Subsequent user allocations (pipe, channel_create,
   open) naturally land at slot 3+ (the existing "skip
   occupied slots" logic in `nx_handle_alloc`).  pipe
   returns Linux-shape fds 4/5 (encoded for slots 3/4),
   so ash's `dup2(p[1]=5, STDOUT_FILENO=1, 0)` doesn't
   collide with anything: newfd=1 → slot 0 (STDOUT) gets
   replaced with the pipe write end, exactly the POSIX
   semantic.

3. **Syscall layer.**  `sys_read` / `sys_write` add a
   `NX_HANDLE_CONSOLE` arm next to the existing CHANNEL
   / FILE arms — dispatch to `nx_console_read` /
   `nx_console_write`.  `sys_handle_close` adds a
   CONSOLE arm (no-op — no underlying object to free;
   `nx_handle_close` still bumps the slot's generation
   so stale handle values to the closed slot don't
   resolve).

   `h == 0` special case in `sys_read` /
   `sys_handle_close` / `sys_dup3`: treat as "look at
   slot 2 directly" / "close slot 2 directly" /
   "install at slot 2 with gen=0" respectively.  The
   special case lives at the syscall-entry decode step,
   before the encoded-handle lookup.

   Existing magic-fd fallbacks deleted:
   - In `sys_read`:
     `if (rc == NX_ENOENT && h == 0) return 0` —
     replaced by the slot-2 lookup, which returns EOF
     iff slot 2 holds an empty/unwritable HANDLE_CONSOLE.
   - In `sys_write`:
     `if (rc != NX_OK && (h == 1 || h == 2)) →
     debug_write` — replaced by HANDLE_CONSOLE dispatch
     at slots 0/1.

   `sys_dup3`'s existing `newfd == 0` special case
   (added in 7.6d.N.6a, currently installs at slot 0)
   is repurposed: now installs at slot 2.  The encoding
   trick (force slot's gen to match newfd's encoded
   gen) still applies for newfd >= 1; for newfd = 0 we
   force gen=0 directly.

4. **`sys_exec` preserves slots 0/1/2 + their console
   identity.**  POSIX behaviour: stdin/stdout/stderr
   survive exec by default (no FD_CLOEXEC bit in our v1
   model; we don't have FD_CLOEXEC at all).  Required
   for stage 2's `dup3(pipe_read, 0, 0)` +
   `exec(/bin/cat)` to actually pipe data through —
   without preservation, the table reset wipes the
   pipe-redirected stdin and cat reads default console
   (EOF immediately, no data).

   Implementation: `sys_exec`'s table-reset loop skips
   slots 0/1/2.  Slots 3+ get the standard close-and-
   reset.  Concretely, change

   ```c
   for (size_t i = 0; i < NX_HANDLE_TABLE_CAPACITY; i++)
       close_and_clear(&t->entries[i]);
   ```

   to

   ```c
   for (size_t i = 3; i < NX_HANDLE_TABLE_CAPACITY; i++)
       close_and_clear(&t->entries[i]);
   ```

5. **`sys_fork` extends 7.6a's inheritance to
   HANDLE_CONSOLE.**  Slice 7.6a only retained CHANNEL
   handles across fork; HANDLE_CONSOLE needs the same
   treatment so child processes start with stdin/stdout/
   stderr (and any redirections via dup3) intact.
   HANDLE_CONSOLE is a singleton with no real refcount
   — the inheritance is just type+rights+object-pointer
   copy via `nx_handle_alloc` in the child's table.

6. **Test updates.**

   Strategy: keep `nx_handle_alloc`'s scan starting at
   slot 0 (so pure unit tests of the allocator continue
   to assert "first allocation lands at slot 0 with
   encoded handle 1").  The process-level reservation
   comes from the console pre-population done by
   `nx_process_create`, which the unit tests don't
   invoke.  This keeps the handle-allocator API neutral
   and pushes the POSIX-fd convention up to the process
   layer where it belongs.

   Tests that DO go through `nx_process_create` (e.g.
   fork/exec ktests, pipe ktests) need their hardcoded
   handle-value expectations bumped: pipe returns
   encoded {4, 5} not {1, 2}; the first user-allocated
   handle in a process is encoded handle 4 = slot 3.

   Concretely:
   - `test/host/handle_test.c` — no changes (operates on
     fresh tables, not process tables).
   - `test/kernel/ktest_handle.c` — same, no changes.
   - `test/kernel/ktest_channel.c`, `test/kernel/
     ktest_posix_pipe.c`, etc. — adjust hardcoded
     expectations if they reference specific encoded
     handle values for first allocations (most use the
     value the syscall returned, so they're symbol-
     value-neutral and need no update).

7. **Re-enable pipe ktest.**  Move
   `ktest_posix_busybox_sh_pipe.c` back into `KTEST_C`
   (delete the `#`-comment block in the Makefile).  Run
   `make test`; expect `[bbsh-pipe-ok]` status 0.  If
   cat still hangs after the console + slot fix (the
   sendfile-fast-path failure noted in 7.6d.N.6a),
   spin a 7.6d.N.6c follow-up: either map `__NR_fstat
   (80)` to a stat-by-handle path that returns
   `S_IFIFO` for HANDLE_CHANNEL (so cat skips sendfile
   on the pipe), or stub `__NR_sendfile (71)` → -ESPIPE
   so cat falls back to read-loop cleanly.

The new `NX_EINVAL = -1` ↔ Linux EPERM = -1 collision
noted in 7.6d.N.6a stays as a separate opportunistic
cleanup — change the kernel's "unknown syscall" return
from `NX_EINVAL` to `-ENOSYS = -38`.  Lands on its own
(no test impact; just makes diagnostic messages stop
lying).

**7.6d.N.6+ — Path to N.final** (DONE through N.11 as
of session 68; the rest is the planned escalation toward
the interactive shell).  Five intermediate sub-slices cover
the kernel-composition gap before tackling the
UART-RX/blocking-console-read/terminal-mode block.  Two are
blocking (signal handler dispatch, tolerable-stub sweep);
two are quality-of-coverage and skippable if N.final is the
only goal; one is technical-debt paydown flagged after N.11.

**7.6d.N.12 — sigaction + rt_sigprocmask + signal-handler
dispatch.**  ash unconditionally installs SIGINT/SIGCHLD
handlers during interactive startup (and during interactive
trap-builtin in non-interactive mode); without
`rt_sigaction`/`rt_sigprocmask` returning 0 plus a real
kernel→user-handler trampoline, ash bails before the prompt
at N.final and `trap` builtin no-ops in `sh -c`.  Workload:
`busybox sh -c "trap 'echo bye' EXIT; echo body"` — `trap`
builtin uses sigaction underneath; EXIT pseudo-signal is
dispatched by ash itself at exit so it forces real handler
dispatch.  Production: new `NX_SYS_RT_SIGACTION` +
`NX_SYS_RT_SIGPROCMASK` (~80 lines combined); per-process
`sigaction[NSIG]` table (extends slice 7.5's
`pending_signals` mask with per-signal handler/flags);
kernel→user trampoline in `framework/syscall.c` that pushes
a synthetic stack frame on return-to-EL0 when a pending
signal has a handler installed.  Real kernel work, ~200
lines.  Builds the dispatch hook that N.final's Ctrl-C path
will reuse.

**7.6d.N.13 — Tolerable-syscall stubs sweep.**  Maps the 9
remaining stub-only syscalls (sigaction family already in
N.12).  Workload: `busybox sh -c "id; uname -a"`.
`set_tid_address (96)`, `getuid (174)` / `geteuid (175)` /
`getgid (176)` / `getegid (177)`, `setuid (146)` /
`setgid (144)`, `getpid (172)` / `getppid (173)`,
`uname (160)`.  All ~5–10 line stubs; total ~80 lines + 9
musl-translation entries.

**7.6d.N.14 — First 3-stage pipe** (quality-of-coverage,
skippable).  Workload: `busybox sh -c "echo hello |
tr a-z A-Z | wc -c"`.  Verifies CHANNEL→CHANNEL→CONSOLE at
three stages; both `tr` and `wc` exercise the
read-loop-on-CHANNEL-stdin pattern in distinct consumer
roles.  New kernel composition vs. N.6b's two-stage.
Likely clean first try (compose-only).

**7.6d.N.15 — Cross-process FILE-fd-through-fork**
(quality-of-coverage; deferred since slice 7.6a, finally
surfaced).  Workload: `busybox sh -c "exec 3< /banner;
head <&3"` — ash opens fd 3 in the parent, then `exec`
replaces the image with `head` which inherits fd 3 and
dup2s onto stdin.  N.9–N.11 all dodged this path because
ash's natural redirect idiom does the open in the post-fork
child.  Production: new `nx_vfs_ops.dup` (~30 lines
vfs_simple + ~30 lines ramfs) that allocates a new
`struct ramfs_open` with the same backing file but an
independent cursor + refs=1; `sys_fork`'s handle-table walk
grows a HANDLE_FILE branch that calls `vops->dup`.  ~50 lines.

**7.6d.N.16 — Drop in-handle generation encoding**
(technical-debt paydown surfaced by N.11; skippable if no
other workload hits the band-aid).  Replace
`(generation << 8) | (idx + 1)` with `idx + 1`; encoded
handles become POSIX-style "lowest available fd" exactly.
Per-slot in-table `entry.generation` stays purely for
stale-handle detection — the lookup path compares
`entry.generation` against a kernel-side side channel NOT
against bits in the encoded value.  Removes the N.11 fcntl
high-arg fallback; matches Linux's "reuse small fd numbers
across close+open" behaviour.  Slice 5.3's stale-handle
invariants restated against the new encoding.  ~30 lines +
slice-5.3 test churn.

**7.6d.N.final — Interactive `busybox sh` over UART**
(sub-slices a/b/c-minimal/d/e closed in Sessions 70–72;
original spec archived in
[HANDOFF-ARCHIVE.md](HANDOFF-ARCHIVE.md#slice-76dnfinal--original-spec)).
Only `c-full` remains forward-looking.

**7.6d.N.final.e — Visible interactive prompt (Session 72,
done).**

`make run-busybox` typed bytes through correctly after Session
71 but ash's prompt was invisible — the headline shell worked
in spirit but felt broken in practice.  Root cause traced
through ash + lineedit + musl stdio: with
`CONFIG_FEATURE_EDITING=y`, ash's prompt write goes through
`libbb/lineedit.c:put_prompt_custom` which calls
`fputs_stdout(prompt)` *without* an `fflush`.  musl's stdout
is line-buffered when `isatty(1) == 1` (.lbf = '\n'), and
ash's prompt has no trailing newline — the bytes sit in the
FILE buffer until the first newline-terminated write flushes
them retroactively (typically the next command's output).
The non-EDITING path at `lineedit.c:3027` already calls
`fflush_all()` after `fputs_stdout(prompt)`; the EDITING path
is the only one missing it.

Patch (one line, plus a 9-line nonux comment block matching
the musl-patch convention used in
`third_party/musl/arch/aarch64/syscall_arch.h`): add
`fflush(stdout);` after `fputs_stdout(...)` in
`put_prompt_custom`.  Use `fflush(stdout)` rather than
`fflush_all()` because `fflush(NULL)` *also* discards
unread bytes in any stdin FILE buffer (glibc's POSIX mode
does this; musl's behaviour is implementation-defined),
which would race against the trickle-fed test harness.

Side fixes in `test/interactive/run.sh`: switch the trickle
from line-burst to per-byte (50 ms gap) because QEMU's
`-serial stdio` chardev does not backpressure when PL011's
16-byte hardware RX FIFO is full — full-line bursts like
`echo PIPE_INPUT | tr a-z A-Z\n` (29 B) silently drop
characters at the host/guest boundary.  The previous
line-burst trickle worked in slice .d only because baseline
busybox ran without a visible prompt and the test only
grep'd for the final pipe output (not echoed input bytes);
once the prompt is visible, the typed-input echo doubles
the UART traffic per line and exposes the FIFO gap.  New
`test/interactive/visible_prompt.{script,expected}`
regression: feed `exit` and assert `# ` (the busybox prompt
prefix) appears in the captured log.

`make test-interactive` 3/3 pass (visible_prompt, echo_hello,
echo_pipe), stable across 3× consecutive runs.  `make test`
unchanged at 432/432 — no kernel changes.

**7.6d.N.final.c-full — Real signal-handler dispatch
trampoline (deferred — focused future session).**

Session 71 shipped the v1 polled approximation
(`c-minimal`): Ctrl-C posts SIGTERM to all ACTIVE non-kernel
processes; Ctrl-D arms a one-shot EOF flag.  This kills the
whole shell session rather than just the foreground command,
because v1 has no per-process signal handlers and no process
groups.

`c-full` promotes `sys_rt_sigaction` from no-op stub to
actually-installs-handler (per-process `sigaction[NSIG]`
table).  On EL0-return, when a pending signal has a handler
installed:

1. Save the trap frame to a slot at the top of the EL0 user
   stack.
2. Set `tf->elr = sa_handler_pc; tf->x[0] = signo;
   tf->sp = sp_after_frame; tf->lr = sigreturn_trampoline_pc`.
3. `sigreturn_trampoline_pc` is a tiny EL0-side stub (single
   instruction `svc #0` with x8 = `NX_SYS_RT_SIGRETURN`)
   embedded in the user-window backing at a known offset by
   `sys_exec` / `nx_process_create`.  Or use musl's
   `sa_restorer` if present.
4. New `NX_SYS_RT_SIGRETURN` restores the saved trap frame
   and re-enters the interrupted code.

Switch Ctrl-C from "kill everyone" to "post SIGINT, deliver
via handler — ash's installed handler then resets line-edit
state and re-prompts."  Estimated ~200–300 lines kernel +
ktest using mock-RX 0x03 injection that confirms an EL0
handler actually ran.

Best done in a dedicated session — the trap-frame
manipulation, AAPCS preservation across handler invocation,
sigreturn semantics from user-stack-corrupted state, and
re-entrancy concerns each want careful focus.

#### Slice 7.7 — Interactive smoke tests

Validate the claim.  Sub-sliced into 7.7a (trivial — no kernel
changes; the workloads work via slice-7.6d kernel surface already)
and 7.7b (mkdir + ps — needs new kernel surface).

1. In the booted shell, run `ls /` and confirm ramfs contents.
2. `echo hello | cat` — pipe works end-to-end.
3. `ps` — at least init + shell appear.
4. `mkdir /tmp` + `ls /tmp` — directory ops.
5. A scripted test harness drives the shell over the serial
   console (`tools/qemu-stdin-feed.sh` from slice 7.6d.N.final.d)
   and compares captured output against expected (substring grep,
   not full-line diff — kernel boot dump shares the UART stream).

**Exit criteria:** All four smoke commands pass in the scripted
harness.

##### Slice 7.7a — Trivial smoke tests (`ls /`, `echo hello | cat`)

Both workloads already work end-to-end at the ktest level (slices
7.6d.N.5 and 7.6d.N.6b respectively).  This sub-slice just
promotes them into the interactive harness shipped in slice
7.6d.N.final.d:

- `test/interactive/ls_root.{script,expected}` — drives `ls /`,
  asserts ramfs entry names appear in the captured log.  Caveat:
  ls's coloured columnar output (`\x1b[1;32m...\x1b[m` around each
  name) doesn't break substring grep.  Choose unique-marker entries
  for the expected file (`argv_child`, `musl_prog`, `banner` —
  `init`/`busybox`/`cat` collide with kernel boot tags).
- `test/interactive/echo_cat.{script,expected}` — drives
  `echo MARKER | cat`, asserts MARKER appears.  Tests the headline
  two-process pipe machinery from slice 7.6d.N.6b end-to-end over
  UART.

No kernel changes.  Exit: `make test-interactive` 5/5 pass (was
3/3); `make test` unchanged at 432/432.

##### Slice 7.7b — `mkdir /tmp` + `ls /tmp` + `ps`

Real kernel work — split into 7.7b.1 (mkdir + paths, **closed in
Session 74**) and 7.7b.2 (ps, **next**).

###### Slice 7.7b.1 — `mkdir` + hierarchical paths in ramfs (CLOSED, Session 74)

Landed:
- New `nx_vfs_ops.mkdir` + `nx_vfs_ops.stat` ops; `nx_vfs_ops.readdir`
  signature gains a `dir_path` argument.  `struct nx_fs_stat = {kind,
  size}` with `NX_FS_KIND_FILE` / `NX_FS_KIND_DIR`.
- New `NX_SYS_MKDIRAT = 40` syscall; musl `__NR_mkdirat = 34 → 40` in
  both `arch/aarch64/syscall_arch.h` and `src/thread/aarch64/syscall_cp.s`.
- ramfs gains a `kind` field per entry; new `ramfs_op_mkdir` +
  `ramfs_op_stat`; rewritten `ramfs_op_readdir` for hierarchical
  projection with O(cookie²) dedup against earlier in-use entries;
  `ramfs_op_open` rejects DIR entries with `NX_EPERM`
  (defence-in-depth).
- Synthesise-on-prefix in `ramfs_op_stat` so any entry with `/bin/X`
  makes `/bin` report as DIR without an explicit entry — saves the
  cpio loader from having to emit per-prefix dir entries; preserves
  `RAMFS_MAX_FILES` budget.
- `sys_open` swaps the slice-7.6d.N.5 `"/"`-only HANDLE_DIR shortcut
  for a generic `vops->stat` probe — any DIR path now allocates
  HANDLE_DIR with `cur->path` set.  `sys_fstatat` collapses its
  open-and-seek dance into one stat call.
- `sys_getdents64` passes `cur->path` to `vops->readdir` and **drops
  the leading-`/` strip hack at `framework/syscall.c:1925-1926`**
  (slice 7.6d.N.5 cleanup target retired).
- `sys_readdir` (legacy) hard-codes `dir_path = "/"`.
- busybox `mkdir` applet wired in via `$(BUSYBOX_BIN):/bin/mkdir`.
- New kernel ktest `posix_busybox_sh_mkdir`: `mkdir /m && > /m/x &&
  ls /m`; asserts `/m` is DIR and `/m/x` is FILE via vfs.
- New interactive script `mkdir_tmp.{script,expected}` (`mkdir /tmp;
  > /tmp/MKDIR_TMP_MARKER; ls /tmp`).
- 6 new conformance / ramfs-specific host TEST cases.

Test totals after 7.7b.1: `make test` 439/439 (was 432);
`make test-interactive` 6/6 (was 5/5).

###### Slice 7.7b.2 — `ps` via procfs (CLOSED, Session 75)

Closes Phase 7's slice-7.7 exit criteria — busybox's stock `ps`
applet runs end-to-end with no vendored patch and prints rows for
the kernel + init + the forked-ps process.

Landed:

1. `components/procfs/procfs.c` (~370 lines + manifest) implements
   `nx_fs_ops` against the live process table:
   - `/proc` (DIR) — readdir lists pid basenames via the new
     `nx_process_for_each` iterator wrapped in a `find_nth(idx)`
     callback pattern.
   - `/proc/<pid>` (DIR) — contains the single child `stat`.
     Trailing-slash variant `/proc/<pid>/` resolves identically;
     critical because busybox's `procps_scan` calls `stat("/proc/
     %u/", &sb)` for the UIDGID column.
   - `/proc/<pid>/stat` (FILE) — Linux-shape 24-field stat line
     synthesised on `open` (per-open buffer, refcounted).  Format:
     `<pid> (<comm>) <state> <ppid> 0 0 ...` covering the slow-path
     fields busybox parses (state `R` for ACTIVE, `Z` for EXITED;
     comm wrapped in parens).
   - Read-only: `mkdir` / `write` / `O_WRITE`-on-open all return
     `NX_EPERM`.
2. New public iterator `nx_process_for_each(cb, ctx)` in
   `framework/process.{h,c}`: visits `g_kernel_process` first, then
   `g_process_table[]` in slot order; cb returns non-zero to stop
   early, return value mirrors the stop code.
3. `vfs_simple` extended with two changes pulled in by serving
   multiple mounts:
   - **Mount table** (`mount_for_path` + `resolve_for_path`):
     `/proc` and `/proc/...` route to `filesystem.proc`;
     everything else stays on `filesystem.root`.  Hard-coded
     rather than table-driven (one non-root mount in v1; promotion
     to a real config-driven table waits for the second mount).
   - **Per-open wrapper** (`struct vfs_simple_open`, pool of 108):
     records the mount slot *name* (string-literal pointer)
     alongside the driver per-open.  Every op re-resolves the
     slot via `nx_slot_lookup` so the late-binding semantics from
     DESIGN §Slot-Based Indirection survive (an `nx_slot_swap
     (slot, NULL)` mid-run causes subsequent reads to fail
     cleanly with `NX_ENOENT`).  Refcount discipline:
     wrapper.refs and driver per-open refs move in lockstep so
     the wrapper is freed exactly when its driver per-open is.
     `procfs` grew a `procfs_op_retain` to keep the lockstep
     invariant clean alongside ramfs's existing one.
4. `kernel.json` gains `filesystem.proc ← procfs`; gen-config
   auto-derives slot iface = `filesystem` from the dotted name.
5. `INITRAMFS_ENTRIES` adds `$(BUSYBOX_BIN):/bin/ps` (16 entries
   in `initramfs-busybox.cpio` total).
6. `test/host/Makefile` picks up `components/procfs/procfs.c` so
   the host build stays clean (no host conformance test for procfs
   in this slice — synthesised + read-only doesn't fit the standard
   write-then-read fixture).

Tests:
- New `test/kernel/ktest_procfs.c` (8 ktests): slot binding, stat
  shape including the trailing-slash variant, readdir of `/proc`
  yields pid 0 first, end-to-end open+read of `/proc/0/stat` with
  the rendered prefix `"0 (kernel) R 0 "`, mount boundary
  (`/banner` still routes to ramfs).
- New `test/interactive/ps_smoke.{script,expected}`: feeds
  `ps\nexit\n`; expected matches `PID   USER     TIME  COMMAND`,
  `[kernel]`, `[init]`.  Tightened from the original `init` only
  draft because the kernel boot message `[init] entering busybox
  sh at EL0` would false-positive on the bare substring (same
  lesson as Session 73).

Result: `make test` → **447/447** (was 439; +8 procfs ktests);
`make test-interactive` → **7/7** (was 6/6).  busybox falls back
to bracketed-comm rendering when cmdline is empty (we don't
implement `/proc/<pid>/cmdline` in v1; `read_cmdline` tolerates
`sz <= 0`); ps prints `[kernel]`, `[init]`, `[init]` (the third
is the forked-ps process — fork inherits parent's name in v1,
exec doesn't update it).

#### Slice 7.8 — Wait queues + `poll`/`ppoll`

Foundation slice — replaces three v1 yield-loop fakes with a
proper kernel blocking primitive, and lands `NX_SYS_PPOLL` on
top of it.  Sequenced *after* 7.7 so we don't half-migrate the
existing yield-loop call sites mid-flight.

**Why this is its own slice (not bundled with c-full or 7.7):**
The yield-loop pattern in `sys_read` CHANNEL arm (slice
7.6d.N.6b), `sys_wait` (slice 7.4b), `nx_console_read` (slice
7.6d.N.final.a) is the same shortcut in three places; cleaning
it up needs a real abstraction.  Real `poll/ppoll` gives any
future EL0 program (not just busybox) the right blocking
semantics, and unblocks `c-full` (signal trampoline can wake
blocked syscalls cleanly via `EINTR`) and Phase 8 drain points.

**7.8a — Wait-queue primitive + `NX_TASK_BLOCKED` state**
**(closed — Session 76, 2026-04-29).**  Real cost: ~190 lines
kernel + ~210 lines ktest.  See [logs/session-76-7.8a-waitq-primitive.md](logs/session-76-7.8a-waitq-primitive.md)
for the full bring-up notes.

- New `core/sched/waitq.{h,c}`: `struct nx_waitq` (intrusive
  list of `struct nx_task *`); `nx_waitq_init`,
  `nx_waitq_wait_with_deadline`, `nx_waitq_wake_one`,
  `nx_waitq_wake_all`, `nx_waitq_tick_deadlines`.  Deadline
  plumbed through `core/cpu/monotonic.h` (same `nx_deadline`
  primitive slice 3.9b.2's `pause_hook` uses).
- `struct nx_task` grew 5 wait-state fields (`wait_q`,
  `wait_deadline`, `wait_has_deadline`, `wait_woken`,
  `deadline_node`); `sched_node` is reused for the waitq
  linkage so a task is on either the runqueue or
  `wq->waiters`, never both.  Indefinite waiters
  (`budget_ns == 0`) skip the global `g_deadline_list` so the
  per-tick deadline walk is O(N_deadline_waiters), not
  O(N_blocked).
- `NX_TASK_BLOCKED` was already in `enum nx_task_state` from an
  earlier slice as a placeholder; this slice activates it.
  `sched_rr_pick_next` defensively walks past `NX_TASK_BLOCKED`
  tasks — in normal flow they're already off the runqueue, but
  the guard prevents a future caller's missed-dequeue bug from
  livelocking the system.
- `wake_one`/`wake_all` are ISR-context-safe via DAIF.I
  save/restore (kernel-only `mrs/msr daif` asm; host stubs).
  `nx_waitq_tick_deadlines` is called from `sched_tick` (after
  `g_sched_ops->tick`) and re-enqueues any waiter whose deadline
  has elapsed, leaving `wait_woken = 0` so the wait function
  returns `NX_EDEADLINE`.
- Reused `NX_EDEADLINE = -9` from `framework/registry.h` for the
  timeout return rather than introducing `NX_ETIMEDOUT`; the
  existing comment "`pause_hook` (or similar) exceeded its
  wall-clock budget" is generic enough to cover waitq-style
  timeouts.
- 5 universal ktests in `test/kernel/ktest_waitq.c`:
  `waitq_wake_one_releases_one`,
  `waitq_wake_all_releases_all`,
  `waitq_wait_returns_on_deadline` (50 ms budget without wake),
  `waitq_wake_after_wait_no_lost_wakeup`,
  `waitq_multi_waiter_fifo_order`.  Test harness pattern:
  spawn-yield-spawn waiters, `WAIT_FOR(cond, budget)` macro
  yields in a bounded loop until each waiter parks, then issue
  wake(s) and yield until completion is observed.  Reap path
  mirrors slice 4.4's pattern (`ops->dequeue` +
  `nx_task_destroy`).
- No production callers yet — the existing yield-loop call
  sites (`sys_read` CHANNEL arm slice 7.6d.N.6b, `sys_wait`
  slice 7.4b, `nx_console_read` slice 7.6d.N.final.a) are
  migrated in slice 7.8c.  `make test` 447 → **452/452 pass**;
  `make test-interactive` unchanged at **7/7**.

**7.8b — `NX_SYS_PPOLL` built on waitqs**
**(closed — Session 77, 2026-04-29).**  Real cost: ~450 lines
across kernel + musl + posix_shim + EL0 test (vs. the plan's
~150 estimate; the listener mechanism is duplicated between two
object types and a thin POSIX-shim wrapper grew alongside the
syscall).  See [logs/session-77-7.8b-ppoll.md](logs/session-77-7.8b-ppoll.md)
for the full bring-up notes.

- New `framework/pollset.h` defines a shared
  `struct nx_pollset_listener { struct nx_list_node node;
  struct nx_waitq *waitq; }` + an inline `nx_pollset_wake_all
  (list)` that walks a per-object listener list and calls
  `nx_waitq_wake_all` on each entry's parent waitq.  Used to be
  per-handle-type `add_to_pollset(handle, pollset)` ops in the
  plan; ended up cleaner as a shared struct that channel +
  console both use.
- `nx_channel_endpoint` grows `pollset_listeners` (initialised
  in `nx_channel_create`); `framework/console.c` gains a
  singleton `g_console_pollset_listeners` (statically self-
  pointing).  Three new public functions per object:
  `register_pollset(listener)` / `unregister_pollset(listener)`
  / `readiness(want) → short`.  FILE / DIR are always-ready (no
  listener registration in `sys_ppoll`).
- `sys_ppoll` (~140 lines) validates `nfds <= NX_PPOLL_MAX_FDS
  = 32`; decodes optional `struct timespec` (16 bytes; budget =
  `tv_sec*1e9 + tv_nsec`); walks handles with the STDIN
  encoded-fd-0 → slot 2 special-case mirroring `sys_read`;
  registers listeners BEFORE the initial readiness check
  (lost-wakeup-safe — a wake during the registration→check
  window reaches the empty pollset waitq as a no-op, but the
  underlying state is mutated, so the readiness check catches
  it); blocks via `nx_waitq_wait_with_deadline` if nothing
  ready and the timeout isn't zero; recomputes readiness on
  wake (NX_OK) or deadline expiry (NX_EDEADLINE); copies
  revents back via `copy_to_user`.  v1 ignores `sigmask` +
  `sigsetsize` — same posture as our other signal stubs.
- Producer-side wakes:
  - `nx_channel_send` after pushing to peer's ring → wake
    peer's listeners (peer-readable transition).
  - `nx_channel_endpoint_close` after the last close → wake
    both endpoints' listeners (peer-closed → POLLHUP on the
    surviving side; the closing side's listeners covered for
    a race).
  - `nx_console_rx_isr` after draining the PL011 FIFO → wake
    if a real byte was pushed OR a Ctrl-D EOF was armed
    (Ctrl-C alone does not change read-readiness so does not
    wake).
  - `nx_console_test_inject_bytes` + `nx_console_test_inject_eof`
    for host-test parity.
- musl translation: `case 73: return 41;` in
  `arch/aarch64/syscall_arch.h` + `cmp x1, #73; b.eq .Lnx_ppoll`
  + `.Lnx_ppoll: mov x8, #41; b .Lnx_run` in
  `src/thread/aarch64/syscall_cp.s`.  `make musl-libc` then
  `touch third_party/musl/lib/libc.a` + `make busybox` to
  relink (lesson from slice 7.7b.1's bring-up).
- POSIX shim (`components/posix_shim/posix.h`) gains a 4-arg
  `nx_posix_svc4` helper + `nx_posix_ppoll` wrapper + matching
  `nx_posix_pollfd` / `nx_posix_timespec` / `NX_POSIX_POLL*`
  constants so EL0 programs can call ppoll without going
  through musl.
- +1 EL0 ktest in `test/kernel/ktest_posix_ppoll.c` runs three
  subtests (initial-ready / 50 ms deadline-expiry / peer-close
  hangup) with discrete failure exit codes (1..11).  Matching
  ktest waits for EXITED via `nx_task_yield(); asm("wfi")`
  loop — yields alone are insufficient because they don't
  advance the wall clock; `wfi` lets the CPU sleep until the
  next timer tick, which is what advances `nx_deadline`'s
  reference.
- Bring-up gotchas: (a) first-iteration `sys_ppoll` called
  `nx_task_current()` — host build doesn't include
  `core/sched/task.h` in syscall.c, switched to
  `nx_process_current()` (always included); (b) `NX_SYSCALL_COUNT`
  bump 41 → 42 needed an explicit `rm test/kernel/ktest_syscall.o`
  because the Makefile's header-dep tracking doesn't catch
  enum-value bumps (the test's "out of range" probe was using
  the cached value 41, now `NX_SYS_PPOLL` returning -EINVAL for
  trash args); (c) `KASSERT_EQ_U(g_ppoll_entry, mmu_user_window
  _base())` was too strict — my ELF has `nx_posix_exit` placed
  at offset 0 (compiler ordered the inline-helper before
  `_start`), so `e_entry == 0x4800000c`; relaxed to "entry
  inside the user window".
- `make test` 452 → **453/453**; `make test-interactive`
  unchanged at **7/7**.  busybox's intended `ask_terminal()`
  `\e[6n` + `fflush_all()` flush path now runs end-to-end
  because `safe_poll(stdin, 0) == 0` succeeds; the slice
  7.6d.N.final.e `fflush(stdout)` patch in
  `libbb/lineedit.c:put_prompt_custom` is still in place (slice
  7.8c reverts it).

**7.8c — Migrate existing yield-loops + revert slice
7.6d.N.final.e patch**
**(closed — Session 78, 2026-04-29).**  Real cost: ~140 lines
added (kernel + new `wait_unless` variant), ~70 lines deleted
(busybox patch + the three yield-loop bodies).  See
[logs/session-78-7.8c-yield-loop-migration.md](logs/session-78-7.8c-yield-loop-migration.md)
for the full bring-up notes.

- New `nx_waitq_wait_unless(wq, budget_ns, pred, ctx)` extends
  the slice 7.8a primitive: re-evaluates `pred(ctx)` *inside*
  the IRQ-disabled + preempt-disabled critical section that
  registers the caller on `wq`.  If it already returns
  non-zero, the caller is NOT enqueued and the function
  returns `NX_OK` immediately (treats the predicate as an
  already-fired wake).  Lost-wakeup-safe: a wake that fires
  between the caller's outer `cond` check and the inner
  predicate check is caught by the predicate.  Internal
  restructuring extracts the shared body into
  `waitq_register_and_wait_locked` — both
  `wait_with_deadline` + `wait_unless` call it.
- `struct nx_process` grows `struct nx_waitq exit_waitq`
  (initialised in `nx_process_create`; statically self-
  pointing for `g_kernel_process`).  `nx_process_exit` walks
  up to `parent_pid` and calls `nx_waitq_wake_all` on the
  parent's `exit_waitq` after handle-table cleanup but before
  the terminal `wfe` loop.  Processes spawned outside fork
  (`parent_pid == 0`) skip the wake.
- `sys_read` CHANNEL arm: replaced `nx_task_yield()` with
  `nx_waitq_wait_unless(&read_wq, 0, sys_read_channel_ready_pred,
  obj)` against a kstack `nx_waitq` + listener pair registered
  on the endpoint's `pollset_listeners` list (slice 7.8b
  infrastructure reused).  No producer-side wake additions
  needed — `nx_channel_send` and `nx_channel_endpoint_close`
  already fan out via `nx_pollset_wake_all`.  Predicate covers
  the "would not return EAGAIN" condition exactly.
- `sys_wait`: replaced both yield-loops (`pid != -1` + `pid ==
  -1` POSIX waitpid-any) with `nx_waitq_wait_unless(&caller->
  exit_waitq, 0, pred, ctx)` against the caller's per-process
  exit_waitq.  Two static-inline predicates above sys_wait
  (target-specific + any-child); both `#if !__STDC_HOSTED__`
  guarded.
- `nx_console_read`: same kstack waitq + listener pattern as
  `sys_read` CHANNEL, registered on `g_console_pollset_listeners`.
  Predicate covers ring-non-empty OR Ctrl-D EOF armed.  The
  Ctrl-D early-return path explicitly unregisters the listener
  before returning 0 (otherwise the per-object list would be
  left pointing at a kstack address about to be invalidated).
- **Reverted the slice 7.6d.N.final.e busybox patch** in
  `third_party/busybox/libbb/lineedit.c:put_prompt_custom`
  (~25 lines including comment block deleted).  Function
  reverts to upstream form.  Prompt visibility now comes from
  busybox's intended `ask_terminal()` `\e[6n` + `fflush_all()`
  path, which works once `safe_poll(stdin, 0) == 0` returns
  success via the slice 7.8b `NX_SYS_PPOLL`.  The
  `test/interactive/visible_prompt.{script,expected}`
  regression script stays as a guard.
- Each yield-loop site uses its own private kstack waitq +
  listener rather than a per-endpoint shared waitq —
  intentionally avoids the thundering-herd issue (a shared
  per-endpoint waitq would wake N concurrent readers for one
  byte; the per-listener fan-out wakes only the active
  waiters).
- `make test` 453/453 (51 python + 283 host + 119 kernel —
  same totals as Session 77; the migration adds no new tests
  but verifies every existing case still passes); `make
  test-interactive` 7/7 — visible_prompt PASSES with stock
  upstream busybox, proving busybox's intended ask_terminal
  flush path runs end-to-end via NX_SYS_PPOLL.
- **Closes Phase 7.**  No more yield-loops in the kernel; no
  more vendored-busybox local diffs.

### Phase 8: Runtime Recomposition and Config Manager

**Goal:** Swap components at runtime. Change IPC modes on the fly.
**Status:** **CLOSED** (Session 104) — all 15 slices (Groups A, B, C) landed.  Plan Session 79; IDL schema Session 80; slot-call API Session 81; Group A Sessions 82–85; Group B Sessions 86–97; Group C Sessions 98–104.  `make test-kernel` **141/141**; `make test-host` **459/459**; 0 drift; 0 findings.

Phase 8 splits into three groups (15 slices total).  The original Phase 8 deliverables (live hot-swap, per-edge mode switching) are **Group C**; Groups A and B exist because today's production paths don't actually flow through the slice-3.8 IPC router — they reach into `slot->active->descriptor->iface_ops` directly (~184 sites across 19 files, all five production components have `handle_msg = NULL`).  A live recompose against that surface is a use-after-free.  Group A builds the generator that makes the migration tractable; Group B routes every cross-component call through `nx_slot_call_sync`; Group C then ships the original Phase 8 deliverables on top.

#### Architectural decisions (locked in at Session 79)

- **Option A (full IPC routing) over pause-aware borrow.**  Long-term DESIGN.md alignment.  Cost absorbed by the generator.
- **JSON IDL per interface, parsed by `tools/gen-iface.py`.**  Per-op metadata (pause-policy override, sync/async preference, trace tags) has no natural home in a C `struct ops`.
- **Sync-mode v1.**  Caller blocks on its own stack; async-mode-per-edge is forward-compatible via DESIGN's existing `mode = IPC_SYNC|IPC_ASYNC` parameter and lands later.
- **Sync-during-pause = block on `slot->resume_waitq`** (slice 7.8a primitive), not literal message buffering.  Async edges keep DESIGN's buffer semantics.  DESIGN.md gets a one-section update with slice 8.0a.
- **Per-op tagged message structs** over generic argv.  Easier to debug, easier for hooks to introspect.

#### Group A — Generator infrastructure (4 slices)

| Slice | Deliverable |
|---|---|
| **8.0pre.1** ✓ | **Landed Session 82.**  `tools/gen-iface.py` (~600 lines) emits four artifacts per IDL: typedef header, msg structs, sender wrappers, dispatch template.  Meta-schema gained one field (`forward_decls_doc`); the *set* of forward declarations is auto-detected by walking op-param `ctype` for `^(struct\|union\|enum)\s+\S+$`.  `interfaces/idl/vfs.json` enriched with verbatim hand-written prose; type-mapping convention codified (`u32 → uint32_t` but `i32 → int`, asymmetric).  `interfaces/vfs.h` canonicalized to generator output (DESIGN.md R7 — IDL is source of truth post-cutover; C-level shape unchanged so callers compile identically).  Makefile: `gen-iface` + `verify-iface-fresh` targets, both hooked into `all` and `test`.  18 new tools tests (`tools/tests/test_gen_iface.py`).  `make test` 453 → **471/471**; `make verify-iface-fresh` clean.  Generated `_msg.h` / `_call.h` / `_dispatch.h` not yet included by production code.  See [Session 82 log](logs/session-82-8.0pre.1-idl-generator.md). |
| **8.0pre.2** ✓ | **Landed Session 83.**  Second IDL (`fs`).  Data shapes split out to a hand-written `interfaces/fs_types.h` (carrying `struct nx_fs_dirent`, `struct nx_fs_stat`, and the constants tightly bound to them — `NX_FS_DIRENT_NAME_MAX`, `NX_FS_KIND_*`); shared between fs.h and vfs.h via the IDL's existing `includes:` array.  Generator extension: emit author-declared `includes:` in **both** typedef header and msg header (msg header embeds struct values by sizeof, needs the full def); suppress auto-detected forward declarations when `includes:` is non-empty (the includes carry the definitions).  No new schema fields — the IDL describes operations only; data layout stays in C.  `interfaces/fs.h` canonicalized to byte-equal generator output (per-value SEEK trailing comments folded into group doc; section divider comments deleted).  +7 tests (`TestFsByteForByte`, `TestIncludesAuthorDeclared`).  `make test` 471 → **478/478**; `make test-interactive` **7/7**; `make verify-iface-fresh` clean.  See [Session 83 log](logs/session-83-8.0pre.2-fs-iface.md). |
| **8.0pre.3** ✓ | **Landed Session 84.**  Three remaining IDLs (`scheduler`, `mm`, `char_device`) + IRQ-entry codegen.  Schema extension: optional `ctype` on `opaque_self_handle` params (e.g. `struct nx_task *task`); optional `ctype` on `void_ptr` returns (e.g. `pick_next` returning `struct nx_task *`); new `u32`/`u64` return types (mm.max_order canonicalizes `unsigned` → `uint32_t`).  Generator extension: emit `framework/<iface>_isr.h` for IDLs with any `context: "irq"` op (declarations + `NX_<IFACE>_ISR_POOL_SIZE`; bodies in 8.0a).  `interfaces/scheduler.h` + `interfaces/mm.h` canonicalized to byte-equal generator output; `interfaces/char_device.h` is a brand-new file authored fresh from IDL (no hand-written predecessor) — first to exercise IRQ-entry shape via `rx_byte`.  +17 tests (`TestSchedulerByteForByte`, `TestMmByteForByte`, `TestCharDeviceByteForByte`, `TestIrqEntryArtifact`, `TestTypedOpaqueSelfHandle`, `TestTypedVoidPtrReturn`, `TestU32Return`).  `make test` 478 → **495/495**; `make test-interactive` **7/7**; `make verify-iface-fresh` clean.  All five production interfaces are now IDL-driven; only Group A's cutover (8.0pre.4) remains.  See [Session 84 log](logs/session-84-8.0pre.3-sched-mm-char-device.md). |
| **8.0pre.4** ✓ | **Landed Session 85.**  Cutover paperwork.  Generator output declared canonical: DESIGN.md gained §"Interface Definition Language" + an updated "Build Flow" diagram that documents `make verify-iface-fresh` as a `make all` / `make test` prerequisite (already wired in slice 8.0pre.1; this slice formalizes the enforcement call-out per the original plan).  R7's table row in DESIGN.md and AI-RULES.md gained a parallel-rule note for IDL artefacts (machine-checked today via `verify-iface-fresh`, since the IDL artefacts *are* committed — unlike manifest-derived `gen/<name>_deps.h` which remains gitignored).  Latent dual-maintenance language pruned: meta-schema's `constants` description rewritten ("when the hand-written header is replaced by the generator output" → "carries the API surface that the IDL declares"); meta-schema's `includes` description expanded with the post-8.0pre.2 reality (suppresses auto-detected forward decls); `tools/README.md` dropped the "post-cutover" hedge from the gen-iface intro and the "match today's hand-written headers" type-system caveat; `tools/gen-iface.py` source comments aligned ("matches today's hand-written conventions" → "the codified rule").  No code-shape changes; no in-tree generated header was modified; `make verify-iface-fresh` clean throughout.  No new tests (the cutover is documentation/spec).  `make test` **495/495**, `make test-interactive` **7/7**, `make verify-registry` 0 findings.  See [Session 85 log](logs/session-85-8.0pre.4-cutover.md). |

**Checkpoint A:** All interfaces declared via IDL; generator is project infrastructure.

#### Group B — IPC migration (5 slices; 8.0a is now sub-sliced)

| Slice | Deliverable |
|---|---|
| **8.0a** | `framework/slot_call.{h,c}` blocking-call infrastructure.  API: **`nx_slot_call_blocking(slot, msg, reply_buf, reply_buf_len)`** (post-Session-85 refinement — reply buffer is an explicit arg pair instead of widening `nx_ipc_message`).  Full spec locked Session 81 in [SLOT-CALL-API.md](SLOT-CALL-API.md).  Resolution 1 model: dispatcher resolves `slot->active` (caller never reads it), reply-via-dispatched-message (Option β).  **Sub-sliced into 8.0a.1 → 8.0a.8** (Session 86) so each landing is a green checkpoint; the eight sub-slices share the slice-8.0a budget (~70 ktests, lands cross-cutting test infrastructure in 8.0a.8).  See sub-rows below. |
| **8.0a.1** ✓ | **Landed Session 86.**  Rename `components/posix_shim/` → `components/libnxlibc/` to free the `posix_shim` name for the kernel-side boundary component.  Path-only shift: 6 file renames, manifest name field, README cross-ref, 30 `test/kernel/*.c` `#include` updates, ~92 lines of Makefile recipe-path updates.  No semantic change; tests stay 495/495.  See [Session 86 log](logs/session-86-8.0a-kickoff.md). |
| **8.0a.2** ✓ | **Landed Session 86.**  IPC + slot scaffolding for the blocking-call infra.  Adds `NX_MSG_FLAG_REPLY_REQUESTED (1u<<2)` request flag in `framework/ipc.h`.  Adds two slot fields in `framework/registry.h`: `struct nx_waitq resume_waitq` (QUEUE-policy callers park here) + `_Atomic(uint32_t) in_flight_calls` (dispatcher-incremented; pause-drain reads it).  `nx_slot_register` now `nx_waitq_init` + `atomic_init` the new fields.  Reuses existing `NX_EABORT = -8` (no new errcode).  Pure additive scaffolding — no consumer yet; `make test` 495/495.  See [Session 86 log](logs/session-86-8.0a-kickoff.md). |
| **8.0a.3** ✓ | **Landed Session 87.**  `framework/slot_call.{h,c}` — header + skeleton body for `nx_slot_call_blocking(slot, msg, reply_buf, reply_buf_len)`.  Body returns `NX_ENOSYS` for any well-formed call (NULL-arg branch returns `NX_EINVAL`); the dispatcher round-trip body lands in 8.0a.6.  Hooks into the kernel build (`framework/slot_call.c` added to `FW_C` after `framework/ipc.c`; same insertion in `test/host/Makefile` `SRCS`); no callers yet.  Doc strings match SLOT-CALL-API.md §"API" verbatim.  Pure scaffolding; tests stay 495/495.  See [Session 87 log](logs/session-87-8.0a.3-slot-call-skeleton.md). |
| **8.0a.4** ✓ | **Landed Session 88.**  Kernel `components/posix_shim/` skeleton — `manifest.json` (4 deps: `vfs` stateful + `scheduler` / `memory.page_alloc` / `char_device.serial` stateless, all `mode: async, policy: queue` per DESIGN.md §"Sync-mode caller must be on a dispatcher"), `posix_shim.c` (init/enable/disable/destroy stubs + `g_posix_shim` singleton accessor set in init / cleared in destroy + `handle_msg` returning `NX_EINVAL` until 8.0a.6 wires the reply-routing body; no `pause_hook` since component doesn't spawn threads), `README.md`.  `kernel.json` gains `posix_shim ← posix_shim` slot binding; booted kernel composition advances to **7 slots / 7 components** with `posix_shim` ACTIVE alongside the existing six.  Auto-generated `gen/posix_shim_deps.h` lands via the new pattern rule `gen/%_deps.h: components/%/manifest.json` — first user of the gen-config DI flow on a manifest with dotted dep keys.  Two paving fixes: (1) `tools/gen-config.py` `parse_dep` now derives a C-identifier `c_field` by replacing dots with underscores (`memory.page_alloc` → `memory_page_alloc`); `render_deps_header` uses `c_field` for the struct field + `offsetof()` arg but keeps the dotted `name` literal in the descriptor for runtime `nx_slot_lookup`; (2) `tools/verify-registry.py` R2 now skips components using the gen-config DI pattern (detected via `^\s+struct\s+\w+_deps\s+\w+\s*;` regex in the .c file) — gen-config itself enforces the parity, re-checking would produce false positives on dot-flattened fields.  `test/kernel/ktest_bootstrap.c` snapshot-JSON buffer bumped 2048 → 4096 bytes.  No callers yet — `framework/syscall.c` keeps its direct `vops->read(self, ...)` calls until slice 8.0c.  Tests stay 495/495.  See [Session 88 log](logs/session-88-8.0a.4-posix-shim-skeleton.md). |
| **8.0a.5** ✓ | **Landed Session 89.**  Per-task `caller_slot` lifecycle.  `struct nx_task` gains embedded `caller_slot` + `reply_waitq` + `in_flight_reply_buf` + `in_flight_reply_buf_len` + `in_flight_reply_rc` + `caller_slot_name[24]` + `caller_slot_active` (the destroy-side gate).  `core/sched/task.h` now includes `framework/registry.h` and `core/sched/waitq.h` so the slot and waitq embed by value (the existing `struct nx_waitq;` forward decl is dropped).  `core/sched/task.c` adds three pieces: `render_caller_slot_name` (synthesizes `task#<id>` into the embedded buffer; libc-free decimal render); `wire_caller_slot` / `unwire_caller_slot` lifecycle pair; integration into `nx_task_create`, `nx_task_create_forked`, and `nx_task_destroy`.  `wire_caller_slot` does `nx_slot_lookup("posix_shim")` first — if posix_shim is absent or unbound, returns `NX_OK` without touching anything (the soft-skip lets host unit tests stay green without running framework_bootstrap).  Otherwise sets the slot's metadata (`name`, `iface = "task"`, `mutability = NX_MUT_HOT`, `concurrency = NX_CONC_SHARED`), `nx_slot_register`s, `nx_slot_swap`s to bind posix_shim's component, then walks posix_shim's outgoing edges via `nx_slot_foreach_dependency` and `nx_connection_register`s a parallel edge per dep with the same `mode/stateful/policy` (per DESIGN.md §"Edge inheritance").  Rolls back on any clone-edge failure.  `unwire_caller_slot` re-fetches the head every iteration (`nx_connection_unregister` mutates the list `nx_slot_foreach_dependency` walks).  R3 cap-scan + pause hold-queue work uniformly for blocking-call senders without per-callsite special cases.  **Tests:** +6 host (`task_test.c`: skip when posix_shim absent, register+clone when bound, edge attribute inheritance, 3-task multi-independence, partial-fixture rollback, `reply_waitq` init); +1 ktest (`bootstrap_caller_slot_create_destroy_round_trip_in_kernel`: same shape but against the live kernel composition where posix_shim is actually bound).  Existing `task_*` tests gained `nx_graph_reset()` at the top of every TEST: pre-existing component-destroy tests register stack-allocated slots and never clean up; `wire_caller_slot`'s `nx_slot_lookup` exposes the latent isolation bug because it traverses the slot list on every `task_create`.  `make test` 495 → **502/502 pass** (93 python + 289 host + 120 kernel); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.  See [Session 89 log](logs/session-89-8.0a.5-caller-slot.md). |
| **8.0a.6** ✓ | **Landed Session 90.**  Reply path end-to-end via dispatcher + posix_shim.  `framework/dispatcher.c` gains a 256-entry reply-message pool (`struct nx_reply_pool_entry { msg + payload[NX_REPLY_PAYLOAD_MAX=512] }`) backed by an atomic-bitmap allocator (lowest-zero-bit CAS, `NX_REPLY_POOL_BITMAP_WORDS = 4`).  Pool exhaustion fires `kpanic` on kernel / `abort()` on host (a dropped reply silently hangs a blocked caller — fail loud).  `nx_dispatcher_pump_once` wraps `handle_msg` in `slot->in_flight_calls++/--` (atomic acq_rel — pause-protocol drain reads this), and on `NX_MSG_FLAG_REPLY_REQUESTED` calls `build_reply(req, rc)` (allocates pool entry, zeros header + 512-byte payload, writes `nx_reply_header.rc`, inverts src/dst so the reply lands at the caller's `caller_slot`, sets `flags = NX_MSG_FLAG_REPLY` — replies never carry `REPLY_REQUESTED` so no infinite reply loops) + `nx_dispatcher_enqueue`.  Pool-owned reply messages are freed after delivery.  IPC_RECV-hook ABORT on a `REPLY_REQUESTED` request synthesizes an `NX_EABORT` reply — caller's `reply_waitq` always wakes (DESIGN.md §"ABORT on a blocking-call edge synthesizes a reply").  `nx_dispatcher_reset` resets the pool too.  `framework/slot_call.c`'s `nx_slot_call_blocking` body lands in full per [SLOT-CALL-API.md §"Body sequence"](SLOT-CALL-API.md#body-sequence): validate (NULL / dst-mismatch / missing-src / src-not-caller_slot / `task->in_flight_reply_buf != NULL` recursive-call guard) → force `NX_MSG_FLAG_REPLY_REQUESTED` → pause-state walk (NONE → break, REJECT → `NX_EBUSY`, REDIRECT → `NX_ENOENT` v1 fail-closed, QUEUE → park on `slot->resume_waitq` indefinitely) → stash `in_flight_reply_buf{,_len}` + zero rc → IPC_SEND hook chain (ABORT → clear buf, return `NX_EABORT` directly — no synthetic reply needed since caller never reaches dispatcher) → `nx_ipc_scan_send_caps` (R3) → `nx_dispatcher_enqueue` → `nx_waitq_wait_with_deadline(&task->reply_waitq, 0)` indefinite block → clear buf on wake → return `task->in_flight_reply_rc`.  Edge lookup helper walks the caller_slot's outgoing deps (the parallel edges slice 8.0a.5's `wire_caller_slot` registered).  `components/posix_shim/posix_shim.c`'s `handle_msg` switches from the slice-8.0a.4 `NX_EINVAL` stub to the real reply-routing body: `task_from_caller_slot` does `container_of` back to `nx_task` (guarded by `slot->iface == "task"` — only `wire_caller_slot` writes that tag, exact back-conversion); reject non-reply messages (tasks are senders, not receivers, except for replies); validate `caller_slot_active && in_flight_reply_buf != NULL`; truncation guard (`payload_len < sizeof(nx_reply_header) || payload_len > in_flight_reply_buf_len || payload == NULL` → set `in_flight_reply_rc = NX_EINVAL` and increment `reply_truncations` but **still wake** the caller so it doesn't hang); happy path `memcpy(in_flight_reply_buf, msg->payload, payload_len)` + `in_flight_reply_rc = ((const struct nx_reply_header *)msg->payload)->rc` + increment `replies_routed`; `nx_waitq_wake_one(&task->reply_waitq)`.  Counters (`messages_handled / replies_routed / reply_truncations / reply_unbound_caller`) + `nx_posix_shim_replies_routed_for_test / _reply_truncations_for_test` accessors.  **Tests:** +16 host (`test/host/slot_call_test.c`, new file 530 lines: pool capacity + reset; six validation paths in `nx_slot_call_blocking`; IPC_SEND-hook ABORT returns `NX_EABORT` without enqueue; dispatcher allocates pool entry only when `REPLY_REQUESTED`; `slot->in_flight_calls` returns to 0 after handler; pool entry freed after reply delivery; IPC_RECV-hook ABORT on `REPLY_REQUESTED` request synthesizes `NX_EABORT` reply; RECV ABORT on plain request does not).  +1 kernel ktest (`slot_call_blocking_round_trip_via_uart_returns_handler_rc` in `test/kernel/ktest_bootstrap.c` — first end-to-end blocking call against the live composition: a fresh kthread issues `nx_slot_call_blocking` to `char_device.serial`, dispatcher runs uart_pl011's smoke-stub `handle_msg`, posts reply, posix_shim copies payload + wakes reply_waitq, caller resumes with rc=0; pool returns to pre-call level — leak-free).  Host has no real `cpu_switch_to`, so the full reply_waitq-park round-trip is exercised only on the kernel side; host TESTs cover dispatcher + validation up to the wait point.  `make test` 502 → **519/519 pass** (93 python + 305 host + 121 kernel); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.  See [Session 90 log](logs/session-90-8.0a.6-reply-path.md). |
| **8.0a.7** ✓ | **Landed Session 91.**  `posix_shim_on_dep_swapped` STATE_LOST handler + `nx_handle_table_invalidate_for_slot()`.  Option B: `struct nx_slot *slot` (NULL = immune) added to `nx_handle_entry`; `nx_handle_alloc` / `nx_handle_table_init` / `nx_handle_close` zero it; `nx_handle_duplicate` propagates it; new `nx_handle_alloc_with_slot()` + `nx_handle_set_slot()` helpers; `sys_open`'s FILE + DIR paths wire `vfs_slot` (~2 call sites).  `nx_handle_table_invalidate_for_slot(dep_slot)` via `nx_process_for_each` callback: bumps `generation` + clears fields on matching entries.  `posix_shim_on_dep_swapped` calls it when `NX_SWAP_STATE_LOST` set.  `framework/component.h` gains `NX_SWAP_STATE_LOST` flag + `on_dep_swapped` callback in `nx_component_ops`.  +9 host tests (slot-wiring + invalidation walk).  `make test` 519 → **528/528 pass**.  Note: `make clean` required after the struct size change.  See [Session 91 log](logs/session-91-8.0a.7-handle-invalidation.md). |
| **8.0a.8** ✓ | **Landed Session 92 — closes 8.0a.**  `test/host/mock_component.h` (header-only programmable mock: `struct mock_handle` with embedded `mock_state` holding `handler_rc`, `call_count`, `last_msg`, optional `custom_fn`; `comp.impl = &state` pattern, two independent mocks coexist without globals); `test/host/hook_inspector.h` (header-only observe-only hook; records up to 32 firings with `ipc_src/dst/msg`; always `NX_HOOK_CONTINUE`); `test/host/slice_8_0a8_test.c` (36 host tests: 6 mock, 8 hook inspector, 3 recompose logger via `nx_graph_subscribe` + `nx_change_log_read`, 8 pause-injector — REJECT→EBUSY, REDIRECT→ENOENT with/without fallback, state transitions, in_flight counter, resume_waitq init, 6 cap-forgery — `nx_ipc_scan_send_caps`/`_recv_caps` R3 enforcement, 5 equivalence-runner — direct-vs-dispatcher call count + rc with/without hooks).  +2 kernel ktests in `ktest_bootstrap.c`: `hook_inspector_observe_only_does_not_alter_blocking_call_result` (IPC_SEND observer fires ≥1, rc unchanged = 0); `cap_scan_rejects_forged_slot_ref_cap_during_blocking_call` (`filesystem.root` cap from caller_slot with no edge to it → NX_EINVAL, in_flight_reply_buf cleared).  Snapshot buffer bumped 4096 → 8192: three un-destroyed kthreads (8.0a.6/.7/.8) each add ~250 bytes of JSON via registered `caller_slot` + 4 edges; ktest linker ordering runs new tests before the snapshot test.  `make test` 528 → **566/566 pass** (93 python + 350 host + 123 kernel; +36 host +2 kernel); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.  See [Session 92 log](logs/session-92-8.0a.8-test-infra.md). |
| **8.0b** ✓ | **Landed Session 93.**  Activated generated `handle_msg` shims on all 5 production components (`vfs_simple`, `ramfs`, `procfs`, `mm_buddy`, `sched_rr`) + replaced `uart_pl011`'s smoke-test handler with the real generated shim.  `reply_payload_len` field added to `nx_ipc_message`; `build_reply` in `dispatcher.c` updated to copy the per-op reply struct in-place when `reply_payload_len > 0`, falling back to header-only for legacy handlers.  `bytes_in`/`bytes_out` params now pointer-encoded as `uint64_t` in msg structs; new `_dispatch_case()` in `gen-iface.py` emits full switch-case bodies with in-place reply writes and `msg->reply_payload_len` assignment.  Components are dual-callable (direct ops + `handle_msg`) ready for 8.0c equivalence tests.  +24 host tests (`test/host/slice_8_0b_test.c`): `handle_msg` non-NULL checks for all 6 components, per-op dispatch routing, `reply_payload_len` correctness, `ASSERT_CALL_EQUIVALENT` direct-vs-dispatch parity.  `make test` 566 → **590/590 pass** (93 python + 374 host + 123 kernel); `make verify-iface-fresh` clean; `make verify-registry` clean.  See [Session 93 log](logs/session-93-8.0b-handle-msg-shims.md). |
| **8.0c** ✓ | **Sessions 94–95 — CLOSED.**  Infrastructure (Session 94): `framework/vfs_call.c` (9 wrapper implementations: host fast-path + kernel `nx_slot_call_blocking`); `DEFAULT_PATH_MAX` 4096→128; call headers include `slot_call.h`; IDL headers regenerated; `test/host/slice_8_0c_test.c` (+23 equivalence tests).  ktest payload fix (commit `5c0de13`).  **`syscall.c` migration (Session 95, commit `217724b`)**: root cause of prior hang identified — `sys_getdents64` had `uint8_t staging[4096]` on the kstack; the init kthread's 4 KiB kstack overflowed (kthread frames + SAVE_TRAPFRAME + getdents64 frame > 4096), and the blocking-call overhead (~500 bytes) made it worse, corrupting adjacent physical memory and hanging the dispatcher.  Fix: heap-allocate staging in `sys_getdents64`.  14 callsites migrated: `resolve_vfs()` removed; every `vops->` call replaced with `nx_vfs_{open,close,retain,read,write,seek,readdir,mkdir,stat}` — `sys_handle_close`, `sys_open`, `sys_read`, `sys_write`, `sys_seek`, `sys_readdir`, `sys_fork`, `sys_fstatat`, `sys_getdents64`, `sys_mkdirat`, `sys_dup2`, `sys_fcntl`.  `make test-tools` **93/93**; `make test-host` **397/397**; `make test-interactive` **7/7**; `make test-kernel` **123/123**; `make verify-iface-fresh` clean; `make verify-registry` clean.  See [Session 94 log](logs/session-94-8.0c-wrappers.md), [Session 95 log](logs/session-95-8.0c-syscall-migration.md). |
| **8.0d** ✓ | Migrate component-to-component calls — `vfs_simple` → ramfs/procfs (mount-table dispatch).  `framework/fs_call.c`: 9 `nx_fs_*` sync-dispatcher wrappers (kernel path: direct `handle_msg`; host path: fast `iface_ops`).  Removed `resolve_fs`/`resolve_for_path` from `vfs_simple.c`.  GCC strict-aliasing barrier required.  421/421 host + 123/123 kernel. — Session 96 |
| **8.0e** ✓ | **Session 97 — CLOSED.**  Added machine-checked R9 to `verify-registry.py` — scans `framework/` + `components/` for `->iface_ops` reads; exempts `dispatcher.c` / `bootstrap.c` by filename, `#if __STDC_HOSTED__` blocks, and C comment lines.  New `REPO_CHECK_FNS` dispatch table; `run_checks` restructured to run repo-level checks from `components_dir.parent`.  +9 `TestR9` unit tests.  0 findings on current codebase.  `make test-tools` **102/102**.  See [Session 97 log](logs/session-97-8.0e-verify-registry-r9.md). |

**Checkpoint B:** Hot-swap is structurally safe.  No production code reaches into ops tables.  The `pause_flag → block-on-resume → drain → swap → flush` protocol is the only path between components.

#### Group C — Runtime recomposition (6 slices)

| Slice | Deliverable |
|---|---|
| **8.1** ✓ | Lift pause/drain/resume from host build into kernel build.  `in_flight_calls` moved to enqueue-time; `slot_drain_cb` yields until counter reaches zero.  3 ktests: basic lifecycle, inflight-MPSC drain, hold-queue flush on resume.  QEMU timeout 300 → 360 s.  126/126 kernel. — Session 98 |
| **8.2** ✓ | `framework/recompose.c` orchestrator — `struct recomp_plan`, `nx_recompose()`, Kahn's topological pause order, rollback on failure, connection rewiring, `slot_clear_pause`, `timer_pause()`/`timer_resume()` bookends, `on_dep_swapped` notification.  6 host + 2 kernel tests.  427/427 host; 128/128 kernel. — Session 99 |
| **8.3** ✓ | `framework/config.c` runtime config manager + `NX_HANDLE_CONFIG` + `NX_SYS_CONFIG_OPEN/QUERY/SWAP` (42–44).  `nx_config_open()`, `nx_config_query_snapshot()`, `nx_config_swap_component()`.  10 host + 4 kernel tests.  437/437 host; 132/132 kernel. — Session 100 |
| **8.4** ✓ | Second scheduler impl — `components/sched_priority/`.  Conformance suite reused from Phase 4.  Kernel boots with `sched_priority` from `kernel.json` start-to-finish.  Standalone validation (no swap yet).  16 host tests; 132/132 kernel. — Session 101 |
| **8.5** ✓ | **Headline:** 3 kthreads survive `sched_priority → sched_rr` mid-run swap via config handle.  `kernel.json` `"alternatives": ["sched_rr"]`; both schedulers compiled in; bootstrap only enables bound component (alternative stays READY); enable hooks call `sched_init`; `sched_rr_purge_user_tasks` dispatches to correct impl; pre-drain on `in_flight_calls` before swap guards against `timer_pause + idle-WFI` deadlock; behavior change observable (`set_priority` NX_OK → NX_EINVAL).  2 kernel tests; 134/134 kernel; QEMU timeout 360 → 450 s. — Session 102 |
| **8.6** ✓ | Runtime async↔sync mode switching.  `build_pause_order` drains `to_slot` on `CONN_REWIRE`; `nx_config_set_conn_mode()` + `NX_SYS_CONFIG_REWIRE` (45); 6 host + 2 kernel tests; QEMU timeout 450 → 900 s. — Session 103 |
| **8.7** ✓ | `NX_HOOK_SYSCALL_ENTER` / `NX_HOOK_SYSCALL_EXIT` hook points.  `sc` arm in `nx_hook_context` (`num`, `a[6]`, `rc*`, `tf`); ENTER ABORT skips body; EXIT can override return value.  5 ktests; 141/141 kernel.  Key finding: `NX_HOOK_POINT_COUNT` shift 8→10 caused stale `component_hook_test.o` to treat value 8 (now `NX_HOOK_SYSCALL_ENTER`) as "bad point" — `make clean` required. — Session 104 |

**Checkpoint C:** Phase 8 closed.  Live recomposition shipping; per-edge mode switching shipping; second scheduler shipping; syscall-boundary observability shipping.

#### Test plan

Migration of 184 callsites and 5 components is high-risk surgery.  Coarse `make test` green is necessary but not sufficient.  ~270 new ktests across the 15 slices; end-state target ~700 (today: 453).

| Group | New ktests | Notes |
|---|---|---|
| Group A — generator | ~50 | Schema validation, round-trip pack/unpack, snapshot diffing, cap-bearing ops, no-context ops, IRQ-entry shape. |
| Slice 8.0a (sync-call infra) | ~70 | Pause-state transitions; queue/reject/redirect policies; in-flight counter; hook chain incl. abort; cap-scan incl. forgery; identity property under `PAUSE_NONE`; re-entrancy; lost-wakeup race; ISR-context guards.  Foundation everything else stands on. |
| Slice 8.0b (receiver shims) | ~50 | Dual-callable equivalence per op per component. |
| Slices 8.0c, 8.0d (callsite migration) | ~54 landed | Per-callsite golden tests pinning behavior *before* migration; 8.0c added 23 equivalence tests, 8.0d added 24 (421/421 host baseline). |
| Slices 8.1–8.6 (Phase 8 proper) | ~100 | Kernel-side pause/drain/resume; recompose orchestrator; config handle API; sched_priority conformance; live-swap demo; mode switching. |

Cross-cutting test infrastructure lands in slice **8.0a.8** (sub-sliced from the original 8.0a — see Group B table): configurable mock component, hook-chain inspector, recompose event logger, pause-injector ktest fixture, cap-forgery harness, per-callsite equivalence-runner macro.  The earlier 8.0a sub-slices (8.0a.1 → 8.0a.7) build the runtime infra these tests exercise.

**Test-first migration philosophy.**  Per-callsite equivalence tests land *before* the migration in slice 8.0c.  Dual-callable window in slice 8.0b is what makes side-by-side assertion possible.  Each slice ends with `make test` count strictly higher than it started — no slice regresses.

#### Risk concentration

- **Slice 8.0a.**  Sync-call API shape must accommodate hooks + pause-flag + fallback redirect + in-flight tracking + cap-scan.  Worth getting right before generator output (Group A) depends on it; in practice, 8.0a's API contract is a prerequisite for 8.0pre.1's wrapper-emission template.
- **Slice 8.0c.**  Hot-path perf checkpoint.  If syscall paths slow significantly, may need wrapper inlining work or a fast-path before continuing to 8.0d.
- **Slice 8.1.**  Kernel-side pause may surface races invisible in host build (preempt/IRQ semantics around the atomic pause flag).  Worth doing first as a de-risking probe before 8.2+.
- **Slices 8.4 → 8.5.**  `sched_priority` must be stable standalone before live-swap can validate correctness.  Rushing 8.4 makes 8.5 hard to debug.

#### Estimated effort

15 slices × ~1–2 sessions each = **~15–25 sessions**.  Front-loaded: Groups A and B together are ~9 slices and roughly the same scope as Group C alone, but they pay off for every future component and every future hot-swap demo.

**Validation (Phase 8 exit):** Scheduler swapped at runtime without crash; tasks continue running under new scheduling policy; per-edge async↔sync mode flips at runtime without restart; `make test` ~700/700; `make test-interactive` 7/7; no production code reaches into `slot->active->descriptor->iface_ops`.

### Phase 9: Per-Process Memory Management Rework

**Goal:** Replace the contiguous 8 MiB-per-process backing model with L3 4 KiB pages, per-process VMAs, demand paging, and copy-on-write fork.  Phase 7 deliberately stayed on the simple model so we could ship busybox; this phase pays it down before Phase 10 benchmarks measure anything.
**Status:** NOT STARTED

The current model (Phase 5–7) gives every process one contiguous 8 MiB physical block, mapped via four 2 MiB L2 block descriptors.  That's functional but rigid:

- **Over-commit.**  Every process eats 8 MiB of physical RAM regardless of actual use (`init_prog` is ~800 bytes; signal demo ~250 bytes).
- **PMM fragmentation.**  `nx_process_create` needs 10 MiB physically contiguous (8 MiB usable + 2 MiB alignment slack); after many fork/exec cycles PMM may not find a contiguous run even with hundreds of MiB free elsewhere.
- **Fork is O(USER_WINDOW_SIZE) memcpy** even when the child overwrites two bytes and exits — no COW.
- **Hard 8 MiB ceiling per process.**  Any binary whose code+data+heap+stack would exceed ~6 MiB of mapped memory just can't run.
- **Wasted alignment slack.**  Each `process_create` carries 2 MiB of slack locked away from PMM until the process is destroyed.

Steps (multi-slice rework):

1. Per-process page tables grow a third level — L3 4 KiB pages instead of L2 2 MiB blocks.  4 KiB granularity makes per-page demand-paging and per-page COW natural.
2. `nx_process` gains a list of VMAs (`{vaddr_lo, vaddr_hi, prot, file_or_anon, offset}`).  ELF loader populates one VMA per PT_LOAD; brk grows a heap VMA up; mmap creates new VMAs.
3. Page-fault handler upgrades from "panic" to "look up VMA, allocate one PMM page, populate L3 entry, return."  COW pages get a "copy on write" flag; first write demotes the entry from RO to RW with a fresh page.
4. `mmu_copy_user_backing` becomes `mmu_clone_user_backing_cow`: walks parent's L3, marks each entry RO in both parent and child, no memcpy.  First write in either side faults, copies one page, splits.
5. `mmap(NULL, size, PROT_*, MAP_ANON|MAP_PRIVATE, ...)` falls out for free as a new VMA + lazy population.  busybox + most real userspace stops needing brk.
6. Migrate every existing test off the contiguous-block assumption (per-process layout constants, `mmu_user_window_*` semantics, the `pmm_reserve_range` boot-time exclusion in `boot.c`).

**Validation:** `make test` still passes; an EL0 demo that mmap's 100 MiB of anonymous memory and touches one page per MiB completes without OOM; fork is O(VMA count) not O(window size); a stress test that forks/execs 1000 times per second runs steadily without PMM exhaustion.

### Phase 9b: Ephemeral Slots and Handles as Capabilities

**Goal:** Eliminate the `switch (handle_type)` dispatch in `sys_read`/`sys_write` by making every open file descriptor an anonymous (unregistered) slot wired to the backing architectural slot.  Handles become capability tokens whose routing is data-driven by the slot graph rather than hard-coded type tags.
**Status:** NOT STARTED — design decision recorded in [DESIGN.md §"Two-Tier Slot Model"](DESIGN.md#two-tier-slot-model-planned--phase-9b).  May be scheduled before or after Phase 9 (MM rework); the two phases are independent.

**Background.** The registry currently requires every slot to be named and globally registered. Placing one slot per open fd would flood the registry with thousands of ephemeral entries.  The minimum structural change needed is smaller than it first appears — no struct migration required.  See DESIGN.md §"Two-Tier Slot Model" for the full rationale.

**Key insight on the edge-lookup direction.**  `nx_slot_call_blocking` uses `find_outgoing_edge(src, dst)` which walks `nx_slot_foreach_dependency(src)` — i.e. the *source* slot's outgoing list.  An anonymous source has no `slot_node`, so this walk finds nothing.  But there is no need to move edge lists onto `struct nx_slot`: the connection also lives in the *target* slot's `incoming` list (via `slot_add_incoming(to_sn, n)` in `nx_connection_register`, which fires unconditionally as long as `to_sn` is registered).  The target IS registered.  Flipping the lookup to walk the target's `incoming` list instead — `find_incoming_edge(dst, src)` matching `c->from_slot == src` — finds the same conn_node with no struct changes at all.

This reduces the implementation to two targeted changes before the handle-embedding work:
1. Relax `nx_connection_register` to accept an unregistered (but non-NULL) `from` slot — currently line 534 rejects it with `NX_ENOENT`; the `slot_add_outgoing` call at line 558 is already conditional on `from_sn != NULL` and becomes a no-op.
2. Replace `find_outgoing_edge(src, dst)` in `slot_call.c` with `find_incoming_edge(dst, src)`.

#### Slice 9b.1 — Anonymous slot API + flip edge lookup

Add `nx_slot_init_anon(struct nx_slot *s, const char *iface, enum nx_slot_mutability m)` — initialises `pause_state` and `in_flight_calls` atomics, zero-fills the name (anonymous), but does NOT call `nx_slot_register`.

Relax `nx_connection_register`: remove the `NX_ENOENT` guard on unregistered `from`; `slot_add_outgoing` becomes a no-op (already conditional).  The conn_node is added to `g_connections` and `to_sn->incoming` as before — the anonymous source simply has no outgoing list entry, which is correct since anonymous slots are leaf nodes never targeted by `nx_slot_foreach_dependency`.

Replace `find_outgoing_edge(src, dst)` in `slot_call.c` with `find_incoming_edge(dst, src)` that calls `nx_slot_foreach_dependent(dst, ...)` matching `c->from_slot == src`.  Functionally identical for registered sources; now also works for anonymous sources.

Add `nx_slot_is_registered(const struct nx_slot *)` predicate.

Host tests: anonymous slot lifecycle (init, connect, disconnect, no registry entry), `nx_slot_call_blocking` succeeds from an anonymous source, drain works (target's `in_flight_calls` reaches zero), pause/drain order never includes anonymous slots.

~50 lines + ~6 host tests.  No struct layout changes.

#### Slice 9b.3 — Embed `struct nx_slot` in handle entries

`struct nx_handle_entry` gains an embedded `struct nx_slot slot` field (zero-initialised until the handle is allocated).  `nx_handle_alloc` for `HANDLE_FILE`, `HANDLE_DIR`, `HANDLE_CHANNEL`, and `HANDLE_CONSOLE` calls `nx_slot_init_anon` on `entry->slot` and registers a connection from `entry->slot` to the relevant architectural slot (vfs, char_device, etc.) via `nx_connection_register`.  `nx_handle_free` (called from `sys_handle_close`) unregisters the connection and zeroes the slot.

`nx_process_create` initialises the three pre-installed console handles (stdin/stdout/stderr at slots 0/1/2) by wiring their embedded slots to `char_device.serial`.

`sys_open` wires the new handle's slot to `vfs` (mode: sync, same as the current `nx_vfs_*` path).  `sys_pipe` wires each end's slot to a channel endpoint — the channel IS the object, so the slot is effectively a named reference to the endpoint.

No routing change yet (type-switch still present); this slice is purely structural.

~120 lines kernel code; ~8 host tests covering embed/wire/unwire lifecycle.

#### Slice 9b.4 — Route `sys_read` / `sys_write` through slot calls

Replace the `switch (handle_type)` in `sys_read`, `sys_write`, `sys_seek`, `sys_readv`, `sys_writev` with `nx_slot_call_blocking(&entry->slot, &msg)`.  The message type encodes the operation (READ / WRITE / SEEK); the dispatcher on the other side routes to the component's op handler.

The `char_device` IDL gains a `read(buf, len) → i64_count_or_status` op (currently only `write` and `rx_byte` exist).  `uart_pl011` implements it via `nx_console_read` (same as today).  The `fs` IDL already has `read`/`write`/`seek`; vfs_simple and ramfs already implement them.  The HANDLE_CHANNEL arm (pipe read) becomes `nx_channel_recv` through the channel slot — the channel endpoint IS the slot's target object.

The POSIX fd-0 magic (stdin special-case) disappears: slot 0's embedded slot already points to the console component, so `nx_slot_call_blocking` routes correctly without the `h == 0` branch.

~100 lines changed in syscall.c; updated IDL + regenerated char_device headers; ~10 ktests for the routed read/write/seek paths.

#### Slice 9b.5 — Cleanup: retire per-type dispatch remnants

- Remove `NX_HANDLE_CONSOLE`, `NX_HANDLE_FILE`, `NX_HANDLE_DIR` type tags from `enum nx_handle_type`; replace with `NX_HANDLE_RESOURCE` (a generic "anonymous slot to some backing component") so `sys_handle_close` has one code path for all of them.
- `NX_HANDLE_CHANNEL` survives — channels have their own close semantics (endpoint refcount).  Its slot still routes via `nx_slot_call_blocking` for read/write, but `sys_handle_close` still calls `nx_channel_endpoint_close`.
- Remove the `copy_path_from_user` HANDLE_DIR special-case in `sys_readdir`; directory cursors become a slot op (`READDIR`) dispatched through the vfs component.
- `verify-registry` R9 rule already exempts the host fast-paths; no rule changes needed — anonymous slots are invisible to it by design.
- `make verify-registry` 0 findings; `make test` all pass; the `HANDLE_FILE`/`HANDLE_DIR`/`HANDLE_CONSOLE` branches in syscall.c are gone.

~80 lines removed; 0 new tests (all coverage comes from earlier slices).

**Checkpoint 9b:** Every open fd is an anonymous slot.  `sys_read(fd)` is `nx_slot_call_blocking`.  The kernel's POSIX shim is free of hard-coded object types; routing is entirely data-driven by the slot graph.  Adding a new "file-like" object (e.g. a network socket) requires only a new component implementing the `fs` interface — no changes to `syscall.c`.

**Test plan**

| Slice | New tests | Notes |
|---|---|---|
| 9b.1 | 0 (regression gate) | All 600 existing tests must pass |
| 9b.2 | ~6 host | Anonymous lifecycle, drain without registration |
| 9b.3 | ~8 host | Embed/wire/unwire; console pre-install; open/close |
| 9b.4 | ~10 kernel | Routed read/write/seek; stdin=slot-0; pipe read via slot |
| 9b.5 | 0 (cleanup) | Verify no HANDLE_FILE/DIR/CONSOLE remnants |

### Phase 10: Integration Tests and Benchmarks

**Goal:** In-kernel integration tests and performance benchmarks. Test infrastructure was built incrementally in Phases 2-9; this phase adds system-level tests and benchmarking.
**Status:** NOT STARTED

Steps:
1. In-kernel integration tests (run in QEMU):
   - End-to-end IPC: userspace→posix_shim→vfs→block_device round trip
   - Hot-swap under load: swap scheduler while tasks are running, verify no leaks/crashes
   - Hook injection: register hooks, verify trace output, test abort behavior
   - Lifecycle stress: cycle all hot-swappable components 100 times under load
2. IPC latency benchmark (async and sync modes)
3. Context switch benchmark
4. Syscall round-trip benchmark
5. `make test` and `make bench` working end-to-end
6. Audit: verify every component has passing conformance, memory, and lifecycle tests

**Validation:** `make test` — all pass (host + kernel), >80% core coverage. `make bench` — produces reproducible report. Every component shows "0 leaks, 0 errors".

### Phase 11: Documentation and AI Operability

**Goal:** Complete documentation. AI agent can build a custom kernel from manifests alone.
**Status:** NOT STARTED

Steps:
1. Write `docs/ARCHITECTURE.md` — minimum modules, module types, composition rules
2. Write `docs/COMPOSITION-GUIDE.md` — how to create a kernel config
3. Write `docs/COMPONENT-TEMPLATE.md` — template for component READMEs
4. Ensure every component has complete README.md (usage, interface, relationships, resource ownership)
5. Ensure every manifest.json is complete and accurate
6. Test: give AI agent only the docs + manifests, ask it to build a kernel

**Validation:** AI agent (Claude) generates valid kernel.json, runs `make validate-config` + `make` + `make run` successfully.

### Future: Dynamic Loading, SMP, Networking, GUI

These are tracked but not planned in detail:
- **Dynamic loading** — Load components from filesystem at runtime (ELF relocations, symbol resolution)
- **SMP** — Multi-core support (per-CPU data, spinlocks, IPI)
- **Networking** — virtio-net driver, TCP/IP stack as a component
- **GUI** — Framebuffer driver, display server as a component

---

## Build & Test

### Building

```bash
# Build kernel image
make

# Build with verbose output
make V=1

# Clean build
make clean && make
```

### Running

```bash
# Boot in QEMU, interactive (Ctrl-A X to exit)
make run                        # or: tools/run-qemu.sh

# Boot, auto-terminate after N seconds (scripted runs — Phase 1 halts in wfe)
tools/run-qemu.sh -t 5

# Pass extra flags through to QEMU
tools/run-qemu.sh -- -append test
tools/run-qemu.sh -t 10 -- -append bench

# Boot with extra RAM
make run QEMU_MEM=2G            # or: QEMU_MEM=2G tools/run-qemu.sh

# Boot with GDB server (for debugging)
make debug
# Then in another terminal: aarch64-linux-gnu-gdb kernel.elf -ex "target remote :1234"
```

`tools/run-qemu.sh` is the single source of truth for QEMU invocation flags (machine, CPU, `-nographic`, memory) for interactive/scripted runs. `make test-kernel` runs QEMU directly with `-semihosting` added so the in-kernel runner can exit via ARM semihosting `SYS_EXIT_EXTENDED` (see TESTING-GUIDE.md). `make bench` will follow the same pattern when benchmarks land.

### Testing

Full details and per-test inventory: [TESTING-GUIDE.md](TESTING-GUIDE.md).

```bash
# Run all tests (host + kernel)
make test

# Host-side only — pure-C algorithm tests, runs in ms
make test-host

# Kernel-side only — boots kernel-test.bin under QEMU -semihosting,
# guest exits VM with pass/fail as QEMU's own exit code
make test-kernel

# Validate kernel config
make validate-config

# Static check: registry rules R1-R7 across framework + components
make verify-registry

# Show component dependency graph
make deps

# Export dependency graph as Graphviz dot
make deps-dot > deps.dot
```

`make verify-registry` is a prerequisite for `make` and `make test` — a component that fails the verifier will not build. This is how AI compliance is enforced: the registry rules are treated like `-Werror`, not like a lint suggestion.

### Benchmarks

```bash
# Run benchmarks
make bench
```

### Quick Validation

```bash
# Full pipeline: validate, verify registry, build, test
make validate-config && make verify-registry && make && make test
```

---

**Last Updated:** 2026-05-03 (Phases 1–7 complete.  Phase 8 in progress.  **Group A complete** (8.0pre.1 → 8.0pre.4, Sessions 82–85).  **Group B — slice 8.0d CLOSED Session 96:** 8.0a (Sessions 86–92) + 8.0b (Session 93) + 8.0c (Sessions 94–95) + 8.0d (Session 96) landed.  Slice 8.0d: `framework/fs_call.c` — 9 `nx_fs_*` sync-dispatcher wrappers; `vfs_simple.c` migrated to use them (removed `resolve_fs`/`resolve_for_path`); 24 new host tests.  Key finding: GCC -O2 strict-aliasing heisenbug required `__asm__ volatile(""::: "memory")` barrier after every `hmsg()` call.  Tests: `make test-tools` **93/93 pass**; `make test-host` **421/421 pass**; `make test-interactive` **7/7 pass**; `make test-kernel` **123/123 pass**; `make verify-iface-fresh` clean; `make verify-registry` clean.  Next forward step: slice 8.0e (`verify-registry` rule banning `iface_ops` access outside `framework/dispatcher.c`).)
