# Session 75 — slice 7.7b.2: `ps` via procfs

**Date:** 2026-04-28
**Slice:** 7.7b.2 — `ps` via procfs
**Outcome:** **Closed.**  busybox's stock `ps` applet runs end-to-end, reads `/proc` + `/proc/<pid>/stat`, and prints rows for the kernel process + init + the forked-ps process.  No vendored-busybox patch.  `make test` 439 → **447/447** (+8 procfs ktests); `make test-interactive` 6/6 → **7/7** (gained `ps_smoke`).

---

## Goal

Slice 7.7b.1 (Session 74) made hierarchical paths real in ramfs.  This slice completes the `ps` exit criterion of slice 7.7 by introducing a synthesised `/proc` mount and routing `/proc/...` through it from `vfs_simple`.

The slice plan (HANDOFF.md Next Actions §0 + IMPLEMENTATION-GUIDE.md slice 7.7b.2) called for:

1. New `components/procfs/procfs.c` implementing `nx_fs_ops` against the live process table.
2. New public process iterator (`nx_process_for_each`) in `framework/process.c`.
3. `vfs_simple` extended with a tiny mount table routing `/proc/...` → `filesystem.proc`.
4. `kernel.json` gains `filesystem.proc ← procfs`.
5. `INITRAMFS_ENTRIES` gains `$(BUSYBOX_BIN):/bin/ps` so the stock applet ships.
6. Test deliverable: `test/interactive/ps_smoke.{script,expected}` taking `make test-interactive` 6/6 → 7/7.

All landed.  Real estimated cost was ~700 lines (vs. the plan's ~250) once the per-open wrapper and the trailing-slash bug surfaced — see *Discoveries* below.

---

## What landed

### `framework/process.{h,c}` — public iterator

New API:

```c
int nx_process_for_each(int (*cb)(struct nx_process *, void *), void *ctx);
```

Visits `g_kernel_process` first, then walks `g_process_table[]` in slot order.  Callback returns 0 to continue, non-zero to stop early; iteration return value mirrors the stop-code (or 0 if completed).  Replaces ad-hoc table walks that were previously private to `process_table_add` / `_remove`.

procfs uses it via a `find_nth` callback pattern (`procfs_find_nth(idx)` builds a `target_idx`/`current_idx` context and stops at the matching slot).  O(n²) per directory listing of `/proc`, fine for n ≤ 129.

### `components/procfs/` — synthesised process filesystem

New component, ~370 lines.  Same `nx_fs_ops` shape as ramfs but every entry is generated on demand from the live process table:

- `procfs_op_stat` recognises `/proc` (DIR), `/proc/<pid>` (DIR if pid exists), `/proc/<pid>/` (same — trailing slash variant, see *Discoveries*), `/proc/<pid>/stat` (FILE).
- `procfs_op_readdir` for `/proc` walks every live process via `nx_process_for_each`, emitting the pid as a decimal basename.  For `/proc/<pid>` it emits the single child `stat`.
- `procfs_op_open` accepts `O_RDONLY` for `/proc/<pid>/stat` only; renders the Linux-shape stat line into a per-open buffer at open time so concurrent reads see a stable snapshot rather than racing the process table.
- `procfs_op_read` returns from the buffer at the cursor; `seek` works as expected.
- Read-only: `mkdir`, `write`, and write/append/create flags on `open` return `NX_EPERM`.
- Per-open pool of 8 fixed-size structs; same allocator-free pattern as ramfs.

The stat line emits 24 fields covering busybox `procps_scan`'s slow path (pid, comm, state, ppid, pgid, sid, tty, tpgid, flags, minflt..rss).  Only pid/comm/state/ppid carry real data — the rest are zeros.  busybox tolerates this: ps shows `0` for VSZ and `0:00` for TIME.

State characters: `R` (running) for `NX_PROCESS_STATE_ACTIVE`, `Z` (zombie) for `NX_PROCESS_STATE_EXITED`.  busybox uses these verbatim in the STAT column.

### `components/vfs_simple/vfs_simple.c` — mount table + per-open wrapper

Two changes pulled in by serving multiple mounts:

1. **Mount table** (`mount_for_path` + `resolve_for_path`).  `/proc` and `/proc/...` route to `filesystem.proc`; everything else stays on `filesystem.root`.  The lookup stays branch-free for the common case (single literal-prefix compare).  Hard-coded rather than table-driven because v1 has exactly one non-root mount and the procfs driver itself encodes the mount path; introducing a config-driven table now would just push the coupling around.  A second non-root mount triggers promotion.

2. **Per-open wrapper** (`struct vfs_simple_open`).  The opaque `void *file` returned by vfs_simple's open() can no longer be the driver's per-open pointer directly: subsequent close/read/write/seek/retain only have that pointer to work with and wouldn't know which driver's op table to dispatch into.  The wrapper records the mount slot name (a string-literal pointer) alongside the driver per-open; every op re-resolves the slot via `nx_slot_lookup` so the slot's late-binding semantics (DESIGN §Slot-Based Indirection) survive.  Pool of 108 wrappers (RAMFS_MAX_OPEN + PROCFS_MAX_OPEN + retain headroom; ~3.5 KB .bss).

   First-iteration of the wrapper cached `ops` and `driver_self` pointers and broke the slice-3.x `slot_swap_to_NULL_clears_handle` test (the test asserts that an `nx_slot_swap(slot, NULL)` mid-run causes subsequent reads to fail with `NX_ENOENT`; cached pointers stayed valid past the swap).  Switched to caching the slot *name* instead; the resolve happens per-op now.  Cost is one extra `nx_slot_lookup` per read/write/seek call, which is a hash-free linear scan over a tiny global table — negligible.

   Refcount discipline: wrapper.refs and driver per-open refs move in lockstep so a wrapper is freed exactly when its driver per-open is.  `open` returns refs=1 on both sides; `retain` calls driver retain (must be non-NULL — every filesystem driver bound under vfs_simple now implements it as of slice 7.7b.2) and bumps wrapper.refs.  `close` always calls driver close and decrements wrapper.refs; the wrapper is returned to the pool when refs reaches zero.

### `kernel.json` — new slot

```json
"filesystem.proc": {
  "impl": "procfs",
  "config": {}
}
```

`gen-config.py`'s slot-iface derivation (`split('.', 1)[0]`) automatically tags it as `iface = "filesystem"`.  No tooling changes needed.

### `Makefile` — new initramfs entry + new ktest source

- `INITRAMFS_ENTRIES` gains `$(BUSYBOX_BIN):/bin/ps` (16 entries total in `initramfs-busybox.cpio`).
- `KTEST_C` gains `test/kernel/ktest_procfs.c`.

### `test/host/Makefile` — host build adds procfs.c

Compiles `components/procfs/procfs.c` so the symbol resolution stays clean even though no host test exercises procfs through vfs_simple in this slice.  vpath updated to find the source.

### `test/kernel/ktest_procfs.c` — fast regression guard (8 ktests)

Covers the procfs bring-up at the vfs layer without paying the 60-second QEMU stdin-feed of the interactive harness:

- `bootstrap_binds_procfs_to_filesystem_proc_slot` — bootstrap binding round-trip.
- `procfs_stat_root_reports_dir` — `/proc` is DIR.
- `procfs_stat_pid_dir_handles_trailing_slash` — `/proc/0` and `/proc/0/` both DIR (busybox calls the trailing-slash variant; without the trailing-slash branch every process is silently skipped).
- `procfs_stat_pid_stat_reports_file` — `/proc/0/stat` is FILE with size > 0.
- `procfs_stat_unknown_pid_returns_enoent` — `/proc/99999/stat` → `NX_ENOENT`.
- `procfs_readdir_root_yields_kernel_pid_first` — first readdir call returns `"0"`.
- `procfs_open_pid0_stat_renders_kernel_R_0_prefix` — the rendered stat line for pid 0 starts with `"0 (kernel) R 0 "` (the prefix busybox's procps_scan parses).
- `procfs_mount_does_not_intercept_ramfs_paths` — `/banner` (initramfs file) still goes to ramfs.

### `test/interactive/ps_smoke.{script,expected}`

```
script:    ps\nexit\n
expected:  PID   USER     TIME  COMMAND
           [kernel]
           [init]
```

Tightened from the original draft (`init` only) after seeing that the kernel boot message `[init] entering busybox sh at EL0` would false-positive on the bare `init` substring — same mechanism that bit slice 7.7a.  The PID column header is unique to ps's output; `[kernel]` only appears as a bracketed comm rendered by busybox when cmdline is empty (which it is for our procfs).

---

## Discoveries

### 1. Trailing-slash stat: busybox calls `stat("/proc/<pid>/", ...)`

First-pass procfs failed silently — `ps` printed only its column header and zero rows.  Tracing through busybox's `libbb/procps.c:354`:

```c
filename_tail = filename + sprintf(filename, "/proc/%u/", pid);
if (flags & PSSCAN_UIDGID) {
    struct stat sb;
    if (stat(filename, &sb))
        continue; /* process probably exited */
```

The trailing slash on `/proc/%u/` is intentional — POSIX says `stat()` on a path ending in `/` requires the path to refer to a directory.  My initial `procfs_op_stat` only matched `/proc/<pid>` (no trailing slash) → the stat returned `NX_ENOENT` → `continue` → every process silently skipped.  Default `ps` requests `pid,user,time,args`; the `user` column needs UIDGID; the UIDGID branch fires for every entry; every entry skipped; ps prints zero rows.

Fix: extended `procfs_op_stat` (and `procfs_op_readdir` for symmetry) to also recognise `tail[0] == '/' && tail[1] == '\0'` as the same DIR result as `tail[0] == '\0'`.  Added a `procfs_stat_pid_dir_handles_trailing_slash` ktest specifically to lock this in.

(Latent same-shape gap exists in ramfs's `stat("/bin/", ...)` — never surfaced because no caller stats a directory with a trailing slash on the ramfs side.  Not fixed in this slice; recorded as a low-priority cleanup.)

### 2. The vfs_simple per-open wrapper had to preserve slot-late-binding

Initial wrapper cached `(ops, driver_self, driver_file)` so per-op dispatch was a single struct field load.  Broke the `slot_swap_to_NULL_clears_handle` host test (component_vfs_simple_test.c:295) which asserts that `nx_slot_swap(&fx.root_slot, NULL)` between two reads on the same handle causes the second read to return `NX_ENOENT`.  With cached ops, the second read happily dispatched to the still-pointed-at driver.

Switched the wrapper to cache only the slot *name* — every op now does `nx_slot_lookup(w->mount)` first, picks up the current binding (or `NULL` if swapped out).  Preserves the DESIGN-mandated late-binding property; cost is one tiny linear lookup per call.

### 3. ps fell back to bracketed comm when cmdline was empty

procfs doesn't implement `/proc/<pid>/cmdline` in v1.  busybox's `read_cmdline` (libbb/procps.c:569) tolerates `sz <= 0` — it just doesn't populate the cmdline buffer.  ps then renders `[<comm>]` (square-bracketed comm) for the `args` column.  This is the standard "kernel-thread style" rendering and is exactly what we want: confirms that procfs's stat line is being parsed and the comm is being extracted correctly.  Output:

```
 # ps
PID   USER     TIME  COMMAND
    0 0         0:00 [kernel]
    1 0         0:00 [init]
    2 0         0:00 [init]
 # exit
```

(pid 2 is the busybox-ps child of pid 1 — fork inherits parent's name in v1, exec doesn't update it.)

### 4. Build cache hazard recurrence

Same as Session 74's `INITRAMFS_ENTRIES` change: editing the Makefile var doesn't touch the cpio rebuild rule, so a stale `initramfs-busybox.cpio` survives across builds.  Workflow now: `rm -f test/kernel/initramfs.cpio test/kernel/initramfs-busybox.cpio` after any `INITRAMFS_ENTRIES` edit.  Recording this for future slices that touch the entry list — long-term fix is to add the Makefile itself as a dep of the cpio target, which is a one-line drive-by some future cleanup slice can land.

---

## Tests

```
make test
  python:  51/51   PASS
  host:   283/283  PASS  (0 leaks, 0 errors)
  kernel: 113/113  PASS  (was 105; +8 procfs ktests)
  total:  447/447  PASS

make test-interactive
  echo_cat:        PASS
  echo_hello:      PASS
  echo_pipe:       PASS
  ls_root:         PASS
  mkdir_tmp:       PASS
  ps_smoke:        PASS  (new in this session)
  visible_prompt:  PASS
  total:    7/7    PASS
```

---

## Files changed

**New (5):**
- `components/procfs/manifest.json`
- `components/procfs/procfs.c`
- `test/kernel/ktest_procfs.c`
- `test/interactive/ps_smoke.script`
- `test/interactive/ps_smoke.expected`

**Modified (6):**
- `framework/process.h` (+`nx_process_for_each` decl)
- `framework/process.c` (+`nx_process_for_each` impl)
- `components/vfs_simple/vfs_simple.c` (mount table + per-open wrapper)
- `kernel.json` (+`filesystem.proc` slot)
- `Makefile` (+`/bin/ps` initramfs entry, +`ktest_procfs.c`)
- `test/host/Makefile` (+`procfs.c` source + vpath)

**Auto-regenerated (3):** `gen/sources.mk`, `gen/slot_table.c`, `gen/config.h` (from `kernel.json` via `gen-config.py`).

---

## Next slice

Slice 7.7 is now **closed** — all four exit criteria of the original slice plan have shipped (`ls /`, `echo | cat`, `mkdir /tmp`, `ps`).  Phase 7 still has slice 7.8 (wait queues + `ppoll`) as the foundation slice that:

- promotes the three v1 yield-loop fakes (`sys_read` CHANNEL arm, `sys_wait`, `nx_console_read`) onto a real kernel blocking primitive;
- lands `NX_SYS_PPOLL`;
- reverts the slice-7.6d.N.final.e busybox `fflush(stdout)` workaround once busybox's intended `ask_terminal()` flush path runs again.

See HANDOFF.md Next Actions §0.1 and IMPLEMENTATION-GUIDE.md §Slice 7.8 for the per-sub-slice plan (a/b/c).
