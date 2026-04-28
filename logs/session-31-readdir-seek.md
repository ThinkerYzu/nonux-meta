# Session 31: slice 6.4 — readdir + seek + directory iteration (closes Phase 6)

**Date:** 2026-04-24
**Phase:** 6 slice 6.4 (closes Phase 6)
**Branch:** master

---

## Goals

Close Phase 6 by rounding out the file API with the two ops that turn
a bare "read/write bytes at a cursor" into the full basic-file
contract: `seek` for positioning and `readdir` for enumeration.  With
these, userspace can now do all five core file operations —
create/open/read/write/list/seek/close — across the vfs_simple →
ramfs stack.

## Scope choices

- **Readdir is filesystem-level, not dir-handle-based.**  `nx_fs_ops.
  readdir` takes a caller-owned `uint32_t *cookie` + a `struct
  nx_fs_dirent *out`.  No dir-handle concept in v1.  Rationale:
  ramfs has a flat namespace (every file lives in one table with
  no subdirectories), so "iterate this directory" reduces to
  "iterate every file in the filesystem" — a dir-handle type
  would double the syscall surface (HANDLE_DIR) and double the
  close-destructor logic for zero v1 benefit.  When a tree-FS
  driver lands, the interface will grow a second `readdir_at(dir,
  cookie, out)` flavour; this flat form stays for flat drivers.
- **Seek is per-open (handle-based).**  Standard POSIX shape:
  SEEK_SET / SEEK_CUR / SEEK_END whence with a signed offset.
  Position clamped to `[0, size]` — past-EOF seeks return
  NX_EINVAL (no hole-filling in v1; callers write in sequence or
  SEEK_END).
- **SEEK right granted implicitly on any readable or writable
  open.**  Simpler than a separate `NX_FS_OPEN_SEEKABLE` flag — in
  practice every seekable stream supports seek, and a future
  non-seekable stream type can drop the right via
  `nx_handle_duplicate`.
- **Fixed-size dirent.**  `struct nx_fs_dirent` carries
  `{uint32_t name_len; char name[NX_FS_DIRENT_NAME_MAX = 64];}`
  inline — no per-entry malloc.  Always NUL-terminated.  Drivers
  with names > 63 chars truncate and report the truncated length
  (ramfs's NAME_MAX is 32, so truncation doesn't happen in v1).
- **Cookie-based, not fd-based, iteration.**  The caller stashes
  the cookie wherever they like; no kernel-side state per iterator
  beyond the cookie value.  Matches how many embedded readdir APIs
  work; simpler than maintaining a dir-handle + its cursor.

## What Was Done

### Interface additions

- `interfaces/fs.h` grows:
  - `NX_FS_SEEK_SET / _CUR / _END` whence values.
  - `NX_FS_DIRENT_NAME_MAX = 64` + `struct nx_fs_dirent`.
  - `nx_fs_ops.seek(self, file, int64_t offset, int whence) →
    int64_t` — returns new absolute position ≥ 0 or negative
    `NX_E*`.
  - `nx_fs_ops.readdir(self, uint32_t *cookie, struct nx_fs_dirent
    *out)` — returns NX_OK (entry populated, cookie advanced) /
    NX_ENOENT (iteration done) / NX_EINVAL.
- `interfaces/vfs.h` mirrors: `NX_VFS_SEEK_*`, forward-declares
  `struct nx_fs_dirent` (full def in fs.h), adds
  `nx_vfs_ops.seek` and `.readdir` with identical semantics —
  vfs_simple forwards verbatim.

### Conformance suite (+5 universal cases)

`test/host/conformance/conformance_fs.{h,c}` gains:

1. `readdir_on_empty_fs_returns_enoent`.
2. `readdir_yields_created_files_then_enoent` — creates three files,
   iterates; asserts each name seen exactly once in any order;
   asserts post-ENOENT cookie is sticky (repeated calls keep
   returning ENOENT).
3. `seek_set_to_zero_restarts_reads`.
4. `seek_end_returns_file_size`.
5. `seek_past_size_returns_einval` (also covers unknown whence →
   EINVAL).

`fs_stub` in `test/host/conformance_fs_test.c` grows matching impls
— seek uses the same clamped-to-[0,size] semantics as ramfs;
readdir iterates the in-use file table.

### Components

- **ramfs** (`components/ramfs/ramfs.c`):
  - `ramfs_op_seek` — switch on whence; compute `base + offset`;
    reject if out of `[0, size]`.
  - `ramfs_op_readdir` — loop from `*cookie` forward through the
    file-table array, skipping empty slots; on the first in-use
    slot found, copy the name + name_len into `*out`, advance
    `*cookie = i + 1`, return NX_OK.  On exhaustion, set `*cookie
    = RAMFS_MAX_FILES`, return NX_ENOENT.
- **vfs_simple** (`components/vfs_simple/vfs_simple.c`):
  - `vfs_simple_seek` / `_readdir` — resolve `filesystem.root` via
    `nx_slot_lookup`, dispatch to the driver's op.  Same late-
    binding discipline as slice 6.2's other methods.

### Syscalls

- `framework/syscall.h` gains `NX_SYS_SEEK = 9` and `NX_SYS_READDIR
  = 10` (numbers stable, channel/open/read/write renumbering
  avoided).
- `framework/syscall.c`:
  - `sys_seek(h, offset, whence)` — `lookup_file_object` with
    `NX_RIGHT_SEEK`, resolve vfs, dispatch.  Returns the driver's
    int64 result (new position or negative NX_E*) as
    `nx_status_t`.
  - `sys_readdir(user_cookie, user_out)` — `copy_from_user` the
    cookie into a kernel-local, dispatch, `copy_to_user` both the
    updated cookie and the populated dirent.  Returns NX_OK /
    NX_ENOENT / NX_EINVAL.
  - `sys_open` now grants `NX_RIGHT_SEEK` implicitly: if any of
    `NX_RIGHT_READ` or `NX_RIGHT_WRITE` is set on the handle, SEEK
    gets set too.

### Host tests (+5 new file_syscall_test.c cases)

- `sys_seek_set_then_read_returns_original_bytes`.
- `sys_seek_end_returns_file_size`.
- `sys_seek_past_size_returns_einval`.
- `sys_readdir_yields_created_files_then_enoent` — note: keeps
  handles open during readdir because `fake_fs` uses per-open
  slots as file storage; a close wipes the name.  Real drivers
  like ramfs separate file-table entries from per-open state (and
  the kernel ktest exercises that path).
- `sys_seek_no_seek_right_returns_eperm` — proves the rights
  check is distinct from the open-implicit-grant path: allocates
  a HANDLE_FILE directly with only `NX_RIGHT_READ`, sys_seek
  returns NX_EPERM.

### EL0 demo (`test/kernel/user_prog_readdir.S`)

PC-relative AArch64 asm that:

1. `sys_open("/rd", WRITE|CREATE)` → `sys_handle_close` — seeds a
   fresh file so readdir has something to find regardless of what
   prior ktests put in ramfs.
2. Loop: `sys_readdir(&_cookie, &_ent)`.  On NX_OK,
   `sys_debug_write(&ent.name, ent.name_len)` + `sys_debug_write
   ("\n", 1)`.  On NX_ENOENT, exit the loop.
3. `sys_debug_write("[el0-rdr-ok]\n", 13)` — tail marker.
4. wfe park.

`_cookie` and `_ent` are embedded in the program's `.rodata` copy
that gets memcopy'd into the user window; ADR gives user-window
addresses (writable from EL0 thanks to the AP=EL0+EL1 RW mapping).

`test/kernel/ktest_vfs.c` gains `el0_readdir_walks_root_and_emits_
names_then_marker`: spawns a kthread that drops to EL0; yields
until the debug_write counter ≥ 3 (minimum: 1 name + 1 newline + 1
tail marker — on a fresh ramfs).  Live ktest log shows `/rd` and
`[el0-rdr-ok]`.

### Test ordering observation

The linker-section ordering of ktests puts `el0_readdir_walks_root_
and_emits_names_then_marker` *before* the other ramfs-modifying VFS
ktests in the current build.  That means at the time readdir runs,
ramfs only contains `/rd` that the program just seeded — the other
files (`/ktest_hello`, `/hello`) from slice-6.2/6.3 tests haven't
been created yet.  The `>= 3` floor accommodates this.  Future
reshuffling would only increase the count (more files → more
debug_writes); the test stays green.

## Key Findings

- **Filesystem-level readdir is simpler than a dir-handle
  abstraction, and v1's flat namespace makes it semantically
  correct.**  The implementation in ramfs is 15 lines; a dir-
  handle version would need a new HANDLE_DIR type, a new close
  destructor, a new `opendir` syscall, and rights-mask decisions
  for it — all for zero incremental functionality in v1.
- **Per-byte cookie copy-in/copy-out is overkill for readdir.**
  Unlike paths (which the first draft of slice 6.3 needed a per-
  byte copy loop for), a 4-byte cookie is safely handled with one
  `copy_from_user` call — bounds check applies once, overflow
  impossible.  Kept the bulk-copy shape.
- **The "sticky NX_ENOENT after exhaustion" conformance case
  matters.**  Without it, a driver could legitimately cycle the
  cookie back to 0 on overflow and start yielding entries again —
  which would silently break any loop that relied on ENOENT as
  the termination condition.  The assertion in
  `readdir_yields_created_files_then_enoent` pins this invariant.

## Decisions Made

- **SEEK right implicit on any file open.**  Alternative: new
  `NX_FS_OPEN_SEEKABLE` flag.  Chose implicit because every file
  in every driver shipping in Phase 6 is seekable.  Why: adding
  the flag introduces a category of user-visible mistakes
  ("forgot to add SEEKABLE, got EPERM") with no real value — SEEK
  has no resource cost, no leak path.  How to apply: a future
  non-seekable stream type can drop the implicit grant and
  require an explicit flag.
- **No hole-filling on seek past EOF.**  POSIX allows it; we
  don't.  Why: ramfs's fixed per-file buffer can't represent
  holes efficiently; implementing them requires branching on
  every read to check "is this byte actually valid or a hole?"
  None of Phase 7's planned consumers (busybox, posix_shim) need
  it.  How to apply: the follow-up that introduces dynamic file
  storage can also introduce holes; conformance case grows a
  `seek_past_size_then_write_extends_file` case at that time.
- **Readdir returns a snapshot, not a live cursor.**  Drivers
  must tolerate concurrent file creation/deletion during
  iteration — the cookie just keeps advancing monotonically, and
  if a file is deleted between iterations its name simply
  doesn't appear.  v1 is single-threaded so this is automatic; a
  future SMP-safe driver will need to think about this explicitly.

## Status at End of Session

- `make test` → **364/364 pass (51 python + 258 host + 55 kernel),
  0 leaks, 0 errors, exit 0**.  +16 from slice 6.3.
- `make run` boots the 5-slot composition cleanly; `make
  test-kernel` ktest log shows four complete EL0 round-trips
  (`[el0] hello`, `[el0-chan-ok]`, `[el0-file-ok]`, `[el0-rdr-ok]`)
  — one per new EL0 capability added across the phase.
- `verify-registry` clean; `validate-config` passes.
- **Phase 6 closes.**

## Next Steps

Phase 7: Process model + POSIX shim.

- `nx_process_create / _start / _wait` — real processes (not just
  kthreads) with their own address spaces.  Per-task TTBR0 (a
  Phase 5 follow-up that 6.3's handle table could also exploit)
  lands here.
- `components/posix_shim/` — maps POSIX syscalls
  (open/read/write/fork/exec/pipe/signal) onto the nonux handle
  operations.  Slice 6.3's NX_SYS_OPEN already matches POSIX
  shape, so the shim's `open(path, flags, mode)` is a thin
  translation.
- Cross-compile busybox for aarch64 (static, minimal config).
- Package busybox + rootfs files into an initramfs that ramfs
  loads at boot.
- Boot → init process → exec busybox shell.  `ls /`, `echo hello
  | cat`, `ps` all work.

Slicing plan for Phase 7 lands when Phase 7 starts.

---

**Files Changed:**
- `sources/nonux/interfaces/fs.h` — `seek` / `readdir` ops; `struct nx_fs_dirent`; SEEK whence values
- `sources/nonux/interfaces/vfs.h` — matching additions; forward-declares dirent
- `sources/nonux/test/host/conformance/conformance_fs.h` + `.c` — 5 new universal cases
- `sources/nonux/test/host/conformance_fs_test.c` — `fs_stub` gains seek + readdir; +5 TEST() wrappers
- `sources/nonux/components/ramfs/ramfs.c` — `ramfs_op_seek`, `ramfs_op_readdir`
- `sources/nonux/test/host/component_ramfs_test.c` — +5 conformance TEST() wrappers
- `sources/nonux/components/vfs_simple/vfs_simple.c` — `vfs_simple_seek`, `vfs_simple_readdir`
- `sources/nonux/framework/syscall.h` — NX_SYS_SEEK / _READDIR
- `sources/nonux/framework/syscall.c` — `sys_seek`, `sys_readdir`; SEEK right granted implicitly on open
- `sources/nonux/test/host/file_syscall_test.c` — `fake_fs` gains seek + readdir; +5 syscall-level TESTs
- `sources/nonux/test/kernel/user_prog_readdir.S` — new (EL0 readdir demo)
- `sources/nonux/test/kernel/ktest_vfs.c` — `el0_readdir_walks_root_and_emits_names_then_marker`
- `sources/nonux/Makefile` — add `user_prog_readdir.S` to `KTEST_S`
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 6.4 rewritten; Phase 6 marked complete
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 26 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
