# Session 33: slice 7.2 — per-process address spaces (TTBR0 per process)

**Date:** 2026-04-24
**Phase:** 7 slice 7.2
**Branch:** master

---

## Goals

Give every `nx_process` its own TTBR0 root so two processes can
hold different bytes at the same VA and context-switching
transparently flips the mapping.  This is the foundation slice
7.3's ELF loader builds on (a loader that can't install into a
process-private address space isn't very interesting).

## Scope choices

- **Full "kernel to TTBR1 high half" rework deferred.**  DESIGN.md
  describes the end state as kernel-only high-half mappings in
  TTBR1 + user-only low-half in TTBR0.  That requires a linker
  script rewrite (kernel symbols at high VAs), boot.c changes,
  and updating every kernel pointer path.  Big scope, not needed
  for the isolation guarantee.  Instead I chose "every per-
  process TTBR0 is a copy of the kernel's identity map + one
  private user window" — simpler, delivers the same user-level
  isolation, costs ~8 KiB of table storage + 2 MiB of backing per
  process.  Left a note in the Phase 7 plan that the full
  TTBR1 migration lands when there's a consumer (probably around
  multi-CPU or a security-boundary consumer).
- **2 MiB user window per process, not 4 KiB.**  Keeping the same
  L2-block granularity as slice 5.5's kernel user window means no
  L3 tables — L2 descriptors alone suffice.  Cost: 2 MiB per
  process.  Benefit: the MMU extension is self-contained in
  ~100 LOC of C instead of needing L3 table allocation + walking.
  The PMM has ~1 GiB to hand out so process count isn't
  practically bounded by this; mm_buddy's 64 KiB pool stays
  untouched.
- **Kernel-level tests only.**  MMU manipulation is inherently
  kernel-side — host has no MMU to test against.  `process_test.c`
  doesn't grow; `ktest_process.c` gets three new tests.  Host
  stubs for `mmu_create_address_space` return 0 so host code
  still compiles and `nx_process_create` still returns a valid
  process (with `ttbr0_root = 0`).
- **Existing EL0 ktests not migrated.**  Channel / file / readdir
  tests keep running in the kernel process — they'd need a
  dedicated per-test process wired up to use their own windows,
  and that's mechanical churn with no test-delta in v1 (they'd
  still see the same bytes at the same VA because nothing else
  is contending).  Slice 7.3's ELF loader replaces the baked-in
  programs; that's the natural moment for the migration.

## What Was Done

### `core/mmu/mmu.{h,c}` — new address-space API

- `mmu_create_address_space()` — allocates a fresh L1 + L2_ram
  (both 4 KiB, page-aligned via `malloc` which routes through
  kheap→PMM for large-class sizes) plus a 2 MiB user-window
  backing chunk (malloced with an extra 2 MiB slack and aligned
  up).  Populates:
  - `L1[0] = DESC_TABLE | &l2_mmio_table` (shared kernel MMIO
    map — read-only reference; we never write through it from a
    per-process root).
  - `L1[1] = DESC_TABLE | &new_l2_ram`.
  - `new_l2_ram[i] = l2_ram_table[i]` for i ≠ 64 (a byte-copy
    of the kernel's RAM map).
  - `new_l2_ram[64] = user_block(user_backing_pa)` — the
    process's private 2 MiB user window.
  - `dsb ish` so the table walker sees the writes before we
    hand the root out.
  Returns the L1 physical address (== virtual address under our
  identity map).  `NULL` return signals allocation failure;
  caller rolls back.
- `mmu_destroy_address_space(root)` — looks up a bookkeeping
  entry keyed by root and frees the L1 + L2_ram + user backing.
  Idempotent on root == 0 and on the static kernel root.
- `mmu_switch_address_space(root)` — writes TTBR0_EL1, barriers
  + `tlbi vmalle1` to discard stale user-region entries.  On
  host: just records the last-passed root for
  `mmu_current_address_space_for_test` to inspect.
- `mmu_kernel_address_space()` — returns `(uintptr_t)l1_table`
  (the static kernel L1 set up by `mmu_init`).  Host returns 0.
- Fixed-size `g_mmu_spaces[16]` bookkeeping table (linear-scan
  lookup).  16 processes is plenty for v1 tests; bumping later
  is a one-line change.

### `framework/process.{h,c}` — `ttbr0_root` field

- `struct nx_process` gains `uint64_t ttbr0_root`.
- `nx_process_create` calls `mmu_create_address_space()` on
  kernel (rolling back on failure: unregister + free).  On host
  stays 0.
- `nx_process_destroy` calls `mmu_destroy_address_space` on
  kernel; no-op on host.

### `framework/bootstrap.c` — seed the kernel process's root

- `nx_framework_bootstrap` sets `g_kernel_process.ttbr0_root =
  mmu_kernel_address_space()` as its first kernel-specific
  action, before slot registration.  `mmu_init` has run by then
  (boot.c calls it before `boot_main` runs the framework).

### `core/sched/sched.c` — TTBR0 flip on context switch

- `sched_check_resched` reads `curr->process` and `next->process`;
  if both are non-NULL, their `ttbr0_root` values differ, and
  the incoming root is non-zero, calls
  `mmu_switch_address_space(next->process->ttbr0_root)` before
  `cpu_switch_to`.
- The flip sits in C code (not `context.S` asm) because there's
  only one call site and doing it before the switch keeps the
  cpu_switch_to asm free of architectural deps.  The kernel
  instruction cache page currently executing is mapped
  identically in every address-space root, so the `isb`
  following the `msr ttbr0_el1` resolves the next instruction
  fetch through the new root with no semantic change.

### Kernel tests (`ktest_process.c`)

- `process_create_allocates_fresh_ttbr0_root` — two new
  processes each get a non-zero root distinct from each other
  and from the kernel root.
- `process_two_address_spaces_hold_different_bytes_at_same_va`
  — manually flips TTBR0 between two processes and confirms
  each sees its own bytes at `user_window_base`.  This is the
  core slice-7.2 isolation claim.  `nx_preempt_disable` wraps
  the flip sequence so a timer tick can't yield the test body
  away in the middle of the experiment (which would auto-flip
  TTBR0 back to the scheduled task's address space).
- `process_context_switch_flips_ttbr0` — spawns two kthreads,
  each pinned to its own process (via `ta->process = pa` after
  `sched_spawn_kthread`).  Each kthread reads `user_va[0]` and
  yields forever.  The test body yields until both have
  recorded; asserts each saw its OWN process's seed byte.  This
  proves the scheduler-driven auto-flip path.

## Key Findings

- **Kernel instruction cache stays valid across TTBR0 flips
  because every per-process root has the same kernel identity
  map.**  This was the scariest invariant of the slice: if the
  kernel code page at some PA were mapped differently in two
  processes' TTBR0s, an `msr ttbr0_el1` instruction would
  potentially invalidate the execution of the very code that
  performed it.  Our design keeps the kernel RAM block identical
  in every per-process `L2_ram[0..63]` + `[65..511]`, so the
  flip is semantically a no-op for the kernel's code page — the
  VA→PA mapping resolves the same way.  This mirrors how Linux
  (pre-PTI) ran with the kernel mapped in every process's table.
- **Preempt-disable around a multi-step TTBR0 experiment is
  essential.**  First draft of the two-processes-hold-different-
  bytes test occasionally failed because a timer tick during
  the write sequence flipped TTBR0 (to the scheduler's pick,
  likely back to the kernel process).  Wrapping the write +
  flip + read sequence in `nx_preempt_disable` / `_enable`
  fixes it.  General rule: any test that observes cross-
  address-space behavior through manual flips must hold
  preemption disabled.
- **`nx_preempt_disable` is safe to call from kernel tests
  because the test runner itself is a kthread.**  It has a
  `preempt_count`; incrementing and decrementing it is standard.
  Without this, a "manual-flip" test would be inherently flaky.

## Decisions Made

- **Per-process TTBR0 shares kernel mappings, not separated
  via TTBR1.**  Why: delivers the isolation guarantee with
  <10% of the code-and-linker-rework cost.  How to apply: the
  full TTBR1-migration lands when a security/multi-CPU consumer
  needs it.  Plan noted in IMPLEMENTATION-GUIDE.md §7.2 "Out of
  scope" section so a future session can pick it up without
  archaeology.
- **Bookkeeping table, not embedded metadata.**  Alternative:
  put a header struct at the front of the L1 page and
  bias-access descriptors past the header.  Chose the external
  `g_mmu_spaces[]` lookup table because the L1 page-table
  descriptors MUST start at a 4 KiB boundary (the TTBR0 format
  requires it), and interleaving the header with descriptors
  introduces permutations of "what's at offset 0" across
  processes — confusing.  Cost of the separate table is ~16 ×
  40 bytes = 640 bytes of BSS.  How to apply: any future MMU-
  level per-process state that doesn't fit a natural field in
  the page-table descriptors themselves follows this pattern.
- **Flip in C, not asm.**  Call site count is 1
  (`sched_check_resched`); the flip is 5-6 asm instructions;
  the only reason to put it in `context.S` would be atomicity
  with the register save, which isn't needed because the flip
  is a per-process-boundary event and the kernel code stays
  reachable either way.  How to apply: other "only-at-switch"
  tweaks (e.g., future ASID assignment, TLB domain flushing)
  can go in the same C site.

## Status at End of Session

- `make test` → **379/379 pass (51 python + 266 host + 62 kernel),
  0 leaks, 0 errors, exit 0**.  +3 kernel tests from slice 7.1.
- `make run` boots cleanly under QEMU — the bootstrap log shows
  the same 5-slot composition as before; the new TTBR0-flip
  machinery is exercised by tests but idle during normal boot
  (there's only one process active until a ktest spawns one).
- Phase 5 follow-up "per-task TTBR0" is now closed as a
  consequence of 7.2 (the `ttbr0_root` field IS per-task in
  practice, since each task points at its process's root).

## Next Steps

Slice 7.3 — ELF loader reading programs from ramfs.

- `framework/elf.{h,c}` parses PT_LOAD segments of a
  position-independent static ELF and maps them into a target
  process's address space.  For v1 only PT_LOAD is handled —
  relocations, dynamic linking, symbol resolution all defer.
- `nx_process_create_from_elf(path)` opens via `vfs_simple`,
  loads the ELF into the new process's user window, returns a
  ready-to-run process.
- Bootstrap seeds ramfs with an `/init` ELF (assembled from a
  simple EL0 test binary) at boot time so the loader has real
  content to exercise.
- Migrates one of the existing EL0 ktests (probably `user_prog`,
  the slice-5.5 hello-world) to use the loader.

---

**Files Changed:**
- `sources/nonux/core/mmu/mmu.h` — new per-process address-space API
- `sources/nonux/core/mmu/mmu.c` — `mmu_create/destroy/switch/kernel_address_space`, bookkeeping table, host stubs
- `sources/nonux/framework/process.h` — `ttbr0_root` field on `struct nx_process`
- `sources/nonux/framework/process.c` — allocate address space in create, free in destroy (kernel only)
- `sources/nonux/framework/bootstrap.c` — seed `g_kernel_process.ttbr0_root` from `mmu_kernel_address_space`
- `sources/nonux/core/sched/sched.c` — TTBR0 flip in `sched_check_resched` before `cpu_switch_to`
- `sources/nonux/test/kernel/ktest_process.c` — three new tests (distinct roots, bytes-at-same-VA, auto-flip on switch)
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.2 rewritten with as-built details; Phase 7 status bumped
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 28 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
