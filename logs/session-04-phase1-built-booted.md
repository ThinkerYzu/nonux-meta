# Session 4: Phase 1 built and booted

**Date:** 2026-04-20
**Phase:** Phase 1 complete
**Branch:** master (sources/nonux)

---

## Goals

- Install the ARM64 cross-toolchain and QEMU
- Build the Phase 1 kernel written in Session 1
- Boot it in QEMU and verify the banner + memory-layout printout
- Add a reusable QEMU run script
- Unblock Phase 2

## What Was Done

### Toolchain verified

User installed the Debian/Ubuntu packages `gcc-aarch64-linux-gnu`, `binutils-aarch64-linux-gnu`, and `qemu-system-arm`. Confirmed present:

- `aarch64-linux-gnu-gcc` 15.2.0
- `qemu-system-aarch64` 10.1.0

### Build

Ran `make` at the top of `sources/nonux/`. Clean compile of `start.S`, `boot.c`, `string.c`, `printf.c`, link with `core/boot/linker.ld`, then `objcopy -O binary` to `kernel.bin`. One benign warning: `kernel.elf has a LOAD segment with RWX permissions` — expected for a freestanding kernel image; not addressing in v1. Artifacts: `kernel.elf` 72320 B, `kernel.bin` 4720 B.

Section layout (from `objdump -h`):

| Section | VMA | Size |
|---|---|---|
| .text | 0x40000000 | 0x870 |
| .rodata | 0x40001000 | 0x12b |
| .eh_frame | 0x40001130 | 0x100 |
| .got | 0x40001230 | 0x28 |
| .got.plt | 0x40001258 | 0x18 |

### Boot in QEMU

Ran `timeout --preserve-status 5 qemu-system-aarch64 -M virt -cpu cortex-a53 -nographic -kernel kernel.bin -m 1G`. Output:

```
========================================
  nonux — composable microkernel
  ARM64 / QEMU virt
========================================

[boot] kernel loaded at 0x40000000
[boot] BSS:  0x0000000040002000 — 0x0000000040002000
[boot] kernel end:    0x0000000040002000
[boot] free memory:   0x0000000040042000

[boot] Phase 1 complete — halting.
```

Kernel then parks in `wfe` as designed; QEMU exits on the 5-second timeout with signal 15, which is expected. The `make run` target has no self-exit — Phase 1's halt is a `wfe` loop, not a QEMU semihosting exit.

### Reusable run script

Added `sources/nonux/tools/run-qemu.sh` so the QEMU invocation flags (`-M virt -cpu cortex-a53 -nographic -kernel ... -m ...`) live in one place instead of being re-typed every run. Features:

- No args → interactive (Ctrl-A X to exit), equivalent to `make run`.
- `-t SECONDS` → wrap in `timeout --preserve-status`. This is the main reason the script exists: until a kernel self-exit path exists, every scripted run needs a timeout, and pasting the full QEMU line into each call site is error-prone.
- `--` passthrough for extra QEMU flags (e.g. `-append test`).
- `QEMU` and `QEMU_MEM` env vars override defaults.
- Verified: `tools/run-qemu.sh -t 3` prints the full boot banner and exits with status 0.

Deliberately **not** included: GDB/`-s -S` mode (the existing `make debug` target already covers that workflow, and combining them would duplicate the invocation). `make run` and `make debug` stay as-is — the script is additive, not a replacement.

## Key Findings

- **BSS is empty** (`0x40002000 — 0x40002000`, zero bytes). No uninitialized globals exist yet in Phase 1 code, so the linker emits no `.bss`. The boot printout is honest — this is not a bug. Phase 2 will add uninitialized globals (PMM bitmap, tracking structures) and the range will populate.
- **Kernel end reported at 0x40002000** even though the last allocated section ends around `0x40001270`. The linker rounds the kernel image up to the next 4 KiB boundary before the stack; Phase 1's printout uses that rounded value, which is the correct "first page after the image."
- **Free memory at 0x40042000** = kernel end + 256 KiB stack (matches linker script).
- **Timeout + `--preserve-status`** is the right invocation for `make run` in scripted contexts until a Phase 2 self-exit path exists.

## Decisions Made

- **Keep the RWX-segment linker warning silent for now.** Splitting `.text` and data into separate PT_LOAD segments is a Phase 2+ concern once the MMU is on; noisy to suppress now and the warning is correct.
- **Don't change the boot printout** even though the empty BSS looks odd. It reflects reality. Phase 2 will exercise it.
- **Add `tools/run-qemu.sh` now, not later.** Original plan was "`make test-kernel` will own the scripted-run wrapper in Phase 2," but the script is trivial and useful for any ad-hoc scripted boot (CI smoke checks, dev loops, bisection). Keeping it narrow: only interactive and timed modes. GDB stays under `make debug`.

## Status at End of Session

- Working: cross-toolchain, `make`, `make run`, `tools/run-qemu.sh` (both interactive and `-t` timed paths verified).
- Not yet working: `make test`, `make test-kernel` (no test harness yet), `make validate-config` (no component manifests parsed yet).
- Tests: none exist yet; Phase 2 introduces host-side tests and `mem_track`.

## Next Steps

- **Phase 2: Test harness + physical memory.**
  1. `test/host/test_runner.c` — minimal host-gcc runner for per-module unit tests
  2. `core/lib/mem_track.c` — tracking layer (counters on every alloc/free, leak report on drain)
  3. `core/pmm/pmm.c` — bitmap allocator with host-side tests first, then kernel-side
  4. `core/irq/gic.c` — GICv2 init
  5. `core/cpu/vectors.S` — exception vectors, trap handler skeleton
  6. ARM Generic Timer tick → first "kernel does something over time" output
- Open question for Session 5: should `mem_track` use C11 `_Atomic` counters from day 1 (consistent with the "preemption requires atomics in v1" rule) even though SMP isn't up yet? Leaning yes — cost is near zero and it matches the project-wide rule.

---

**Files Changed:**
- `sources/nonux/tools/run-qemu.sh` — **new**, executable. Single source of truth for QEMU invocation flags.
- `sources/nonux/README.md` — Quick Start block mentions the timed-run option.
- Build artifacts produced (not committed): `kernel.elf` 72 KB, `kernel.bin` 4720 B.
- `proj_docs/nonux/HANDOFF.md` — status updated to "Phase 1 booted", Next Actions advanced to Phase 2, Session 4 entry added, `tools/run-qemu.sh` listed under What We Have.
- `proj_docs/nonux/README.md` — status line + "Last Updated" bumped.
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — top status line bumped; Phase 1 marked DONE; Running section documents the script.
