# Session 53: slice 7.6d.3a — signal-on-fault

**Date:** 2026-04-26
**Phase:** 7 slice 7.6d.3a (busybox integration scaffolding, sub-slice 1 of 4)
**Branch:** master

---

## Goals

The BLOCKING precondition for re-enabling slice 7.6d.2c's busybox
test, and the standard POSIX-shaped fault-to-exit conversion that
turns "kernel halt" into "parent observes 128+signo" for any EL0
program that misbehaves.

Concrete change list:
- Update `on_sync` in `core/cpu/exception.c` to convert lower-EL
  fault EC values into `nx_process_exit(128 + signo)`:
  - EC 0x20 (Inst Abort, lower EL) → SIGSEGV (signo 11) → exit 139
  - EC 0x24 (Data Abort, lower EL) → SIGSEGV → exit 139
  - EC 0x00 (Unknown reason) from EL0 (per saved `tf->pstate.M`) →
    SIGILL (signo 4) → exit 132
  - EC 0x21 / 0x25 (current-EL faults) → kernel panic (unchanged)
  - EC 0x00 from EL1 → kernel panic
- Two new ktests that EL0-fault on purpose and assert the parent
  observes the matching `128 + signo` exit code:
  - `posix_segfault_prog` — child writes to PA 0 via VA 0
    (device_block, AP=EL1-only) → SIGSEGV → 139.
  - `posix_undef_prog` — child executes `.word 0` (UDF #0) →
    SIGILL → 132.
- Re-enable `ktest_posix_busybox` — slice 7.6d.2c left it out of
  `KTEST_C` because the unconverted fault would `halt_forever` and
  break `make test`.  With 7.6d.3a in place the fault converts
  cleanly to status 139 and the test passes.

## Scope choices

- **Convert at `on_sync` (the C-level dispatcher), not in
  `_sync_stub` (the asm vector).**  All current-EL and lower-EL
  synchronous-exception entries already share `_sync_stub` →
  `SAVE_TRAPFRAME` → `bl on_sync` → `RESTORE_TRAPFRAME` → `eret`,
  so the trap frame is on the kstack and the C code has
  `tf->pstate` to differentiate ECs that don't carry the EL in
  ESR (notably EC=0x00 "Unknown reason").  Splitting the asm
  vector for lower-EL would require duplicating SAVE/RESTORE in
  a per-EL copy + a redundant dispatcher; not worth it for the
  small sliver of EC values we handle.

- **Use `tf->pstate` (saved SPSR_EL1) to differentiate EC=0x00
  origin.**  The CPU saves the source PSTATE into SPSR_EL1 at
  exception entry, including the M[3:0] bits (mode + EL + SP
  selector).  M=0b0000 = EL0t — the only EL0 mode we use.  Any
  other value means EL1.  EC=0x00 from EL0 is "EL0 ran an
  undefined instruction"; from EL1 is "kernel ran one" (= bug).

- **Hardcode signo values 11 / 4 in `exception.c`, don't pull
  in `framework/syscall.h`'s `NX_SIGKILL`/`NX_SIGTERM`.**  Those
  constants are for the existing slice-7.5 polled-signal path
  (delivered via `sys_signal`); fault-to-signal is a separate
  path.  Tying the two together would mean a future signal
  renumbering in one path silently changes exit codes in the
  other.  Keep the well-known POSIX signo values explicit and
  named (`NX_FAULT_SIGSEGV = 11`, `NX_FAULT_SIGILL = 4`).

- **`deliver_el0_fault_signal` is `__attribute__((noreturn))`.**
  `nx_process_exit` doesn't return (parks in `wfe`).  Without
  the annotation, gcc complains about implicit fallthrough in
  the switch statement (`-Werror=implicit-fallthrough`).  The
  attribute also lets gcc skip generating dead code after the
  call sites.  A `halt_forever()` follows the
  `nx_process_exit` as a safety net in case nx_process_exit
  ever grows a return path.

- **Two-test minimum, not one combined.**  SIGSEGV via data
  abort and SIGILL via undef are two distinct EC codepaths
  (0x24 vs 0x00, with 0x00 needing the SPSR check).  Separate
  tests pinpoint regressions: if someone breaks the SPSR
  check, `ktest_posix_undef` fails specifically; if they break
  the data-abort case, `ktest_posix_segfault` fails
  specifically.  No combined "executes both fault types"
  demo.

- **The default branch in the EC switch falls through to
  panic, doesn't convert.**  Other lower-EL EC values
  (FP/SIMD trap 0x07, watchpoint 0x34, HVC/SMC 0x12+, etc.)
  exist but no current demo trips them.  Defaulting to
  SIGSEGV would mask kernel bugs (e.g. an unexpected
  watchpoint hit).  Defer the conversion until a real demo
  needs it.

## What Was Done

### `core/cpu/exception.c`

- Added `framework/process.h` include (for `nx_process_exit`).
- Added named EC constants: `ESR_EC_UNKNOWN`, `ESR_EC_SVC64`
  (existing), `ESR_EC_INST_ABORT_EL0`, `ESR_EC_INST_ABORT_EL1`,
  `ESR_EC_DATA_ABORT_EL0`, `ESR_EC_DATA_ABORT_EL1`.
- Added `NX_FAULT_SIGSEGV = 11`, `NX_FAULT_SIGILL = 4` constants
  (POSIX values, hardcoded — see Scope choices).
- Added `saved_pstate_was_el0()` static inline that masks
  `pstate & 0xf` and compares to 0 (M[3:0] = EL0t).
- Added `deliver_el0_fault_signal(signo, reason, esr, far, pc)`
  noreturn helper that prints a one-line marker and calls
  `nx_process_exit(128 + signo)`.  Includes a
  `halt_forever()` safety net.
- Rewrote `on_sync`'s fault path: SVC dispatch unchanged (still
  the early `return`).  After the FAR read, a switch on EC
  routes EC 0x20 / 0x24 directly to the fault helper, EC 0x00
  through the SPSR check + helper, and any other unhandled EC
  through the existing `kprintf` + `halt_forever`.

### `test/kernel/posix_segfault_prog.c` + `posix_segfault_prog_blob.S` + `ktest_posix_segfault.c`

- Demo: parent forks; child writes `*(volatile int *)0 = 42`
  (PA 0, slot 0 of L2_mmio, AP=EL1-only — guaranteed
  permission fault from EL0).  Parent waits + asserts status
  == 128 + SIGSEGV (= 139), emits `[segv-ok]`.
- Three markers expected: `[segv-parent][segv-child][segv-ok]`.
- ktest matches the existing libnxlibc-linked-EL0 scaffolding
  pattern (musl_exec_parent, posix_busybox).  Independent
  cross-check: walk the process table, find a child of `segv`
  with `exit_code == 139`.

### `test/kernel/posix_undef_prog.c` + `posix_undef_prog_blob.S` + `ktest_posix_undef.c`

- Demo: parent forks; child executes `asm volatile (".word 0")`
  (encoding 0x00000000, the canonical UDF #0).  Parent waits +
  asserts status == 128 + SIGILL (= 132), emits `[undef-ok]`.
- Three markers expected: `[undef-parent][undef-child][undef-ok]`.
- ktest mirrors the segfault one; cross-check looks for
  `exit_code == 132`.

### `Makefile`

- KTEST_C list extended: `ktest_posix_segfault.c`,
  `ktest_posix_undef.c`, `ktest_posix_busybox.c`.
- KTEST_S list extended: matching `posix_*_blob.S` files.
- The "intentionally NOT in KTEST_C" comment block from slice
  7.6d.2c removed — busybox test now belongs in the build.
- Build rules added for `posix_segfault_prog.{o,elf}` +
  `posix_segfault_prog_blob.o`, `posix_undef_prog.{o,elf}` +
  `posix_undef_prog_blob.o` (template from the existing
  libnxlibc-linked demos).
- Clean rule extended.

## Drive-by gotchas

- **`-Werror=implicit-fallthrough` caught my missing
  `noreturn`.**  First compile attempt failed because gcc
  thought the switch cases would fall through after the helper
  call.  Adding `__attribute__((noreturn))` to the helper
  fixed it cleanly.  Worth keeping in mind for future fault-
  path additions.

## Verification

- `make test` → **410/410** (51 python + 275 host + 84 kernel),
  0 leaks, 0 errors, exit 0.  Up from 407/407 — the three new
  ktests plus the 7.6d.2c busybox test (now passing).

- Live ktest log fragments from the new tests:

```
posix_segfault_child_data_abort_becomes_sigsegv_exit_139 [segv-parent][segv-child]
[EXC] EL0 data_abort ESR=9200004e FAR=0 ELR=48000070 -> exit 139
[segv-ok]PASS

posix_undef_child_executes_udf_becomes_sigill_exit_132 [undef-parent][undef-child]
[EXC] EL0 undef ESR=2000000 FAR=0 ELR=4800006c -> exit 132
[undef-ok]PASS

posix_busybox_help_parent_forks_and_execs_busybox [busybox-parent]
[EXC] EL0 data_abort ESR=9200004e FAR=20 ELR=48002460 -> exit 139
[busybox-status=8b][busybox-help-failed]PASS
```

The busybox fault is the same `__memset_generic+0xa0` write to
`0x20` that 7.6d.2c captured — but now it's a clean status-139
exit instead of `halt_forever`, and `[busybox-status=8b]` (139
in hex) confirms the parent observed it.

## What Was Not Done

- **No fix for the busybox crash itself.**  That's still
  7.6d.3b (richer AUXV) + 7.6d.3c (TPIDR_EL0 init).  This
  slice only changes "the kernel halts when busybox crashes"
  to "the kernel converts busybox's crash into a 139 the
  parent observes."  Big difference operationally — busybox
  runs no further than before, but every other test in the
  suite stays runnable.
- **No support for handler-registered signals** (POSIX
  `signal()` / `sigaction()`).  Faults always force-exit.
  Catchable signals require a signal-handler framework
  (per-process disposition table, signal mask, return-from-
  handler trampoline) — bigger than this slice.  Kernel signals
  with handlers land in a future slice when first user pulls
  one in.
- **No FAR-based attribution** — we just print FAR in the
  marker line.  Future "page-fault-vs-permission-fault" or
  "missing-page-could-be-mapped" logic uses the DFSC field
  in ESR ISS; not needed today.

## Test Counts

- Python: 51 (unchanged)
- Host: 275 (unchanged)
- Kernel: 84 (+3 — segfault, undef, busybox-help re-enabled)
- **Total: 410/410 PASS**

## Next Steps

- **7.6d.3b** — Push richer AUXV (`AT_HWCAP` + `AT_HWCAP2` +
  `AT_PLATFORM`) so musl's `__init_libc` doesn't read garbage
  feature flags or crash on a missing AT_PLATFORM string.
  May not change the busybox failure mode by itself — but
  it's a quick win with low risk.
- **7.6d.3c** — Pre-initialize `TPIDR_EL0` to a kernel-
  allocated zeroed thread-control area before `eret` to EL0.
  This is the actual fix for the captured TLS-gap crash;
  busybox's first errno-touch will then resolve to a valid
  PA inside the buffer rather than VA 0x20 in MMIO space.
- After 7.6d.3b + 7.6d.3c, re-test busybox: probably it
  reaches further into init before another (TBD) gap.

---

**Last Updated:** 2026-04-26
