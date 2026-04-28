# Session 5: Phase 2 built — test harness, mem_track, PMM, GIC, timer

**Date:** 2026-04-20
**Phase:** Phase 2 complete
**Branch:** master (sources/nonux)

---

## Goals

- Ship the host-side test infrastructure (runner + mem_track).
- Write the bitmap PMM with host-side tests.
- Stand up EL1 exception vectors, GICv2, and the ARM Generic Timer.
- Prove the kernel takes an IRQ: periodic tick prints in QEMU.

## What Was Done

### Host test harness (`test/host/test_runner.{h,c}`, `main.c`)

Minimal, self-registering test framework. `TEST(name) { … }` emits a
`struct test_entry` into a `test_registry` linker section; the runner walks
from `__start_test_registry` to `__stop_test_registry` at startup. Each
test gets a fresh `mt_reset()` before running and an `mt_check_leaks()`
after. Assertions (`ASSERT`, `ASSERT_EQ_U`, `ASSERT_EQ_PTR`, `ASSERT_NULL`,
`ASSERT_NOT_NULL`) long-jump out on failure so test functions can `return`
early without per-ASSERT cleanup.

### Memory tracker (`test/host/mem_track.{h,c}`)

Wraps `malloc`/`free` with:
- per-block header + red zones (`REDZONE_SIZE = 16`, `REDZONE_BYTE = 0xFE`)
- quarantine on free (payload filled with `0xDE`) — live blocks go on one
  doubly-linked list, freed blocks on another; `mt_reset()` returns both to
  libc
- double-free, write-after-free, red-zone-overflow checks; all abort the
  test binary with a diagnostic naming alloc+free sites
- `mt_live_count()` for lifecycle tests

Not yet needed: full UAF on *reads* (the payload fill catches *writes* post-
free, which is the important case for Phase 3+ components).

### PMM (`core/pmm/pmm.{h,c}`)

Bitmap allocator over a contiguous region. Bitmap lives at the base of the
region; the pages covering the bitmap are marked allocated at init so they
never get handed out. Interface:

```
pmm_init(base, size);
pmm_alloc_page(); pmm_free_page(p);
pmm_alloc_pages(n); pmm_free_pages(p, n);
pmm_free_count(); pmm_total_count();
```

Concurrency: the bitmap uses `__atomic_fetch_or` / `__atomic_fetch_and` on
bytes; the free counter is `_Atomic size_t`. This satisfies the project-
wide "preemption requires atomics in v1" rule without adding a lock.
Allocation is first-fit with an advisory `g_hint` biased toward recently-
freed indices so repeated alloc/free doesn't walk the whole bitmap.

### PMM + mem_track host tests (`test/host/pmm_test.c`)

11 tests, all green (`PASS: 11/11 tests passed, 0 leaks, 0 errors`):

- `pmm_init_counts_pages_and_reserves_bitmap`
- `pmm_alloc_page_returns_nonnull_and_aligned`
- `pmm_alloc_distinct_pages`
- `pmm_alloc_until_exhausted_then_null`
- `pmm_free_then_realloc` (verifies hint reuse)
- `pmm_full_cycle_no_leak`
- `pmm_alloc_pages_contiguous_two`
- `pmm_alloc_pages_too_big_returns_null`
- `pmm_alloc_pages_finds_contiguous_range_after_fragmentation`
- `mem_track_tracks_live_and_released`
- `mem_track_zero_initializes_payload`

A 1 MiB `posix_memalign`'d region per test stands in for physical memory.

### Host test Makefile (`test/host/Makefile`)

Builds with the system `cc` (not the cross-compiler). Uses
`vpath %.c ../../core/pmm ../../core/lib` to pull in the code-under-test
from the kernel tree **without** VPATH'ing `.o` files — otherwise stale
cross-compiled objects left by `make` at the top would silently poison the
link. The top-level `make test-host` already delegates here.

### Exception vectors (`core/cpu/vectors.S`, `core/cpu/exception.{h,c}`)

Standard ARM64 16-entry table, 2 KiB aligned. Each vector is a single
`b` to an out-of-line stub; the `SAVE_TRAPFRAME` / `RESTORE_TRAPFRAME`
macros are far larger than the 32-instruction per-entry budget, so inlining
them in the vector wouldn't fit.

Trap frame: 272 bytes (x0..x30 + SP_EL0 placeholder + ELR_EL1 + SPSR_EL1),
16-byte aligned. C handlers (`on_sync`, `on_irq`, `on_fiq`, `on_serror`,
`on_unimpl`) receive a `struct trap_frame *`.

`vectors_install()` writes VBAR_EL1 and issues an ISB.

### IRQ framework + GICv2 (`core/irq/irq.{h,c}`, `core/irq/gic.c`)

Fixed-size dispatch table (`IRQ_TABLE_SIZE = 128`) indexed by IRQ number.
`irq_dispatch()` reads GICC_IAR (which acknowledges), looks up the
handler, calls it, writes GICC_EOIR. Unhandled IRQs are masked at the GIC
rather than spinning.

GICv2 driver is deliberately minimal: distributor enable (`GICD_CTLR = 3`
for both groups), CPU interface enable (`GICC_CTLR = 3`), PMR = 0xFF
(accept all priorities). `gic_enable(irq)` sets priority 0 and ISENABLER
bit; `gic_disable` / `gic_ack` / `gic_eoi` the rest.

### ARM Generic Timer (`core/timer/timer.{h,c}`)

Uses the EL1 physical timer (PPI 30). `timer_init(hz)` reads CNTFRQ_EL0,
computes `interval = freq / hz`, registers `on_tick` as IRQ 30, enables
the IRQ at the GIC, arms `CNTP_TVAL_EL0`, and sets `CNTP_CTL_EL0 = 1`.
Handler rearms the TVAL first (to minimize drift) and increments an
`_Atomic uint64_t` tick counter.

### Boot wiring (`core/boot/boot.c`)

New sequence: UART → banner/memory-map print → PMM over
`[__free_mem_start, 0x80000000)` → `vectors_install()` → `gic_init()` →
`timer_init(10)` → `irq_enable_local()` → `wfi` loop.

## Key Findings

- **QEMU `-kernel` loads at `RAM_BASE + 0x80000`, not `RAM_BASE`.** Root-
  caused the entire IRQ-not-firing stall to this. QEMU injects a small
  bootloader at 0x40000000 that jumps to the kernel at 0x40080000; our
  linker had `. = 0x40000000`, so every absolute address (VBAR_EL1 and
  every `ldr =symbol`) pointed 0x80000 short of the real code. `-d int`
  showed ELR at 0x40000a80 faulting with "Undefined Instruction" and
  ESR=0x2000000 (EC=0, "Unknown reason"); `-d in_asm` then revealed
  `OBJD-T: 00000000` at that address — zero padding. Fixed by
  `. = 0x40080000` in linker.ld. A future cleanup can add a proper
  AArch64 Linux Image header so the image self-describes the offset.
- **GCC 15 emits outline-atomics calls into libgcc.** In a freestanding
  kernel that isn't linked against libgcc, that turns every atomic into
  an `__aarch64_ldset1_acq`-shaped undefined reference. Fix: add
  `-mno-outline-atomics` to CFLAGS. Keeps LL/SC (ldxrb/stxrb loops) for
  pre-ARMv8.1 cores and avoids the libgcc dependency.
- **QEMU `virt` defaults to GICv3 in TCG mode.** Must be forced with
  `-M virt,gic-version=2` or our MMIO-based GICv2 driver silently does
  nothing (GICv3 moves the CPU interface to system registers). Updated
  both Makefile and `tools/run-qemu.sh`.
- **CNTHCTL_EL2.EL1PCEN must be set before dropping to EL1** or else EL1
  writes to `CNTP_CTL_EL0` / `CNTP_TVAL_EL0` trap to EL2. Added the
  `mrs … ; orr #3 ; msr` in `start.S`'s `from_el2` path, plus
  `msr cntvoff_el2, xzr`.
- **`.align 2048` on the vector table drives the whole `.text`
  section's alignment** in the ELF. Our file offset of `.text` bumped to
  0x10000 as a side effect; didn't cause any functional problem because
  `objcopy -O binary` writes by VMA, not file offset — but it's a thing
  to know.

## Decisions Made

- **Use `__atomic_*` builtins, not `<stdatomic.h>`.** Works in freestanding
  without an include, encodes acquire/release explicitly where it matters
  (bitmap OR = acquire, AND = release, counter = relaxed), and matches
  what Linux kernel code does. `_Atomic size_t` is fine for the counter
  because it compiles to the same code on GCC.
- **Bitmap PMM, not buddy, for v1.** `kernel.json` names `mm_buddy` as a
  component (Phase 3+ will plug one in on top of PMM). The bottom
  physical allocator just needs to hand out pages; first-fit bitmap with
  a hint is ~150 lines and easy to reason about.
- **No serial output in `irq_dispatch` fast path.** The per-tick
  `[tick] N` print at 1 Hz is fine — `n % 10 == 0` keeps it bounded. A
  per-IRQ `[irq-dbg]` line was used while debugging and removed.
- **Force GICv2 at the CLI, not in the driver.** Writing a GICv3 driver
  was tempting but the project targets QEMU virt as the primary platform
  and v2 is universally supported. When we need GICv3 (real hardware,
  GICv3-only virt config), it slots in behind the same `irq_enable`/
  `irq_dispatch` interface.
- **Keep the RWX-segment linker warning.** Still benign; revisit when
  the MMU lands in Phase 5 and we can split `.text` / data into separate
  PT_LOAD segments.

## Status at End of Session

**Host side:**
- `make test-host` → `PASS: 11/11 tests passed, 0 leaks, 0 errors` ✓
- mem_track detects leaks, double-free, red-zone overflow (verified by
  exercising happy paths; adversarial test is fine to add in Phase 3 when
  components actually use `TRACKED_ALLOC`).

**Kernel side:** `tools/run-qemu.sh -t 4` gives:

```
========================================
  nonux — composable microkernel
  ARM64 / QEMU virt
========================================
[boot] kernel loaded at 0x40080000 (QEMU's -kernel offset)
[boot] BSS:  0x40084000 — 0x40084840
[boot] kernel end:    0x40085000
[boot] free memory:   0x400c5000 — 0x80000000
[pmm]  total=261939 free=261939 pages (1047756 KiB)
[cpu]  exception vectors installed at 0x40080800
[gic]  distributor + CPU interface enabled
[timer] cntfrq=62500000 Hz, interval=6250000 ticks, rate=10 Hz
[boot] Phase 2 ready — tick prints once/sec.

[tick] 10
[tick] 20
[tick] 30
```

Three ticks in 4 seconds = 10 Hz confirmed. IRQ path fully functional:
timer → GIC → vector → C dispatch → rearm → ERET back to `wfi`.

## Next Steps

- **Phase 3: component framework.** Lifecycle state machine (init / enable
  / pause / resume / disable / destroy), IPC router (async + sync),
  dependency resolver (topo sort + injection), registry (per Session 2
  decision — v1, not deferred), `gen-config.py`, `validate-config.py`,
  `make verify-registry`. Wire `pause_hook` into `COMPONENT_REGISTER`
  (Session 3 follow-up). This is the biggest phase on the roadmap.
- **Infrastructure polish (opportunistic):**
  - Add an AArch64 Linux Image header so `-kernel` self-describes the
    load offset instead of us hardcoding `0x40080000` in linker.ld.
  - Host test `test-host COMPONENT=X` filter (SPEC lists it as a doc'd
    feature; trivial to implement).
  - Kernel-side PMM smoke test under `make test-kernel` once we have a
    test harness in the kernel.

---

**Files Changed:**
- **new:** `test/host/test_runner.{h,c}`, `test/host/mem_track.{h,c}`,
  `test/host/pmm_test.c`, `test/host/main.c`, `test/host/Makefile`
- **new:** `core/pmm/pmm.{h,c}`
- **new:** `core/cpu/vectors.S`, `core/cpu/exception.{h,c}`
- **new:** `core/irq/irq.{h,c}`, `core/irq/gic.c`
- **new:** `core/timer/timer.{h,c}`
- **edited:** `core/boot/start.S` — CNTHCTL_EL2.EL1PCEN + CNTVOFF_EL2 = 0
- **edited:** `core/boot/boot.c` — Phase 2 bring-up wiring
- **edited:** `core/boot/linker.ld` — load address moved to 0x40080000
- **edited:** `Makefile` — new CORE_S/CORE_C, `-mno-outline-atomics`,
  `-M virt,gic-version=2`
- **edited:** `tools/run-qemu.sh` — `-M virt,gic-version=2`
