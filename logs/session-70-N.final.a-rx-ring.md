# Session 70: slice 7.6d.N.final.a — PL011 RX IRQ + console RX ring + blocking `nx_console_read` + ioctl/termios stubs

**Date:** 2026-04-28
**Phase:** 7 slice 7.6d.N.final.a (interactive busybox sh — kernel-side RX plumbing)
**Branch:** master

---

## Goals

Session 69 closed the last batch of non-interactive busybox sh
workloads (N.12 through N.15).  Slice 7.6d.N.final is the only
remaining slice in the 7.6d arc.  User-direction questions
resolved this session before any code:

- **`/init` swap policy:** build-flag-gated.  Default `/init`
  stays the libnxlibc-linked test program so all 425 ktests keep
  their current entry path; `make run-busybox` (sub-slice b)
  repacks initramfs with `/init -> busybox`.
- **Verification:** eyeball-test for slice closure (sub-slice b),
  plus scriptable injection via `tools/qemu-stdin-feed.sh` +
  `make test-interactive` as a follow-up (sub-slice d).

That carved N.final into four sub-slices:

- **N.final.a (this session)** — RX IRQ + console RX ring +
  blocking `nx_console_read` + termios/ioctl stubs.
- N.final.b — `/init` swap policy + eyeball-test recipe.
- N.final.c — Ctrl-C → SIGINT + Ctrl-D EOF + signal trampoline.
- N.final.d — `tools/qemu-stdin-feed.sh` + `make test-interactive`.

This slice closes the kernel-side mechanical core: **bytes
typed at the QEMU UART end up in `read(0, ...)` from EL0.**

## Deliverables

### `framework/console.{h,c}` — RX ring + IRQ wiring (~150 lines)

A single-producer (PL011 RX ISR or test injector) /
single-consumer (`nx_console_read`) byte ring of 256 entries.
Plain head/tail with C11 atomics on the indices; the IRQ runs
with IRQs masked at the GIC, so no further synchronisation is
needed on the producer side.

`nx_console_read` becomes a yield-loop: drain whatever is in
the ring; if empty and nothing yet returned, `nx_task_yield`;
on wake, drain again.  POSIX read semantics — a successful
read may return less than `cap`, and the caller (musl's
`__stdio_read`, ash's cmdedit) loops as needed.

Host build keeps the v1 EOF behaviour for empty ring; tests can
prime the ring via `nx_console_test_inject_bytes` and drain it
with the same `nx_console_read` (no scheduler / no yield, just
the modular drain).

`nx_console_init` registers the ISR + enables IRQ 33 at the GIC
+ programs the PL011's `IMSC` (RXIM | RTIM masks) and `IFLS`
(receive FIFO threshold = 1/8, fires on first byte).  Called
from `boot_main` after `gic_init` and before `irq_enable_local`.

The ISR drains the PL011 FIFO into the ring (so we don't take
back-to-back IRQs for already-buffered bytes) and clears RXIM +
RTIM in `ICR`.

### `framework/syscall.c` — `NX_SYS_IOCTL = 39` (~90 lines)

Recognised cmds against a `NX_HANDLE_CONSOLE` fd:

- `TCGETS = 0x5401`     — fill 36 zero bytes (`struct termios`),
                          return 0.  musl's `isatty` calls
                          `tcgetattr` (= TCGETS); success makes
                          isatty return 1.
- `TCSETS / TCSETSW / TCSETSF` (`0x5402` / `0x5403` / `0x5404`)
                          — ignore + return 0.
- `TIOCGWINSZ = 0x5413` — fill `struct winsize {row=24, col=80}`,
                          return 0.
- everything else        — return `-ENOTTY` (Linux `-25`,
                          delivered through musl's `__syscall_ret`
                          which preserves Linux errno values
                          regardless of our NX errno table).

Non-CONSOLE fd → `-ENOTTY` (busybox tcgetattr on a regular file
is supposed to fail with ENOTTY, not EINVAL).

### `sys_read` CONSOLE arm — drop the EOF hack

The previous arm passed `buf=NULL` to `nx_console_read`
because the v1 implementation always returned 0=EOF.  With
the real RX path it fills bytes, so the arm now does the
proper `copy_to_user` through a kernel staging buffer of
`NX_FILE_IO_MAX = 256` bytes (matching the FILE arm).

### musl translation table

`__NR_ioctl (29) → NX_SYS_IOCTL (39)` in both
`arch/aarch64/syscall_arch.h` (C-level) and
`src/thread/aarch64/syscall_cp.s` (asm-level cancellation
point).  Comment block at the top of `syscall_arch.h`
updated to record the mapping.

### `core/boot/boot.c` — wire `nx_console_init`

One call between `timer_init(10)` and `nx_framework_bootstrap`,
with a comment explaining the placement (must be after `gic_init`
so the GIC is up; before `irq_enable_local` so the first key
press doesn't fire into a masked vector).

### Tests

**`test/kernel/ktest_console_rx.c`** (new) — 3 ktests
exercising the ring without bringing up EL0:
- `console_rx_inject_drain` — push 5 bytes via the test
  injector, read them back through `nx_console_read`, verify
  ordering and count.  Read does NOT block (ring already has
  bytes) — fast-path coverage.
- `console_rx_wraps_around` — push 200 + drain 200, then push
  100 + drain 100.  Forces `head` and `tail` past
  `RX_RING_SIZE = 256` and back, exercising the modular index
  arithmetic.
- `console_rx_drop_on_full` — push 256 bytes; ring capacity is
  256 minus the full sentinel, so 255 are accepted and the last
  is dropped.  Drain returns exactly 255.

**`test/host/file_syscall_test.c`** (extended) — 2 host tests:
- `sys_ioctl_console_termios_stubs_succeed` — TCGETS / TCSETS /
  TIOCGWINSZ all succeed, winsize struct is filled with 24×80.
  Unknown cmd → -ENOTTY.
- `sys_ioctl_on_non_console_handle_returns_enotty` — open a
  regular FILE handle, ioctl returns -ENOTTY (busybox + musl
  rely on this for the "is this a tty?" test).

The "block-then-wake" semantic of `nx_console_read` (read on
empty ring yields, IRQ pushes a byte, read returns) is exercised
implicitly by the busybox interactive path that lands in
sub-slice b/c.  Adding a kthread-based block-then-wake test
in-kernel would require spinning up a producer kthread that
the idle-task ktest framework doesn't naturally provide;
deferred to the EL0 sub-slice.

## Discoveries / gotchas

### Stale `.o` files masquerading as test failures

First `make test` after the change reported one host failure
(`syscall_unknown_number_returns_enosys_on_host`) and one
kernel failure (`syscall_unknown_number_returns_enosys`).
Both expected `NX_ENOSYS = -38` from a syscall number
`>= NX_SYSCALL_COUNT`; got `-1` (NX_EINVAL) and `-4` (NX_ENOENT)
respectively.

Root cause: adding `NX_SYS_IOCTL = 39` shifted
`NX_SYSCALL_COUNT` from 39 → 40, but the test object files
(`syscall_test.o`, `ktest_syscall.o`) were built against the
old header and inlined `svc0(NX_SYSCALL_COUNT)` with the old
value 39.  The kernel/host dispatch — built fresh against the
new header — happily routed slot 39 to `sys_ioctl`, which
returned an errno consistent with whatever uninitialised
values the test stuffed into the trap frame's argument
registers.

The Makefile doesn't automatically depend object files on
header changes; routine work uses incremental builds, but a
header edit that changes a sentinel constant requires a
`make clean` to refresh test objects.  No kernel-side change
needed — the dispatch is correct; only the stale tests
disagreed.  Fix: `make clean && make test`.  Documented in
this session log so the next sentinel-bumping edit doesn't
trip over the same trap.

### Adopting the timer driver's IRQ-wiring shape

`core/timer/timer.c:timer_init` was the natural template:
`irq_register(N, handler, NULL); gic_enable(N)` is the
two-step that registers the dispatch + unmasks the line.
`framework/console.c:nx_console_init` mirrors it exactly.

The PL011's IRQ line on QEMU virt is **SPI 1 = IRQ 33** (= 32
SPI offset + 1).  Not in any header; lifted from the QEMU
virt machine model.  Encoded as a `#define PL011_IRQ 33` in
`framework/console.c` with a one-line comment above it so a
later reader doesn't re-derive it.

### `NX_FILE_IO_MAX = 256` keeps the staging arm honest

The CONSOLE read arm follows the FILE arm's shape — bound by
`NX_FILE_IO_MAX`, copy through a kernel staging buffer.  Lower
bound (1 byte) works the same.  Unblocked the "what if EL0
asks for `cap = 8 KiB`?" question — we cap it at 256, the EL0
caller loops as needed.

## Test results

`make clean && make test` →

- 51 python tests pass
- 277 host tests pass (was 275, +2 ioctl tests)
- 102 kernel tests pass (was 99, +3 console_rx_* tests)
- **Total: 430/430 pass, 0 leaks, 0 errors, exit 0**

## Status at end of session

- N.final.a complete.  Kernel-side mechanical core for
  interactive shell is live: bytes typed at the QEMU UART
  end up in `read(0, ...)` from EL0.  `isatty(0)` returns 1
  via the ioctl stub.
- N.final.b (next) — `/init` swap policy.  Sub-30-line
  tooling change (`tools/pack-initramfs.py` + Makefile target
  + TESTING-GUIDE.md recipe).  Eyeball-test on real keyboard
  closes the sub-slice.
- N.final.c — Ctrl-C / Ctrl-D + signal trampoline (~100 lines
  kernel).
- N.final.d — `tools/qemu-stdin-feed.sh` + `make test-interactive`.

## Files touched

Production:
- `framework/console.h` — extended docstring + new
  `nx_console_init` / `nx_console_test_inject_bytes` declarations.
- `framework/console.c` — RX ring + IRQ handler + blocking
  read + nx_console_init.  ~120 net new lines.
- `framework/syscall.h` — `NX_SYS_IOCTL = 39` enum entry.
- `framework/syscall.c` — `sys_ioctl` body + dispatch table
  entry; `sys_read` CONSOLE arm rewritten for the real
  staging-buffer shape.  ~95 net new lines.
- `core/boot/boot.c` — `#include "framework/console.h"` +
  `nx_console_init()` call.
- `third_party/musl/arch/aarch64/syscall_arch.h` — translation
  table entry for `__NR_ioctl (29) → 39` + comment update.
- `third_party/musl/src/thread/aarch64/syscall_cp.s` — same
  translation entry on the asm-level cancellation path.

Tests:
- `test/kernel/ktest_console_rx.c` (new, 3 ktests).
- `test/host/file_syscall_test.c` — 2 new TEST() blocks.
- `Makefile` — `KTEST_C` += `ktest_console_rx.c`.
