# Session 28: slice 6.1 — fs-driver interface + conformance harness

**Date:** 2026-04-23
**Phase:** 6 slice 6.1 (opens Phase 6)
**Branch:** master

---

## Goals

Open Phase 6 (VFS + ramfs) the same way slice 4.2 opened Phase 4: ship
the filesystem-driver interface header and a universal conformance suite
against an in-file stub driver, with no new components or bootstrap
changes. Slice 6.2 then plugs ramfs into the same harness unchanged.

## Scope choices

Deliberately narrow:

- **Lower API only.** Only `struct nx_fs_ops` — the API that filesystem
  drivers (ramfs, future tmpfs/ext2) implement. The upper `nx_vfs_ops`
  API that vfs_simple will expose is NOT defined here; it emerges when
  vfs_simple is actually written in 6.2.
- **Four ops.** `open / close / read / write`. `readdir` and `seek`
  land in 6.4 — adding them now without a real driver or consumer
  would just be untested scaffolding.
- **No mount op.** A v1 driver instance has at most one mount; the VFS
  layer in 6.2 owns the single-mount invariant. Multi-mount = Phase 8+.
- **In-file stub, not a new component.** `fs_stub` is a 180-line
  in-memory dictionary of named byte buffers living entirely inside
  `test/host/conformance_fs_test.c`. Follows the slice-4.2 `sched_fifo`
  pattern (harness exerciser, not production code).

## What Was Done

### `interfaces/fs.h` — new

- `struct nx_fs_ops { open, close, read, write }`.
- Open flags: `NX_FS_OPEN_READ / _WRITE / _CREATE`. Absent `CREATE`,
  `open` on a missing path returns `NX_ENOENT` — matches the POSIX
  `O_CREAT` meaning. Truncate semantics deliberately unspecified in v1.
- Per-open state is driver-owned between `open` and `close`. Two
  concurrent opens on the same path produce two distinct file-state
  objects with independent cursors — the interface is explicit about
  this so a driver that accidentally shared cursor state would fail
  conformance case 6.
- Byte-count ops use the channel-send/recv return convention: `>= 0`
  on success, negative `NX_E*` on failure. Status ops use plain `int`.

### `test/host/conformance/conformance_fs.{h,c}` — new

Seven universal cases covering:

1. `open_create_on_fresh_path_succeeds` — CREATE on new path works.
2. `open_without_create_on_missing_path_returns_enoent` — absent
   CREATE, missing file → NX_ENOENT.
3. `fresh_file_reads_zero_bytes` — just-created empty file reads 0.
4. `write_then_read_after_reopen_roundtrips` — write bytes, close,
   reopen READ-only (no CREATE), read back identical bytes. Also
   checks that a trailing read sees EOF (0).
5. `read_past_eof_returns_zero` — repeated reads past EOF stay at 0.
6. `two_opens_have_independent_cursors` — read on open A doesn't
   advance open B's cursor.
7. `write_without_write_right_returns_eperm` — write on a READ-only
   open returns NX_EPERM.

Case 7 is the only one that tests the rights/flags plumbing — the
others test data flow. That keeps the suite small but covers both
axes of the interface.

### `test/host/conformance_fs_test.c` — new

In-file `fs_stub` driver:

- Fixed 8-file table with 256-byte per-file buffers.
- Per-open state (`struct stub_open`) malloc'd in `open`, freed in
  `close`. Carries the `flags` so `read` / `write` can enforce the
  per-right check without re-consulting the path.
- Seven TEST() wrappers around the conformance helpers, plus two
  internal smoke tests (unknown flag bit → NX_EINVAL; write past
  file capacity → NX_ENOMEM).

### `test/host/Makefile` — add new sources

`conformance/conformance_fs.c` and `conformance_fs_test.c` appended to
the SRCS list. No other Makefile edits needed — the conformance vpath
already covers the `conformance/` subdirectory.

### `IMPLEMENTATION-GUIDE.md §Phase 6` — sliced 6.1–6.4

Replaced the five-bullet sketch with a sliced plan mirroring the
Phase 5 shape: 6.1 (this slice), 6.2 (vfs_simple + ramfs components),
6.3 (file syscalls + HANDLE_FILE destructor + EL0 round-trip),
6.4 (readdir + seek + directory iteration). `Last Updated` bumped.

## Key Findings

- **The fs-driver API is fundamentally different from the vfs API.**
  Both will have `open` / `read` / `write`, but the driver's `self` is
  a single mount instance; the VFS layer's `self` is a mount table.
  Defining only the lower API in 6.1 avoids committing to the
  multi-mount question before it has a consumer.
- **Landing the harness before either component lets us validate the
  interface shape in isolation.** The universal cases caught two early
  API drafts (one where `open` wrote the file pointer on failure, one
  where `write` could silently advance the cursor past the file
  capacity) without having to go through the component lifecycle.

## Decisions Made

- **Path handling is driver-local.** The VFS layer in 6.2 will strip
  mount prefixes before calling into the driver. In v1 the only mount
  is `/`, so the distinction is only shape-level — but getting it
  right in the header now means ramfs doesn't need a rewrite when a
  second mount point lands.
- **Truncate-on-open deferred to 6.4.** POSIX's `O_TRUNC` is a common
  request but adding it in 6.1 would force an EPERM interaction check
  (truncate on a READ-only open) that isn't exercised by any consumer
  yet. Slice 6.4 adds an explicit `NX_FS_OPEN_TRUNCATE` bit with the
  first consumer.
- **Write-past-capacity → NX_ENOMEM, not a short write.** The v1 fs
  driver may choose: either (a) short-write to signal "backing store
  filled, retry with less" or (b) NX_ENOMEM if no bytes can be
  written at all. The stub takes (a); ramfs will do the same. The
  contract documents both possibilities.

## Status at End of Session

- `make test` → **315/315 pass (51 python + 217 host + 47 kernel), 0
  leaks, 0 errors, exit 0**. +9 host tests from slice 6.1 (7 universal
  + 2 stub-internal). Kernel side unchanged — 6.1 is pure host-side.
- `verify-registry` unchanged (no manifests in 6.1).
- `interfaces/fs.h`, `test/host/conformance/conformance_fs.{h,c}`, and
  `test/host/conformance_fs_test.c` in place; Makefile wired.
- `IMPLEMENTATION-GUIDE.md` reflects the 6.1–6.4 slicing.

## Next Steps

Slice 6.2: `components/vfs_simple/` + `components/ramfs/`.

- vfs_simple binds to the `vfs` slot, consumes the `nx_fs_ops` of its
  configured root_fs driver.
- ramfs is a filesystem-driver component that exposes itself to
  vfs_simple via a `NX_FS_DRIVER_REGISTER(name, ops)` macro populating
  a linker section (vfs_simple looks up `root_fs` config knob by name
  in that section).
- Conformance suite runs unchanged against ramfs.
- Bootstrap brings both to `NX_LC_ACTIVE`; kernel test proves a
  create/write/read round-trip through the full VFS stack.

---

**Files Changed:**
- `sources/nonux/interfaces/fs.h` — new (fs-driver interface + open flags)
- `sources/nonux/test/host/conformance/conformance_fs.h` — new (7 cases)
- `sources/nonux/test/host/conformance/conformance_fs.c` — new (impl)
- `sources/nonux/test/host/conformance_fs_test.c` — new (in-file `fs_stub` fixture + 7 conformance + 2 internal TEST()s)
- `sources/nonux/test/host/Makefile` — SRCS += new conformance files
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Phase 6 sliced 6.1–6.4
- `proj_docs/nonux/HANDOFF.md` — current status + phase checklist + session-log entry
