# Session 54: slice 7.6d.3b + 7.6d.3c — busybox actually runs

**Date:** 2026-04-27
**Phase:** 7 slice 7.6d.3b + 7.6d.3c (busybox integration scaffolding, sub-slices 2 + 3 of 4)
**Branch:** master

---

## Goals

The user's "why don't you finish 7.6d.2?" pushed me to keep
going past 7.6d.3a's signal-on-fault until busybox actually
runs `--help` end-to-end (rather than just cleanly faulting and
dying).  This session combines 7.6d.3b (richer AUXV) and 7.6d.3c
(pre-init `TPIDR_EL0`) plus a third gap that surfaced during
debugging — busybox was being linked against **glibc, not musl**
because the cross-toolchain's library search paths beat
`--sysroot` for `-lc` / `-lm` resolution.  Closing all three
gets `[busybox-status=00][busybox-help-ok]` in the live ktest
log alongside the full `BusyBox v1.36.1 ...` banner.

## Scope choices

- **Combine 7.6d.3b + 7.6d.3c into one session.**  Originally
  planned as separate sub-slices; the session-53 plan was
  "land each 7.6d.3.x independently."  But after 7.6d.3a I
  could see all three (AUXV + TLS + glibc-vs-musl) needed to
  land together for busybox to actually run, and shipping
  intermediate "still crashes, just at a different EC"
  milestones isn't useful.  Single session, single commit.

- **TLS area at user-window offset 5 MiB.**  Lives in the unused
  gap between busybox's data+bss segment (ends at +1.91 MiB)
  and the brk heap region (`[+6 MiB, +7.5 MiB)`).  4 KiB is
  plenty for musl's pre-`__set_thread_area` lifetime — musl
  touches `errno` (offset 0x20) plus a handful of small
  fields, all within the first ~256 bytes of `struct pthread`.
  Once `__init_libc` calls `__set_thread_area(td)`, musl
  manages its own larger pthread struct in the heap region
  and the kernel's pre-init buffer is dead.

- **Save/restore `TPIDR_EL0` per task in sched_check_resched.**
  Per-task save/restore (around `cpu_switch_to`) rather than
  per-process — once musl runs `__set_thread_area`, each task
  owns a distinct TPIDR value.  Initial value is the kernel-
  pre-init TLS area for fresh tasks; fork copies the parent's
  current TPIDR_EL0; exec resets to the fresh kernel-pre-init
  area.

- **Push AT_PHDR + AT_PHENT + AT_PHNUM at minimum.**  musl's
  `static_init_tls` walks the program-header table via these
  three AT_* values to find PT_TLS.  Without them, the loop
  zero-iterates and downstream TLS-allocation math gets
  garbage state.  AT_PHDR is computed as `window_base +
  e_phoff` — for our static-no-pie binaries the ELF's first
  PT_LOAD starts at file offset 0 and maps to `window_base`,
  so file offset `e_phoff` lives at VA `window_base + e_phoff`.
  No AT_HWCAP / AT_PLATFORM / AT_ENTRY pushed yet — busybox
  doesn't require them for `--help`; defer until a real demo
  needs them.

- **Discovered + fixed: busybox was glibc-linked, not musl.**
  The slice-7.6d.1 build script passed `--sysroot=$MUSL_SYSROOT`
  and `-static`, expecting that to bias gcc's library search
  toward musl.  It doesn't — `gcc -print-search-dirs --sysroot=...`
  shows the cross-toolchain's bundled glibc paths
  (`/usr/aarch64-linux-gnu/lib/...`) come BEFORE the sysroot
  paths in the library search order, so `-lc` resolved to
  glibc's libc.a and `-lm` to glibc's libm.a.  The resulting
  busybox linked `__libc_setup_tls` + `_IO_*` (glibc internals)
  and never touched musl's syscall translation — every syscall
  went via the raw Linux number, returned -ENOSYS, and
  busybox crashed early in init regardless of what the kernel
  did.  Fix: a `tools/musl-aarch64-gcc.sh` wrapper that adds
  `-nostdinc -isystem $SYSROOT/include` (force only musl headers)
  + `-isystem /usr/aarch64-linux-gnu/include` (kernel headers
  the cross-toolchain ships, which musl doesn't replicate)
  + `-B $SYSROOT/lib/` (force musl's crt1/crti/crtn/libc.a to
  be found before glibc's).  Plus symlinks
  `lib<m,pthread,rt,dl,util,resolv,crypt>.a → libc.a` in the
  sysroot — musl bundles those into libc.a but busybox's link
  line passes `-lm` etc., and the symlinks let `-l<x>` resolve
  to musl's combined archive.  Build script switches to
  `CC="$MUSL_GCC"`.

- **Compute `e_phoff` / `e_phentsize` / `e_phnum` via the existing
  ELF parser, not by re-reading raw header bytes.**  Extended
  `struct nx_elf_info` (in `framework/elf.h`) with three new
  fields; `nx_elf_parse` populates them.  `sys_exec` already
  parses the ELF header via `nx_elf_parse` — the new fields
  fall out for free.

## What Was Done

### `framework/process.h`

- Added `NX_PROCESS_TLS_OFFSET = 5 MiB` and `NX_PROCESS_TLS_SIZE
  = 4 KiB` macros.  Comment block describes the layout choice +
  the musl-init lifetime pattern.

### `core/mmu/mmu.c`

- `mmu_create_address_space` zeroes a 4 KiB region at
  `user_pa + NX_PROCESS_TLS_OFFSET` after the user-backing
  malloc.  PMM hands out un-zeroed pages, so without this
  musl's first errno read returns garbage and trips downstream.
- New include of `framework/process.h` for the TLS macros.

### `core/sched/task.h` + `core/sched/task.c`

- `struct nx_task` gains `uint64_t tpidr_el0` field.
- `nx_task_create` initializes `tpidr_el0 =
  mmu_user_window_base() + NX_PROCESS_TLS_OFFSET` for fresh
  tasks (host build sets 0).
- `nx_task_create_forked` reads the *current* `TPIDR_EL0` (via
  `mrs`) and copies it into the new child task.  Real Linux's
  fork preserves TLS identically; we match.

### `core/sched/sched.c`

- `sched_check_resched` saves outgoing task's `TPIDR_EL0` and
  restores incoming task's, alongside the existing TTBR0 flip.
  Two `mrs`/`msr` per context switch — cheap.  Documented
  alongside the comment for the TTBR0 flip.

### `framework/syscall.c`

- `sys_exec` resets `TPIDR_EL0` (CPU register + the per-task
  field) to `mmu_user_window_base() + NX_PROCESS_TLS_OFFSET`
  after `mmu_switch_address_space`.  Necessary because exec
  replaces the entire image — the pre-exec process may have
  called `__set_thread_area` and TPIDR_EL0 might point into
  the now-freed old user_backing.
- AUXV layout extended.  `fixed_size = 8u * (15u + argc)`
  (was `8 * (9 + argc)`) to fit three more AT pairs.  New
  entries:
  - `AT_PHDR (3) → window_base + info.phoff`
  - `AT_PHENT (4) → info.phentsize`
  - `AT_PHNUM (5) → info.phnum`
  Comment block explains the AT_* trio and the
  `window_base + e_phoff` derivation.

### `framework/elf.h` + `framework/elf.c`

- `struct nx_elf_info` gains `phoff` (uint64_t), `phentsize`
  (uint16_t), `phnum` (uint16_t).
- `nx_elf_parse` populates them from the ELF header.

### `tools/musl-aarch64-gcc.sh` (new)

- gcc wrapper that biases the search order toward musl:
  - `-nostdinc -isystem $SYSROOT/include` (musl headers only
    for libc-namespace stuff)
  - `-isystem /usr/aarch64-linux-gnu/include` (kernel headers
    the cross-toolchain ships — `linux/*.h`, `asm/*.h` —
    that busybox needs and musl doesn't replicate)
  - `-B $SYSROOT/lib/` (musl's crt + libc found before glibc's)
- 36-line script with a 25-line preamble explaining why the
  plain `--sysroot` approach didn't work.

### `tools/build-busybox.sh`

- Switched busybox build from `EXTRA_CFLAGS=--sysroot=...` to
  `CC="$MUSL_GCC"` (the wrapper).  EXTRA_LDFLAGS keeps `-static
  -Wl,-Ttext-segment=$USER_WINDOW_BASE` (no `--sysroot` —
  the wrapper handles that).
- `HOSTCC=cc` added explicitly so busybox's host-side helpers
  (kconfig, applet_tables) still build with the system gcc.
- musl-libm/pthread/rt/dl/util/resolv/crypt symlinks created in
  `$SYSROOT/lib/` — all point at `libc.a`, since musl bundles
  these into the combined archive.

## Drive-by gotchas

- **busybox shrinks from 2.29 MB to 1.22 MB.**  Half the size,
  because it's no longer pulling in glibc's stdio infrastructure
  (`_IO_*` was huge — buffered streams, locale, wide-char
  conversions, etc.).  musl's stdio is a fraction of glibc's.
  Pleasant side effect: the binary fits with more headroom in
  our 8 MiB user window.

- **Initial musl wrapper attempt missed kernel headers.**
  `-nostdinc` removed everything including `linux/kd.h` etc.
  busybox's kbd_mode applet needs them.  Fix: add
  `-isystem /usr/aarch64-linux-gnu/include` (cross-toolchain's
  kernel headers).  The order matters — musl's headers come
  first so `stdio.h` etc. resolve to musl, but `linux/*` falls
  through to the cross-toolchain.

- **Initial AUXV-only attempt didn't fix anything.**  Before
  realizing the busybox-vs-musl link issue, I pushed AT_PHDR/
  PHENT/PHNUM expecting the captured fault to move.  It didn't:
  busybox kept faulting at `__memset_generic+0xa0` writing to
  `0x20`.  Once I dumped x0/x1/x2/x30 from the trap frame, I
  saw the caller (`x30=0x48007124`) was `__libc_setup_tls`
  (a glibc symbol).  That's when the glibc-vs-musl confusion
  clicked.  Lesson: dumping caller-PC + arg registers on every
  fault is high-leverage diagnostic info.

- **Initial busybox test "passed" with status 139.**  Slice
  7.6d.3a's signal-on-fault converted busybox's TLS-gap
  fault into exit 139, and the ktest assertion was satisfied
  with `parent exit_code == 0`.  Busybox didn't actually run
  — it just cleanly died.  The user's "why don't you finish
  7.6d.2?" was the right kick to push past this.

- **`make test` not picking up new busybox.**  After fixing
  the link issue, `make test` still showed the old failure.
  Reason: `initramfs.cpio` and `kernel-test.elf` weren't
  re-linked because their dep on `busybox` is via the
  `$(BUSYBOX_BIN)` variable but the underlying file timestamp
  hadn't changed in a way make noticed.  `rm -f
  test/kernel/initramfs.cpio kernel-test.elf kernel-test.bin
  && make test` forced the rebuild.  Worth a Makefile
  cleanup pass to make this dependency cascade reliable —
  filed implicitly via this drive-by.

## Verification

- `make test` → **410/410** pass (51 python + 275 host + 84
  kernel), 0 leaks, 0 errors, exit 0.  Same count as session
  53 — but now `posix_busybox_help_parent_forks_and_execs_busybox`
  passes with the busybox `--help` actually completing, not
  just cleanly faulting.

- Live ktest log fragment from the busybox test:
```
posix_busybox_help_parent_forks_and_execs_busybox [busybox-parent]BusyBox v1.36.1 (2026-04-27 01:11:03 CST) multi-call binary.
Usage: busybox [function [arguments]...]
   or: busybox --list[-full]
   or: busybox --show SCRIPT
   or: busybox --install [-s] [DIR]
        ...
        tsort, tty, ttysize, ubiattach, ubidetach, ubimkvol, ubirename,
        ubirmvol, ubirsvol, ubiupdatevol, uevent, umount, uname, unexpand,
        uniq, unix2dos, unlink, unlzma, unshare, unxz, unzip, uptime, users,
        usleep, uudecode, uuencode, vi, volname, w, wall, watch, watchdog, wc,
        which, who, whoami, xargs, xxd, xz, xzcat, yes, zcat
[busybox-status=00][busybox-help-ok]PASS
```

- `nm busybox | grep _IO_` still shows `_IO_*` symbols, but
  they're WEAK aliases that musl provides for glibc-source
  compatibility (a few apps use these directly).  `__libc_setup_tls`
  (glibc-only) is GONE.  busybox is musl-linked.

- busybox binary size: 1,224,312 bytes (was 2,292,464 — drop
  of 47%).

## What Was Not Done

- **Slice 7.6d.N** — `/init = busybox sh` boots to a shell
  prompt over serial.  busybox's `sh` applet needs symlink
  dispatch (`/bin/sh -> busybox`) which neither cpio's flat-
  file model nor our ramfs supports yet.  Plus `sh` will fault
  on any number of new gaps (`mmap`, `clone`, terminal ioctls).
  Whichever fault surfaces becomes the next sub-slice.
- **No `mmap` syscall.**  busybox `--help` doesn't need it
  (musl's mallocng falls back to brk for small allocations);
  the shell will.
- **No catchable signals.**  Fault delivers SIGSEGV/SIGILL via
  `nx_process_exit(128 + signo)`, force-exit only.  busybox's
  shell wants `signal(SIGINT, SIG_IGN)` etc. — needs handler
  framework, deferred.
- **AT_HWCAP / AT_HWCAP2 / AT_PLATFORM not pushed.**  Originally
  in the 7.6d.3b plan; turned out busybox `--help` doesn't
  need them.  Push when first user does.
- **Makefile dep cascade for busybox → initramfs → kernel-test
  not robust.**  Manual `rm -f` workaround above.

## Test Counts

- Python: 51 (unchanged)
- Host: 275 (unchanged)
- Kernel: 84 (unchanged — busybox ktest already in the count
  from session 53; now actually passes with `--help` output
  instead of with the captured-fault status)
- **Total: 410/410 PASS**

Build outputs that changed:
- `third_party/busybox/busybox` — 2,292,464 → 1,224,312 bytes
  (now properly musl-linked).
- `test/kernel/initramfs.cpio` — proportionally smaller.

## Slice 7.6d.2 — morally closed

The original 7.6d.2 goal (per IMPLEMENTATION-GUIDE) was:
> Try to boot busybox `--help` from a libnxlibc parent.

That's now done end-to-end.  Sub-slices that took us there:
- 7.6d.1 — vendor + cross-compile busybox
- 7.6d.2a — re-link at user-window VA
- 7.6d.2b — grow user window to 8 MiB
- 7.6d.2c — first attempt + capture failure (TLS gap)
- 7.6d.3a — signal-on-fault (so the kernel survives EL0
  faults instead of halt_forever)
- 7.6d.3b + 7.6d.3c (this session) — push AT_PHDR/PHENT/PHNUM,
  pre-init TPIDR_EL0, force musl-linking via gcc wrapper

The combined arc closes "make busybox `--help` actually run."
Slice 7.6d.N (boot to shell prompt) is the next milestone but
a separate goal.

## Next Steps

- **7.6d.N** — `/init = busybox sh` boots to a shell prompt
  over serial.  Needs:
  - busybox applet-symlink layout in initramfs (cpio doesn't
    natively model symlinks across our flat ramfs; either
    duplicate file entries per applet name OR teach ramfs
    symlink semantics)
  - whichever new musl/busybox features `sh` requires
    (probably `mmap` for shell job control + signal handlers
    for SIGINT)
- **Slice 7.7** — interactive smoke tests once shell is up.

---

**Last Updated:** 2026-04-27
