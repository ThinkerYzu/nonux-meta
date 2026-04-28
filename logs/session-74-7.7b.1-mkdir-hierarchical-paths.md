# Session 74: Slice 7.7b.1 — `mkdir` + hierarchical paths in ramfs

**Date:** 2026-04-28
**Phase:** 7 (process model + POSIX shim)
**Branch:** master

---

## Goals

- Land the `mkdir` half of slice 7.7b: real kernel surface for
  `NX_SYS_MKDIRAT`, hierarchical paths in ramfs, and the syscall-layer
  rewiring that lets ash's `mkdir /m && > /m/x && ls /m` round-trip
  end-to-end.
- Keep the `ps` work for slice 7.7b.2 (next session) — splitting per
  IMPLEMENTATION-GUIDE's "1–2 sessions depending on whether path-rework
  lands in 7.7b or splits into 7.7b.1 / 7.7b.2" note.
- Drop the `sys_getdents64` leading-`/` strip hack from slice 7.6d.N.5
  (called out as cleanup debt in the slice 7.7b plan).

## What Was Done

### Interface additions (`interfaces/fs.h`, `interfaces/vfs.h`)

- `nx_fs_ops` / `nx_vfs_ops` gain three changes:
  - `readdir` signature changes from `(self, *cookie, *out)` to
    `(self, dir_path, *cookie, *out)` — dir_path is the absolute path
    of the directory to enumerate, basename projection happens
    driver-side.
  - New op `mkdir(self, path)` — creates a directory entry; returns
    `NX_EEXIST` against existing entries, `NX_ENOMEM` when storage
    exhausted.
  - New op `stat(self, path, *out)` — returns `{kind, size}` via
    new `struct nx_fs_stat` with `NX_FS_KIND_FILE` / `NX_FS_KIND_DIR`.
- The contract for `mkdir` in v1 is permissive: it does **not**
  enforce parent-directory existence (consistent with how the cpio
  loader stores `/bin/busybox` without an explicit `/bin` dir).

### ramfs (`components/ramfs/ramfs.c`)

- `struct ramfs_file` gains a `uint32_t kind` field
  (`RAMFS_KIND_FILE` / `RAMFS_KIND_DIR`).
- New `ramfs_classify(s, path)` helper recognises a path as DIR if:
  - Root (`"/"`),
  - Some entry stores it verbatim with kind=DIR,
  - **OR** some entry has it as a parent prefix (`"/bin"` is a
    synthesised dir if any `/bin/sh`-style entry exists).  This
    synthesis lets the existing cpio loader keep storing flat full
    paths without a per-prefix dir-entry expansion.
- `ramfs_op_open` rejects opening a DIR entry with `NX_EPERM` —
  the syscall layer routes DIR opens through `HANDLE_DIR` instead.
- New `ramfs_op_mkdir` allocates a DIR entry; rejects pre-existing
  entries (explicit or synthesised) with `NX_EEXIST`.
- New `ramfs_op_stat` returns kind+size; synthesises DIR kind for
  parent prefixes that are not stored explicitly.
- `ramfs_op_readdir` rewritten for hierarchical projection:
  - Walks the file table from `*cookie`.
  - For each entry, computes the suffix relative to `dir_path`,
    takes the first segment (up to next `/` or end-of-string).
  - Deduplicates against earlier in-use entries (O(cookie²) per
    call — fine for `RAMFS_MAX_FILES = 24`).
  - Yields the basename, never the full path; the leading-`/` strip
    in `sys_getdents64` becomes unnecessary.
- The cpio loader's flat-namespace comment is updated to point at
  the new synthesise-via-stat behaviour.

### vfs_simple (`components/vfs_simple/vfs_simple.c`)

- Adds pass-through `vfs_simple_mkdir`, `vfs_simple_stat`.
- Updates `vfs_simple_readdir` to take `dir_path`.
- All three new ops NULL-guard the underlying driver op so a fixture
  driver that doesn't implement them returns `NX_EINVAL` rather than
  segfaulting.

### syscall layer (`framework/syscall.c` + `.h`)

- `struct nx_dir_cursor` grows a `path[NX_PATH_MAX]` field — getdents64
  now passes this to `vops->readdir` so iteration is per-directory.
- `sys_open` swaps the slice-7.6d.N.5 `"/"`-only HANDLE_DIR shortcut
  for a generic `vops->stat`-driven probe: if stat reports DIR,
  allocate HANDLE_DIR with the path; otherwise fall through to
  `vops->open`.  Host-build branch stays gated under `!__STDC_HOSTED__`
  (no kheap on host, fake_fs returns ENOENT from stat anyway).
- `sys_fstatat` replaces its open-and-seek dance with a single
  `vops->stat` call that returns kind+size — collapses the `"/"`
  special case and the file-size SEEK_END trick into one shape.
- `sys_getdents64` passes `cur->path` to `vops->readdir`; drops the
  leading-`/` strip hack at the old line range.
- `sys_readdir` (legacy `NX_SYS_READDIR = 10`) hard-codes
  `dir_path = "/"` — the only consumers (`ktest_vfs` and the
  user_prog_readdir.S EL0 demo) only ever iterated the root anyway.
- New `NX_SYS_MKDIRAT = 40` + `sys_mkdirat` (~25 lines): copies path,
  resolves vfs, calls `vops->mkdir`, translates `NX_E*` to Linux
  `-errno` (`-17` EEXIST, `-12` ENOMEM, `-2` ENOENT) so musl's
  `mkdir(2)` reports the right errno.
- Dispatch table picks up `[NX_SYS_MKDIRAT] = sys_mkdirat`.
- musl translation: `__NR_mkdirat = 34 → NX_SYS_MKDIRAT = 40` in
  `arch/aarch64/syscall_arch.h` and the `syscall_cp.s` cancellation
  path.

### Tests

- Conformance suite (`test/host/conformance/conformance_fs.{h,c}`):
  - Updates the two readdir cases for the new `(dir_path, cookie, out)`
    signature; expects basenames (`"a"`, not `"/a"`).
  - Adds two new universal cases: `mkdir_creates_dir_visible_in_readdir`
    and `stat_reports_kind_for_files_and_dirs`.  Driven by ramfs's
    fixture; fs_stub fixture also wires through.
- `test/host/conformance_fs_test.c` adds stub `mkdir_op` + `stat_op`
  + path-aware `readdir_op` + the two new TEST() wrappers.
- `test/host/component_ramfs_test.c` gains:
  - Two new conformance TEST() wrappers (mkdir + stat).
  - `ramfs_hierarchical_readdir_dedups_synthesised_dirs` —
    creates `/bin/sh` + `/bin/cat`, asserts `readdir("/")` yields
    `bin` exactly once and `readdir("/bin")` yields both basenames.
  - `ramfs_open_on_directory_returns_eperm` — guard against the
    syscall layer routing a DIR entry through the file-open path.
- `test/host/file_syscall_test.c` updates `fake_fs_ops`:
  - `fake_readdir` takes `dir_path` and projects basenames.
  - New `fake_stat` reports FILE for stored entries, DIR for `"/"`.
  - The existing `sys_readdir_yields_created_files_then_enoent` test
    flips its name-match expectations from `/alpha` / `/beta` to
    `alpha` / `beta`.
- New kernel test:
  - `test/kernel/posix_busybox_sh_mkdir_prog.c` — libnxlibc parent
    that `execve`s `/bin/busybox sh -c "mkdir /m && > /m/x && ls /m"`.
  - `test/kernel/posix_busybox_sh_mkdir_prog_blob.S` — `.incbin` blob
    wrapper.
  - `test/kernel/ktest_posix_busybox_sh_mkdir.c` — drops to EL0,
    waits for parent EXIT(0), then re-opens via vfs_simple to assert
    `stat("/m") == DIR` and `stat("/m/x") == FILE`.
  - Makefile wired into `KTEST_C`, `KTEST_S`, the build recipe block,
    and the `clean` target.
- `INITRAMFS_ENTRIES` in the Makefile gains
  `$(BUSYBOX_BIN):/bin/mkdir` so busybox's `mkdir` applet dispatches
  via argv[0] basename matching (busybox's standard
  `mkdir`-as-cpio-duplicate pattern, same as `/bin/cat`, `/bin/ls`,
  ...).
- New interactive script:
  - `test/interactive/mkdir_tmp.script` — `mkdir /tmp; > /tmp/MKDIR_TMP_MARKER; ls /tmp`.
  - `test/interactive/mkdir_tmp.expected` — `MKDIR_TMP_MARKER`.

## Key Findings

- The cpio loader's flat-namespace comment from slice 6.2 was the
  load-bearing stop-gap that `ramfs_op_stat`'s synthesise-on-prefix
  path implicitly retires: today's initramfs entries (`/bin/busybox`,
  `/banner`, ...) stay stored verbatim, and `stat("/bin")` /
  `readdir("/bin")` work correctly without needing the cpio loader
  to also emit explicit `/bin` directory entries.  Net effect: zero
  ramfs-storage overhead for the hierarchical-paths upgrade beyond
  the new `kind` field.
- `make test-interactive` got 1 new script (`mkdir_tmp`) for a 5/5 →
  6/6 jump, NOT the 5/5 → 7/7 originally projected for the full
  slice 7.7b.  The remaining script (`ps_smoke`) lands with slice
  7.7b.2's procfs / `getprocs` work next session.
- The kernel rebuild dependency chain has a soft spot:
  `INITRAMFS_ENTRIES` is a Makefile-internal variable, so adding
  `/bin/mkdir` to it does NOT trigger a re-pack of `initramfs.cpio`
  on its own — touching the cpio file or running `rm initramfs*.cpio`
  before `make` was the manual nudge needed during this session.  Not
  worth fixing immediately (the symptom is a clear "mkdir: not found"
  rather than a silent wrong result), but a `.PHONY: initramfs` or
  a `Makefile` order-only dep could prevent the foot-gun later.
- musl's libc.a rebuild rule does pick up syscall-table changes via
  the explicit dep on `syscall_arch.h` + `syscall_cp.s`, but busybox's
  rebuild against the new musl needs a fresh stamp on `MUSL_LIBC` —
  in practice this means after editing the translation table you
  also need either `touch third_party/musl/lib/libc.a` or `make
  busybox` explicitly before `make test`.  Same pattern as slice
  7.6d.N.13's musl-translation-sweep.

## Decisions Made

- **Split slice 7.7b into 7.7b.1 (mkdir + paths) + 7.7b.2 (ps).**
  Each half is a real kernel-surface change with its own test
  surface; bundling them into one session would have made the diff
  unreadable and the kernel-test rebuild loop slow.  Per the
  IMPLEMENTATION-GUIDE note this is an explicitly-supported split.
- **Synthesise intermediate dirs in `ramfs_op_stat` rather than
  emit explicit DIR entries from the cpio loader.**  Saves
  `RAMFS_MAX_FILES` budget (already at 24, headroom is tight for
  the v1 file table); keeps the cpio loader's logic local; any
  future tree-fs replacement can re-use the same stat synthesis
  pattern when migrating between flat and hierarchical layouts.
- **Refuse `open` on a DIR entry with `NX_EPERM`** rather than
  returning a HANDLE_FILE pointing at zero-byte data.  The syscall
  layer's stat probe is the routing point; ramfs's open guard is
  defence-in-depth so a buggy caller (or a future syscall variant
  that bypasses the probe) doesn't get a meaningless file handle.
- **Hard-code `dir_path = "/"` in `sys_readdir` (the legacy syscall)**
  rather than wiring a path argument through.  Only `ktest_vfs` and
  the user_prog_readdir.S EL0 demo consume it, both iterate root,
  and slice 7.6d.N.5 already declared `sys_getdents64` as the
  forward-going API for arbitrary directories.

## Status at End of Session

- `make test` → **439/439 passing** (51 python + 283 host + 105 kernel),
  0 leaks, 0 errors, exit 0.  Was 432.
- `make test-interactive` → **6/6 passing** (echo_cat, echo_hello,
  echo_pipe, ls_root, mkdir_tmp, visible_prompt).  Was 5/5.
- Slice 7.7b.1 is closed at v1 quality.  Slice 7.7b.2 (ps via procfs
  OR getprocs syscall) is the next forward step; the
  `test/interactive/ps_smoke.{script,expected}` deliverable lives
  there.

## Next Steps

- **Slice 7.7b.2 — `ps` via procfs (preferred) OR `NX_SYS_GETPROCS`**.
  Lifts the IMPLEMENTATION-GUIDE 7.7b plan's procfs route now that
  hierarchical paths are real: a new `components/procfs/` exposes
  `/proc/<pid>/stat` files synthesised on demand from the process
  table; `vfs_simple` grows a tiny mount table to route `/proc/...`
  paths to the new slot while everything else stays on
  `filesystem.root`.  busybox's stock ps applet just works against
  the synthesised files; one new interactive script
  (`ps_smoke.{script,expected}`) takes `make test-interactive`
  6/6 → 7/7.

---

**Files Changed:**

- `interfaces/fs.h`, `interfaces/vfs.h` — `mkdir` + `stat` ops; new
  `struct nx_fs_stat`, `NX_FS_KIND_*`; `readdir` signature gains
  `dir_path`.
- `components/ramfs/ramfs.c` — `kind` field on `ramfs_file`; new
  `ramfs_classify` / `ramfs_op_mkdir` / `ramfs_op_stat`; rewritten
  `ramfs_op_readdir` for hierarchical projection + dedup.
- `components/vfs_simple/vfs_simple.c` — pass-through `mkdir` / `stat`;
  `readdir` takes `dir_path`.
- `framework/syscall.c` — `nx_dir_cursor.path`; `sys_open` /
  `sys_fstatat` use `vops->stat`; `sys_getdents64` passes path
  + drops leading-`/` strip; `sys_readdir` hard-codes `/`; new
  `sys_mkdirat`; dispatch table entry.
- `framework/syscall.h` — `NX_SYS_MKDIRAT = 40`.
- `third_party/musl/arch/aarch64/syscall_arch.h`,
  `third_party/musl/src/thread/aarch64/syscall_cp.s` —
  `__NR_mkdirat = 34 → 40`.
- `test/host/conformance/conformance_fs.{h,c}` — readdir signature
  + 2 new universal cases (mkdir, stat).
- `test/host/conformance_fs_test.c` — fs_stub gains `mkdir_op` /
  `stat_op` + path-aware readdir + 2 new TEST() wrappers.
- `test/host/component_ramfs_test.c` — 2 new conformance wrappers +
  hierarchical-readdir + open-on-dir tests.
- `test/host/file_syscall_test.c` — fake_fs gets `stat` + path-aware
  readdir; sys_readdir test name-match flipped to basenames.
- `test/kernel/posix_busybox_sh_mkdir_prog.c`,
  `test/kernel/posix_busybox_sh_mkdir_prog_blob.S`,
  `test/kernel/ktest_posix_busybox_sh_mkdir.c` (new) — kernel
  end-to-end coverage.
- `Makefile` — `KTEST_C` + `KTEST_S` + recipe block + `clean` target
  for the new mkdir kernel test; `INITRAMFS_ENTRIES` adds
  `/bin/mkdir` cpio duplicate.
- `test/interactive/mkdir_tmp.script`,
  `test/interactive/mkdir_tmp.expected` (new) — sixth interactive
  smoke script.
