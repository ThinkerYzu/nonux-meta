# Session 116: `chdir` and `getcwd`

**Date:** 2026-05-06
**Phase:** Post-9b (between Phase 9b and Phase 9)
**Branch:** master

---

## Goals

- Add per-process current working directory (CWD) with `chdir` and `getcwd` syscalls so busybox `cd` works.

## What Was Done

### Part 1 — `struct nx_process` CWD field

Added `char cwd[NX_PROCESS_CWD_MAX]` (128 bytes) to `struct nx_process` in `framework/process.h`, with a matching `#define NX_PROCESS_CWD_MAX 128u`.

- `g_kernel_process` static initialiser: `.cwd = "/"`.
- `nx_process_create`: initialises `cwd` to `"/"`.
- `nx_process_fork`: inherits parent's CWD via `memcpy`.
- `nx_process_reset_for_test`: resets `g_kernel_process.cwd` to `"/"`.

### Part 2 — `path_make_absolute` helper (`framework/syscall.c`)

Added `static void path_make_absolute(char kpath[NX_PATH_MAX])` just after `path_normalize`.  If `kpath[0] == '/'` it simply calls `path_normalize`.  Otherwise it prepends the current process's CWD (stripping any trailing slash) and a `/`, appends `kpath`, then calls `path_normalize`.  Result is always an absolute, normalised path within `NX_PATH_MAX`.

Updated all path-consuming syscalls to call `path_make_absolute` instead of `path_normalize`:
- `sys_open` (line ~478)
- `sys_fstatat` (line ~1899)
- `sys_mkdirat` (line ~2127)

`sys_openat` is unaffected because it delegates to `sys_open`.

### Part 3 — `NX_SYS_CHDIR = 46` and `NX_SYS_GETCWD = 47` (`framework/syscall.h`)

Added with full doc comments. `NX_SYSCALL_COUNT` is now 48.

### Part 4 — `sys_chdir` and `sys_getcwd` implementations

`sys_chdir(a0)`:
1. `copy_path_from_user` + `path_make_absolute`.
2. `nx_vfs_stat` to verify the resolved path exists and is `NX_FS_KIND_DIR`.
3. On success copy `kpath` into `proc->cwd`; return 0.
4. On ENOENT return Linux `-2`; if not DIR return Linux `-20` (ENOTDIR); otherwise Linux `-22`.

`sys_getcwd(a0, a1)`:
1. Validate `buf` and `size`.
2. `copy_to_user` the current process's `cwd` (including NUL).
3. Return the number of bytes written.  musl's `getcwd(3)` treats any non-negative return as success.

Both added to `g_syscall_table`.

`nx_syscall_reset_for_test` now also resets `proc->cwd` to `"/"`.

### Part 5 — musl translation table (`third_party/musl/arch/aarch64/syscall_arch.h`)

Added to `__nx_translate()`:
- `case 17: return 47;  /* __NR_getcwd -> NX_SYS_GETCWD */`
- `case 49: return 46;  /* __NR_chdir  -> NX_SYS_CHDIR  */`

### Part 6 — Tests

**Host tests** (9 new tests in `test/host/file_syscall_test.c`):
- `sys_getcwd_returns_initial_cwd_as_root`
- `sys_getcwd_null_buf_returns_einval`
- `sys_getcwd_zero_size_returns_einval`
- `sys_chdir_to_root_succeeds_and_getcwd_confirms`
- `sys_chdir_to_nonexistent_path_returns_enoent`
- `sys_chdir_to_file_returns_enotdir`
- `sys_chdir_null_path_returns_einval`
- `sys_open_relative_path_resolved_against_cwd`
- `sys_fstatat_relative_path_resolved_against_cwd`

**Kernel test** (1 new test in `test/kernel/ktest_vfs.c`):
- `sys_getcwd_returns_per_process_cwd` — verifies that directly updating `proc->cwd` is reflected by `sys_getcwd` via SVC.  Full `sys_chdir` path (uses blocking IPC, needs a live kthread task) is covered by host tests; kernel test restricts to the non-IPC `sys_getcwd` path which can be driven from the synchronous ktest body.

## Test Results

- `make test-tools` → **102/102 pass**
- `make test-host` → **485/485 pass** (+9 new tests, was 476)
- `make test-kernel` → **152/152 pass** (+1 new test, was 151)
- `make verify-iface-fresh` → 0 drift
- `make verify-registry` → 0 findings

### Part 7 — Makefile dependency fix

`make test` was not rebuilding `initramfs-busybox.cpio` after a musl/busybox change, because `kernel-busybox.bin` was not in the `test` prerequisites.  The dependency chain was structurally correct (kernel-busybox.bin → initramfs-busybox.cpio → busybox → musl → syscall_arch.h) but only triggered by `make test-interactive` or `make run-busybox`, not by `make test`.

Fix: added `kernel-busybox.bin` to the `test` target and updated the stale comment on the busybox rule.

## Key Findings

1. **Stale .o files**: After adding new entries to `NX_SYSCALL_COUNT`, host and kernel test `.o` files compiled against the old count will call the wrong handler for the sentinel value.  Fixed by `make clean` in `test/host/` and by forcing a rebuild of `ktest_syscall.o`.  The project should add automatic header dependency tracking to the test Makefile.

2. **Blocking IPC from ktest body**: The synchronous ktest runner executes before any kthread is scheduled, so `TPIDR_EL1 = 0` → `nx_task_current() = NULL` → any syscall that calls `nx_slot_call_blocking` (including `sys_chdir`'s `nx_vfs_stat`) returns `NX_EINVAL`.  Pattern: tests that need blocking IPC must use `sched_spawn_kthread` (see ktest_9b_3.c pattern).

3. **Stale initramfs-busybox.cpio**: The interactive kernel image embeds busybox via `initramfs-busybox.cpio`, but this was only rebuilt by `make test-interactive`/`make run-busybox`, not by `make test`.  After a musl change, `make test` correctly rebuilt busybox (via `kernel-test.bin` → `initramfs.cpio` → `busybox`), but left `initramfs-busybox.cpio` stale — so the interactive shell still ran the old binary.  Fixed by adding `kernel-busybox.bin` to the `test` target.

## Next Actions

Continue with Phase 9 (per-process MM rework — L3 4 KiB pages, VMAs, demand paging, COW fork).
