# Session 51: slice 7.6d.2b — grow user window 2 MiB → 8 MiB + relocate brk

**Date:** 2026-04-26
**Phase:** 7 slice 7.6d.2b (busybox integration, sub-sub-slice 2 of 3)
**Branch:** master

---

## Goals

Slice 7.6d.2b is the second of three sub-sub-slices that together
(7.6d.2a / 7.6d.2b / 7.6d.2c) make `posix_busybox_help_prog`'s
fork+exec attempt possible.  7.6d.2a re-linked busybox at the
user-window VA; this slice grows the window itself and relocates
the brk region so busybox's ~1.91 MiB text+data span has room for
stack + heap.

The minimum-viable change list (per the IMPLEMENTATION-GUIDE):
- Bump `USER_WINDOW_SIZE` from 2 MiB to 8 MiB (1 → 4 contiguous
  L2-block slots).
- Move `NX_PROCESS_HEAP_OFFSET` and `NX_PROCESS_HEAP_LIMIT` past
  busybox's data+bss span (`+1 MiB / +1.5 MiB → +6 MiB / +7.5 MiB`).
- Trim alignment slack in `mmu_create_address_space`'s malloc
  (`USER_WINDOW_SIZE * 2` → `USER_WINDOW_SIZE + USER_BLOCK_SIZE`)
  so per-process backing footprint stays bounded.
- Update the only hard size assertion (`ktest_el0.c`) and the
  comments that mention "2 MiB user window".

That's the headline change.  The slice ended up uncovering and
fixing two latent bugs that the size bump exposed but didn't
itself create:
1. A spawn-side race in `sched_spawn_kthread` callers (the
   process-binding assignment after spawn could lose to the
   scheduler picking the kthread).
2. A **self-aliasing memcpy** in `mmu_copy_user_backing` whose
   write VAs landed on the user-window slot of the *current*
   process's TTBR0 (overriding the kernel-identity PA).
3. A second-order alias: kernel allocations (kstacks, page
   tables, slab pages) at PA inside the user-window VA range
   silently aliased through per-process user_block overrides
   when accessed under any process TTBR0.

All three needed fixing for `make test` to stay green at 407/407.

## Scope choices

- **8 MiB, not 4 MiB or 16 MiB.**  4 MiB barely fits busybox
  (1.91 MiB text+data + ~2 MiB for stack+heap is tight).  16 MiB
  pre-allocates more than we need today and doubles the per-
  process malloc footprint.  8 MiB is the smallest power-of-two
  that comfortably fits everything our test programs do, with
  6 MiB left for stack+heap after busybox's image.

- **`USER_WINDOW_BLOCKS = 4` macro instead of inlining `4`
  everywhere.**  Two places need the count: `mmu_init`'s loop
  to mark slots `USER_WINDOW_INDEX..+BLOCKS-1` as user_block in
  the kernel L2_ram, and `mmu_create_address_space`'s loop that
  populates the same slots in the per-process L2.  A single
  named constant means the two stay in sync if the size ever
  changes again.

- **Trim malloc alignment slack from `USER_WINDOW_SIZE` to
  `USER_BLOCK_SIZE`.**  Each L2 block is 2 MiB-granular, so the
  four contiguous slots just need the START address to be 2 MiB-
  aligned; subsequent slots follow by stride.  The previous
  formula (`malloc(USER_WINDOW_SIZE * 2)`) wasted up to one full
  window per allocation — wasteful at 2 MiB, unaffordable at
  8 MiB.  New formula: `malloc(USER_WINDOW_SIZE + USER_BLOCK_SIZE)`
  with alignment-up by `USER_BLOCK_SIZE`.  Worst-case per-process
  footprint: `8 + 2 = 10 MiB` raw.

- **Keep kernel L2_ram[USER_WINDOW_INDEX] alone, leave 65/66/67
  as `normal_block` (kernel-only).**  Originally I thought the
  kernel's identity slots 64..67 should all be `user_block` for
  symmetry with the per-process L2.  That made the second-order
  alias bug worse: kernel-internal allocations at PA in the
  user-window range would be EL0-accessible via the kernel's own
  identity (a security smell as well as the trap-frame issue
  described below).  Reverting to `user_block(slot 64) +
  normal_block(slots 65..67)` keeps the slice-5.5-era drop_to_el0-
  with-kernel-TTBR0 working (the drop_to_el0 case in `ktest_el0`)
  while not extending EL0 access to additional kernel-identity
  ranges.  Per-process L2 still overrides all four slots as
  user_block — that's the EL0 path for processes.

- **Spawn-process-binding API change, not preempt-disable
  bandaids.**  `sched_spawn_kthread` grew a 4th parameter
  `struct nx_process *process`.  NULL means "inherit from
  caller's task" (production default — dispatcher kthread,
  ktest_el0, ktest_channel, ktest_sched all pass NULL).  Non-
  NULL is set on the task BEFORE enqueue, eliminating the race
  between spawn-return and the test code's `t->process =
  X` assignment.  Updated all 23 call sites in one pass: 18
  ktests pass an explicit process; 5 pass NULL.  The race had
  been latent for many slices but never reliably triggered
  until the 8 MiB user-window made the kernel's identity-map
  slot 64 garbage routinely point at fresh user-backing
  contents (uninitialized = mostly zeros = `udf #0`).

- **Switch to kernel TTBR0 around the `memcpy` in
  `mmu_copy_user_backing`, not page-by-page rewriting.**  The
  memcpy's destination kernel pointer (= the dst's PA, since
  the kernel runs identity-mapped) can span the user-window VA
  range when dst PA is in `[user_window_base, +size)`.  Under
  the caller's TTBR0, writes to those VAs go through the
  caller's user_block override → the *caller's* user_pa, not
  dst's.  With kernel TTBR0, slot `USER_WINDOW_INDEX` is the
  kernel's identity (`user_block(PA RAM_BASE+slot*2MiB)`), so
  writes land at the intended PA.  Used raw `msr ttbr0_el1` +
  `isb` rather than the full `mmu_switch_address_space`
  helper — the latter does a `tlbi vmalle1` which is overkill
  in a hot path; non-user-window TLB entries stay valid because
  they're identical across all roots.

- **Reserve `[user_window_base, +size)` from PMM at boot,
  not "after first leak".**  This stops kernel-internal
  allocations (kstacks, page tables, slab pages) from EVER
  landing at PA in the user-window range, which would silently
  alias through per-process user_block overrides when the
  kernel runs under any TTBR0 and accesses its own state via
  identity-VA in that range.  Cost: 8 MiB of physical RAM
  permanently off-limits — small compared to the 1 GiB we
  have, and absolutely correct rather than "mostly works".
  Implementation: new `pmm_reserve_range(pa, bytes)` API,
  called once from `boot.c` after `pmm_init`.

## What Was Done

### `core/mmu/mmu.c`

- **`USER_WINDOW_BLOCKS = 4`** macro (count of 2 MiB blocks).
  `USER_WINDOW_SIZE` redefined as `USER_WINDOW_BLOCKS * BLOCK2_SHIFT`
  → 8 MiB.  `USER_BLOCK_SIZE` macro (a single 2 MiB block) for
  alignment math.

- **`mmu_init`** unchanged from the slice-5.5 era for slots
  `USER_WINDOW_INDEX+1..` — they stay as `normal_block` (kernel-
  only).  Only slot `USER_WINDOW_INDEX` (64) is `user_block` in
  the kernel L2_ram, preserving the slice-5.5 drop_to_el0-with-
  kernel-TTBR0 path used by `ktest_el0` and `ktest_channel`.

- **`mmu_create_address_space`** populates per-process L2 slots
  `USER_WINDOW_INDEX..+BLOCKS-1` as `user_block(user_pa + i*2MiB)`
  (loop instead of single assignment).  Malloc footprint trimmed
  to `USER_WINDOW_SIZE + USER_BLOCK_SIZE = 10 MiB`; alignment
  rounded up to `USER_BLOCK_SIZE = 2 MiB`.

- **`mmu_copy_user_backing`** wraps the memcpy in
  `msr ttbr0_el1, kernel_root` / `msr ttbr0_el1, saved_root`
  (with `isb` between).  Justification documented in a 14-line
  comment that names the slice and the failure shape.

### `framework/process.h`

- **`NX_PROCESS_HEAP_OFFSET` 1 MiB → 6 MiB**, `NX_PROCESS_HEAP_LIMIT`
  1.5 MiB → 7.5 MiB.  Heap region: `[base+6 MiB .. base+7.5 MiB)`,
  1.5 MiB total (was 512 KiB).  Comment block updated to describe
  the new layout: code/data 0..6 MiB, heap 6..7.5 MiB, stack
  7.5..8 MiB.

### `core/sched/sched.{h,c}` + 23 call sites

- **`sched_spawn_kthread` gains a 4th parameter
  `struct nx_process *process`.**  NULL = inherit (default);
  non-NULL = set `t->process = process` before enqueue.  Header
  comment cross-references this slice's session log for the
  failure-mode rationale.

- **All 23 call sites updated** in one pass (1 grep, 23 targeted
  Edits):
  - 5 pass NULL: `dispatcher.c`, `ktest_channel.c`,
    `ktest_el0.c`, `ktest_sched.c` (×3 calls), `ktest_vfs.c`
    (×2 calls).
  - 18 pass an explicit process: every fork/exec/posix/musl
    EL0 ktest, plus `ktest_process.c`'s exiter / cs-a / cs-b
    kthreads.

### `core/pmm/pmm.{h,c}`

- **New `pmm_reserve_range(pa, bytes)` API.**  Atomically claims
  every page in `[pa, pa+bytes)` from the bitmap, decrements
  `g_free`.  Idempotent for already-allocated pages; clamps to
  the PMM pool's bounds.  Documented as the user-window PA
  reservation use case in the header.

- **`boot.c` calls it** right after `pmm_init`, with
  `mmu_user_window_base()` + `mmu_user_window_size()`.  Boot
  log gains `[user-window reserved]` suffix.  At 8 MiB
  reserved, that's 2048 pages off-limits to the kernel.

### `test/kernel/ktest_el0.c`

- Hard size assertion bumped: `KASSERT(size == (8UL * 1024 * 1024))`
  (was `2 MiB`).  Comment notes the slice that changed it.

### Comment refreshes (no semantic change)

- `framework/elf.h`, `framework/elf.c`, `framework/syscall.c`,
  `framework/syscall.h`, `framework/process.h`,
  `core/mmu/mmu.c` — every comment that said "2 MiB user
  window" or "2 MiB user backing" updated to reflect the new
  size or generalized to `mmu_user_window_size()` where
  appropriate.

## Drive-by gotchas

This slice took multiple debugging rounds to land.  Three
distinct bugs surfaced in sequence; each had to be fixed in turn
to make `make test` green again.

- **Bug 1: spawn race.**  Symptom: `posix_signal` test (and
  potentially others under racy timing) saw EXC at `ELR=0x48000008`
  with `ESR=0x02000000` (EC=0 "Unknown reason") right after the
  test name printed.  Cause: `sched_spawn_kthread` enqueues the
  task, then the test code does `t->process = X` *after*.  If
  the scheduler picks the task between enqueue and assignment,
  the task runs with its inherited (kernel-process) TTBR0,
  drops to EL0 at the user-window VA, and reads from
  `kernel_identity_PA(0x48000000)` — uninitialized memory that
  often decoded as `udf #0`.  Latent for many slices; reliably
  triggered post-8-MiB-bump because user-backings now routinely
  span PA `0x48000000+`, leaving uninitialized heap data at
  the kernel's identity-map slot 64.  Fix: the
  `sched_spawn_kthread` API extension above.

- **Bug 2: self-aliasing memcpy in `mmu_copy_user_backing`.**
  Symptom: after fix-1, EXC moved to `ELR=0x48000008` again
  but with parent's program code reading as zero (verified by
  printing `parent[0]` before/after each step of fork).
  Cause: kernel runs `mmu_copy_user_backing` with the *caller's*
  TTBR0 still active.  memcpy iterates byte-by-byte; for the
  last 2 MiB of an 8 MiB copy, dst+i lands in the user-window
  VA range (e.g., `0x48000000-0x481FFFFF`) when dst PA is
  `0x47A00000` (= dst+6 MiB = `0x48000000`).  Caller's L2[64]
  is `user_block(caller's_user_pa)`, so the write to VA
  `0x48000000` translates back to *caller's* user_pa, not
  dst's PA — silently zeroing the caller's program code with
  the bytes from dst+6 MiB (heap area, uninitialized).  Fix:
  switch to kernel TTBR0 around the memcpy.

- **Bug 3: kernel kstacks at PA in user-window VA range.**
  Symptom: after fix-2, EXC moved AGAIN, this time at
  `ELR=0` with `ESR=0x8200000e` (EC=0x20 instruction abort,
  IFSC=0x0e permission fault L2).  Cause: forked child's
  kstack PA happened to land at `0x483CF000` (in user-window
  PA range slot 65).  `RESTORE_TRAPFRAME` in `nx_task_fork_child_entry`
  runs with child's TTBR0 active, so kstack VA `0x483CFEF0`
  goes through child's L2[65] = `user_block(child_user_pa +
  2 MiB)` = a totally different PA.  RESTORE_TRAPFRAME reads
  zeros for ELR/SPSR/registers, then `eret` jumps to PC=0
  (MMIO range, UXN, instruction abort).  Fix: reserve
  `[user_window_base, +size)` from PMM at boot so no kernel
  allocation can land there.  Generalizes beyond kstacks —
  page tables, slab pages, anything — all get pushed to PAs
  outside the dangerous range.

- The debugging path used `kprintf` instrumentation at four
  points (in `mmu_create_address_space`, `mmu_switch_address_space`,
  `mmu_copy_user_backing` pre/post-memcpy, `nx_process_fork`
  pre/mid/end, `nx_task_create_forked` after byte-copy).  Each
  iteration narrowed the bug location by ~2x.  All
  instrumentation removed before commit.

## Verification

- `make test` → 407/407 (51 python + 275 host + 81 kernel),
  0 leaks, 0 errors, exit 0.  Identical pass count to slice
  7.6d.2a — the size bump is invisible to existing tests.

- `[pmm]` boot log now reads:
  ```
  [pmm]  total=NNN free=NNN pages (NN KiB) [user-window reserved]
  ```
  Exact numbers shift by 2048 pages (8 MiB / 4 KiB) from prior
  builds.

- Spot-check: `posix_signal_sigterm_kills_forked_child_with_status_143`
  now prints `[signal-parent][signal-child][signal-ok]` in order
  in the live ktest log (was crashing with `[signal-parent]` then
  EXC mid-slice).

## What Was Not Done

- **No busybox-exec attempt yet.**  That's 7.6d.2c.
- **No `RAMFS_FILE_CAP` or `SYS_EXEC_MAX_FILE` bump yet.**
  These need to grow to ≥ 4 MiB for busybox (2.29 MB binary).
  7.6d.2c.
- **No initramfs busybox seeding yet.**  7.6d.2c.
- **No mmap, no clone.s/vfork.s/__unmapself.s/restore.s
  patches.**  Whichever 7.6d.2c surfaces, becomes 7.6d.3.x.
- **TTBR1 high-half kernel mapping** (the "real" fix for the
  kernel-data-vs-user-window aliasing).  PMM reservation works
  for v1 but it's a workaround.  Long-term, kernel data should
  live in TTBR1 high half so per-process TTBR0 overrides can't
  alias it under any circumstance.  Tracked as a Phase 8+
  follow-up.

## Test Counts

- Python: 51 (unchanged)
- Host: 275 (unchanged)
- Kernel: 81 (unchanged)
- **Total: 407/407 PASS**

## Next Steps

- **7.6d.2c** — Bump `RAMFS_FILE_CAP` (8 KiB → ≥ 4 MiB) + bump
  `SYS_EXEC_MAX_FILE` (8 KiB → ≥ 4 MiB) + add busybox to
  `tools/pack-initramfs.py`'s entry list as `/bin/busybox` + write
  `posix_busybox_help_prog.c` (libnxlibc parent that forks +
  `nxlibc_execve("/bin/busybox", { "/bin/busybox", "--help",
  NULL }, NULL)`) + write a kernel test that captures whatever
  failure mode surfaces.  This is the actual discovery sub-slice
  that was originally 7.6d.2's whole job.

---

**Last Updated:** 2026-04-26
