# Session 29: slice 6.2 — vfs_simple + ramfs (first real VFS composition)

**Date:** 2026-04-23
**Phase:** 6 slice 6.2
**Branch:** master

---

## Goals

Land the first real VFS layer + the first real filesystem driver on
top of slice 6.1's `nx_fs_ops` interface. Two new components wired
into `kernel.json`, brought up through the bootstrap topo sort, and
exercised end-to-end with a live create/write/read round-trip in the
kernel test. Slice 6.3 then plugs the syscall layer into this.

## Scope choices

Several scope-narrowing decisions vs. the original plan:

- **Ramfs is static, no `memory.page_alloc` dep.**  The plan had
  ramfs allocating backing pages through the `memory` slot.  Dropped
  it — a fixed 8-file × 256-byte inline table inside `struct
  ramfs_state` is enough to pass every conformance case and keeps 6.2
  focused on "first filesystem component + first VFS component"
  rather than "first component with a runtime-allocated backing
  store".  A future slice can swap to dynamic storage without
  touching the interface.
- **No `NX_FS_DRIVER_REGISTER` linker section.**  The plan had ramfs
  populate a separate fs-driver section that vfs_simple would walk at
  init time.  Dropped — the component graph already has everything
  needed: ramfs binds to `filesystem.root`, and vfs_simple dispatches
  through `slot->active->descriptor->iface_ops` (DESIGN §Slot-Based
  Indirection).  One fewer concept, fewer moving parts.
- **No `root_fs: "ramfs"` config knob.**  Follows from the previous
  decision: the slot binding *is* the selector.  Swapping to a
  different filesystem is `kernel.json`'s `filesystem.root: { impl:
  "tmpfs" }`, not a new config string.
- **No DI dep in vfs_simple's manifest.**  `gen-config.py`'s kernel
  mode doesn't emit per-component `gen/<name>_deps.h` files yet
  (only `config.h` / `sources.mk` / `slot_table.c`).  Adding DI here
  would force extending gen-config — out of scope.  Instead
  vfs_simple late-binds via `nx_slot_lookup("filesystem.root")` at
  every call, which is also the correct long-term behaviour: it
  means future hot-swap of the root filesystem doesn't need any
  invalidation inside vfs_simple.
- **Path handling is minimal.**  vfs_simple rejects relative paths
  with `NX_EINVAL` and passes absolute paths through to the driver
  unchanged (the leading `/` ends up part of the filename in ramfs's
  flat namespace).  Mount-prefix stripping lands when there's more
  than one mount (Phase 8+).

## What Was Done

### `interfaces/vfs.h` — new

- `struct nx_vfs_ops { open, close, read, write }` — the upper API
  the syscall layer will dispatch through in slice 6.3.
- `NX_VFS_OPEN_READ / _WRITE / _CREATE` defined as `(1U << 0) .. (1U
  << 2)` — bit-compatible with `NX_FS_OPEN_*` from `interfaces/fs.h`
  so vfs_simple forwards the flag word verbatim, no translation.
- Kept structurally separate from `fs.h` (even though v1 shapes
  match) because the two interfaces will diverge in slice 6.4:
  `nx_vfs_ops.readdir` iterates across mount boundaries,
  `nx_fs_ops.readdir` stops at the current filesystem's boundary.

### `components/ramfs/` — new

- `ramfs.c` (~250 LOC including headers/comments): implements
  `nx_fs_ops` on a static fixed-capacity inode table (`RAMFS_MAX_FILES
  = 8`, `RAMFS_FILE_CAP = 256`, `RAMFS_MAX_OPEN = 32`).
- Per-open state (`struct ramfs_open`) lives in a pool inside the
  instance state — no dynamic allocator, no `memory.page_alloc` dep.
  An exhausted pool returns `NX_ENOMEM` from `open`.
- `NX_COMPONENT_REGISTER_NO_DEPS_IFACE(ramfs, …, &ramfs_fs_ops)`
  publishes the fs ops through the descriptor's `iface_ops` field —
  the same wiring sched_rr uses for `nx_scheduler_ops`.
- `manifest.json`: `{ "name": "ramfs", "version": "0.1.0", "iface":
  "filesystem" }`.
- `README.md` documents interface, state layout, and the
  slice-6.1 conformance relationship.

### `components/vfs_simple/` — new

- `vfs_simple.c` (~150 LOC): thin dispatch layer.  Every op calls
  `resolve_root_fs()` which does `nx_slot_lookup("filesystem.root")`
  and extracts `slot->active->descriptor->iface_ops` as `struct
  nx_fs_ops *` plus `slot->active->impl` as `self`.  `NX_ENOENT` if
  the slot is unregistered or unbound (i.e. no filesystem mounted).
- `NX_COMPONENT_REGISTER_NO_DEPS_IFACE(vfs_simple, …,
  &vfs_simple_vfs_ops)` — published through the `vfs` slot's iface_ops
  so the syscall layer in 6.3 reads it the same way.
- `manifest.json`: `{ "name": "vfs_simple", "version": "0.1.0",
  "iface": "vfs" }`.
- `README.md` explains the late-binding discipline and why
  vfs_simple doesn't have a DI dep.

### `kernel.json` — two new slots

Added `filesystem.root ← ramfs` and `vfs ← vfs_simple`.
`gen-config.py` auto-derives their ifaces from the slot names
(`filesystem.root` → iface `filesystem`; `vfs` → iface `vfs`) and
emits the expected `gen/slot_table.c` entries.

Booted kernel now reports:

```
[fw]   composition (gen=25, 5 slots, 5 components):
{..."slots":[{"name":"vfs",...,"active":{"manifest":"vfs_simple"}},
             {"name":"scheduler",...},
             {"name":"memory.page_alloc",...},
             {"name":"filesystem.root",...,"active":{"manifest":"ramfs"}},
             {"name":"char_device.serial",...}],
 "components":[{"manifest":"vfs_simple",...,"state":"active"},
               {"manifest":"uart_pl011",...,"state":"active"},
               {"manifest":"sched_rr",...,"state":"active"},
               {"manifest":"ramfs",...,"state":"active"},
               {"manifest":"mm_buddy",...,"state":"active"}]}
```

### Tests

- **`test/host/component_ramfs_test.c`** — 10 tests: 7 conformance
  helpers (slice 6.1 suite), 1 lifecycle-100× cycling, 2 ramfs-
  specific exhaustion smoke tests (file-table full → NX_ENOMEM;
  per-open pool full → NX_ENOMEM).
- **`test/host/component_vfs_simple_test.c`** — 7 tests via an
  in-test fake fs driver injected into the `filesystem.root` slot:
  lifecycle counter tick, path/flags forwarding, byte-count
  forwarding on write/read, relative-path validation,
  null-path validation, no-mount-→-NX_ENOENT, and the late-binding
  property (unbind the slot mid-run → subsequent ops see NX_ENOENT
  even though earlier ops on the same handle succeeded).
- **`test/kernel/ktest_vfs.c`** — 6 tests: 4 bootstrap assertions
  (both slots registered + bound in ACTIVE state with expected
  manifest names), 1 end-to-end create/write/close/reopen/read
  round-trip through vfs_simple → ramfs on the live bootstrap
  instance, 1 relative-path validation.

### Makefile wiring

- `test/host/Makefile`: SRCS gets the two new component source
  files and the two new test files; `vpath %.c` extends to
  `../../components/ramfs ../../components/vfs_simple`.
- Top Makefile: `KTEST_C` += `test/kernel/ktest_vfs.c`; the
  new component `.c` files are discovered automatically through
  the gen-config-emitted `gen/sources.mk`.

## Key Findings

- **`vfs_simple_state` is private to vfs_simple.c** — its host tests
  treat it as opaque (`calloc(desc->state_size)`, cast to
  `unsigned *` to peek at the counter block).  Same technique the
  slice-5.2 mm_buddy tests use.  When tests need lifecycle-counter
  introspection on a state struct the framework hides, this is the
  pattern.
- **`core/lib/lib.h` doesn't export `memcmp`**.  Hit this writing
  `ktest_vfs`.  Inlined a byte loop instead of adding it to the lib
  — memcmp has enough subtle semantics that the first consumer
  shouldn't be a one-off test.
- **`kernel.bin` must be rebuilt separately from `make test`.**  The
  test target rebuilds `kernel-test.bin` only; the live boot image
  needs an explicit `make`.  Found this when the first `tools/run-
  qemu.sh` after the kernel.json change booted the old 3-slot
  composition.  Added the `make` step to the todo list for the
  record.

## Decisions Made

- **Slot dispatch via `descriptor->iface_ops`, not a hand-rolled
  driver table.**  The scheduler already uses this pattern
  (sched_rr → `sched_rr_scheduler_ops` published through the
  descriptor; `sched_init` reads it off the active slot at boot).
  Reusing it for filesystems keeps "how do components publish their
  interface ops?" a single, well-understood answer.  Why this is
  OK: a component's iface is declared in its manifest, so the
  framework already knows what type `descriptor->iface_ops` points
  at per bound slot.  How to apply: any future component that
  exposes a typed ops surface (block_device, net_stack, …) goes
  through this same channel.
- **Vfs_simple caches nothing.**  Paid a few ns per call (slot
  lookup + pointer chase) in exchange for honest late-binding
  semantics.  Why: if vfs_simple cached `filesystem.root`'s
  iface_ops at init-time, a future `nx_slot_swap` of the root
  filesystem would leave vfs_simple holding a stale pointer.  How
  to apply: components consuming other components' ops should
  follow this rule unless profiling shows the lookup cost matters.

## Status at End of Session

- `make test` → **338/338 pass (51 python + 234 host + 53 kernel), 0
  leaks, 0 errors, exit 0**.  +17 host + 6 kernel from slice 6.2.
- `verify-registry` green (ramfs and vfs_simple both
  `pause_hook: false, spawns_threads: false` — no R2/R4 findings).
- `tools/run-qemu.sh -t 3` boots through to the idle loop with the
  5-slot composition log and both new components `state:active`.
- IMPLEMENTATION-GUIDE.md §Slice 6.2 rewritten with the
  as-built details (no `NX_FS_DRIVER_REGISTER`, no `root_fs`
  config knob).

## Next Steps

Slice 6.3: file syscalls + HANDLE_FILE destructor.

- Add `NX_SYS_OPEN / _READ / _WRITE` to `framework/syscall.c`.
  Follow the slice-5.6 channel-syscall pattern: look up the handle,
  `copy_from_user` the path into a kernel buffer, dispatch through
  `nx_slot_lookup("vfs")->active->descriptor->iface_ops` (as
  `struct nx_vfs_ops *`).
- Extend the type-aware `sys_handle_close` with a `HANDLE_FILE`
  branch that calls `vfs_simple_vfs_ops.close` on the object.
- Open assigns `NX_RIGHT_READ` / `NX_RIGHT_WRITE` based on the
  open flags; slice 6.4 adds `NX_RIGHT_SEEK`.
- Baked-in EL0 program opens `/hello` with `CREATE|WRITE`, writes
  `"world"`, closes, reopens READ-only, reads, forwards payload to
  UART as `[el0-file-ok]`.

---

**Files Changed:**
- `sources/nonux/interfaces/vfs.h` — new (`nx_vfs_ops`, flags)
- `sources/nonux/components/ramfs/ramfs.c` — new (driver impl)
- `sources/nonux/components/ramfs/manifest.json` — new
- `sources/nonux/components/ramfs/README.md` — new
- `sources/nonux/components/vfs_simple/vfs_simple.c` — new (dispatch layer)
- `sources/nonux/components/vfs_simple/manifest.json` — new
- `sources/nonux/components/vfs_simple/README.md` — new
- `sources/nonux/kernel.json` — `filesystem.root ← ramfs`, `vfs ← vfs_simple`
- `sources/nonux/test/host/component_ramfs_test.c` — new (10 tests)
- `sources/nonux/test/host/component_vfs_simple_test.c` — new (7 tests)
- `sources/nonux/test/kernel/ktest_vfs.c` — new (6 tests)
- `sources/nonux/test/host/Makefile` — add new sources + vpath
- `sources/nonux/Makefile` — `KTEST_C += ktest_vfs.c`
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 6.2 rewritten with as-built details
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 24 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
