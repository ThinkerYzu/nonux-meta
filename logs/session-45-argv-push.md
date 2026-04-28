# Session 45: slice 7.6c.4 — sys_exec argv push

**Date:** 2026-04-25
**Phase:** 7 slice 7.6c.4 (inserted ahead of 7.6c.3 musl pin)
**Branch:** master

---

## Goals

The slice 7.6c.0 crt0 had to fake argv = `{ "nonux", NULL }` because
sys_exec ignored the user-supplied argv pointer.  busybox + any
real shell needs the kernel to forward argv across exec, so close
that gap before the musl pin lands.  This is a discrete kernel-ABI
extension — clear scope, single-session-shaped.

## Scope choices

- **System V layout on the user stack, not a separate argv buffer.**
  Could have allocated kernel-side memory for argv and exposed it
  via a per-task pointer.  But every existing libc (musl,
  glibc, picolibc, etc.) reads argc/argv from `[sp]` per the
  System V ABI.  Using the same layout means crt0 + the eventual
  musl pin both work without translation.
- **Cap argc at 16, argv strings at 1024 bytes.**  Stack-local
  staging buffer in `sys_exec` makes the limits configurable but
  keeps allocation off the kheap.  Linux's MAX_ARG_STRLEN is
  much larger (128 KiB) but v1 doesn't need the headroom — even
  busybox's invocations rarely exceed a dozen short args.
- **Copy argv BEFORE the TTBR0 flip; lay out AFTER.**  Two
  separate operations:
  1. Pre-flip: argv pointers and strings live in the OLD user
     address space.  copy_from_user reads them (with the existing
     bounds check against `mmu_user_window_*`).
  2. Post-flip: write the System V layout into the NEW user
     backing through the kernel-visible alias from
     `mmu_address_space_user_backing`.  Same trick the ELF
     loader uses for PT_LOAD copies.
  Mixing the two would mean either accessing OLD-AS user memory
  after the flip (impossible) or accessing the NEW backing via
  user VAs before the flip (impossible).
- **crt0 reads argc from `[sp]` with argc==0 fallback.**  For
  drop_to_el0 entries (no exec), the user backing is calloc'd,
  so `[sp]` reads as 0 — that's the cue to use the synthesized
  default.  For sys_exec entries, sys_exec ensures argc > 0 (at
  worst synthesizes `{ path, NULL }` if user passed NULL), so
  the stack-read path is always correct.
- **Bump v1 caps as needed; defer real reap.**  Cumulative test
  usage of stranded processes hit `NX_PROCESS_TABLE_CAPACITY` =
  16.  Real fix is reaping on `wait()`, but that's been deferred
  since slice 7.4 because the test convention deliberately
  strands tasks (cross-test stable state).  Bumping the cap to
  32 buys us another dozen tests of headroom; the proper fix
  lands when slice 7.6d busybox's exit-heavy workload makes the
  current convention untenable.

## What Was Done

### `framework/syscall.{h,c}` — NX_SYS_EXEC ABI extension

- Header comment for `NX_SYS_EXEC = 14` rewritten to document the
  new (path, argv, envp) signature + the System V layout that
  sys_exec builds on the user stack.
- `sys_exec` body — added two new sections:

  1. **Pre-flip argv copy.**  Walks `user_argv[0..]`:
     - copy_from_user(&user_str_ptr, user_argv + i*8, 8) — read
       the i'th pointer from the user array.
     - If NULL, terminate.
     - copy_path_from_user(staging + offset, remain,
       user_str_ptr) — copy the string body into kernel staging.
     - Track per-string offsets in `argv_offsets[]`.
     Caps: 16 entries, 1024 bytes total of string data.

  2. **Synthesised argv = { path, NULL } when user passes NULL.**
     Matches Linux convention.  Programs that don't supply argv
     still see argv[0] == path in their main.

  3. **Post-flip layout build.**  After the TTBR0 flip:
     - Strings copied to `backing + (window_size - argv_str_len)
       & ~7` (high end, 8-aligned).
     - argc + argv pointer array + envp NULL placed below the
       strings, 16-aligned for AAPCS.
     - `tf->sp_el0` = window_base + sp_offset (the argc slot's
       user VA).

### `components/posix_shim/posix.h`

- `nx_posix_execve` now passes argv as the second SVC arg.  envp
  is the third (reserved).
- Header comment notes the slice 7.6c.4 contract.

### `components/posix_shim/crt0.S`

- `_start` reads argc from `[sp]` first.  Two paths:
  - Stack-pushed (`cbnz x0, ...`) → x1 = sp+8 (argv start),
    x2 = sp+16+8*argc (envp start).
  - Synthesised (`cbz x0, _synth_argv`) → existing fallback
    that loads `_argv_storage` from `.rodata`.
- Both paths converge at `_call_main` → `bl main`.

### v1 cap bumps

- `framework/process.c`: `NX_PROCESS_TABLE_CAPACITY` 16 → 32.
- `core/mmu/mmu.c`: `MMU_MAX_ADDRESS_SPACES` 16 → 32.
- `components/ramfs/ramfs.c`: `RAMFS_FILE_CAP` 4096 → 8192.

Each bump documented in a comment with the reason.

### Stack-boundary fix (15 ktest_*.c callers)

Every drop_to_el0 caller used to set `sp_el0 = (base + size) &
~0xf` — top-of-window (exclusive).  crt0's new `ldr x0, [sp]`
faults on that boundary.  Bulk-updated via sed:

```sh
sed -i 's|sp_el0 = (base + size) & ~|sp_el0 = (base + size - 16u) \& ~|' \
    test/kernel/ktest_*.c
```

15 files updated.  Now sp_el0 = top-16, well within the window;
crt0 reads `[sp]` as 0 (zero-initialized backing) and falls
back to the synthesized default.

### Demo `argv_child_prog.c` + `argv_parent_prog.c`

Two new C-compiled EL0 binaries:

- **`argv_child_prog.c`** — exec'd target.  Uses `int main(int
  argc, char **argv, char **envp)`, prints argc + each argv slot
  via printf, validates each slot byte-for-byte against
  `{ "/argv_child", "hello", "world" }`, exits with `argc + 60`
  (success → 63).  Six distinct error sentinel codes (81..85)
  pinpoint which slot regressed.  Embedded in initramfs as
  `/argv_child` (alongside `/init` and `/banner`).
- **`argv_parent_prog.c`** — drop-to-EL0 entry point.  fork()s;
  child execve's `/argv_child` with the explicit argv; parent
  emits `[argv-parent]`, waits, asserts status == 63, emits
  `[argv-ok]`.

### `test/kernel/ktest_argv_push.c`

Loads `argv_parent_prog.elf`, drops to EL0, asserts ≥ 7
debug_writes (parent + child markers) + an EXITED process with
exit_code == 63.

### Initramfs

- New entry `/argv_child` mapped to `argv_child_prog.elf`.
- Initramfs grew from 2 entries to 3.

## Key Findings

- **Off-by-one in the user-stack `fixed_size` formula.**  First
  pass had `fixed_size = 16 + 8*argc`.  Should be `8*(3+argc)`
  — slot count is `argc` (argv body) + 1 (argv NULL terminator)
  + 1 (envp NULL) + 1 (argc cell) = `argc + 3`.  The bug
  caused the argv pointer area to overlap the string area's
  first 8 bytes, so argv[0] read as zeros while argv[1+] read
  correctly.  Surfaced via a debug `[argv0 bytes=...]` print
  that showed all zeros despite argv[0] pointing at the right
  address.
- **drop_to_el0 callers' sp_el0 was AT the boundary, not below.**
  Pre-slice-7.6c.4 crt0 didn't read `[sp]`, so the boundary
  bug went undetected.  My new `ldr x0, [sp]` instantly
  faulted with ESR=9200000e (data abort, lower EL,
  permission-fault) at FAR=0x48200000 (one past the user
  window).  Easy fix once diagnosed; sed-bulk-updated 15
  files.
- **MMU_MAX_ADDRESS_SPACES is a separate cap from
  NX_PROCESS_TABLE_CAPACITY.**  First cap-bump-attempt missed
  this — bumped only the process-table cap to 32, but MMU
  topped out at 16 and `mmu_create_address_space` returned 0,
  causing `nx_process_create` to fail.  Both caps need to
  match for any process-allocation path to work.
- **RAMFS_FILE_CAP needs to fit a real C-compiled program.**
  argv_child_prog.elf is 4568 bytes (libnxlibc + main +
  printf).  RAMFS_FILE_CAP was 4096 from slice 7.4c, sized
  for init_prog.elf (~800 bytes).  My ramfs slurp silently
  skips entries that exceed the cap, so /argv_child wasn't
  in the live FS — sys_exec returned NX_ENOENT, child took
  the exit(99) fallback, parent's wait saw status != 63.
  Bumped to 8192.

## Decisions Made

- **Insert this as 7.6c.4, not 7.6c.3.**  Slice 7.6c.3 is the
  musl pin — a multi-day integration task that's been deferred
  for good reason.  argv push is a discrete kernel ABI
  extension that takes a single session and is independently
  useful.  Renumbering preserves the original "musl is 7.6c.3"
  promise while documenting the actual landing order.
- **`int main(int argc, char **argv, char **envp)` in the
  demo, not `(int argc, char **argv)`.**  Three-arg form
  matches POSIX exactly + lets us probe envp's NULL pass-
  through (no envp support yet, but the slot is reserved
  for slice 7.6c.3 musl).
- **Bump caps to 32 not 64.**  32 gives ~14 tests of headroom
  (currently 18 user-process-creating tests).  64 would
  postpone the proper-reap fix indefinitely; 32 keeps it as
  an actionable next step.
- **Roll Session 40 to archive.**  HANDOFF.md's session list
  caps at 5; after inserting Session 45, Session 40 (slice
  7.6a fork handle inheritance) moves to the archive.

## Status at End of Session

- `make test` → **403/403 pass (51 python + 274 host + 78
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test.
- `make run` boots cleanly.
- 27 distinct EL0 markers across the test suite (now adds
  `[argv-parent]`, `[argv-child argc=N]`, `[argv-child
  argv[i]=...]`, `[argv-child-ok]`, `[argv-ok]`).
- The C-runtime story is now end-to-end: a freshly-exec'd C
  program receives real argc/argv from the kernel, exactly
  the way busybox's main expects.

## Next Steps

**Slice 7.6c.3 — vendor + cross-build musl** (next).  Plan
unchanged: vendored snapshot, patched syscall_arch.h, swap
libnxlibc for libc.a on the demo link lines.  argv push being
in place means musl's `__libc_start_main` will read argc/argv
from `[sp]` exactly as it expects — no kernel-side change for
musl.

**Deferred (unchanged):**

- Proper cross-test task reap on `wait()`.  Now visibly hitting
  caps; should land before slice 7.6d which will create many
  more processes.
- HANDLE_FILE fork inheritance.

---

**Files Changed:**
- `sources/nonux/framework/syscall.h` — NX_SYS_EXEC ABI doc updated
- `sources/nonux/framework/syscall.c` — argv copy + System V layout build
- `sources/nonux/framework/process.c` — NX_PROCESS_TABLE_CAPACITY 16 → 32
- `sources/nonux/core/mmu/mmu.c` — MMU_MAX_ADDRESS_SPACES 16 → 32
- `sources/nonux/components/ramfs/ramfs.c` — RAMFS_FILE_CAP 4096 → 8192
- `sources/nonux/components/posix_shim/posix.h` — nx_posix_execve passes argv
- `sources/nonux/components/posix_shim/crt0.S` — read argc from [sp] with argc==0 fallback
- `sources/nonux/test/kernel/ktest_*.c` — 15 callers: sp_el0 = top-16
- `sources/nonux/test/kernel/argv_child_prog.c` — new (exec'd child, validates argv)
- `sources/nonux/test/kernel/argv_parent_prog.c` — new (forks + execs child)
- `sources/nonux/test/kernel/argv_parent_prog_blob.S` — new (.incbin wrapper)
- `sources/nonux/test/kernel/ktest_argv_push.c` — new (kernel test)
- `sources/nonux/Makefile` — argv_child_prog + argv_parent_prog rules; initramfs entry; KTEST_C/S; clean
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.6c.4 inserted; numbering updated
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 40 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
