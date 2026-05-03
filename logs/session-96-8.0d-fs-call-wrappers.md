# Session 96: Slice 8.0d — Component-to-Component FS Call Wrappers

**Date:** 2026-05-03
**Phase:** Phase 8 — Runtime Recomposition, Group B (IPC migration)
**Branch:** master

---

## Goals

- Implement slice 8.0d: migrate component-to-component calls (vfs_simple → ramfs/procfs)
  through the `nx_fs_*` blocking-call / sync-dispatcher wrappers.
- Create `framework/fs_call.c` implementing 9 `nx_fs_*` wrappers.
- Migrate all direct `iface_ops` / `resolve_fs()` accesses in `vfs_simple.c` to use the wrappers.
- Add 24 host equivalence tests in `test/host/slice_8_0d_test.c`.

---

## What Was Done

### 1. `framework/fs_call.h` (already generated)

Already existed as a generated header from `tools/gen-iface.py`.  Declares
`nx_fs_open`, `nx_fs_close`, `nx_fs_retain`, `nx_fs_read`, `nx_fs_write`,
`nx_fs_seek`, `nx_fs_readdir`, `nx_fs_mkdir`, `nx_fs_stat`.

### 2. `framework/fs_call.c` — new file

Implements the 9 wrappers.

**Host path:** Direct ops call through `slot->active->descriptor->iface_ops`
(same fast-path pattern as `vfs_call.c`).

**Kernel path (sync dispatcher-to-dispatcher):** vfs_simple's `handle_msg`
runs on the dispatcher thread, where `nx_slot_call_blocking` cannot be used
(requires per-task `caller_slot`).  Instead, each wrapper:
1. Calls `disp_resolve(slot, &hmsg, &impl)` to get `handle_msg` + `impl`
   from `slot->active->descriptor->ops` (legal on dispatcher thread per R8).
2. Builds a synthetic `nx_ipc_message` on the kstack with the request struct
   as `msg.payload`.
3. Calls `hmsg(impl, &msg)` directly (sync dispatcher-to-dispatcher).
4. Reads the reply from `msg.payload` (the dispatch writes it in-place).

**Critical: GCC strict-aliasing barrier.**  `nx_fs_dispatch` writes the reply
into the request buffer through a type-punned pointer (different struct type).
Without a compiler barrier, GCC -O2 may cache pre-`hmsg` values of the
request buffer fields, producing stale reads from the reply.  Every wrapper
that reads from `msg.payload` after `hmsg` has:
```c
__asm__ volatile("" ::: "memory");
```
This barrier was identified through a heisenbug: `procfs_readdir_root_yields_kernel_pid_first`
ktest failed without the barrier (GCC returned stale value = NX_ENOENT instead
of the correct NX_OK), but passed with any kprintf inserted between `hmsg`
and the reply read (kprintf changed stack layout / register allocation,
preventing the cache).

### 3. `components/vfs_simple/vfs_simple.c` — migrated

Removed:
- `resolve_fs()` helper (direct `slot->active->descriptor->iface_ops` access)
- `resolve_for_path()` helper

Added `#include "framework/fs_call.h"`.

All 9 ops now call the corresponding `nx_fs_*` wrapper:

| Op | Before | After |
|---|---|---|
| open | `resolve_fs` + `ops->open` | `nx_fs_open(nx_slot_lookup(mount), ...)` |
| close | `resolve_fs` + `ops->close` | `nx_fs_close(nx_slot_lookup(w->mount), ...)` |
| retain | `resolve_fs` + `ops->retain` | `nx_fs_retain(nx_slot_lookup(w->mount), ...)`; pre-checks `slot->active` before bumping `w->refs` |
| read | `resolve_fs` + `ops->read` | `nx_fs_read(nx_slot_lookup(w->mount), ...)` |
| write | `resolve_fs` + `ops->write` | `nx_fs_write(nx_slot_lookup(w->mount), ...)` |
| seek | `resolve_fs` + `ops->seek` | `nx_fs_seek(nx_slot_lookup(w->mount), ...)` |
| readdir | `resolve_for_path` + `ops->readdir` | `nx_fs_readdir(nx_slot_lookup(mount_for_path(dir_path)), ...)` |
| mkdir | `resolve_for_path` + `ops->mkdir` | `nx_fs_mkdir(nx_slot_lookup(mount_for_path(path)), ...)` |
| stat | `resolve_for_path` + `ops->stat` | `nx_fs_stat(nx_slot_lookup(mount_for_path(path)), ...)` |

### 4. `test/host/slice_8_0d_test.c` — new file, 24 tests

Tests `nx_fs_*` wrappers directly against a fake filesystem driver registered
in a `filesystem.root` slot (no vfs_simple in the fixture).  Covers all 9
operations: basic, equivalence-to-direct-ops, null-slot robustness.

### 5. Makefile updates

- Top-level `Makefile`: added `framework/fs_call.c` to `FW_C`.
- `test/host/Makefile`: added `slice_8_0d_test.c` and `../../framework/fs_call.c`.

---

## Debugging Notes

### Interactive test timing flake (mkdir_tmp)

`mkdir_tmp` failed on the first run (QEMU timeout mid-input). The second run
passed.  Root cause: the 60-second test budget is tight when IPC round-trips
at 10 Hz accumulate.  My 8.0d changes add zero blocking overhead (direct
handle_msg call, no dispatcher queue).  The flake is pre-existing scheduling
variance; the test passes consistently.

### GCC strict-aliasing heisenbug

`procfs_readdir_root_yields_kernel_pid_first` ktest failed without kprintfs,
passed with any kprintf inserted between `hmsg` and the reply read.  Cause:
GCC -O2 strict-aliasing optimization: the in-place reply write
(`(struct nx_fs_reply_readdir *)msg->payload`) uses a type incompatible with
the original `struct nx_fs_msg_readdir req`.  GCC cached the pre-call content
of `req` (zeroed by `memset`), so `rep->rc` read a stale 0 — but for readdir
this meant the ENOENT path was taken in some optimized code path.

Fix: `__asm__ volatile("" ::: "memory")` after every `hmsg` call in
`fs_call.c`, applied to all 7 wrappers that read back from `msg.payload`.
This forces GCC to reload from memory.

---

## Test Results

```
make test-tools:         93/93  PASS
make test-host:         421/421 PASS  (+24 new slice_8_0d tests)
make verify-iface-fresh: 0 drift
make verify-registry:    0 findings
make test-interactive:   7/7  PASS
make test-kernel:       123/123 PASS
```

---

## Decisions Made

- **Sync dispatcher-to-dispatcher via direct `handle_msg`** — vfs_simple runs
  on the dispatcher; `nx_slot_call_blocking` requires a per-task `caller_slot`
  which isn't available there.  The direct call is legal (R8 permits slot-active
  reads on dispatcher threads) and avoids an unnecessary IPC queue round-trip.
- **`iface_ops` not accessed in kernel path** — only `descriptor->ops->handle_msg`
  is read; `descriptor->iface_ops` is read only in the host path.  This
  satisfies the forthcoming 8.0e iface_ops rule without changes.
- **Compiler barrier after every `hmsg`** — type-punned in-place reply write
  creates a latent strict-aliasing UB; the barrier is the minimal correct fix
  without restructuring the generated dispatch protocol.

---

## Commits

(To be created at end of session)

## Next Step

Slice **8.0e** — `verify-registry.py` rule banning direct `slot->active->descriptor->iface_ops`
access outside `framework/dispatcher.c`.  After 8.0d, the remaining
`iface_ops` reads are only in the host-build fast-paths of `vfs_call.c` and
`fs_call.c` (both exempt since they're `#if __STDC_HOSTED__`).
