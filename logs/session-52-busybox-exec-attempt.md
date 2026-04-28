# Session 52: slice 7.6d.2c — first busybox exec attempt + failure capture

**Date:** 2026-04-26
**Phase:** 7 slice 7.6d.2c (busybox integration, sub-sub-slice 3 of 3 — closes 7.6d.2)
**Branch:** master

---

## Goals

The actually-discovery-driven sub-slice that 7.6d.2's original plan
was about.  Preconditions from 7.6d.1 (vendor + cross-compile)
+ 7.6d.2a (re-link at user-window VA) + 7.6d.2b (grow window +
fix latent bugs) are all in place.  This session adds the first
attempt at exec'ing the cross-compiled busybox and captures
whatever failure mode surfaces first.

Concrete change list:
- Bump `RAMFS_FILE_CAP` 8192 → 4 MiB so the 2.29 MB busybox binary
  fits at a ramfs path.
- Bump `SYS_EXEC_MAX_FILE` 8192 → 4 MiB to match.
- Add busybox to `tools/pack-initramfs.py`'s entry list as
  `/bin/busybox`.
- Write `posix_busybox_help_prog.c` (libnxlibc-linked parent that
  forks + execs `/bin/busybox --help`).
- Write `ktest_posix_busybox.c` that drives the parent program
  through the standard libnxlibc-linked-EL0-test scaffolding.
- Capture the failure: an unhandled EL0 data abort inside busybox's
  musl init.

## Scope choices

- **`RAMFS_FILE_CAP = 4 MiB`, not a smaller bump.**  busybox is
  2.29 MB; round up to the next clean power-of-2 with headroom for
  later busybox versions or other large binaries.  Cost: 32 MiB
  static .bss for the per-instance file table (8 slots × 4 MiB
  each).  Cheap on a 1 GiB-RAM system; expensive in concept.  The
  v1-hack character of this is documented in the new comment;
  long-term direction is per-file dynamic allocation as part of
  the Phase-8+ memory rework already tracked in HANDOFF.md.

- **`SYS_EXEC_MAX_FILE = 4 MiB` matches `RAMFS_FILE_CAP`.**  Per-
  call kheap allocation; no static cost.  Each `sys_exec` briefly
  holds a 4 MiB buffer.

- **`/bin/busybox`, not `/busybox`.**  busybox itself dispatches
  applets by `argv[0]`'s basename; the conventional layout under
  busybox-as-init is `/bin/busybox` plus applet symlinks
  (`/bin/sh -> busybox`, `/bin/ls -> busybox`, etc.).  We pre-
  position the convention now so 7.6d.N's symlink work has a
  natural home.  ramfs treats path strings as flat names (no
  hierarchy) — `/bin/busybox` is just a string, no `/bin/`
  directory needed.

- **Don't add the test to `KTEST_C` yet.**  busybox crashes under
  musl init (see "Captured failure mode" below); our v1 kernel's
  EL0 fault handler `halt_forever`s, which would break `make
  test` 407/407.  The test source + demo source + blob.S all stay
  in the tree as the discovery deliverable; 7.6d.3.x re-enables
  the test after wiring (a) signal-on-fault and (b) the musl-
  init gap.

- **Status marker as hex byte, not decimal.**  Hand-formatted
  two-hex-digit byte (`[busybox-status=NN]` with NN in hex)
  rather than pulling in libnxlibc's printf.  Keeps the demo
  binary tiny + makes the marker grep-friendly in the live log.

## What Was Done

### `components/ramfs/ramfs.c`

- `RAMFS_FILE_CAP` bumped from `8192u` to `(4u * 1024u * 1024u)`
  (4 MiB).  Comment block extended with the bump history (256 →
  4096 → 8192 → 4 MiB) and the v1-hack rationale (32 MiB static
  cost; long-term ramfs rework would dynamically allocate per-
  file).

### `framework/syscall.c`

- `SYS_EXEC_MAX_FILE` bumped from `8192u` to `(4u * 1024u * 1024u)`
  (4 MiB).  Comment notes per-call kheap allocation, no static
  cost.

### `Makefile`

- Initramfs entry list gains `$(BUSYBOX_BIN):/bin/busybox`.
  `initramfs.cpio` rule deps on `$(BUSYBOX_BIN)` so a busybox
  rebuild correctly triggers an initramfs rebuild.
- Build rules added for `test/kernel/posix_busybox_help_prog.{o,elf}`
  + `posix_busybox_help_prog_blob.o` (same shape as the existing
  libnxlibc-linked demos: `POSIX_PROG_CFLAGS` for the .c, `LD -n -T
  init_prog.ld $(O) -lnxlibc` for the .elf).
- `KTEST_C` and `KTEST_S` lists deliberately do NOT include the new
  ktest + blob.S — see scope choice above.  A 9-line comment in
  `KTEST_C` explains the reason and the file paths so future readers
  see the discovery files immediately.
- Clean rule extended with `posix_busybox_help_prog.elf`.

### `test/kernel/posix_busybox_help_prog.c`

- New libnxlibc-linked EL0 program.  `main()` forks; child
  `nxlibc_execve("/bin/busybox", { "/bin/busybox", "--help",
  NULL }, NULL)`; parent emits `[busybox-parent]`, waits, prints
  `[busybox-status=NN]` (NN as a hex byte), prints either
  `[busybox-help-ok]` (status == 0) or `[busybox-help-failed]`.
  Child has a fall-through sentinel after `execve` returns
  (`[busybox-exec-failed]` + `nxlibc_exit(96)`) so the marker
  pattern distinguishes "exec returned (couldn't even start)"
  from "exec succeeded but busybox's main exited non-zero".

### `test/kernel/ktest_posix_busybox.c`

- New ktest that spawns the parent kthread under `nx_process_create
  ("bbh")`, waits up to 32768 yields for the parent to EXITED
  (higher than other tests because exec'ing 2.29 MB takes
  considerably longer than the 7.3 KB / 37 KB demos), asserts
  parent exit_code == 0 and `nx_syscall_debug_write_calls() >= 3`.
  Same shape as `ktest_musl_exec.c` modulo the marker count + yield
  budget.

### `test/kernel/posix_busybox_help_prog_blob.S`

- Standard `.incbin` wrapper between weak symbols, matching every
  other demo's blob.S.

## Captured failure mode

When the test is enabled (commenting it back into `KTEST_C` for a
manual run), the live ktest log shows:

```
posix_busybox_help_parent_forks_and_execs_busybox [busybox-parent]
[EXC] sync  ESR=9200004e FAR=20 ELR=48002460 SPSR=20000000
```

Decoded:
- `ESR = 0x9200_004e`
  - EC bits[31:26] = `0b100100` = `0x24` → **Data Abort, lower EL**
  - IL bit[25] = 1 (32-bit instruction)
  - ISS[6] (WnR) = 1 → **write** access caused the fault
  - ISS[5:0] (DFSC) = `0x0e` → **Permission fault, level 2**
- `FAR = 0x20` — faulting VA (writes to a low MMIO-range address)
- `ELR = 0x48002460` — EL0 PC at fault time (offset `0x2460` into
  busybox's text)
- `SPSR = 0x2000_0000` — EL0t (M=0), V flag set

Disassembly at `0x48002460` shows `__memset_generic+0xa0`:
```
48002460: 3d800000 str q0, [x0]   ← faulting instruction
```

`x0` is the destination pointer.  FAR=`0x20` means `x0 = 0x20` at
fault time — i.e., the caller of `memset` passed a destination of
`0x20`, a low MMIO-range address with no EL0 write access.

This is the classic symptom of musl's TLS being uninitialized.
musl on AArch64 uses `TPIDR_EL0` to point at the thread structure;
TLS-relative variables (notably `errno` via `__errno_location()`,
which returns `&__pthread_self()->errno`) resolve to small offsets
from the TLS pointer.  If `TPIDR_EL0` is 0 (uninitialized at EL0
entry — we don't write it), then `errno` lives at a small
positive offset from NULL — `0x20` is consistent with the offset
of `errno` (or a similar field) within the `pthread` struct.

busybox's main() and the surrounding musl init touch errno during
just about every syscall return path.  The first `memset` of a
zero-initialized heap allocation that lands inside a TLS-relative
struct is what trips the fault.

## What Was Not Done

- **No fix for the crash itself.**  That's 7.6d.3.x.  The likely
  fix is one or more of:
  1. **Push more AUXV** — at minimum `AT_HWCAP` (musl checks
     `hwcap & HWCAP_ATOMICS` etc.); maybe `AT_BASE`, `AT_PHDR`,
     `AT_PHENT`, `AT_PHNUM`, `AT_ENTRY` for static-PIE-style
     binaries.  Our current AUXV is just `AT_PAGESZ` + `AT_RANDOM`
     + `AT_NULL`.
  2. **Initialize TPIDR_EL0** to point at a valid (zeroed)
     thread-control area before `eret` to EL0.  musl's `__init_libc`
     calls `__set_thread_area(td)` which boils down to `msr
     tpidr_el0, x0`; but this only runs after the program has
     executed enough init to construct `td`, which itself uses
     `errno` ⇒ a chicken-and-egg.  The kernel-side option is to
     pre-zero TPIDR_EL0 and let musl's init populate it.
  3. **Patch musl's init paths** that use TLS before TLS is
     initialized.  Less likely needed; musl is well-tested.

- **No EL0-fault recovery.**  Today `on_sync` for non-SVC EC
  prints the EXC and `halt_forever`s the kernel.  A signal-on-
  fault model (POSIX SIGSEGV → `nx_process_exit(128 + signo)`)
  would let the parent's `wait()` collect the child's death and
  report it through the normal exit path.  Tracked as a 7.6d.3.x
  prerequisite (without it, we can't even add the busybox test
  to `make test`).

- **No `/bin/busybox` applet symlinks** (`/bin/sh -> busybox`
  etc.).  That's 7.6d.N's job.  The path layout is positioned for
  it.

## Verification

- `make test` → 407/407 pass (51 python + 275 host + 81 kernel).
  Same count as slice 7.6d.2b — the discovery files are in the
  tree but not wired into the test build.
- `make busybox` → unchanged 2,292,464-byte binary at
  `third_party/busybox/busybox`.
- `make test/kernel/initramfs.cpio` → 2,305,888 bytes, 5 entries +
  trailer.
- Manual enable of `ktest_posix_busybox` (uncomment from `KTEST_C`
  + add blob to `KTEST_S`) reproduces the captured EXC reliably.

## Test Counts

- Python: 51 (unchanged)
- Host: 275 (unchanged)
- Kernel: 81 (unchanged — discovery test held out)
- **Total: 407/407 PASS**

## Next Steps — 7.6d.3 enumeration based on actual observation

The originally-speculative 7.6d.3+ list is now grounded in real
data.  Concrete sub-slices:

- **7.6d.3a (BLOCKING) — Signal-on-fault.**  Replace `on_sync`'s
  `halt_forever` for non-SVC + lower-EL fault with `nx_process_exit
  (128 + signo)` semantics so the parent's `wait()` collects the
  death.  Maps:
  - EC `0x20` (Instruction Abort, lower EL) → SIGSEGV (signo 11) →
    exit 139
  - EC `0x21` (Instruction Abort, current EL) → kernel panic
    (kernel itself faulted, halt_forever stays)
  - EC `0x24` (Data Abort, lower EL) → SIGSEGV (signo 11) → exit 139
  - EC `0x25` (Data Abort, current EL) → kernel panic
  - EC `0x00` (Unknown reason) from EL0 → SIGILL (signo 4) → exit 132
  Without this, no test that exercises a misbehaving EL0 program
  can run cleanly.  Unblocks the busybox test re-enable.
- **7.6d.3b — Push richer AUXV.**  Add at minimum `AT_HWCAP`
  (zero is acceptable — busybox just won't take ATOMICS path) +
  `AT_HWCAP2` (zero) + `AT_PLATFORM` (point at a static "aarch64\0"
  string).  Per-architecture musl needs vary; check
  `third_party/musl/src/env/__init_tp.c` and friends.
- **7.6d.3c — Pre-initialize TPIDR_EL0** to a kernel-allocated
  zeroed thread-control area before `eret` to EL0.  musl can then
  read errno + write its own TLS pointer once `__init_libc` runs.
- **7.6d.3.x (later) — `mmap` / `clone.s` / `vfork.s`** — only if
  busybox's `--help` path actually calls them after the TLS gap
  is closed.  Re-test, observe what fails next.
- **7.6d.N — `/init = busybox sh`** — once `--help` runs to
  completion, drive the actual shell.  Initramfs gains applet
  symlinks (or duplicate-file entries — TBD when symlink support
  is needed).

---

**Last Updated:** 2026-04-26
