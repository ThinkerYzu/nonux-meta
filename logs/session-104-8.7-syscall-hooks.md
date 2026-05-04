# Session 104: Slice 8.7 — NX_HOOK_SYSCALL_ENTER / _EXIT Hook Points

**Date:** 2026-05-04
**Phase:** 8 (Runtime Recomposition + Config Manager)
**Branch:** master

---

## Goals

- Implement `NX_HOOK_SYSCALL_ENTER` and `NX_HOOK_SYSCALL_EXIT` hook points, closing slice 8.7 and Phase 8.

## What Was Done

### Hook infrastructure extension (framework/hook.h)

Added two new enum values before `NX_HOOK_POINT_COUNT`:
- `NX_HOOK_SYSCALL_ENTER` (8) — fires from `nx_syscall_dispatch` immediately before the syscall body
- `NX_HOOK_SYSCALL_EXIT` (9) — fires immediately after the syscall body

Added a `sc` arm to `struct nx_hook_context.u`:
```c
struct {
    uint64_t          num;   /* x8 — syscall number */
    uint64_t          a[6];  /* snapshot of x0..x5 at SVC entry */
    int64_t          *rc;    /* &local rc — aliases nx_status_t */
    struct trap_frame *tf;
} sc;
```

Added forward declaration for `struct trap_frame` (alongside existing `struct nx_task` forward decl).

`rc` is a pointer into the dispatch-local accumulator: ENTER hooks may set `*rc` then return `NX_HOOK_ABORT` to skip the syscall body entirely; EXIT hooks may overwrite `*rc` to change what EL0 sees in x0.

### Dispatch integration (framework/syscall.c)

Added `#include "framework/hook.h"`.

Rewrote `nx_syscall_dispatch()`:
- Initialises `rc = NX_OK` (was uninitialised before hook dispatch)
- Builds a `struct nx_hook_context` and calls `nx_hook_dispatch(ENTER)`
- If ENTER returns `NX_HOOK_ABORT`, skips the syscall table lookup entirely; `*rc` is whatever the ENTER hook left
- Flips `ctx.point` to `NX_HOOK_SYSCALL_EXIT` and calls `nx_hook_dispatch(EXIT)` after the body
- EXIT hooks may overwrite `*ctx.u.sc.rc` to change the EL0 return value

### Tests (test/kernel/ktest_syscall.c)

Added `#include "framework/hook.h"` and 5 new `KTEST` cases:

1. `syscall_hook_enter_fires` — verifies ENTER hook sees correct `num` and `a[0]`
2. `syscall_hook_exit_fires` — verifies EXIT hook sees the body's return value in `*rc`
3. `syscall_hook_enter_abort_skips_body` — ENTER returns ABORT with `*rc = NX_ENOENT`; verifies body doesn't run (debug_write counter stays flat) and EL0 gets `NX_ENOENT`
4. `syscall_hook_exit_can_override_result` — EXIT hook sets `*rc = 0x7EEF`; verifies EL0 sees `0x7EEF`
5. `syscall_hook_enter_and_exit_both_fire` — both hooks registered; both fire on a single SVC

All hooks use `nx_hook_unregister()` for teardown (no `nx_hook_reset()` which would clobber other chains).

## Key Findings

- **Stale-object problem (same as Session 100):** `NX_HOOK_POINT_COUNT` increased from 8 to 10. The host build's `component_hook_test.o` was compiled before the change — it encoded `NX_HOOK_POINT_COUNT = 8` into the `bad_point.point` field. After the enum change, value 8 became `NX_HOOK_SYSCALL_ENTER` (a valid point), so `nx_hook_register` accepted it instead of rejecting it. Fix: `make clean` when header enums change. The kernel build also needed `touch framework/hook.c` to force recompilation of hook.o (which sizes `g_chains[]` by `NX_HOOK_POINT_COUNT`).
- **`int64_t *rc` alias:** The `sc.rc` field uses `int64_t *` rather than `nx_status_t *` to avoid adding `framework/syscall.h` as a dependency of `framework/hook.h`. The cast `(int64_t *)&rc` in `syscall.c` is safe — `nx_status_t` is `typedef int64_t`.

## Status at End of Session

- `make test-tools` → **102/102**
- `make test-host` → **459/459** (after `make clean` to flush stale object)
- `make test-kernel` → **141/141** (+5 new ENTER/EXIT hook tests)
- `make verify-iface-fresh` → 0 drift
- `make verify-registry` → 0 findings (R2, R4, R9)

**Phase 8 is closed.** All 15 slices (Groups A, B, C) landed.

## Next Steps

- Phase 9: per-process memory management rework (L3 4 KiB pages, VMAs, demand paging, COW fork) — see IMPLEMENTATION-GUIDE.md §Phase 9.

---

**Files Changed:**
- `framework/hook.h` — added `NX_HOOK_SYSCALL_ENTER`, `NX_HOOK_SYSCALL_EXIT` enum values; `sc` union arm; `struct trap_frame` forward decl
- `framework/syscall.c` — added `#include "framework/hook.h"`; rewrote `nx_syscall_dispatch` to fire ENTER/EXIT hooks
- `test/kernel/ktest_syscall.c` — added `#include "framework/hook.h"`; 5 new KTEST cases for slice 8.7
