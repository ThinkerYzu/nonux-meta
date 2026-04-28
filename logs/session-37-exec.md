# Session 37: slice 7.4c — `NX_SYS_EXEC` + ramfs bump + sched_rr quantum drop

**Date:** 2026-04-24
**Phase:** 7 slice 7.4c
**Branch:** master

---

## Goals

Close the fork/exec/wait triad by landing `NX_SYS_EXEC(path)` so a
forked child can replace its address space with a fresh ELF loaded
from the vfs.  Absorb the deferred 7.3.5 ramfs bump (needed so a
static ELF fits) along the way, since the exec test needs a real
ramfs-hosted `/init` to load.

## Scope choices

- **Read-then-parse, not mmap-and-replay.**  sys_exec slurps the
  full ELF into a kheap buffer (capped at `SYS_EXEC_MAX_FILE =
  8192`), then `nx_elf_parse`s it.  A real kernel would map the
  ELF's PT_LOAD segments directly from the page cache; we don't
  have demand paging yet and our ELFs are tiny, so byte-copy is
  simpler and correct.
- **Handle table carries over.**  POSIX `execve` closes handles
  flagged close-on-exec and preserves the rest.  v1 has no
  close-on-exec flag, so all handles carry through.  A later slice
  can plumb the flag into `nx_handle_entry` and have sys_exec walk
  the table.
- **Handle-table duplication in fork stays empty.**  Noted in
  slice 7.4a — child inherits an empty handle table.  That means
  any handle a child uses must be opened after fork.  Not an exec
  concern specifically, but it's the current state that
  constrains any test that goes fork → child opens → exec.  Our
  init_prog opens nothing, so unaffected.
- **Trap-frame clobber, not a separate eret path.**  `sys_exec`
  rewrites the current task's saved trap frame (zeroed x[0..30],
  `pc = entry`, `sp_el0 = top-of-user-window`, `pstate = 0`) so
  the existing `RESTORE_TRAPFRAME + eret` in the sync-exception
  stub delivers control to the new ELF's entry without any
  asm-level work specific to exec.  Same pattern slice 7.4a used
  for fork's child — reuse the exception-return machinery we
  already have.
- **Destroy old address space inline, not lazily.**  After flipping
  TTBR0 to the new root, sys_exec calls
  `mmu_destroy_address_space(old_root)`.  Idle tasks (which may
  still have the old TTBR0 register-loaded because
  `g_kernel_process.ttbr0_root` is 0 and the scheduler's flip
  condition short-circuits on that) are harmless so long as they
  don't dereference user VAs — which kernel code doesn't.  A real
  kernel would defer the destroy until no scheduled context
  references the root, but v1's single-CPU cooperative model
  makes the inline destroy safe enough.
- **sched_rr quantum: 10 ticks → 2 ticks.**  The original 1-second
  quantum (10 × 100 ms) was fine with a handful of cooperating
  tasks but starved the exec test once fork + wait had left their
  forked-child tasks idling in `wfe` on the runqueue.  Each
  stranded task burned a full quantum before the scheduler
  rotated past it, pushing the run past the 15-second QEMU
  timeout.  200 ms per slice is still long enough for the slice
  4.4 preemption demo to visibly advance both tasks, short enough
  for slice-7.4 ktests to cycle through 4+ stranded tasks within
  the budget.
- **Test-harness purge helper, not a general scheduler feature.**
  New `sched_rr_purge_user_tasks(self, keep)` walks the runqueue
  and dequeues every task whose `task->process != &g_kernel_
  process` and whose task != `keep`.  Declared directly in the
  test file (not in sched_rr's public ops surface) because it's
  a policy-specific escape hatch — a proper cross-test reap
  lands when `wait()` collects status and destroys the child's
  process + task.  For now `ktest_exec` calls it before spawning
  its own kthread so the fork-child and wait-child leftovers
  don't starve the exec test's kthread.

## What Was Done

### `framework/syscall.{h,c}` — `NX_SYS_EXEC`

- `NX_SYS_EXEC = 14` appended to the enum.  Header comment
  documents the "noreturn on success, NX_E* on failure"
  contract.
- `sys_exec(a0 = const char *path)`:
  1. `copy_path_from_user` into a 128-byte kernel buffer
     (`NX_PATH_MAX`).
  2. Resolve the `vfs` slot; `vops->open(path, READ)`.
  3. Loop `vops->read` up to `SYS_EXEC_MAX_FILE = 8192` bytes into
     a freshly-`malloc`'d kernel buffer.
  4. `vops->close`.
  5. `nx_elf_parse` for validation.
  6. `mmu_create_address_space()` → new_root.
  7. `current->ttbr0_root = new_root` so
     `nx_elf_load_into_process`'s calls to
     `mmu_address_space_user_backing` resolve the new backing.
     Rollback path restores old_root on failure.
  8. `nx_elf_load_into_process(current, buf, len, &entry)`.
  9. Rewrite the current trap frame via
     `nx_syscall_current_tf()`: zero every x-reg, set `pc =
     entry`, `sp_el0 = top-of-user-window`, `pstate = 0`.
  10. `mmu_switch_address_space(new_root)`.
  11. `mmu_destroy_address_space(old_root)`.
  12. Return `NX_OK`.  Dispatcher writes x[0] = 0, then
      `RESTORE_TRAPFRAME + eret` delivers EL0 at the ELF entry.
- Host stub returns `NX_EINVAL`.
- `nx_syscall_current_tf()` return type flipped from `const
  struct trap_frame *` → `struct trap_frame *` so sys_exec can
  write.  `g_current_tf` storage type matched.

### Absorbed slice 7.3.5

- `components/ramfs/ramfs.c`: `RAMFS_FILE_CAP` raised 256 →
  4096.  Now comfortably fits our ~800-byte init_prog.elf.
- `Makefile`: `init_prog.elf` link rule grows `-n` (`--nmagic`)
  so the phdr table and the single LOAD segment pack directly
  after the header — no 4 KiB page alignment, which had been
  inflating the embedded blob to ~66 KiB (the default page
  alignment on LOAD boundaries).  Post-fix: ~800 bytes.
- `test/kernel/init_prog.S`: appends `NX_SYS_EXIT(17)` after the
  `[el0-elf-ok]` write.  Needed because the exec test in
  `ktest_exec` asks the parent to wait for the exec'd child
  and assert status == 17.  Without the explicit exit, init_prog
  would park in `wfe` forever and the parent's wait would never
  complete.

### Scheduler tuning

- `components/sched_rr/sched_rr.c`: `SCHED_RR_DEFAULT_QUANTUM_
  TICKS` lowered 10 → 2.  Rationale in the inline comment.
- New test-only helper:
  ```c
  void sched_rr_purge_user_tasks(void *self, struct nx_task *keep);
  ```
  Walks the runqueue, dequeues any task whose `task->process !=
  &g_kernel_process` and whose task != `keep`.  Storage is not
  freed — only unlinked from the runqueue.  Imports
  `framework/process.h` for the `g_kernel_process` symbol.

### EL0 + kernel test

- `test/kernel/user_prog_exec.S`:
  - `fork`.
  - Child branch: `exec("/init")` via `NX_SYS_EXEC`; on
    failure falls through to `exit(99)` as a sentinel.
  - Parent branch: save child pid in x19; debug_write(
    "[exec-parent]"); wait(x19, &status_buf); load w2 from
    status_buf; compare to 17 (init_prog's exit code); if equal,
    debug_write("[exec-ok]"); park.
- `test/kernel/ktest_exec.c`:
  - Calls `sched_rr_purge_user_tasks(sself, NULL)` first to
    drain stranded fork-child + wait-child tasks from earlier
    ktests.
  - Seeds ramfs with `/init` by writing the `init_prog.elf`
    blob (embedded via `.incbin` in a new
    `init_prog_blob.S` section) chunk-by-chunk through
    vfs_simple's ops directly — no syscalls, since those need
    a live process context and we're still in the test body.
  - Creates `exec-parent` process, spawns kthread pinned to
    it, drops to EL0.
  - Yields up to 2048 times waiting for debug_write counter ≥
    3 (`[exec-parent]` + `[el0-elf-ok]` + `[exec-ok]`).
  - Independently verifies a process with exit_code == 17
    exists in the process table — the exec'd child.  Searches
    pids > exec_parent_pid so stranded processes from earlier
    tests can't spoof the match.
  - Dequeues the exec parent task; child task stays stranded
    (same convention as earlier EL0 ktests).
- `Makefile`: adds `test/kernel/ktest_exec.c` +
  `test/kernel/user_prog_exec.S` to the KTEST sources.

## Key Findings

- **Quantum-per-task × stranded-count × timer-rate = test time
  budget.**  Each test that forks leaves its child task in the
  runqueue (we have no reap).  The scheduler rotates through the
  runqueue every quantum; at `SCHED_RR_DEFAULT_QUANTUM_TICKS =
  10` (1 second), the slice-7.4c test would have to cycle past
  ~4 stranded EL0-wfe tasks before its own kthread got picked —
  and each `nx_task_yield()` in the main test body blocks until
  main cycles back, which takes `num_stranded × quantum` ≈ 4
  seconds per yield.  Multiplied across the 15-second global
  timeout, the test exhausts its budget before anything useful
  happens.  Drop to 2 ticks (200 ms) and the same test finishes
  in well under 15 s.
- **Markdown numbering vs sequential list semantics.**
  HANDOFF-ARCHIVE.md's inline list renders fine even when the
  source has duplicate "2." entries — CommonMark auto-numbers
  based on position, so the raw digits only need the first item
  to be "1.".  (Still cleaner to keep source-level numbering
  monotonic, but a duplicate isn't a rendering bug — it's just a
  cosmetic source-editor issue.)
- **`nx_syscall_current_tf` const-ness was too strict.**  Until
  7.4c, the accessor returned `const struct trap_frame *` because
  fork only *reads* the parent's frame to clone it.  Exec needs
  to *write* the frame (clobber pc/sp_el0/pstate/regs).  The fix
  is a one-line header change; no other caller relied on the
  const contract.

## Decisions Made

- **Lower sched_rr quantum globally, don't per-test-override.**
  A per-test quantum override would need either (a) a testing
  API to poke into sched_rr's state, or (b) a kernel.json knob
  (once gen-config emits per-component config macros).  Lowering
  the global default is simpler and has no downside — 200 ms is
  still plenty for interactive work, and the slice 4.4
  preemption demo still shows both tasks visibly advancing.
- **Purge stranded tasks at ktest start, not at ktest end.**
  Either order would work.  Purging at start is more defensive:
  every test declares "I don't need anything from prior runs" at
  its top.  Purging at end relies on every test remembering to
  clean up, which is easy to miss.
- **Trust markdown auto-numbering in HANDOFF-ARCHIVE.md.**  The
  duplicate "2." post-insertion renders fine.  Chasing a
  full-file renumber is busywork.  (Previous sessions have done
  the renumber when they noticed — it's a judgment call, not a
  hard requirement.)
- **No `nx_process_create_from_elf(path)` helper.**  The
  original plan (deferred 7.3.5) sketched a helper that would
  open an ELF from a vfs path and load it into a fresh process.
  `sys_exec` ended up absorbing that logic inline — building a
  standalone helper now would just duplicate it.  If a second
  caller shows up (spawn-from-ELF?  live-load?), refactor then.

## Status at End of Session

- `make test` → **393/393 pass (51 python + 274 host + 68
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test.
- `make run` boots cleanly.
- EL0 markers now visible across the run:
  `[el0] hello` · `[el0-chan-ok]` · `[el0-file-ok]` ·
  `[el0-rdr-ok]` · `[el0-elf-ok]` · `[fork-parent]` ·
  `[fork-child]` · `[wait-parent]` · `[wait-child]` ·
  `[wait-ok]` · `[exec-parent]` · `[exec-ok]` (the `[el0-elf-
  ok]` marker now fires twice — once from the slice 7.3 ktest
  and once from init_prog running under the exec'd child).
- Fork + exec + wait + exit form the full Unix triad.  A
  shell written on top of our NX_SYS_* surface could now spawn
  a child from an ELF path, wait for it, and read its exit
  code.

## Next Steps

Slice 7.4d — `components/posix_shim/`.

- POSIX API surface wrapping the NX_SYS_* set.  Wraps `fork /
  waitpid / execve / exit / read / write / open / close / seek /
  readdir` as C APIs callable from user C code.
- Manifest declares the shim as a library-style component (no
  kthreads, no dependencies on real slots — just an ABI layer).
- Kernel is unchanged.  Purely a user-facing packaging slice.
- Unblocks 7.5 (pipes + signals need a POSIX-ish surface to
  bolt onto) and 7.6 (busybox cross-compile needs the POSIX
  wrapper).

Follow-ups that don't block 7.4d:

- Proper cross-test task reap.  The `sched_rr_purge_user_tasks`
  helper in `ktest_exec` is a band-aid.  Real reap: when
  `wait()` collects status, destroy the child's process and
  free its task.  Falls out naturally when slice 7.5/7.6 need
  more concurrent processes than the 16-slot table holds.
- Per-test quantum override.  The `SCHED_RR_DEFAULT_QUANTUM_
  TICKS = 2` lowering is a project-wide default change.  If a
  future test case wants a longer or shorter slice for its
  own cycle-accurate checks, a kernel.json knob (once
  gen-config emits per-component config macros) would let it
  tune per-build.

---

**Files Changed:**
- `sources/nonux/framework/syscall.h` — `NX_SYS_EXEC = 14`
- `sources/nonux/framework/syscall.c` — `sys_exec` + dispatch entry; `nx_syscall_current_tf` drops const
- `sources/nonux/components/ramfs/ramfs.c` — `RAMFS_FILE_CAP` 256 → 4096
- `sources/nonux/components/sched_rr/sched_rr.c` — quantum 10 → 2; `sched_rr_purge_user_tasks`
- `sources/nonux/test/kernel/init_prog.S` — appends `NX_SYS_EXIT(17)`
- `sources/nonux/test/kernel/user_prog_exec.S` — new (EL0 fork+exec+wait demo)
- `sources/nonux/test/kernel/ktest_exec.c` — new (kernel test, incl. init_prog seed)
- `sources/nonux/Makefile` — `init_prog.elf` link adds `-n`; `KTEST_S += user_prog_exec.S`; `KTEST_C += ktest_exec.c`
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.4c filled in with as-built details
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 32 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
