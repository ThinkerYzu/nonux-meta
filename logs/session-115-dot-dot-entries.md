# Session 115: `.` and `..` directory entries

**Date:** 2026-05-06
**Phase:** Post-9b (between Phase 9b and Phase 9)
**Branch:** master

---

## Goals

- Add POSIX `.` and `..` entries to ramfs directory listings so `ls -a` works correctly.

## What Was Done

### Part 1 — `ramfs_op_readdir`: yield `.` and `..`

Reserved cookie values 0 and 1 for synthetic dot entries, shifting all real file-table entries to cookie ≥ 2:

- `cookie == 0` → yield `.`, advance to 1
- `cookie == 1` → yield `..`, advance to 2
- `cookie >= 2` → iterate `files[cookie-2]`; yield at index `j` sets `cookie = j+3`
- Terminal: `cookie = RAMFS_MAX_FILES + 2`, return `NX_ENOENT`

The existing `cur->cookie--` rewind in `sys_getdents64` (for staging-buffer-full case) still works correctly because each dot entry is exactly one cookie-step apart.

### Part 2 — `sys_getdents64`: set `d_type = DT_DIR` for `.` and `..`

Added a name check: if `name == "."` or `name == ".."`, emit `NX_LINUX_DT_DIR` instead of `NX_LINUX_DT_REG`. Without this, terminals and tools that consult `d_type` for coloring/classification would mis-identify the dot entries.

### Part 3 — Conformance tests updated

Two conformance test functions updated to tolerate dot entries from POSIX-compliant drivers:
- `nx_conformance_fs_readdir_on_empty_fs_returns_enoent`: drains up to 2 dot entries before expecting `NX_ENOENT`.
- `nx_conformance_fs_readdir_yields_created_files_then_enoent`: skips dot entries in the match loop; iteration limit raised from 10 to 12.

### Part 4 — `path_normalize` + call in `sys_open` and `sys_fstatat`

**Root cause of `ls -a /` still failing:** busybox `ls -a` calls `stat` on every dirent entry using its full path. For entries `.` and `..` in directory `/`, that means `stat("/.")` and `stat("/..")`. The VFS/ramfs stat handler had no handling for these path components, returning `ENOENT`, causing busybox to print `ls: /.: No such file or directory`.

Added `path_normalize(char *path)` — a 50-line in-place normalizer that resolves `.` and `..` segments:
- `/./foo` → `/foo`
- `/.` → `/`
- `/..` → `/` (root's parent is root)
- `/foo/../bar` → `/bar`
- `/foo/./bar/..` → `/foo`

Uses a `uint8_t stk[]` segment-position stack (64 entries, 64 bytes) and a `char out[NX_PATH_MAX]` scratch buffer (128 bytes); total stack cost ~200 bytes, well within the SVC handler budget.

Called after `copy_path_from_user` in both `sys_open` (covers `sys_openat`) and `sys_fstatat`.

## Key Findings

- **Two-part bug.** The readdir fix alone was not sufficient: even though getdents64 returned `.` and `..` entries, busybox's subsequent `stat` calls on `"/."` and `"/.."` failed because the FS layer had no path normalization. The real fix required both halves.
- **`kernel-busybox.bin` is the interactive binary.** `make` rebuilds `kernel.bin` (test kernel); the interactive `make run-busybox` target uses `kernel-busybox.bin`, which is a separate build target.
- **Cookie offset is rewind-safe.** The `cur->cookie--` rewind in `sys_getdents64` (for when the staging buffer is full) works unchanged with the 2-offset cookie scheme because each entry advances the cookie by exactly 1.

## Decisions Made

- **Normalize in syscall layer, not FS driver** — `path_normalize` lives in `syscall.c` and is called before any VFS/FS call. This fixes the issue for all FS drivers at once and keeps drivers simple.
- **Dot entries only in ramfs, not procfs** — procfs has its own internal cookie scheme; adding dot entries to procfs would break existing ktests that check `cookie=0` yields the first PID. Deferred.
- **Conformance tests: tolerate, not require** — Updated conformance tests to skip dot entries if present rather than mandating them. `fs_stub` (used by the conformance suite for VFS tests) doesn't yield dot entries and doesn't need to.

## Status at End of Session

- `ls -a /` in busybox shell shows `.` and `..` (colored blue as DT_DIR).
- `make test-host` → **476/476 pass**
- `make test-kernel` → **151/151 pass**
- `make test-interactive` → **7/7 pass**

## Next Steps

- Phase 9 — per-process MM rework: L3 4 KiB pages, VMAs, demand paging, COW fork. See [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework).

---

**Files Changed:**
- `components/ramfs/ramfs.c` — `ramfs_op_readdir`: cookie 0→`.`, cookie 1→`..`, real entries shifted to cookie ≥ 2.
- `framework/syscall.c` — `path_normalize()` helper + calls in `sys_open` and `sys_fstatat`; `sys_getdents64` sets `d_type = DT_DIR` for `.`/`..`.
- `test/host/conformance/conformance_fs.c` — two conformance test functions updated to tolerate dot entries.
