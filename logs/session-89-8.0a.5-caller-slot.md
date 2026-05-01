# Session 89: slice 8.0a.5 — per-task `caller_slot` lifecycle + edge inheritance

**Date:** 2026-05-01
**Phase:** Phase 8 — Group B (slice 8.0a, sub-slice 5 of 8)
**Branch:** master (source-side commit `0917023` in `sources/nonux/`; this session's checkpoint is the proj_docs paperwork that records it)

---

## Goals

- Land slice **8.0a.5** — promote every task to a slot-bearing graph
  entity by embedding a `caller_slot` in `struct nx_task`.  The slot is
  registered at `nx_task_create`, bound to the singleton `posix_shim`
  component, and walks posix_shim's outgoing edges to register parallel
  `(caller_slot → svc_slot)` connections that inherit `mode/stateful/
  policy` verbatim.  `nx_task_destroy` unwires in reverse order.
- Plumb the per-task blocking-call carrier alongside the slot:
  `reply_waitq`, `in_flight_reply_buf`, `in_flight_reply_buf_len`,
  `in_flight_reply_rc`.  Slice 8.0a.6 will populate them when
  `nx_slot_call_blocking` body lands.
- Keep host coverage green.  `task_test.c` runs before
  `nx_framework_bootstrap`, so the kernel registry is empty when those
  tests fire — `wire_caller_slot` must soft-skip gracefully when
  posix_shim isn't bound rather than fail every existing task test.
- Add ~10 ktests to pin down the per-task slot lifecycle (host: skip
  vs. wire, edge clone, multi-task independence, partial-fixture
  rollback, `reply_waitq` init; kernel: live-composition round-trip).

## What Was Done

### Slice 8.0a.5 (source-side commit `0917023`, `sources/nonux/`)

Pure additive lifecycle plumbing.  No production callers consume the
new slot yet — `framework/syscall.c` still uses direct `vops->read(self,
…)` calls; that migration lands in slice 8.0c.  The slot/edge graph
just gets bigger by one slot + four edges per task.

**`core/sched/task.h` (+46 lines).**  Headers grow from forward decls
to full includes (`framework/registry.h` and `core/sched/waitq.h`) so
`struct nx_slot` and `struct nx_waitq` can be embedded by value.  Six
new fields on `struct nx_task` per [SLOT-CALL-API.md](../SLOT-CALL-API.md)
§"Per-Task `caller_slot`":

```c
struct nx_slot      caller_slot;
char                caller_slot_name[NX_TASK_SLOT_NAME_MAX];
bool                caller_slot_active;

struct nx_waitq     reply_waitq;
void               *in_flight_reply_buf;
size_t              in_flight_reply_buf_len;
int                 in_flight_reply_rc;
```

`NX_TASK_SLOT_NAME_MAX = 24` covers `task#` (5) + `uint32_t` decimal (10)
+ NUL with comfortable headroom.  The slot's `const char *name` points
at the embedded buffer so the registry's caller-owned-name discipline
holds across the slot's whole lifetime.

`caller_slot_active` is the destroy-side gate: tasks that never wired
(host tests with no posix_shim, the static `g_idle_task`,
`nx_task_create_forked` failure paths) skip the unwire entirely so
`nx_task_destroy` is safe regardless of registry state.

**`core/sched/task.c` (+173 lines).**  Three additions:

1. **`render_caller_slot_name`** — synthesizes `task#<id>` into the
   embedded buffer.  Decimal render is hand-rolled (no libc on the
   freestanding build); truncation is benign because uniqueness is
   the only contract.

2. **`wire_caller_slot` / `unwire_caller_slot`** — the lifecycle pair.
   `wire` does a `nx_slot_lookup("posix_shim")` first; if the slot is
   absent or unbound (`active == NULL`), returns `NX_OK` without
   touching anything (the soft-skip).  Otherwise it sets the slot's
   metadata (`name`, `iface = "task"`, `mutability = NX_MUT_HOT`,
   `concurrency = NX_CONC_SHARED`), calls `nx_slot_register` (which
   `atomic_init`s `pause_state` / `in_flight_calls` and `nx_waitq_init`s
   `resume_waitq`), `nx_slot_swap`s to bind posix_shim's component, then
   walks posix_shim's outgoing edges via `nx_slot_foreach_dependency`
   and `nx_connection_register`s a parallel edge per dep with the same
   `mode/stateful/policy`.  On any failure during cloning, rolls back —
   walks the partial outgoing list and unregisters each, unbinds, and
   unregisters the slot.

   `unwire` re-fetches the head of the outgoing list every iteration
   (it's mutated by each `nx_connection_unregister`), then unbinds and
   unregisters the slot.

3. **Integration into `nx_task_create` and `nx_task_create_forked`** —
   both call `wire_caller_slot` once the rest of the task is set up
   (id assigned, name copied, cpu_ctx written, process inherited,
   `tpidr_el0` set).  On failure, both free the kstack + struct and
   return `NULL`.  Both also `nx_waitq_init(&t->reply_waitq)` and
   zero the in-flight reply trio inline next to the existing field
   inits.

   `nx_task_destroy` calls `unwire_caller_slot` first (no-op if
   `caller_slot_active == false`), so the registry never holds a
   dangling slot pointer once the task struct is freed.

**`test/host/task_test.c` (+276 lines).**  Six new TESTs cover the
caller_slot surface end-to-end against a synthesized posix_shim
fixture (a slot bound to a component plus 4 dep slots and 4
`posix_shim → svc` edges with manifest-matching `mode/stateful/policy`):

| Test | Coverage |
|---|---|
| `caller_slot_skipped_when_posix_shim_absent` | Empty registry → soft-skip; `caller_slot_active == false`; slot/conn counts unchanged across create+destroy. |
| `caller_slot_registers_and_clones_edges_when_posix_shim_bound` | Fixture present → `caller_slot_active == true`; `+1` slot, `+POSIX_SHIM_DEP_COUNT` (=4) edges; bound to fixture's posix_shim component; lookup-by-`task#<id>` finds the embedded slot; destroy returns counts. |
| `caller_slot_inherits_edge_attributes` | Captures parent/child outgoing edges; pairs them by `to_slot`; asserts `mode/stateful/policy` match verbatim and `from_slot` is the per-task slot. |
| `caller_slot_multiple_tasks_each_get_independent_slot_and_edges` | Three concurrent tasks → 3 slots + 12 edges; names distinct; reverse-order destroy unwinds in 4-edge chunks. |
| `caller_slot_destroy_after_partial_fixture_no_registry_leak` | posix_shim bound but with zero outgoing edges (rare bring-up window) — clone-edges loop is a no-op, but slot still registers/unregisters cleanly. |
| `caller_slot_reply_waitq_initialized_for_blocking_call_path` | `reply_waitq.waiters.n` self-linked (init'd); `in_flight_reply_buf == NULL`, `_len == 0`, `_rc == 0`. |

The five existing `task_*` tests gained `nx_graph_reset()` at the top
of each TEST body.  Pre-existing component-destroy tests deliberately
register stack-allocated `struct nx_slot` instances and never clean up
(stack frames die, registry nodes still point at them); previously
this was invisible because no test consumed the registry across a
task_test boundary, but `wire_caller_slot` does an `nx_slot_lookup`
which traverses every slot node and `strcmp`s the dangling `name`
field.  The targeted fix (graph_reset per task test) is the minimal
blast radius — broader cleanup of the destroy tests is out of scope.

**`test/kernel/ktest_bootstrap.c` (+66 lines).**  Two changes:

1. **`bootstrap_snapshot_json_contains_bound_impl` comment** — extended
   to note that slice 8.0a.5 adds per-task slots on top of the 7-slot
   composition; the 4096-byte buffer (bumped from 2048 in slice 8.0a.4)
   still fits.  No semantic change to the assertion.

2. **`bootstrap_caller_slot_create_destroy_round_trip_in_kernel`** — new
   ktest; mirrors `caller_slot_registers_and_clones_edges_when_posix_shim_bound`
   on the live kernel composition.  Creates a fresh kthread (entry =
   `ktest_dummy_entry`, never scheduled), asserts `caller_slot_active`
   true, slot count grew by 1, conn count grew by 4, `caller_slot.active`
   resolves to the `posix_shim` component, lookup-by-name returns the
   embedded slot, all 4 outgoing edges originate from that slot.
   Destroy returns both counts.  This is the only ktest needed for
   8.0a.5 — host coverage already pins down the matrix.

   An earlier draft asserted against `nx_task_current()` to verify the
   dispatcher kthread was wired at boot; pulled it because `ktest_main`
   resets TPIDR_EL1 to `&g_idle_task` before each test, and `g_idle_task`
   is statically allocated → never went through `nx_task_create` →
   `caller_slot_active == false` by definition.  The fresh-kthread
   path tests the same wire logic without depending on bootstrap order.

### Key decisions

- **Soft-skip vs. hard-fail when posix_shim is absent.**  Chose
  soft-skip.  Production builds always have posix_shim bound (kernel.json
  + slice 8.0a.4 wired it); host unit tests don't run framework_bootstrap
  and have a clean registry.  Hard-failing would have forced a
  fixture-init prologue into every task_test.c TEST and into
  conformance_scheduler_test.c (although that file doesn't actually call
  `nx_task_create` in v1, only references it in a comment).  Soft-skip
  also mirrors how `framework/process.c` handles the kernel process
  before bootstrap.

- **Embed `struct nx_slot` and `struct nx_waitq` by value.**
  SLOT-CALL-API.md §"Per-Task `caller_slot` — Data structure" specifies
  embedded; the alternative (heap-allocated slot, pointer in nx_task)
  would have been clean for forward decls but incurs an extra alloc on
  every fork/exec.  Embedding adds ~250 bytes per `struct nx_task`,
  which is negligible against the 4 KiB kstack.

- **`task#<id>` slot name format.**  Matches SLOT-CALL-API.md's prose;
  the embedded buffer is sized for `uint32_t` so a future bump of
  `g_task_id_seq` to `uint64_t` would need to grow `NX_TASK_SLOT_NAME_MAX`.

- **Where to call `wire_caller_slot` in `nx_task_create_forked`.**  After
  `tpidr_el0` is mirrored from the parent — fork inherits TLS; everything
  graph-related is independent of the parent.

- **`nx_graph_reset()` in `task_test.c`** instead of broader cleanup.
  See above; targeted fix for a pre-existing test-isolation bug that
  only became visible because slice 8.0a.5 reads the registry from
  `nx_task_create`.

### Tests at end of Session 89

- `make test` → **502/502 pass** (93 python + **289** host (was 283; +6) +
  **120** kernel (was 119; +1)).
- `make test-interactive` → **7/7 pass**.
- `make verify-iface-fresh` → 0 drift.
- `make verify-registry` → 0 findings.

## Next Steps

Slice **8.0a.6** — reply path end-to-end.  The big remaining piece of
the blocking-call infra:

- Per-CPU reply-message pool sized 256, matching the existing ISR
  message-pool discipline; assertion on exhaustion.
- Dispatcher integration: increment/decrement `slot->in_flight_calls`
  around `handle_msg`; on `NX_MSG_FLAG_REPLY_REQUESTED`, post a reply
  message back to the caller's `caller_slot` via `posix_shim_handle_msg`.
- Fill in `nx_slot_call_blocking` body: validate args + assert no
  recursive call, read pause_state, set up reply buf, fire
  `NX_HOOK_IPC_SEND` chain, scan caps, enqueue, block on
  `task->reply_waitq`, clear reply buf on wake, return
  `task->in_flight_reply_rc`.  ABORT path synthesizes a reply with
  `rc = NX_EABORT` so the caller wakes cleanly.

After 8.0a.6, the first end-to-end blocking call works in a ktest;
remaining slices in Group B activate generated `handle_msg` shims on
all 5 production components (8.0b), migrate framework production paths
(8.0c), migrate component-to-component calls (8.0d), and add the
`verify-registry` rule banning direct ops access (8.0e).

---

**Files Changed (`sources/nonux/`, commit `0917023`):**
- `core/sched/task.h` — embedded caller_slot + reply_waitq + in-flight reply trio + slot-name buffer.
- `core/sched/task.c` — wire_caller_slot / unwire_caller_slot helpers + integration into create / create_forked / destroy.
- `test/host/task_test.c` — `nx_graph_reset()` per existing test + 6 new TESTs covering the caller_slot surface.
- `test/kernel/ktest_bootstrap.c` — new ktest for live-composition round-trip + comment note on snapshot test for slice 8.0a.5.

**Files Changed (`proj_docs/nonux/`, this commit):**
- `logs/session-89-8.0a.5-caller-slot.md` — this log.
- `HANDOFF.md` — Current Status / Phase checklist / Next Actions / Session Logs advanced; Session 84 archived.
- `IMPLEMENTATION-GUIDE.md` — slice 8.0a.5 row marked ✓; Phase 8 status + Last Updated footer refreshed.
- `HANDOFF-ARCHIVE.md` — Session 84 entry pulled forward from HANDOFF per the keep-last-5 convention.
