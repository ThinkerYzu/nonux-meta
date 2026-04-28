# Session 49: slice 7.6d.1 — cross-compile busybox

**Date:** 2026-04-25
**Phase:** 7 slice 7.6d.1 (busybox integration, sub-slice 1 of N — opens 7.6d)
**Branch:** master

---

## Goals

Slice 7.6d (busybox cross-compile + boot to shell) is large enough to
warrant the same vendor-then-discover decomposition that 7.6c.3 used.
This sub-slice covers the build infrastructure only:

- Vendor busybox 1.36.1 under `third_party/busybox/` (mirrors how
  musl was vendored in slice 7.6c.3a — full source tree checked in,
  patches live in tree).
- Commit a minimal `nonux_defconfig` derived from upstream defconfig
  with networking, modutils, login utilities, mail/print/runit, and
  syslog stripped out.
- Write `tools/build-busybox.sh` that drives the cross-compile against
  our patched musl via a private sysroot.
- Wire top-level Makefile `busybox` / `busybox-clean` targets matching
  the existing `musl-libc` / `musl-clean` block.
- Verify the output is a static aarch64 ELF.

Explicitly out of scope: actually running busybox against the kernel.
That's slice 7.6d.2's job — and where the discovery-driven follow-ups
(mmap, the four svc-direct asm files, process-table caps) get
triaged.  No new EL0 demos and no new ktests in this slice; sanity
check is `file` + `readelf -h` on the resulting binary.

## Scope choices

- **busybox 1.36.1, not 1.37.0.**  1.37 (July 2024) is a noisy series
  — it rewrote substantial parts of the shell (the ash variant we
  care about) and reshuffled several Config.in trees.  1.36.1 (May
  2023) is the most-deployed version across embedded distros and is
  by far the most-exercised musl combination upstream, which is what
  matters when our musl patches collide with libc-internals
  assumptions.  Bumping later is cheap (drop a new tarball into
  `third_party/busybox/` + re-run oldconfig); going earlier would
  buy nothing.
- **Vendor the full source tree under `third_party/busybox/`.**
  Mirrors the musl pattern from slice 7.6c.3a.  20 MB on disk,
  trivially worth it for the property that anyone cloning the repo
  can `make busybox` without a network round-trip.  Future patches
  to busybox internals (anticipated in 7.6d.2+) live in tree
  alongside the vendored source.
- **`make defconfig` then strip, not `make allnoconfig` then add.**
  defconfig is busybox's "sensible kitchen-sink" preset and is the
  config most upstream patches are written against.  Starting from
  it and disabling problem subsystems gives us a config where every
  on-by-default applet stays on (so our smoke-test set — `sh`, `ls`,
  `echo`, `cat`, `ps`, `mkdir`, `rm`, `mv`, `cp`, `pwd`, `true`,
  `false` — falls out of the box without our having to remember to
  enable each one).  allnoconfig demands you re-enable each applet
  explicitly + chase down its libbb dependencies; way too much
  manual surface for a slice that's supposed to be infrastructure
  only.
- **Strip via `# CONFIG_X is not set` markers, not `CONFIG_X=n`.**
  busybox's kconfig (like the Linux kernel's) ignores `CONFIG_X=n`
  and re-defaults the symbol on the next `oldconfig`.  Force-off
  requires the comment marker.  This is also what `make
  menuconfig` writes when you toggle a symbol off interactively.
- **Disable problem subsystems by directory, not by hand-picked
  applet list.**  Walked `find networking modutils mailutils
  printutils runit sysklogd loginutils selinux -name 'Config.*'`
  and forced off every `config FOO` symbol declared in those
  trees (~278 keys).  Plus a handful of cross-cutting symbols
  (`PAM`, `FEATURE_USE_INITTAB`, the four `*RPC*` keys, the
  `MOUNT_NFS` family).  Result: 875 → 659 enabled symbols.  None
  of the disabled subsystems would compile against musl on a
  freestanding kernel anyway (the first failure I observed was
  `networking/tc.c` referencing `TCF_CBQ_LSS_BOUNDED`, a kernel-
  header constant musl doesn't ship).
- **Private musl sysroot at `third_party/musl/_sysroot/`.**  busybox
  needs musl's headers to compile against, and they don't exist
  until `make install-headers` runs.  Approach:
  - `install-headers prefix=$(MUSL_DIR)/_sysroot` populates `include/`.
  - `mkdir -p $(SYSROOT)/lib && ln -sf ../../lib/{libc,crt1,crti,crtn}.{a,o}
    $(SYSROOT)/lib/` makes the libs visible at the sysroot-standard
    location without copying — symlinks back to `musl/lib/` so the
    sysroot stays coherent with whatever the top-level `make
    musl-libc` last produced.
  - Build with `EXTRA_CFLAGS="--sysroot=$SYSROOT"` and matching
    `EXTRA_LDFLAGS="--sysroot=$SYSROOT -static"`.  `--sysroot` is
    cleaner than `-nostdinc -isystem` because it covers both
    include search and library search in one switch and matches
    the standard cross-toolchain idiom.
- **busybox is NOT (yet) a dep of `make test`.**  The build takes
  ~30 s and the artefact isn't consumed by any ktest in 7.6d.1; no
  reason to penalize the test loop.  It becomes a test dep when
  7.6d.2 first execs busybox from a libnxlibc parent.

## What Was Done

### `third_party/busybox/` — vendored source

- Full extraction of `busybox-1.36.1.tar.bz2` (sha256
  `b8cc24c9574d809e7279c3be349795c5d5ceb6fdf19ca709f80cde50e47de314`,
  official 2023-05-19 release from busybox.net).  ~20 MB on disk;
  no in-tree `.gitignore` (busybox 1.36.1 doesn't ship one).

### `third_party/busybox/configs/nonux_defconfig`

- Generated by:
  1. `make defconfig` in the busybox tree → 875 enabled symbols.
  2. Force off every `config FOO` declared in the
     `networking / modutils / mailutils / printutils / runit /
      sysklogd / loginutils / selinux` directory trees (~278
     symbols total) by rewriting each `CONFIG_FOO=y` line to
     `# CONFIG_FOO is not set`.
  3. Force off the cross-cutting handful: `CONFIG_PAM`,
     `CONFIG_FEATURE_USE_INITTAB`, `CONFIG_FEATURE_HAVE_RPC`,
     `CONFIG_FEATURE_INETD_RPC`, `CONFIG_FEATURE_MOUNT_NFS`,
     `CONFIG_NFSMOUNT`.
  4. `yes "" | make oldconfig` to fill in any new-default symbols
     and resolve `select`-clause dependencies.
- 1223-line snapshot with 659 enabled symbols.  Committed as the
  authoritative config; the build script copies it into `.config`
  + re-runs `oldconfig` each invocation so symbol-tree changes in
  future busybox bumps surface as nonux_defconfig diffs.

### `tools/build-busybox.sh`

- Idempotent build driver.  Five steps:
  1. Verify `third_party/musl/lib/libc.a` exists (top-level `make
     musl-libc` must have run).  Fail fast with a clear error
     otherwise.
  2. Populate the private sysroot lazily — `install-headers` only
     runs if `$SYSROOT/include/stdio.h` is missing; lib symlinks
     are recreated unconditionally (cheap, and it covers the case
     where someone manually nuked `_sysroot/lib/`).
  3. Stage the config: `cp configs/nonux_defconfig .config` then
     `yes "" | make oldconfig`.  pipefail is briefly disabled
     around the pipe because `yes` exits with SIGPIPE (141) once
     `oldconfig` closes its stdin — that's expected, not an
     error.
  4. Real build: `make -j$(nproc) ARCH=arm64
     CROSS_COMPILE=aarch64-linux-gnu- EXTRA_CFLAGS=--sysroot=...
     EXTRA_LDFLAGS="--sysroot=... -static" SKIP_STRIP=y`.  The
     SKIP_STRIP=y leaves debug info in the binary so 7.6d.2 can
     `readelf` segments and follow musl's call graph when things
     trap.
  5. `file busybox` + size print as a self-test; `set -e` catches
     a corrupt link upstream of the file check.

### Top-level `Makefile`

- New block right after `musl-clean`:
  - `BUSYBOX_DIR := third_party/busybox`
  - `BUSYBOX_BIN := $(BUSYBOX_DIR)/busybox`
  - `BUSYBOX_CONFIG := $(BUSYBOX_DIR)/configs/nonux_defconfig`
  - `BUSYBOX_BUILD := tools/build-busybox.sh`
  - `$(BUSYBOX_BIN)` rule depends on the script + config + the
    musl libc archive (so a musl rebuild correctly triggers a
    busybox rebuild against the new libs).
  - `busybox` (.PHONY) and `busybox-clean` (.PHONY).  busybox-clean
    runs busybox's own `make clean` + drops the staged `.config`.
- `musl-clean` extended to `rm -rf $(MUSL_DIR)/_sysroot` so the
  private sysroot doesn't outlive the libc it was carved from.

### `.gitignore`

- Added a new section "Slice 7.6d.1" with:
  - `third_party/musl/_sysroot/` (build output, regenerated on
    demand).
  - `third_party/busybox/busybox` + `busybox_unstripped` +
    `.config*` (busybox's top-level outputs that don't carry
    extensions covered by the existing `*.o` / `*.a` rules).
  - The handful of generated headers under
    `third_party/busybox/include/` (`autoconf.h`, `applets.h`,
    `applet_tables.h`, `bbconfigopts.h`, `bbconfigopts_bz2.h`,
    `common_bufsiz.h`, `embedded_scripts.h`, `NUM_APPLETS.h`,
    `usage_compressed.h`).
  - The kconfig parser-generated bits
    (`scripts/basic/fixdep`, `scripts/kconfig/conf`,
    `scripts/kconfig/zconf.tab.c`, `scripts/kconfig/lex.zconf.c`,
    `scripts/kconfig/zconf.hash.c`).
  - `applets_sh/applet_tables`.

## Drive-by gotchas

- **`yes "" | make oldconfig` exits 141 under pipefail.**  `yes`
  receives SIGPIPE the moment `oldconfig` closes stdin; with
  `set -o pipefail` (which the script uses for the rest of its
  body) that propagates as failure.  Bracketing the one pipe
  with `set +o pipefail` / `set -o pipefail` is the cleanest fix
  — the alternatives (`(yes "" || true) | ...`, counting the
  oldconfig question count and using `printf '\n%.0s' {1..N}`
  instead) are all worse.
- **busybox 1.36.1 has no `olddefconfig`.**  That target was added
  to the kernel-derived kconfig in a later release.  `oldconfig`
  with empty stdin is the equivalent.  Documented as a comment
  next to the `set +o pipefail` to head off the next person who
  tries to "fix" the script by switching to olddefconfig.
- **musl's `install-headers` lays out `prefix/include/`, not
  `prefix/usr/include/`.**  Standard sysroot convention is the
  latter, so when wiring `--sysroot` you have to also pass
  `-isystem $SYSROOT/include` explicitly … except gcc accepts
  `--sysroot` pointing at any prefix where `include/` and `lib/`
  sit at the top, which is what we have.  No explicit
  `-isystem` needed.
- **No qemu-user-static available locally.**  Couldn't run
  busybox under user-mode qemu for an extra-belt-and-braces
  sanity check.  Live ktest run is what 7.6d.2 will provide;
  for now `file` + `readelf -h` are sufficient.

## Verification

- `make busybox` (clean → built):
  ```
  /home/thinker/.../third_party/busybox/busybox: ELF 64-bit LSB
  executable, ARM aarch64, version 1 (GNU/Linux), statically linked,
  BuildID[sha1]=991ca039e6590e9a586e9babaca2610e2931b433, for
  GNU/Linux 3.7.0, not stripped
  build-busybox.sh: ok (2292464 bytes)
  ```
  - 2.29 MB static aarch64 ELF.
  - `readelf -h` shows entry point `0x4005c0`, two LOAD segments
    at `0x400000` (text/rodata) and `0x5d9770` (data/bss), no
    dynamic section (`There is no dynamic section in this file.`).
  - "GNU/Linux 3.7.0" in `file` is musl's identification of the
    `.note.ABI-tag` section musl emits — not a kernel-version
    requirement.  The binary makes only NX_SYS_* syscalls (musl
    is patched to translate Linux numbers into ours before `svc 0`),
    so it'll go wherever the patched libc.a goes.
- `make test` → **407/407 pass** (51 python + 275 host + 81 kernel),
  0 leaks, 0 errors, exit 0.  Identical to slice 7.6c.3c's
  baseline — no regression from the new build infrastructure.

## What Was Not Done

- **No EL0 demo runs busybox.**  By design — that's slice 7.6d.2's
  one job, and it's where the kernel-side discoveries happen.
- **No ktests added.**  Same reason.
- **No initramfs integration.**  busybox doesn't appear in
  `tools/pack-initramfs.py`'s list yet.  7.6d.2 will add it (or
  not — depending on whether we can drive busybox via drop_to_el0
  during the discovery sub-slice without going through the
  initramfs path; TBD).
- **No clone.s / vfork.s / __unmapself.s / restore.s patches** to
  musl.  Predicted in slice 7.6c.3a + flagged in the HANDOFF
  crystal-ball, but the demo set in 7.6d.1 didn't pull them in.
  busybox links against musl but doesn't *run* in this slice, so
  the asm files aren't on the call graph yet.  They land in
  whichever 7.6d.3.x sub-slice 7.6d.2 surfaces them.

## Test Counts

- Python: 51 (unchanged)
- Host: 275 (unchanged)
- Kernel: 81 (unchanged)
- **Total: 407/407 PASS**

Build outputs new to this slice:
- `third_party/busybox/busybox` — 2.29 MB static aarch64 ELF
  (gitignored).
- `third_party/musl/_sysroot/` — musl headers + lib symlinks
  (gitignored).

## Next Steps

- **7.6d.2** — Try to boot busybox `--help` from a libnxlibc
  parent.  Discovery-driven sub-slice.  Small libnxlibc parent
  forks + execs `/bin/busybox` with argv `{ "/bin/busybox",
  "--help", NULL }` against a ramfs-seeded busybox.  Expected:
  some combination of (a) musl tries `mmap` for mallocng's first
  slab → ENOSYS, (b) musl's `__clone` reachable through some
  pthread-init code path → SVC trap with un-translated x8, (c)
  process-table or address-space caps exhaust after a few
  iterations.  Deliverable: kernel test capturing the failure
  mode + written list of follow-up sub-slices.
- **7.6d.3+** — Close whichever gaps 7.6d.2 surfaces.  Most-likely
  candidates: `NX_SYS_MMAP`, the four svc-direct musl asm files,
  process-table cap bump or proper reap-on-wait.
- **7.6d.N** — `/init = busybox sh` boots to a shell prompt.
  `tools/pack-initramfs.py` learns to emit the `/bin/busybox` +
  applet-symlink layout; production `make run` lands on a shell
  prompt over serial.

---

**Last Updated:** 2026-04-25
