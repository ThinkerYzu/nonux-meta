# Session 22: slice 5.1 — MMU + identity map

**Date:** 2026-04-22
**Phase:** 5 slice 5.1
**Branch:** master

---

## Goals

Turn the kernel MMU on at EL1 with a simple identity-mapped address
space, drop `-mstrict-align` from the Makefile, and verify the machine
still boots cleanly + passes every existing test. This slice unblocks
the rest of Phase 5 (per-process page tables, handle table, EL0) by
giving us a real virtual-memory substrate.

## What Was Done

### `core/mmu/{mmu.h,mmu.c}` — new

Boot-time identity-mapped page table:

- 4 KiB granule, 39-bit VA (TCR.T0SZ=25), 3 levels, walk starts at L1.
- **L1** (512 entries, 1 GiB each): entries [0] and [1] wired, rest
  invalid.
- **L2_mmio** (covers 0x00000000..0x40000000): 512 × 2 MiB Device-nGnRnE
  blocks, AP=RW at EL1, PXN=UXN=1. Covers GIC (0x08000000) and UART0
  (0x09000000) on QEMU virt.
- **L2_ram** (covers 0x40000000..0x80000000): 512 × 2 MiB Normal
  WB-WA Inner-Shareable blocks, AP=RW at EL1, PXN=0 (kernel executable),
  UXN=1 (no EL0 execution yet).

All three tables are in BSS (`__attribute__((aligned(4096)))`) so they
get zero-initialised by `start.S` and have stable addresses. No PMM
allocation needed — keeps the MMU bring-up path independent of anything
else.

MAIR_EL1 configuration:

- Attr0 = 0x00 (Device-nGnRnE)
- Attr1 = 0xFF (Normal Inner+Outer WB-WA)

TCR_EL1 configuration:

- T0SZ=25 → 39-bit VA
- IRGN0=ORGN0=01 (table walks Normal WB-WA cacheable)
- SH0=11 (inner shareable)
- TG0=00 (4 KiB)
- EPD1=1 (disable TTBR1 walks — high half unused for now)
- IPS=001 (36-bit PA; QEMU virt only has 1 GiB RAM + a few MMIO regions)

Enable sequence (in C, no separate `.S`):

1. `dsb ish` so table writes are visible before the walker starts.
2. Write MAIR / TCR / TTBR0; `isb`.
3. `tlbi vmalle1; dsb ish; ic iallu; dsb ish; isb`.
4. Read-modify-write SCTLR_EL1 setting M, C, I bits; `isb`.

VA = PA everywhere we map, so turning the MMU on does not change any
symbol's numeric address — only its cacheability, ordering semantics,
and alignment rules. The `isb` after SCTLR write flushes the pipeline
and the next instruction is fetched through the translation table.

### `core/boot/boot.c` — wiring

`mmu_init()` is called right after `uart_init()`, before anything else.
Everything downstream (PMM, vectors, GIC, timer, framework bootstrap,
scheduler start, ktest) runs with the MMU on, D-cache on, I-cache on.

### `Makefile`

- `core/mmu/mmu.c` added to `CORE_C`.
- `-mstrict-align` **dropped** from `CFLAGS`. Safe now because every
  memory region we map is either Normal (where unaligned access is
  legal when `SCTLR.A=0`) or Device (where MMIO register accesses
  already use naturally-aligned `volatile uint32_t *` everywhere).

### Tests (+4 kernel)

New `test/kernel/ktest_mmu.c`:

- `mmu_sctlr_has_m_c_i_bits_set` — reads `SCTLR_EL1`, asserts M/C/I.
- `mmu_is_enabled_helper_returns_true` — exercises the public helper.
- `mmu_ttbr0_points_into_ram` — sanity-checks TTBR0 landed in the
  0x40000000..0x80000000 range (L1 table lives in BSS).
- `mmu_unaligned_access_on_normal_memory_succeeds` — deliberately
  misaligned `uint32_t` load + store on a `static volatile uint8_t`
  buffer. Pre-slice-5.1 this would have faulted with SError; now it
  round-trips as expected (little-endian check on both directions).

Net: `make test` → **248/248** (51 python + 167 host + 30 kernel), up
from 244/244.

## Key Decisions

- **2 MiB blocks at L2, not 4 KiB pages at L3.** Block mappings keep
  the page table footprint tiny (3 × 4 KiB = 12 KiB) and cover all of
  RAM + MMIO with no L3 tables. Fine-grained permissions (split
  .text/.rodata/.data/.bss) come in slice 5.2 once `mm_buddy` owns
  page-table growth.
- **Kernel RAM is mapped RWX at EL1 for now.** PXN=0 across the whole
  RAM region because slice 5.1's goal is getting the MMU on, not
  enforcing W^X. The `LOAD segment with RWX permissions` linker warning
  therefore still fires — listed as a follow-up in HANDOFF for slice 5.2.
- **BSS-resident page tables, not PMM-allocated.** Keeps `mmu_init`
  independent of `pmm_init` and callable before it. The cost is a few
  pages of always-resident tables, which is acceptable at this stage.
- **EPD1=1.** TTBR1_EL1 is explicitly disabled because nothing maps the
  high half yet. Flipping EPD1 to 0 is a one-line change when per-process
  page tables land in slice 5.5.
- **MMU init in C, not assembly.** The enable sequence is short enough
  that inline-asm barriers inside a normal C function are clearer than a
  separate `.S` file. Because VA=PA everywhere we map, there's no
  "branch into a mapped region" dance to orchestrate — the `isb` after
  SCTLR write just fetches the next instruction through the TLB. If the
  enable sequence ever needs to also switch stacks or jump to a high
  half, it becomes an `.S` file.
- **`mmu_init` called before PMM.** Rest of bring-up then runs with
  caches on. PMM doesn't care — it's just marking pages free/used in
  its own bitmap.

## Known Issues / Follow-ups

- `LOAD segment with RWX permissions` linker warning is still present.
  Splitting kernel segments into separate `PHDRS` entries in `linker.ld`
  and giving `.text`/`.rodata` PXN=0+RO and `.data`/`.bss` PXN=1+RW
  mappings lands in slice 5.2 (or as soon as we have a real need, e.g.
  the first SEGV-like test).
- `mmu_init()` currently takes no configuration — it hardcodes the
  QEMU-virt layout. When we gain DTB parsing (Phase 5 infrastructure
  polish), the RAM range becomes dynamic.
- Page-table alloc is BSS-static for three hard-coded tables. Dynamic
  page-table growth (for per-process address spaces, VMO mapping, etc.)
  is slice 5.2–5.5 territory.

## Next Actions

**Slice 5.2 — `components/mm_buddy/`.** Buddy allocator on top of the
PMM shipped as a swappable component (`interfaces/mm.h`, conformance
suite, 100× lifecycle cycling). `kernel.json` binds
`memory.page_alloc ← mm_buddy`. Will also replace the BSS-resident page
tables with dynamically-allocated ones so we can grow the address space
beyond today's 2 × 1 GiB block layout.

## Files Touched

- `core/mmu/mmu.h` — **new**
- `core/mmu/mmu.c` — **new**
- `core/boot/boot.c` — call `mmu_init()` right after `uart_init()`
- `Makefile` — add `core/mmu/mmu.c` to `CORE_C`; drop `-mstrict-align`;
  add `ktest_mmu.c` to `KTEST_C`
- `test/kernel/ktest_mmu.c` — **new**, 4 cases
