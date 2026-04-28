# Session 35: slice 7.4a — `NX_SYS_FORK`

**Date:** 2026-04-24
**Phase:** 7 slice 7.4a
**Branch:** master

---

## Goals

Implement fork — the trickiest of the fork/exec/wait trio because
of the "two return values from one syscall" semantics.  When a
parent calls `fork()`, control must land back at the same EL0
instruction twice: once in the parent with `x0 = child_pid`, once
in the child (running in its own address space) with `x0 = 0`.
This slice builds the trap-frame replay machinery that makes
that possible.

## Scope choices

- **7.4a only.**  The original §7.4 plan bundled fork + exec +
  wait + posix_shim into one slice.  That's too much for one
  session, and fork's mechanics (trap-frame replay, new kstack
  entry-point thunk, per-task address-space duplication) deserve
  isolated verification before layering exec's address-space
  reset on top.  Sub-sliced: 7.4a (fork, this slice), 7.4b (wait,
  next), 7.4c (exec — also gated on bumping `RAMFS_FILE_CAP` so
  the ELF can live at a ramfs path), 7.4d (posix_shim component).
- **Empty child handle table.**  POSIX fork inherits all open
  file descriptors.  For v1 the simpler "child starts with empty
  handles" is enough for the vast majority of fork+exec use
  cases (child immediately execs, which rebuilds its handles
  anyway).  Real handle-table duplication lands with 7.4c where
  exec has a reason to consult what the child inherited.
- **No COW / page-fault machinery.**  The address-space copy is
  eager — `mmu_copy_user_backing` memcopies the full 2 MiB user
  window.  Copy-on-write would cut fork cost dramatically for
  large programs but requires page-fault handling which v1
  doesn't have (only SVC decodes ESR.EC; other sync exceptions
  halt).  Eager copy is correct and works fine at our current
  2 MiB-per-process scale.

## What Was Done

### `core/mmu/mmu.{h,c}` — `mmu_copy_user_backing`

New helper: byte-copies the 2 MiB user window between two
address spaces with the same I-cache flush discipline the ELF
loader uses (`dsb ish ; ic iallu ; dsb ish ; isb`).  Safe
against NULL / identical source+dest (no-op in those cases).
Host stub is a no-op.

### `framework/process.{h,c}` — `nx_process_fork`

- `nx_process_fork(parent)` returns a fresh child process.
  Internally: `nx_process_create` allocates the struct + fresh
  address space + empty handle table; then
  `mmu_copy_user_backing(parent->ttbr0_root, child->ttbr0_root)`.
- Handle table left empty — documented in the header, with a
  pointer to where duplication will land (7.4c).

### `core/cpu/vectors.S` — `nx_task_fork_child_entry` thunk

A new global label at the end of the file.  Contents:

```asm
nx_task_fork_child_entry:
    RESTORE_TRAPFRAME     ;; existing macro from _sync_stub's tail
    eret
```

When the scheduler first picks a fork-child task, `cpu_switch_to`
restores its callee-saved regs and `ret`s to LR — which `nx_task_
create_forked` populates with this thunk's address.  The thunk
expects SP to already point at a valid trap frame; the forked-
child task is created with exactly that layout.  RESTORE_TRAPFRAME
unpacks the frame into the CPU state, eret delivers control to
EL0 at the parent's fork SVC's follow-up instruction.

Reusing RESTORE_TRAPFRAME (instead of duplicating the macro
body) means the trap-frame layout stays single-sourced — if we
ever change the frame, both `_sync_stub` and the fork thunk
pick up the new shape for free.

### `core/sched/task.{h,c}` — `nx_task_create_forked`

- Allocates a kstack (1 page).
- Reserves `sizeof(struct trap_frame) = 272` bytes at the top
  (aligned to 16).
- Byte-copies `parent_tf` into that reservation; rewrites
  `x[0]` to 0 (the child's fork return value).
- Wires `cpu_ctx.sp = frame_addr`, `cpu_ctx.x30 =
  &nx_task_fork_child_entry`, `cpu_ctx.daif = 0`, all other
  callee-saved regs zeroed (the thunk immediately restores
  from the trap frame so they don't matter).
- Inherits `task->process` from the caller; caller
  (specifically `sys_fork`) overrides it to the forked child
  before enqueueing.

Host stub returns NULL — no trap frame or EL0 on host.

### `framework/syscall.c` — per-call trap-frame stash + `sys_fork`

- New `static const struct trap_frame *g_current_tf` +
  `nx_syscall_current_tf()` accessor.  `nx_syscall_dispatch`
  writes the current tf into the stash before calling a
  handler and clears it after.  This is the bridge between
  "dispatcher sees the trap frame" and "individual handlers
  only receive 6 arg registers".  Single-CPU v1; multi-CPU
  wraps the stash in per-CPU storage without changing the
  accessor's signature.
- `sys_fork`:
  1. `nx_task_current()->process` = parent.
  2. `nx_syscall_current_tf()` = parent's trap frame.
  3. `nx_process_fork(parent)` → child process.
  4. `nx_task_create_forked(name, parent_tf)` → child task.
  5. `child_task->process = child`.
  6. `g_sched_ops->enqueue(sched_self, child_task)`.
  7. Return child's pid (goes into parent's `tf->x[0]`).
  Any allocation failure rolls back cleanly.
- Host: returns `NX_EINVAL` (no MMU, no trap frame, no fork).

### Syscall number + EL0 demo

- `NX_SYS_FORK = 12` added to the enum.
- `test/kernel/user_prog_fork.S`: single `svc #0` with `x8 = 12`,
  then `cbz x0, _fork_child` to split parent vs child branches;
  each emits a distinct marker via `NX_SYS_DEBUG_WRITE` and parks.
- `test/kernel/ktest_fork.c`: creates a parent process, memcopies
  the EL0 program into its user window, spawns a kthread pinned
  to the parent, drops to EL0.  Yields until the debug_write
  counter reaches 2, then dequeues the parent task.  The child
  task (enqueued from within `sys_fork`) remains in its wfe park
  — same convention every other EL0 ktest uses for stranded
  tasks.

## Key Findings

- **Trap-frame replay is the entire mechanism.**  The "magic"
  of fork returning twice is really just: save the parent's
  trap frame twice, modify one copy's `x[0]` to 0, plant that
  copy on a new task's kernel stack with an entry-point thunk
  that erets.  No clever control-flow manipulation, no
  setjmp/longjmp — just careful stack setup.
- **`cpu_ctx.sp` → top-of-kstack-minus-frame works cleanly.**
  The trap-frame layout from `SAVE_TRAPFRAME` matches the
  `struct trap_frame` shape exactly (31 × uint64_t + sp_el0 +
  pc + pstate = 272 bytes).  Placing the frame at
  `kstack_top - 272` and pointing `cpu_ctx.sp` at it lets the
  thunk's `RESTORE_TRAPFRAME` pop the frame with its built-in
  `add sp, sp, #272` — returning the stack to its top, ready
  for the next exception-entry save.
- **I-cache flush after user-backing copy is mandatory.**
  Without `ic iallu` after the copy, the child's fork-return
  path fetched stale instruction bytes (from the parent's
  stale cache lines covering VAs that now resolve via the
  child's TTBR0).  Same issue slice 7.3's ELF loader hit;
  same fix.

## Decisions Made

- **Per-call trap-frame stash, not handler-signature change.**
  Alternative: change every syscall handler signature to take
  `struct trap_frame *tf` instead of 6 `uint64_t` args.  Chose
  the stash because (a) only `sys_fork` actually needs the
  full frame — all other handlers are happy with their 6 arg
  registers, (b) changing every handler's signature is
  mechanical churn for one consumer, (c) the stash pattern
  extends naturally to multi-CPU without touching handlers.
- **`nx_task_create_forked` separate from `nx_task_create`.**
  Alternative: add parameters to the existing `nx_task_create`
  so it can produce either a fresh kthread or a fork replay.
  Chose separate function because the entry-point / stack /
  context semantics differ enough that one function would
  branch on "is this fork?" internally — ugly.  The two
  functions share `alloc_task_struct` / `alloc_kstack` but
  diverge on everything else.  How to apply: any future "new
  task with a different resume style" (e.g., `nx_task_create_
  from_elf_entry`) gets its own creator rather than
  overloading the kthread constructor.
- **Child task enqueued in sys_fork, not by the caller.**
  Alternative: return the child task from sys_fork and let
  the caller enqueue.  Chose internal enqueue because
  sys_fork is THE caller — nothing else in the kernel
  manufactures fork-children.  How to apply: syscall
  handlers that create follow-up tasks enqueue them
  themselves rather than returning them up the stack.

## Status at End of Session

- `make test` → **391/391 pass (51 python + 274 host + 66
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test.
- `make run` boots cleanly.  The new thunk is only exercised
  when `sys_fork` creates a child task; normal boot doesn't
  reach it.
- Seven EL0 markers now visible in the ktest log:
  `[el0] hello` · `[el0-chan-ok]` · `[el0-file-ok]` ·
  `[el0-rdr-ok]` · `[el0-elf-ok]` · `[fork-parent]` ·
  `[fork-child]`.  Each a distinct syscall-family or
  process-mechanism milestone.

## Next Steps

Slice 7.4b — `NX_SYS_WAIT(pid, *status)`.

- Look up target process by pid.
- If `state == EXITED`, copy exit_code to user + return pid.
- If `state != EXITED`, yield and retry.  Bounded yield count
  or wait queue — v1 keeps it as a cooperative poll.
- Test: fork + child calls `NX_SYS_EXIT(42)` + parent calls
  `NX_SYS_WAIT(child_pid, &status)` + asserts status == 42.
  Closes the fork/wait pair sufficient for many shell-style
  use cases even without exec yet.

Slice 7.4c — `NX_SYS_EXEC(path)`.

- Needs `RAMFS_FILE_CAP` bump (the deferred 7.3.5).  An ELF
  binary doesn't fit in 256-byte ramfs files.
- Allocate a fresh address space via slice 7.2's primitives;
  load ELF via slice 7.3's loader; swap new address space in
  via `mmu_switch_address_space` + `process->ttbr0_root =
  new`; free old address space; eret to the ELF's entry.
- Handle table carries over (when 7.4a eventually duplicates
  handles, exec is the place where empty-vs-preserved
  diverges).

---

**Files Changed:**
- `sources/nonux/core/mmu/mmu.h` — add `mmu_copy_user_backing` prototype
- `sources/nonux/core/mmu/mmu.c` — implementation
- `sources/nonux/framework/process.h` — add `nx_process_fork` prototype
- `sources/nonux/framework/process.c` — implementation
- `sources/nonux/core/cpu/vectors.S` — add `nx_task_fork_child_entry` thunk
- `sources/nonux/core/sched/task.h` — add `nx_task_create_forked` prototype
- `sources/nonux/core/sched/task.c` — implementation
- `sources/nonux/framework/syscall.h` — `NX_SYS_FORK = 12`
- `sources/nonux/framework/syscall.c` — `g_current_tf` stash + `sys_fork`
- `sources/nonux/test/kernel/user_prog_fork.S` — new (EL0 fork demo)
- `sources/nonux/test/kernel/ktest_fork.c` — new (kernel test)
- `sources/nonux/Makefile` — `KTEST_S += user_prog_fork.S`; `KTEST_C += ktest_fork.c`
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.4 split into 7.4a–7.4d, 7.4a marked complete
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 30 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
