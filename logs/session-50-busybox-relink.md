# Session 50: slice 7.6d.2a — re-link busybox at user-window VA

**Date:** 2026-04-25
**Phase:** 7 slice 7.6d.2a (busybox integration, sub-sub-slice 1 of 3 — opens 7.6d.2)
**Branch:** master

---

## Goals

7.6d.2 (try to boot busybox `--help` from a libnxlibc parent) was
originally one sub-slice.  Planning surfaced two preconditions that
have to land before any runtime discovery can happen:

1. **Address-space mismatch.**  busybox 1.36.1 is linked at VA
   `0x400000` (kbuild default for `-static -no-pie` aarch64).  Our
   user window sits at `0x48000000`.  Loading busybox-as-built would
   write to `0x400000`, which is in MMIO space — guaranteed fault.
2. **Window too small.**  Even after re-linking, busybox's
   text+data segments span ~1.91 MB, while the user window is only
   2 MiB total — leaves no room for stack/heap.

So 7.6d.2 was further sliced 7.6d.2a (re-link) / 7.6d.2b (grow
window + relocate brk) / 7.6d.2c (caps + ramfs-seed + try to exec
+ capture failure).  This session lands 7.6d.2a — the smallest of
the three, build-script-only, no kernel changes, no ktest changes.

## Scope choices

- **`-Wl,-Ttext-segment=0x48000000` over a custom linker script.**
  busybox's kbuild link command is generated from `Makefile.flags`
  and `scripts/trylink`; injecting a full linker script there would
  require either patching the Makefile or replacing it with a custom
  one.  `-Ttext-segment=...` rides through cleanly in `EXTRA_LDFLAGS`,
  which busybox already plumbs to its link line.  The flag pins
  segment 1 (text+rodata, R+E) at the requested VA; segment 2
  (data+bss, RW) follows the standard alignment offset (`p_align =
  0x10000` per the program header) so it lands at
  `0x481d9770`, well above segment 1's end at `0x481c6434`.
- **Hardcode `0x48000000` in build-busybox.sh, not derive from a
  kernel header.**  The VA is also hardcoded in `init_prog.ld`'s
  `. = 0x48000000;` and in `core/mmu/mmu.c`'s `USER_WINDOW_INDEX
  = 64` macro arithmetic.  Three places that have to track each
  other — but extracting a shared `USER_WINDOW_BASE` constant
  would force build-time generation of the linker script and the
  shell script from a single source, which is more plumbing than
  we want for a one-line value.  The script comment flags the
  cross-reference; the next time the user window moves, this is
  one of the call sites the grep will find.
- **No size-hardening yet.**  The re-linked binary is the same
  size as the un-re-linked one (2,292,464 bytes — the only diff
  is the segment header VAs and a few absolute addresses in the
  text).  No need to investigate `--gc-sections` / `-Os` /
  applet trimming yet; that's only worth doing if 7.6d.2c
  surfaces a runtime size constraint we hadn't predicted.

## What Was Done

### `tools/build-busybox.sh`

- Added `USER_WINDOW_BASE="0x48000000"` constant + extended the
  `EXTRA_LDFLAGS` line to include `-Wl,-Ttext-segment=$USER_WINDOW_BASE`.
- 14-line comment above the change documents the cross-reference
  to `core/mmu/mmu.c` + `init_prog.ld` and the rationale for not
  factoring the constant.

That's the entire code change.  No Makefile change (the existing
`$(BUSYBOX_BIN)` rule already invokes the script with the right
deps).  No kernel change.  No new ktest.

## Verification

- `make busybox-clean && make busybox` succeeds; output ELF is
  the same 2,292,464 bytes as Session 49's first build.
- `aarch64-linux-gnu-readelf -h` shows the new entry point:
  ```
  Entry point address:   0x480005c0
  ```
  (was `0x4005c0`).
- `aarch64-linux-gnu-readelf -l` shows segments at the new VAs:
  ```
  LOAD  Offset 0x000000   VirtAddr 0x48000000   FileSiz 0x1c6434  R E
  LOAD  Offset 0x1c9770   VirtAddr 0x481d9770   FileSiz 0x82aa    RW
  ```
  Both segments now sit inside the user-window VA range
  (`0x48000000` – `0x48200000`).
  - Segment 1 ends at `0x481c6434` (~1.86 MB into the window).
  - Segment 2 ends at `0x481e8930` (~1.91 MB into the window).
  - Top of window is `0x48200000` (2 MiB) — leaves ~84 KiB for
    stack + heap.  Confirms the window-size issue 7.6d.2b will
    address.
- `make test` → 407/407 pass (51 python + 275 host + 81 kernel),
  0 leaks, 0 errors, exit 0.  No regression from the re-link.

## Drive-by gotchas

None.  The change went in clean on first try; busybox's build
system handled the new flag without complaint.

## What Was Not Done

- **No window growth.**  busybox still doesn't fit the 2 MiB window
  alongside heap+stack — that's 7.6d.2b's job.
- **No exec attempt.**  ramfs is still capped at 8 KiB per file;
  trying to seed a 2.29 MB busybox would silently truncate.
  7.6d.2c bumps the caps + ramfs-seeds + tries to exec.
- **No source-tree constants extracted.**  The `0x48000000` hardcode
  lives in three places (build-busybox.sh + init_prog.ld +
  mmu.c arithmetic) and that's not changing this slice.

## Test Counts

- Python: 51 (unchanged)
- Host: 275 (unchanged)
- Kernel: 81 (unchanged)
- **Total: 407/407 PASS**

Build outputs:
- `third_party/busybox/busybox` — re-linked at `0x48000000`,
  2,292,464 bytes (identical size to pre-relink).

## Next Steps

- **7.6d.2b** — Grow user window 2 MiB → 8 MiB; relocate brk
  region.  `USER_WINDOW_SIZE` bumps to 4 contiguous L2 ram blocks
  (indexes 64..67); per-process backing alloc footprint scales 4×
  (~32 MiB raw + 8 MiB alignment slack); brk region moves from
  `[base+1 MiB .. base+1.5 MiB)` to `[base+6 MiB .. base+7.5 MiB)`
  (well past busybox's segment-2 end at ~`base+1.91 MiB`); `sp_el0`
  formula stays `top - 16` (since the formula references
  `mmu_user_window_base() + mmu_user_window_size()` rather than a
  hardcoded VA).  Risk surface: every existing EL0 ktest depends on
  the 2 MiB layout — has to stay green.
- **7.6d.2c** — Bump `RAMFS_FILE_CAP` + `SYS_EXEC_MAX_FILE`;
  ramfs-seed `/bin/busybox`; libnxlibc parent execs it; capture
  failure mode.  This is where the actual runtime discovery happens
  and 7.6d.3+ sub-slices get enumerated.

---

**Last Updated:** 2026-04-25
