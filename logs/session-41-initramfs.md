# Session 41: slice 7.6b — initramfs format (cpio-newc) + ramfs slurp

**Date:** 2026-04-25
**Phase:** 7 slice 7.6b
**Branch:** master

---

## Goals

Land an initramfs format and a ramfs that slurps it at boot.  Slice
7.6c (musl ↔ NX_SYS_*) and 7.6d (busybox cross-compile) both need
to ship a binary set inside `kernel-test.bin` — busybox itself, plus
applet symlinks, plus a few rootfs files.  Hand-seeding via
vfs_simple syscalls (the slice-7.4c pattern) doesn't scale past a
handful of files; we want a one-time blob walk at component init.

## Scope choices

- **cpio-newc, not a custom format.**  Three reasons:
  1. busybox already speaks it (its `cpio` applet emits + parses
     newc natively).  Slice 7.6d can hand the initramfs over to
     busybox without any format gymnastics.
  2. Linux's initramfs is also cpio-newc, so any tooling we
     build now (the packer + the parser) is reusable if we ever
     boot nonux as a kernel underneath a Linux-style userspace.
  3. The parser is tiny — 110-byte ASCII-hex header, no
     compression, no checksums beyond a fixed-zero `c_check`
     field.  Fits in ~50 lines of C without any
     freestanding-unsafe library calls.
- **Generate the cpio at build time, embed via `.incbin`.**  Same
  pattern slice 7.3 used for `init_prog.elf`: a Python tool emits
  the binary blob; an asm file pulls it into `.rodata` between
  named labels; the C side declares the labels as externs and
  walks the bytes.  Means no runtime cpio writer to worry about.
- **Weak fallbacks for production safety.**  ramfs is built
  into both `kernel.bin` and `kernel-test.bin`.  Production
  doesn't have an initramfs and shouldn't carry one.  Marking
  the blob symbols `__attribute__((weak))` lets `kernel.bin`
  link cleanly without `initramfs_blob.S`; the weak fallbacks
  resolve to NULL, ramfs_init's `blob_end > blob_start` guard
  short-circuits, and the slurp does nothing.  Footprint on
  shipping kernels: zero (a few hundred bytes of code that
  never runs without the blob).
- **Regular files only in v1.**  cpio entries can be dirs,
  symlinks, special files, and FIFOs.  ramfs is flat-namespace
  + no symlink type, so anything except `S_IFREG` is silently
  skipped.  busybox doesn't actually need directory entries —
  its applets resolve absolute paths against the live FS, and
  the live FS is whatever ramfs has — so this is enough for
  slice 7.6d's needs.  When hierarchical paths land later, the
  slurp learns to mkdir on its own; the on-disk format is
  unchanged.
- **Test files: `/init` + `/banner`.**  `/init` is the existing
  `init_prog.elf` blob (lets `ktest_exec` drop its bespoke
  seeding code).  `/banner` is a 21-byte ASCII payload
  (`hello from initramfs\n`) — a positive-control file that
  proves the slurp creates files via the public ramfs API and
  not through some side path.  Originally named `/hello` but
  renamed after a registration-order race with `ktest_vfs`
  (which independently creates `/hello` via vfs_simple);
  disjoint namespaces are safer.
- **Drop `seed_init_file` from `ktest_exec`, but keep
  `init_prog_blob.S`.**  `ktest_elf` still uses the embedded
  ELF blob directly (it tests the parser + loader against an
  in-memory blob, not via vfs).  Two distinct test scenarios,
  one shared source — `init_prog.elf` lives on disk twice
  (once as a `.incbin`, once inside the cpio) but the build
  rule generates both from the same canonical `.elf` so
  there's no drift.

## What Was Done

### `tools/pack-initramfs.py` — new

~120 lines of stdlib-only Python.  Each invocation:

1. Iterates over `path-on-disk:archive-name` entries on argv.
2. For each, opens the file, builds an entry record:
   header (110 bytes ASCII-hex), name + NUL, padding to 4-byte
   boundary, data, padding to 4-byte boundary.
3. Appends a `TRAILER!!!` sentinel record at end.
4. Writes the blob to the output path.

c_mtime is fixed to 0 for reproducible builds.  Verified with
`cpio -tv -F` on the system cpio — the system tool sees the
same entries we encoded.

### `components/ramfs/ramfs.c` — cpio-newc parser

Three new helpers:

- `cpio_hex8(p)` — parse 8 ASCII-hex chars into a uint;
  returns sentinel `0xFFFFFFFFu` on a non-hex char.
- `cpio_align4(v)` — round up to the next 4-byte boundary.
- `ramfs_slurp_initramfs(s, blob, total)` — the walk.  For each
  record: validate magic, parse mode/filesize/namesize, find
  the name, stop at `TRAILER!!!`, optionally seed the file (only
  for `S_IFREG` mode bits and `filesize <= RAMFS_FILE_CAP`),
  advance past data + padding, repeat.

`ramfs_init` calls the slurp if `__ramfs_initramfs_blob_end >
__ramfs_initramfs_blob_start` (with the `&__sym[0]` form to
evade `-Werror=array-compare`).  The weak `extern char
__ramfs_initramfs_blob_*[]` declarations are local to ramfs.c.

### `test/kernel/initramfs_blob.S` — new

`.incbin` of `test/kernel/initramfs.cpio` between
`__ramfs_initramfs_blob_start` / `_end` labels.  Mirrors
`init_prog_blob.S`'s shape exactly.

### Makefile

- New `test/kernel/banner.txt` rule emits the ASCII payload at
  build time (so the file isn't checked into git).
- `test/kernel/initramfs.cpio` depends on the packer +
  `init_prog.elf` + `banner.txt`; runs the packer with the
  manifest entries.
- `test/kernel/initramfs_blob.o` depends on the cpio so an
  edit triggers a rebuild.
- `KTEST_S += test/kernel/initramfs_blob.S`.
- `KTEST_C += test/kernel/ktest_initramfs.c`.
- `clean` drops the cpio + banner.txt.

### `test/kernel/ktest_initramfs.c` — new

Two tests:

- `initramfs_seed_makes_banner_readable_through_vfs` — opens
  `/banner`, reads 21 bytes, asserts `n == 21` and the bytes
  match `"hello from initramfs\n"`.
- `initramfs_seed_includes_init_with_elf_magic` — opens
  `/init`, reads 4 bytes, asserts the ELF magic.

Both use `nx_slot_lookup("vfs")` + the descriptor's
`iface_ops` — same path real syscalls take.

### `test/kernel/ktest_exec.c` — refactor

Dropped `seed_init_file` (28 lines).  Updated the comment block
to note that `/init` is now seeded by the slurp at boot.
Removed includes of `framework/component.h`,
`framework/registry.h`, `interfaces/vfs.h` that were only
needed for the deleted seeding path.

## Key Findings

- **`-Werror=array-compare` warns on `weak_array_end >
  weak_array_start`.**  When both operands are declared as
  array-typed externs, gcc 12+ assumes you meant to compare
  array contents (which is meaningless) and warns.  The fix is
  to write `&__sym[0]` on each side — the compiler now sees
  explicit pointer operands and accepts the comparison.  The
  linker-resolved address comparison is identical at runtime;
  this is purely a frontend gripe.
- **KTEST registration order is link-order-dependent and
  matters when tests share filesystem state.**  Original cpio
  shipped `/hello`; `ktest_vfs` independently creates a
  `/hello` file via vfs_simple's create flow.  Whichever test
  runs first wins, the other reads the wrong content.  Renaming
  to `/banner` keeps the namespaces disjoint.  Worth a comment
  in the ktest header so a future reader doesn't try to
  "harmonize" the names back together.
- **Build deps for `.incbin` need explicit `make` rules.**  GNU
  as doesn't track `.incbin` dependencies — it just emits the
  bytes at assembly time.  If the embedded file changes, the
  `.S` doesn't, the assembler thinks the `.o` is up to date,
  and the build silently links a stale blob.  Explicit
  prerequisite (`initramfs_blob.o: ... initramfs.cpio`) forces
  the assembler to re-run on every cpio rebuild.

## Decisions Made

- **Packer in Python, not in C.**  Build-time tooling that
  reads file content + writes binary output is much simpler in
  Python; `tools/gen-config.py` and `validate-config.py` are
  already Python, so adding another script keeps the toolbox
  uniform.  The packer is stdlib-only — no jsonschema or
  similar runtime dep.
- **Fixed c_mtime = 0 for reproducibility.**  Real-time
  timestamps would make the cpio non-reproducible (same inputs
  → different outputs across builds), which complicates `make`
  dependency tracking and CI cache invalidation.  busybox cpio
  doesn't care about mtime (it accepts whatever the field
  says); we accept the cosmetic loss for a clean reproducibility
  story.
- **Skip non-regular files silently rather than warn.**  The
  parser-side data structure is regular-files-only.  A noisy
  `kprintf("[ramfs] skipping dir entry...")` would clutter the
  boot log without giving the developer any new information.
  When dirs / symlinks become real, the parser learns to seed
  them and the silent skip becomes a positive action.
- **Don't move `init_prog_blob.S` into the initramfs.**
  `ktest_elf` still embeds `init_prog.elf` directly because
  it tests the parser + loader against an in-memory pointer
  (not via vfs).  Two distinct test scenarios, both using the
  same canonical `init_prog.elf` source.

## Status at End of Session

- `make test` → **399/399 pass (51 python + 274 host + 74
  kernel), 0 leaks, 0 errors, exit 0**.  +2 kernel tests.
- `make run` boots cleanly (production kernel doesn't pull in
  the cpio).
- `make kernel-test.bin` reports
  `pack-initramfs.py: wrote test/kernel/initramfs.cpio (1180
  bytes, 2 entries + trailer)` — `/init` (~800 bytes) +
  `/banner` (21 bytes), header overhead included.
- ktest_exec is the first slice-7.x test relying on
  initramfs-seeded files instead of bespoke vfs writes.

## Next Steps

**Slice 7.6c — musl → NX_SYS_* adapter.**

Pin musl as a git submodule under `third_party/musl`.  Patch
`arch/aarch64/syscall_arch.h` to emit our SVC numbers (musl's
internal numbering matches Linux's; we layer a translation
table or a `__nx_syscall(nr, ...)` shim).  Validate with a
tiny C-against-musl program that calls `write(1, ...)` +
`_exit(...)`; ends up dispatching our `NX_SYS_DEBUG_WRITE` +
`NX_SYS_EXIT`.

The build will need:
- a musl submodule (or vendored snapshot) under
  `third_party/musl/`;
- a separate cross-compile flow (musl produces its own libc.a,
  linkable by user programs);
- a small bootstrap header that maps Linux SYS_* → NX_SYS_*.

No kernel changes — slice 7.6c is purely a build-system +
libc-config slice.

**Slice 7.6d — busybox cross-compile + boot to shell.**

Build busybox against the patched musl, drop the binary into
the initramfs at `/bin/busybox` + applet symlinks, set `/init`
to busybox's `sh`.  `make run` lands on a shell prompt.

**Deferred (unchanged):**

- HANDLE_FILE fork inheritance.
- C-level crt0 for `main(argc, argv)`.
- Proper cross-test task reap.
- Per-test `SCHED_RR_DEFAULT_QUANTUM_TICKS` override.

---

**Files Changed:**
- `sources/nonux/tools/pack-initramfs.py` — new (cpio-newc packer)
- `sources/nonux/components/ramfs/ramfs.c` — cpio-newc parser + slurp at init()
- `sources/nonux/test/kernel/initramfs_blob.S` — new (.incbin wrapper)
- `sources/nonux/test/kernel/ktest_initramfs.c` — new (slurp validation)
- `sources/nonux/test/kernel/ktest_exec.c` — drop seed_init_file (already-seeded /init)
- `sources/nonux/Makefile` — banner.txt + initramfs.cpio + initramfs_blob.o rules; KTEST_C/S updates; clean
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.6b filled in with as-built details
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 36 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
