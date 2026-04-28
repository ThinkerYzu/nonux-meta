# Session 17: slice 4.2 — scheduler interface + host conformance suite

**Date:** 2026-04-21
**Phase:** 4 — Context Switch and Scheduler
**Branch:** master

---

## Goals

- Define `struct nx_scheduler_ops` in a proper `interfaces/` header with
  the per-op error-code contract that every scheduler policy is
  expected to honour.
- Ship the host-side conformance harness that any scheduler impl must
  pass before it's allowed to bind to the `scheduler` slot.
- Prove the harness itself works end-to-end by running every case
  against a trivially-correct in-file FIFO fixture (`sched_fifo`).

No scheduler component yet — that's slice 4.3.

## What Was Done

### `interfaces/scheduler.h`

First real file in the (previously empty) `interfaces/` directory.
`struct nx_scheduler_ops` with six fn pointers:

- `pick_next(self) -> struct nx_task *` — idempotent, side-effect-free,
  returns borrow.
- `enqueue(self, task)` — `NX_OK` / `NX_EEXIST` / `NX_EINVAL`.
- `dequeue(self, task)` — `NX_OK` / `NX_ENOENT` / `NX_EINVAL`.
- `yield(self)` — voluntary quantum surrender (wired to the reschedule
  shim in 4.4).
- `set_priority(self, task, prio)` — `NX_OK` if the policy implements
  priorities, `NX_EINVAL` uniformly if it doesn't. Consistent-across-
  tasks requirement is an explicit contract.
- `tick(self)` — timer-ISR path hook, bounded, non-blocking.

Every task pointer is `borrow`: the scheduler may keep the pointer on
its runqueue but never owns task storage. `nx_task_create` /
`nx_task_destroy` remain the sole owners.

### `test/host/conformance/conformance_scheduler.{h,c}`

New subdirectory `test/host/conformance/` (first occupant). Seven
universal cases, each exposed as its own function so callers can wrap
each in a TEST() and get per-case pass/fail reporting through the
host test framework's ASSERT machinery:

1. `pick_on_empty_returns_null`
2. `enqueue_then_pick_returns_task`
3. `dequeue_removes`
4. `dequeue_nonexistent_returns_enoent`
5. `set_priority_returns_consistent_status`
6. `roundtrip_100_residue_free` (100-task enqueue/dequeue → pick-empty;
   mem_track catches any leak)
7. `borrow_preserves_task_pointer` (scheduler must not mutate
   caller-visible task fields beyond `sched_node`)

The cases use minimal stack-allocated `struct nx_task` instances —
`nx_task_create`'s kstack + id-seq machinery isn't interesting to the
interface contract, so we skip it and focus on the op table.

Policy-specific invariants (FIFO order for round-robin, priority
inversion handling, etc.) are deliberately **not** in conformance —
those belong in each component's own test file.

### `test/host/conformance_scheduler_test.c`

Two roles:

1. **Smoke-test the harness itself.** Runs every conformance case
   against an in-file `sched_fifo` fixture — the world's dumbest
   correct scheduler (intrusive-list FIFO, no priorities). All 7 cases
   pass.
2. **Worked example for slice 4.3.** `components/sched_rr/` will have
   the same shape: per-case `TEST()`s that delegate to the matching
   conformance helper against an `sched_rr_fixture`.

Plus two internal fixture sanity checks
(`sched_fifo_internal_empty_queue_is_detected`,
`sched_fifo_internal_duplicate_enqueue_is_eexist`) — if the fixture is
broken, these fail loudly before the conformance cases run, so it's
easy to tell "harness bug" from "fixture bug."

### Makefile wiring (`test/host/Makefile`)

- `SRCS` gains `conformance/conformance_scheduler.c` and
  `conformance_scheduler_test.c`.
- `vpath %.c` gains `conformance` so the `%.o: %.c` pattern can
  locate the subdirectory source after `$(notdir ...)` flattens
  the object basenames.

### Test counts

- Python: **51** (unchanged).
- Host:   **143** (134 → +9: 7 conformance + 2 fixture sanity).
- Kernel: **13** (unchanged).
- Total:  **207 / 207 pass**, 0 leaks, 0 errors.
- `make verify-registry`: 0 findings.
- QEMU boot and ktest unchanged.

## Decisions

**One helper per case, not one bundled `conformance_scheduler_run`.**
The DESIGN.md sketch (`test_scheduler_conformance(&sched_rr_ops)`) is
one call; real-world test frameworks want per-case reporting. Host's
ASSERT macros `return;` on failure, so a bundled runner would only
surface the first failing case. Seven separate helpers + seven
TEST()s per caller gives clean granularity for ~15 lines of
boilerplate — worth it.

**Fixture lives inline in the test file, not in `test/host/fixtures/`.**
`fixtures/` to date has held only JSON dep descriptors for slice 3.4;
adding C code there would mix two conventions. Slice 3.4's `trivial_ops`
precedent is "fixture inline in the test file." Followed that.

**No `sched_null`.** My pre-implementation plan called for a
deliberately-trivial "always returns NULL" fixture as a negative test
of the harness. On write-up it turned out that sched_null *fails*
most conformance cases by design, so using it to "prove the harness
works" is awkward — a passing fixture is a cleaner proof. Dropped.

**Conformance cases use stack `struct nx_task`.** Going through
`nx_task_create` means allocating a kstack per task, 100 times for
case 6 — wasted effort when the scheduler only touches `sched_node`,
`id`, and `state`. `init_test_task()` is four lines.

**`set_priority` contract: `NX_OK` or `NX_EINVAL`, but consistent.**
The scheduler.h docs promised "NX_OK on accept, NX_EINVAL if the
policy doesn't support priorities." The conformance case asserts
*both tasks* get the same status — catches the "priorities for some
tasks but not others" misbehaviour without requiring callers to know
whether their policy is priority-aware.

**Fixture implements duplicate-enqueue detection (`NX_EEXIST`) even
though no conformance case exercises it.** The contract in scheduler.h
promises this code; correct fixtures model the full interface, not
just what's tested. Slice 4.3's sched_rr will need it for real once
task state transitions (BLOCKED → READY re-enqueue, etc.) land.

## What's Next

**Slice 4.3 — `components/sched_rr/`.** First real policy component.
Manifest + impl + README; passes the conformance suite. Adds
`NX_HOOK_CONTEXT_SWITCH` enum + `csw` arm (dispatch stubbed until
4.4). Binds to the `scheduler` slot in `kernel.json`.

**Opportunistic polish (not blocking):**
- Hook the slice-4.2 conformance case file path into
  AI-RULES.md §R0-ish (interface conformance) — not a rubric rule
  today but useful breadcrumb.
- Consider moving `sched_fifo` into `test/host/fixtures/` if a second
  caller (hypothetical future test that wants a trivial scheduler for
  some non-conformance purpose) wants it. No caller today, so inline.

## Files Touched

- `interfaces/scheduler.h` — new (first file in this dir).
- `test/host/conformance/conformance_scheduler.{h,c}` — new.
- `test/host/conformance_scheduler_test.c` — new.
- `test/host/Makefile` — SRCS + vpath.
- `proj_docs/nonux/HANDOFF.md` — slice 4.2 checkbox flipped, session
  log link added, Next Actions refreshed.
- `proj_docs/nonux/README.md` — status + last-updated.
- `proj_docs/nonux/logs/session-17-sched-conformance.md` — this log.
