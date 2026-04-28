# Session 6: Kernel-side test harness

**Date:** 2026-04-20
**Phase:** Phase 2 polish (completed before moving on to Phase 3)
**Branch:** master (sources/nonux)

---

## Goals

- Add an in-kernel test harness so we can exercise code paths the host-side
  framework can't reach (real IRQ dispatch, real PMM against real physical
  RAM, timer behaviour).
- Make `make test-kernel` actually work — previously a stub that was
  supposed to pipe QEMU output through a Python checker.
- Keep the production `kernel.bin` slim (no test code in the shipping
  image).

## What Was Done

### Harness (`test/kernel/ktest.{h,c}`, `ktest_main.c`)

Same self-registering shape as the host runner:

```c
KTEST(name) { … }
```

expands to a `struct ktest_entry` placed into a `kernel_test_registry`
linker section. `ktest_main()` walks from `__start_kernel_test_registry`
to `__stop_kernel_test_registry` and runs each entry.

Assertions: `KASSERT`, `KASSERT_EQ_U`, `KASSERT_NOT_NULL`, `KASSERT_NULL`.
On failure they print file:line + the expression, set
`ktest_current_failed`, and `return` from the test. No setjmp/longjmp —
test authors are expected to `return` immediately after a failed
assertion (the macros do that implicitly).

### Exit path — ARM semihosting

`ktest_exit(int status)` invokes `SYS_EXIT_EXTENDED` (op 0x20) via
`hlt #0xf000` with x1 pointing at a `{ADP_Stopped_ApplicationExit,
exit_code}` param block. QEMU, launched with `-semihosting`, propagates
the `exit_code` as its own process exit code. That makes
`make test-kernel` a clean pass/fail for CI: exit 0 = all tests passed,
exit 1 = one or more failed.

Verified both paths: happy path → `make` exits 0, single injected
`KASSERT_EQ_U` failure → `make: *** [Makefile:115: test-kernel] Error 1`.

### Two binaries, minimal duplication (`Makefile`)

`kernel.bin` (production) and `kernel-test.bin` (test build) share every
object file **except** `boot.o`:

- `core/boot/boot.o` — plain compile, used by `kernel.bin`
- `core/boot/boot-test.o` — recompiled from the same `boot.c` with
  `-DNX_KTEST`; inside, a `#ifdef NX_KTEST` branch at the end of
  `boot_main` calls `ktest_main()` instead of entering the wfi loop

`kernel-test.elf` links the shared objects + `boot-test.o` + the
`ktest_main.o` / test-case objects. No recursive make, no build-dir
variables. Trade-off: sharing `.o` files means you can't mix a test and
production build in the same tree without a `make clean` when switching
CFLAGS — fine in practice since the two builds are reached via different
`.o` names (`boot.o` vs `boot-test.o`) and the rest is identical.

### `make test-kernel` target

```make
test-kernel: kernel-test.bin
	timeout --preserve-status 15 \
	    $(QEMU) -M virt,gic-version=2 -cpu cortex-a53 -nographic \
	    -m $(QEMU_MEM) -semihosting -kernel kernel-test.bin
```

`--preserve-status` so a well-behaved guest exit (status 0/1 via
semihosting) bubbles up unchanged; the outer `timeout 15` catches hangs
from infinite fault loops (an SError / sync fault in a test would re-
enter the vector forever).

### Initial test cases (6 total)

**PMM (`test/kernel/ktest_pmm.c`)** — 4 tests:
- `pmm_alloc_page_is_aligned_and_above_kernel`
- `pmm_alloc_page_is_writable_and_readable`
- `pmm_full_cycle_restores_free_count` (128 alloc / 128 free)
- `pmm_contiguous_alloc_is_contiguous` (4-page block, touch first and
  last byte)

**IRQ (`test/kernel/ktest_irq.c`)** — 2 tests:
- `irq_timer_ticks_advance_under_busy_wait` — spin 250 ms on
  `CNTPCT_EL0`, confirm `timer_ticks()` advanced (proves the full IRQ
  path from timer hardware through the vector to the handler is live)
- `irq_wfi_wakes_on_timer` — `wfi` and confirm tick count changed on
  wakeup

All pass. Running `make test` in the clean tree:

```
PASS: 11/11 tests passed, 0 leaks, 0 errors    ← host
...
ktest: 6/6 PASSED, 0 failures                  ← kernel
```

### Linker-section wiring

`kernel_test_registry` is explicitly placed in `.rodata` with
`__start_/__stop_` markers defined in `core/boot/linker.ld`. GNU ld only
auto-generates those markers for *orphan* sections — once we KEEP the
pattern inside an output section, the auto-markers don't apply.

### TESTING-GUIDE.md filled in

Previously a template placeholder. Now documents both harnesses, the full
17-test inventory (11 host + 6 kernel), how to run subsets, expected
output, how to add a new test, and troubleshooting entries for the
actual failure modes we'd seen (hang vs semihosting missing vs stale
VPATH .o, etc.).

## Key Findings

- **Semihosting exit-code propagation just works on QEMU virt.** No
  special `-semihosting-config` needed — plain `-semihosting` is enough
  for SYS_EXIT_EXTENDED on aarch64 at EL1 TCG.
- **`__start_/__stop_` auto-markers don't cross orphan-vs-placed
  boundaries.** Placing `KEEP(*(kernel_test_registry))` inside `.rodata`
  made the section non-orphan and the linker stopped auto-generating
  markers; you must define them in the script when you `KEEP` like
  that. Learned this the usual way — a failed link with
  `undefined reference to __start_kernel_test_registry`.
- **Our `kprintf` doesn't support `%-40s`.** Literal `%-40s` came out
  in the first test run. The project's minimal `kprintf` only parses
  `%s %d %u %x %p %c %lu %lx %ld` without width modifiers. Worked
  around with a manual `uart_putc(' ')` loop in the runner (good
  enough; extending `kprintf` is out of scope). Documented in
  TESTING-GUIDE troubleshooting.
- **Sharing `.o` between two kernel binaries is clean as long as only
  the *entry* module differs.** Only `boot.c` needs the `-DNX_KTEST`
  branch; every other subsystem (PMM, GIC, timer, vectors, printf)
  compiles once and links into both. That's a much simpler build than
  maintaining parallel object directories.

## Decisions Made

- **No setjmp/longjmp in kernel tests.** Kernel exception state makes a
  clean unwind hairy; tests just `return` on a failed KASSERT.
- **Tests run with IRQs live and the timer armed.** A separate "tests
  under masked IRQs" mode is more ceremony than it's worth at Phase 2;
  the current tests want IRQs on anyway. If we ever need quiescence we
  can wrap a test body in `irq_disable_local()/irq_enable_local()`.
- **No per-test filter yet.** `make test-kernel` always runs the full
  suite. Adding a `-ksuite` argument parser for QEMU-forwarded flags
  (or a `NX_KTEST_FILTER=name` env var baked into the image) is
  easy but not warranted until we have >20 tests.
- **Minimal first-round test coverage.** Four PMM + two IRQ tests are
  a smoke check for the harness itself more than a thorough PMM suite.
  Host PMM already has 9 algorithm tests; kernel tests only need to
  prove the *integration* (real memory, real IRQ path) is sound.
- **Outer `timeout 15`.** If a broken test causes a recursive sync
  fault, QEMU would otherwise spin forever. 15 s is generous enough
  for full-suite runs on slow CI boxes and short enough that a hang
  is painful to live with, encouraging a real fix.

## Status at End of Session

- `make test` → **17/17 pass (11 host + 6 kernel), exit 0** ✓
- `make` still builds production `kernel.bin`; `tools/run-qemu.sh -t 4`
  still shows tick prints ✓
- Production `kernel.bin` has no test code linked ✓
- Failure path verified by injecting a broken assertion → `Error 1`
  propagated ✓

## Next Steps

- **Phase 3: component framework.** The harness is now available to
  Phase 3's conformance tests (scheduler / VFS / etc. interfaces will
  each have a generic test suite that any implementation must pass —
  see DESIGN.md "Interface Conformance Tests"). Per-component lifecycle
  health tests will also live in `test/kernel/` once the component
  framework exists.
- **Opportunistic polish:**
  - Extend `kprintf` with width modifiers (`%-Ns`, `%5d`) if more tests
    want aligned columns.
  - Per-test filter for kernel tests when suite grows past ~20.
  - Kernel-side equivalent of `mem_track` (tracked `kmalloc`) — needed
    once Phase 3 components start allocating dynamically.

---

**Files Changed:**
- **new:** `test/kernel/ktest.h`, `test/kernel/ktest_main.c`,
  `test/kernel/ktest_pmm.c`, `test/kernel/ktest_irq.c`
- **edited:** `core/boot/boot.c` — `#ifdef NX_KTEST` branch at end of
  `boot_main`
- **edited:** `core/boot/linker.ld` — `kernel_test_registry` placement
  with explicit `__start_/__stop_` markers inside `.rodata`
- **edited:** `Makefile` — `kernel-test.bin` target, `boot-test.o`
  rule, rewritten `test-kernel` target with `-semihosting` and
  `timeout --preserve-status 15`, `clean` now also removes
  `kernel-test.{elf,bin}`
- **edited (proj_docs):** `TESTING-GUIDE.md` filled in from template
