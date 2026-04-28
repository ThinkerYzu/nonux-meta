# Session 25: slice 5.4 — SVC syscall entry

**Date:** 2026-04-23
**Phase:** 5 slice 5.4
**Branch:** master

---

## Goals

Wire up the first syscall path: `svc #0` from EL1 kernel code →
synchronous-exception vector → `on_sync` → `nx_syscall_dispatch` →
syscall body → return value in x0 → `eret`. Two real syscalls
(`nx_debug_write`, `nx_handle_close`) prove the plumbing end-to-end.
Slice 5.5 will add the EL0 side of the same handler.

## What Was Done

### `framework/syscall.{h,c}` — new

- `typedef int64_t nx_status_t` — signed 64-bit return type so negative
  NX_E* codes survive the x0 trip through register storage
  unambiguously.
- `enum nx_syscall_number`: `NX_SYS_RESERVED_0`, `NX_SYS_DEBUG_WRITE`,
  `NX_SYS_HANDLE_CLOSE`, sentinel `NX_SYSCALL_COUNT`. Number 0 is
  deliberately reserved — any `svc #0` with x8 unset lands on a NULL
  slot and dispatches to NX_EINVAL rather than silently invoking op 0.
- `void nx_syscall_dispatch(struct trap_frame *tf)` — reads `tf->x[8]`
  as the syscall number, validates against the table bound, calls the
  handler with `tf->x[0..5]` as args, writes the result to `tf->x[0]`
  so it lands in x0 at `eret`.
- `nx_syscall_current_table()` — returns the caller's handle table.
  Slice 5.4 returns a single static global (`g_kernel_handles`); slice
  5.5 will rewrite the body to `&nx_task_current()->handle_table`.
  Every other call site stays unchanged.
- `nx_syscall_reset_for_test()` — tests call this between cases to
  wipe the global table so leftover handles don't bleed.

Syscall bodies:

- `sys_debug_write(buf, len)` — per-byte `uart_putc` on kernel; returns
  byte count. Host build returns `len` without touching stdout so
  dispatch tests still validate arg pass-through without racing with
  the test runner's output.
- `sys_handle_close(h)` — thin wrapper around `nx_handle_close` against
  the current table.

The dispatch table is `const`, so dispatch itself is lock-free by
construction. Handlers may touch shared state (UART, the global
handle table) — those touch points manage their own invariants.

### `core/cpu/exception.c` — SVC routing

`on_sync` now reads ESR_EL1, decodes `EC = (esr >> 26) & 0x3f`, and
dispatches to `nx_syscall_dispatch(tf)` when EC == 0x15 (SVC AArch64).
Non-SVC synchronous exceptions keep the existing halt-and-log
behaviour — real faults are still fatal. The vector-table entry
already tells us which EL the exception came from; we only use EC to
distinguish SVC from data/instruction aborts at whichever EL.

### Host vs kernel trap-frame shape

`core/cpu/exception.h` has AArch64 inline asm (`daifclr/daifset`), so
the host build of `framework/syscall.c` can't include it. Instead:

- Kernel: `#include "core/cpu/exception.h"` for `struct trap_frame`.
- Host: locally mirror the same layout inside the `#if __STDC_HOSTED__`
  block so host dispatch tests can construct synthetic frames without
  touching ARM headers. Both layouts encode `x[31]; sp_el0; pc;
  pstate` — any drift surfaces as test failures.

### Kernel tests (+7 in `test/kernel/ktest_syscall.c`)

Inline-asm SVC wrappers (`svc0`, `svc1`, `svc2`) pin `x8` to the
syscall number, pin args to `x0`/`x1`, issue `svc #0`, and read back
x0 as the result. Kept test-local because production kernel code
should never issue SVCs (userspace does that).

- `syscall_unknown_number_returns_einval` — reserved 0, out-of-range,
  and exactly-at-sentinel all reject with NX_EINVAL.
- `syscall_debug_write_returns_byte_count` — UART gets the message
  (visible in the kernel-test log), return value equals byte count.
- `syscall_debug_write_zero_length_returns_zero` — NULL + len=0 is OK,
  returns 0.
- `syscall_debug_write_null_buf_nonzero_len_returns_einval`.
- `syscall_handle_close_through_svc_closes_handle_in_kernel_table` —
  end-to-end: alloc a handle in the current table, SVC-close it, look
  up again expecting NX_ENOENT.
- `syscall_handle_close_invalid_handle_returns_einval`.
- `syscall_resumes_at_instruction_after_svc` — sentinel check that
  `eret` lands at the instruction after `svc #0`, not before (would
  mean `elr_el1` is wrong) and preserves pstate.

### Host tests (+6 in `test/host/syscall_test.c`)

Feed synthetic `struct trap_frame_host` into `nx_syscall_dispatch`
(no SVC on host) — exercises the dispatch-table logic and argument
decoding in isolation.

- Unknown-number EINVAL (three cases: 0, out-of-range, sentinel).
- NULL trap frame is a no-op, not a crash.
- `debug_write` returns len with correct arg pass-through.
- `debug_write` with NULL buf + nonzero len rejects.
- `handle_close` routes through `nx_syscall_current_table` and really
  closes the handle.
- `nx_syscall_reset_for_test` clears the kernel table.

### Build wiring

- `Makefile`: `FW_C += framework/syscall.c`; `KTEST_C += test/kernel/ktest_syscall.c`.
- `test/host/Makefile`: SRCS += `syscall_test.c`, `../../framework/syscall.c`.

Net: `make test` → **293/293** (51 python + 197 host + 44 kernel), up
from 279/279 — **+6 host + 7 kernel = +13 tests**.

## Key Decisions

- **Dispatch decision in C, not assembly.** The existing `_sync_stub`
  already saves a full trap frame and calls `on_sync`. Adding a
  separate SVC-only assembly stub would duplicate the save/restore
  machinery for no gain. Reading ESR + branching in C costs one
  unpredicted branch per syscall — negligible next to the SVC
  instruction itself.
- **Linux AArch64 ABI (x8 = syscall number, x0..x5 = args).** Slice
  5.5 and beyond will have real EL0 userspace issuing SVCs; using the
  de facto Linux ABI means compilers and libcs that already target
  Linux AArch64 need no adapter.
- **Reserved syscall 0 slot.** A `const` NULL in the dispatch table
  means a stray `svc #0` with x8 unset (or zero-initialised memory
  passing as x8) returns NX_EINVAL rather than invoking whatever op
  happens to sit at index 0. Avoids an entire class of "buggy caller
  accidentally invoked a real op" footguns.
- **Global kernel handle table in 5.4.** `nx_syscall_current_table`
  returns `&g_kernel_handles`. This is placeholder scaffolding —
  slice 5.5 makes the current-table a per-task pointer. Isolating
  that under a single function means the dispatcher doesn't change
  shape in 5.5.
- **Int64 `nx_status_t`, not plain `int`.** AArch64 x0 is 64-bit;
  returning `int` works but means callers see sign-extended
  low-32-bit values. Committing to int64 up-front avoids width
  surprises when future syscalls return negative values like handle
  IDs (which are uint32 at the C level but arrive in x0 with the
  upper 32 bits zero — indistinguishable from small positive ints).
- **Tests issue SVC directly in inline asm, no libc wrappers.** Per-
  syscall wrapper generation (à la Linux's `glibc` syscall stubs)
  lands when slice 5.5 introduces EL0 userspace. At the kernel-test
  layer, raw inline asm is simpler and more honest about what's
  being tested.

## Known Issues / Follow-ups

- No user-pointer validation yet. `sys_debug_write` trusts `buf` is
  readable for `len` bytes. Kernel-issued SVCs pass kernel pointers,
  so no attack surface today — but slice 5.5 must add a `copy_from_user`
  primitive before EL0 callers land.
- The ESR.EC decode happens on every synchronous exception, not just
  SVC. Non-SVC hot paths (page faults, once they exist) might want a
  dedicated assembly stub that jumps straight to the page-fault
  handler instead of round-tripping through `on_sync`. Non-blocking
  for 5.5; bench it when it matters.
- No SVC `#imm16` handling. The instruction can carry a 16-bit
  immediate encoded in ESR.ISS, which some ABIs use to distinguish
  syscall classes. We read the immediate implicitly (always 0 in our
  inline asm) but ignore it. If a future phase wants fast-path
  inline-asm shims that encode the syscall number in the `#imm16`
  instead of `x8`, the dispatcher already has the ESR handy and can
  prefer ISS over x8 with one extra line.

## Next Actions

**Slice 5.5 — First EL0 process.** `core/cpu/el0_entry.S` for the
drop-to-EL0 path via `eret` with SPSR_EL1 set to EL0t. Per-task TTBR0
page table. Baked-in EL0 test program that issues `nx_debug_write`
then exits via a new `nx_process_exit` syscall. `sched_spawn_el0_task`
extends 4.4's kthread primitive. Kernel test verifies the EL0 program
runs, UART logs the expected output, and control returns to the idle
task cleanly.

## Files Touched

- `framework/syscall.h` — **new**
- `framework/syscall.c` — **new**
- `core/cpu/exception.c` — SVC routing in `on_sync`
- `test/host/syscall_test.c` — **new**, 6 cases
- `test/kernel/ktest_syscall.c` — **new**, 7 cases
- `Makefile` — `FW_C += framework/syscall.c`; `KTEST_C += ktest_syscall.c`
- `test/host/Makefile` — SRCS + vpath for syscall_test.c and framework/syscall.c
