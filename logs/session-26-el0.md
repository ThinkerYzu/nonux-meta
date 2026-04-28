# Session 26: slice 5.5 — first EL0 process

**Date:** 2026-04-23
**Phase:** 5 slice 5.5
**Branch:** master

---

## Goals

Run the first real EL0 userspace program. EL0 → SVC → EL1 → syscall
dispatch → EL0 round-trip, with the user program's bytes flowing
through UART visibly in the test log. Slice 5.4 wired SVC from EL1;
this slice extends the same handler to SVC from EL0.

## What Was Done

### Narrowed scope vs the original plan

The IMPLEMENTATION-GUIDE's slice 5.5 called for per-task TTBR0 page
tables, which requires moving the kernel to the high half (TTBR1) —
a substantial restructure. This slice deliberately narrows to a
single shared TTBR0 identity map with one EL0-accessible window.
Per-task TTBR0 is deferred to a follow-up. Exit-path mechanics (task
reap, `nx_process_exit`, `copy_from_user`) also deferred to later
slices; the EL0 task here lives forever in a `wfe` loop.

### `core/mmu/{mmu.h,mmu.c}` — user window

- New `user_block(pa)` descriptor builder: AP = 0b01 (EL1 + EL0 RW),
  UXN = 0 (EL0 may execute), PXN = 0, SH = Inner Shareable, AttrIdx
  = Normal.
- One L2_ram entry (index 64 → VA 0x48000000) is upgraded from
  `normal_block` to `user_block` during `mmu_init`. Still identity-
  mapped — VA == PA.
- `mmu_user_window_base()` / `mmu_user_window_size()` exposed so the
  EL0 test can locate the window without another macro.

### `core/cpu/vectors.S` — lower-EL sync route + SP_EL0 preservation

- The four "Lower EL using AArch64" vector slots (SVC / IRQ / FIQ /
  SError from EL0) now branch to the same stubs as the EL1-origin
  slots: `_sync_stub`, `_irq_stub`, `_fiq_stub`, `_serror_stub`. The
  sync handler already decodes ESR.EC, so SVCs from either EL route
  to `nx_syscall_dispatch` uniformly.
- `SAVE_TRAPFRAME` now writes `mrs sp_el0` into offset 0x0F8 instead
  of storing `xzr` (which was a no-op placeholder). `RESTORE_TRAPFRAME`
  writes the saved value back via `msr sp_el0`. Preserving SP_EL0
  across every exception is cheaper than branching on origin-EL, and
  EL1-origin paths are unaffected (their SP_EL0 is unused anyway).

### `core/cpu/el0_entry.S` — `drop_to_el0(pc, sp)`

- One-way helper that sets `ELR_EL1 = pc`, `SP_EL0 = sp`, `SPSR_EL1 =
  EL0t | DAIF unmasked`, zeroes x0..x18 + x29 + x30 for hygiene, then
  `eret`s.
- DAIF set to 0 (IRQs unmasked at EL0) so the timer tick preempts EL0
  tasks correctly. Exception entry from EL0 still sets DAIF.I, so
  there's no double-handling.

### `framework/syscall.{h,c}` — observable debug_write counter

- New atomic `g_debug_write_calls` counter bumped on the happy path of
  `sys_debug_write`. Exposed via `nx_syscall_debug_write_calls()` and
  reset by `nx_syscall_reset_for_test()`. Replaces the need to scrape
  UART output for verification — tests read the counter to confirm
  EL0 code reached the SVC.

### `test/kernel/user_prog.S` — baked-in EL0 program

- Assembled into `.rodata` of `kernel-test.bin` between
  `__user_prog_start` and `__user_prog_end` markers. Program:
  - `adr x0, _user_msg; mov x1, #len; mov x8, #1; svc #0` (debug_write).
  - `wfe; b .` — park forever at EL0 waiting for timer IRQs.
- All address computations are PC-relative (`adr`, `b`) so the test
  can `memcpy` the bytes into the user window at a different VA and
  the relocated copy executes correctly.

### `test/kernel/ktest_el0.c` — +2 kernel tests

1. `el0_user_window_vars_are_sensible` — quick sanity: window base is
   in RAM, size is 2 MiB, base is 2 MiB-aligned, user program fits.
2. `drop_to_el0_runs_user_program_which_reaches_debug_write` — the
   end-to-end test. Resets the syscall counter, spawns a kthread
   whose entry memcopies `__user_prog_*` into the window and calls
   `drop_to_el0(base, base + size)`. Parent yields from the ktest
   (idle) context until `nx_syscall_debug_write_calls()` rises, then
   dequeues the EL0 task so it doesn't steal timeslices from later
   tests.

The live `ktest` log shows `[el0] hello` — first EL0 userspace bytes
reaching UART via the SVC dispatcher.

### Build wiring

- `Makefile`: `CORE_S += core/cpu/el0_entry.S`; `KTEST_S` holds
  `test/kernel/user_prog.S` assembled into `kernel-test.bin` only;
  `KTEST_C += ktest_el0.c`.

Net: `make test` → **295/295** (51 python + 197 host + 46 kernel), up
from 293/293 — **+2 kernel tests**.

## Key Decisions

- **Single shared user window, not per-task TTBR0.** The full "each
  EL0 task gets its own address space" vision requires the kernel to
  live on TTBR1 (high half). That's a rewrite I don't want to tangle
  with a first-EL0-program slice — the address-space separation work
  stands on its own. The slice's exit criterion ("EL0 program runs
  and makes syscalls") is met without it.
- **`memcpy` user prog into the window at runtime, rather than link
  it at the user VA.** Placing `.user_text` at 0x48000000 via the
  linker would force the ELF to carry ~128 MiB of zero padding before
  it, bloating `kernel-test.bin` for a flat-binary target. The
  runtime memcpy is one page of code and zero-cost at boot.
- **EL0 program loops in `wfe` forever, no exit syscall.** A proper
  `nx_process_exit` needs scheduler cooperation (dequeue self + yield,
  or a TASK_ZOMBIE state). That plus task reap is a separate slice.
  For "did EL0 actually run?" the counter check is sufficient.
- **Dequeue but don't destroy the EL0 task after the test.** The
  task's kstack still has a live trap frame from the preempting tick;
  `nx_task_destroy` would free it out from under the saved context.
  Dequeue just removes it from the runqueue so the scheduler stops
  picking it — the kstack + state are leaked. Acceptable at this
  stage.
- **DAIF = 0 in SPSR at `eret`**. EL0 needs IRQs unmasked so the
  timer tick preempts it. On exception entry the hardware masks
  DAIF.I automatically, so there's no reentry problem.
- **User window at VA 0x48000000 (128 MiB into RAM).** Chosen far
  enough from the PMM's near-term allocations (kernel ends ~0x40096000,
  PMM hands out contiguously starting ~0x400d6000) that a collision
  is improbable. A proper solution pairs with per-task TTBR0 — then
  each user window can be allocated from PMM like any other page.

## Known Issues / Follow-ups

- **No per-task TTBR0.** All tasks share the same TTBR0. Tracked as
  a follow-up; requires moving the kernel to TTBR1.
- **No EL0 task exit path.** The spawned EL0 task is stranded in a
  `wfe` loop after the test; dequeue prevents it from running more
  but the task struct + kstack are leaked.
- **User window physical memory is PMM-unmanaged.** If the PMM ever
  hands out a page at 0x48000000+ it will collide with user code.
  Low risk for now (PMM is near-empty by boot end) but needs proper
  reservation before more EL0 consumers land.
- **No user-pointer validation.** `sys_debug_write` trusts its
  `buf` is readable for `len` bytes. Kernel-issued SVCs pass kernel
  pointers; the slice 5.5 EL0 program passes a pointer inside the
  (shared) user window, which happens to also be EL1-accessible.
  When EL0 gets its own TTBR0 — or when an EL0 program passes a bad
  pointer — a `copy_from_user` primitive with fault handling is
  required. Explicitly called out in the slice 5.4 log as a
  follow-up.

## Next Actions

**Slice 5.6 — channels.** `HANDLE_CHANNEL` + `nx_channel_create /
_send / _recv` on top of the existing IPC router. First real
EL0-consumable API: an EL0 program creates a channel, sends a message
to an in-kernel echo component, receives the reply. This exercises
the handle framework (channel endpoints as handles), the syscall
dispatcher (new `NX_SYS_CHANNEL_*` numbers), and the IPC router all
end-to-end from userspace. `copy_from_user` lands here as the first
real user-pointer consumer.

## Files Touched

- `core/mmu/mmu.h` — expose `mmu_user_window_{base,size}`
- `core/mmu/mmu.c` — `user_block()` + upgrade L2_ram[64] to user-RW
- `core/cpu/vectors.S` — route lower-EL sync vectors; preserve SP_EL0
- `core/cpu/exception.h` — `drop_to_el0` prototype
- `core/cpu/el0_entry.S` — **new**, the drop-to-EL0 helper
- `framework/syscall.h` — expose `nx_syscall_debug_write_calls()`
- `framework/syscall.c` — atomic counter bumped on happy path
- `test/kernel/user_prog.S` — **new**, the baked-in EL0 program
- `test/kernel/ktest_el0.c` — **new**, 2 cases
- `Makefile` — `CORE_S += el0_entry.S`; `KTEST_S` + `KTEST_C += ktest_el0.c`
