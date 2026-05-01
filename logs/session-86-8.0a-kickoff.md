# Session 86: Group B kickoff — slice 8.0a sub-sliced; 8.0a.1 + 8.0a.2 land

**Date:** 2026-04-30
**Phase:** Phase 8 — Runtime Recomposition + Config Manager (Group B starts)
**Branch:** master (both `proj_docs/nonux/` and `sources/nonux/`)

---

## Goals

- Reconcile post-Session-85 design refinements that had accumulated in the
  working tree of `proj_docs/nonux/` (DESIGN.md / SLOT-CALL-API.md /
  HANDOFF.md / IMPLEMENTATION-GUIDE.md) and commit them as a clean
  spec-refinement checkpoint before any kernel-side code lands.
- Land slice **8.0a.1** — rename `components/posix_shim/` →
  `components/libnxlibc/` so the `posix_shim` name is free for the
  new kernel-side boundary component.  Source-repo work was already
  drafted (staged renames + Makefile/test path updates, uncommitted).
- Kick off Group B with the smallest substantive code change — slice
  **8.0a.2** — adding the IPC + slot scaffolding (`NX_MSG_FLAG_REPLY_REQUESTED`
  request flag + `resume_waitq` and `_Atomic in_flight_calls` fields on
  `struct nx_slot`) that subsequent slices wire up.
- Decide the sub-slicing structure for the rest of slice 8.0a (8.0a.3 → 8.0a.8)
  and update the planning docs so future sessions can pick up the next
  green checkpoint without rederiving the breakdown.

## What Was Done

### Doc-side spec refinement (commit `37a70f3`, `proj_docs/nonux/`)

Reviewed the four uncommitted files in `proj_docs/nonux/` and committed
them as one logical checkpoint — they form the spec ground that slice
8.0a's implementation builds on.  Key changes:

**DESIGN.md** — four new subsections + several local fixes:

1. §"Sync-mode caller must be on a dispatcher" — bounds `mode: sync`
   to dispatcher-thread callers only.  Rules out syscall-entry edges
   (the syscall caller's task is not a dispatcher; running a receiver
   handler on its kstack would let `slot->active` be read off-dispatcher,
   breaking R8 and the drain step's completeness guarantee).  Concrete
   consequence: every `posix_shim → service` edge in slice 8.0a's
   `kernel.json` is `mode: async`; the kernel-side blocking-call infra
   implements "block on reply waitq, dispatcher runs handler" semantics
   that look synchronous to the syscall caller but route through the
   dispatcher.
2. §"Tasks as IPC Senders" — formalizes per-task `caller_slot` as
   a graph entity.  A task is not a component (no lifecycle, no
   manifest, no descriptor) but it can be the *origin* of a cross-
   component call; for the registry's foundational invariants to hold
   on those edges, the task must be a graph entity with its own slot
   identity.  Slice 8.0a embeds a `caller_slot` in `struct nx_task`,
   created at `nx_task_create` and unregistered at `_destroy`; its
   `active` is bound to the singleton `posix_shim` component.
3. §"Multi-slot binding (N:1)" + §"Slot-side pause + blocking-call
   metadata" — documents the existing `component_node.active_in:
   list<slot>` model that slice 8.0a relies on (a single posix_shim
   instance is the active impl of N task slots, where N can reach
   `NX_PROCESS_TABLE_CAPACITY = 128`).  Also documents two new slot
   fields with slice provenance: `resume_waitq` (slice 8.0a; QUEUE-policy
   callers blocked on a paused slot park here) and `_Atomic in_flight_calls`
   (slice 8.0a; dispatcher-incremented counter that pause-protocol's
   drain step waits for).  Explicit claim: reading these atomic fields
   off the dispatcher is permitted because they don't touch `slot->active`.
4. §"Edge inheritance: per-task slots clone the boundary component's
   outgoing edges" — covers the registry mechanics for the per-task
   `caller_slot` model.  Each task slot inherits `posix_shim`'s outgoing
   edge attributes (`mode`, `stateful`, `policy`) verbatim; cloning
   happens at `nx_task_create` and undone at `_destroy`.  R3 cap-scan
   and the pause hold-queue's `(src, dst)` keying work without per-callsite
   special cases for blocking-call senders.

Plus four local fixes:

- Invariant 1 extended to mention task-embedded `caller_slot`.
- ABORT-on-blocking-call rule added to the hook-chain notes:
  dispatcher synthesizes a reply with `rc = NX_EABORT` whenever a
  SEND-leg or post-handler RECV-leg ABORT fires on a message
  carrying `NX_MSG_FLAG_REPLY_REQUESTED` — otherwise the caller's
  reply-waitq hangs.
- `posix_shim → scheduler` flips `mode: sync → async` in the two
  example sites that listed it as sync; explanatory note added.
- `NX_HOOK_SYSCALL_ENTER`/`_EXIT` retagged Phase 7 → Slice 8.7
  in the hook-points table (matches Session 81's slice-plan update).

**SLOT-CALL-API.md** — three coupled refinements that resolved late
review during the working-tree review:

1. `nx_slot_call_blocking` signature gained explicit `(reply_buf,
   reply_buf_len)` arg pair instead of widening `nx_ipc_message`.
   Rationale: the IPC carrier should stay generic; the reply-buffer
   concept is specific to the blocking-call wrapper protocol.
   Symmetric pair (buf + len) keeps the truncation guard inside the
   wrapper without leaking a length convention into all senders.
2. New `NX_MSG_FLAG_REPLY_REQUESTED` request flag is added alongside
   the existing `NX_MSG_FLAG_REPLY` reply flag.  The two are *not*
   interchangeable — `_REPLY` is set on the reply message itself;
   `_REPLY_REQUESTED` is set on the request to tell the dispatcher
   whether to post a reply leg back after the handler returns.
3. `NX_EABORT = -10` proposal dropped.  Investigation showed `NX_EABORT
   = -8` had landed in slice 3.x for the existing hook-chain ABORT
   path (`framework/registry.h:40`), and slot `-10` was already taken
   by `NX_EPERM` (`registry.h:42`).  Slice 8.0a reuses the existing
   `-8`; no new errcode this slice.

Plus housekeeping: `in_flight_reply_buf_len` field added to `struct
nx_task` (truncation guard input); `posix_shim_handle_msg`
`if (n > task->in_flight_reply_buf_len) return NX_EINVAL` truncation
guard added; `nx_slot_call_blocking` body sequence gained the
recursive-call assertion + ABORT-path buf-clear steps.

**HANDOFF.md / IMPLEMENTATION-GUIDE.md** — slice 8.0a row + immediate-next-step
text reconciled with the SLOT-CALL-API.md refinements; Session 81 entry
got a post-Session-85 reconciliation footnote for traceability.

### Slice 8.0a.1: rename `components/posix_shim/` → `components/libnxlibc/` (commit `5f0b3e0`, `sources/nonux/`)

Source-repo work had been staged before this session (rename plus
Makefile/test-path updates, all uncommitted).  Verified path-only nature
by sampling test diffs (every modified `test/kernel/*.c` was a single-
line `#include` change), staged the unstaged Makefile + test files, and
landed the commit.

Scope: 41 files changed, 142 insertions / 135 deletions.

- 6 file renames: `components/posix_shim/{README.md,crt0.S,manifest.json,
  nxlibc.c,nxlibc.h,posix.h}` → `components/libnxlibc/...`.
- `components/libnxlibc/manifest.json`: `"name": "posix_shim"` → `"libnxlibc"`.
- `components/libnxlibc/README.md`: rewritten with explicit cross-ref
  to slice 8.0a.2 ("renamed from `posix_shim` in slice 8.0a.1 — the
  `posix_shim` name is reserved for the kernel-side boundary component").
- `components/libnxlibc/nxlibc.c`: include-path updates.
- 30 `test/kernel/*.c` files: include paths
  `components/posix_shim/{posix,nxlibc}.h` → `components/libnxlibc/...`.
- `Makefile`: ~92 lines of recipe-path updates.

Frees the `posix_shim` name for the new kernel-side boundary component
that slice 8.0a.4 introduces.  No semantic shift.

### Slice 8.0a.2: IPC + slot scaffolding (commit `93aa7b5`, `sources/nonux/`)

Pure additive scaffolding — no consumer yet.  Three additive primitives
that the rest of 8.0a (.3 through .8) will wire up:

**`framework/ipc.h`** — new `NX_MSG_FLAG_REPLY_REQUESTED (1u << 2)`
request flag.  Sender of a blocking call sets this on the request;
dispatcher uses it to decide whether to post a reply leg back to the
caller's `caller_slot` after the handler returns (or synthesize an
ABORT reply on a hook-chain ABORT).  Distinct from the existing
`NX_MSG_FLAG_REPLY` which is set on the *reply* message — the two are
not interchangeable.

**`framework/registry.h`** (struct `nx_slot`) — two new fields:

- `struct nx_waitq resume_waitq` — `QUEUE`-policy callers blocked on
  a paused slot park here; resume's `nx_waitq_wake_all` releases all
  blocked callers atomically.  Safe to read from caller (non-dispatcher)
  context — atomic-load semantics preserve R8.
- `_Atomic(uint32_t) in_flight_calls` — dispatcher kthread increments
  before invoking a slot's handler and decrements after.  Pause-protocol
  drain step waits for the counter to reach 0 before transitioning
  `DRAINING → DONE`.  Slot-side rather than component-side because
  handler-in-progress is a slot invariant (mid-swap, the slot is between
  two components).

`registry.h` gained `#include "core/sched/waitq.h"`; `nx_waitq` is just
a `struct nx_list_head` (two pointers, header-only inline init), already
linked into both kernel and host builds.

**`framework/registry.c`** — `nx_slot_register` now `nx_waitq_init`s the
new waitq and `atomic_init`s the new counter (mirrors the existing
`atomic_init(&s->pause_state, ...)` pattern).

### Sub-slicing decision: 8.0a → 8.0a.1 → 8.0a.8

Original slice 8.0a's scope from Session 81 — kernel `posix_shim`
component + `framework/slot_call.{h,c}` + per-task `caller_slot` +
reply pool + STATE_LOST handler + cross-cutting test infrastructure
+ ~70 ktests — is too large for a single commit.  Sub-sliced into 8
micro-slices so each landing is a green-test checkpoint.  Mapping:

| Sub-slice | Content |
|---|---|
| 8.0a.1 ✓ | Rename `posix_shim` → `libnxlibc` (path-only). |
| 8.0a.2 ✓ | IPC + slot scaffolding (this session). |
| 8.0a.3 | `framework/slot_call.{h,c}` skeleton (decl + body that returns `NX_ENOSYS` until 8.0a.6). |
| 8.0a.4 | Kernel `components/posix_shim/` skeleton + manifest + `kernel.json` wiring. |
| 8.0a.5 | Per-task `caller_slot` in `struct nx_task` + `nx_task_create`/`_destroy` lifecycle + edge cloning. |
| 8.0a.6 | Reply-message pool + dispatcher integration + `nx_slot_call_blocking` body end-to-end. |
| 8.0a.7 | `posix_shim_on_dep_swapped` STATE_LOST handler + `nx_handle_table_invalidate_for_slot()`. |
| 8.0a.8 | Cross-cutting test infrastructure (mock component, hook-chain inspector, recompose event logger, pause-injector fixture, cap-forgery harness, equivalence-runner macro).  Closes 8.0a. |

Total scope and ~70-ktest budget unchanged — sub-slicing is structural,
not scope-expanding.

IMPLEMENTATION-GUIDE.md §"Group B" table grew to show all 8 sub-rows;
HANDOFF.md Phase checklist tracks each.  This session log captures the
sub-slicing decision so subsequent sessions can pick up the next green
checkpoint without re-deriving the breakdown.

## Key Findings

- **Stale host `.o` files cause stack-smash detection.**  Slice 8.0a.2
  bumped `sizeof(struct nx_slot)` by ~24 bytes (added `nx_waitq` +
  `_Atomic uint32_t`).  After editing `framework/registry.h`,
  `make test` recompiled `framework/registry.c` (kernel + host) but
  the `test/host/Makefile` lacks `-MMD -MP` header-dep tracking, so
  the test `.o` files retained their old `struct nx_slot` size in
  stack-allocated test fixtures (`SLOT(a, "a")` at the start of each
  registry test).  When `nx_slot_register` then wrote past the old
  end of the struct (`nx_waitq_init(&s->resume_waitq)` →
  `s->resume_waitq.waiters.n.next = ...`), it corrupted the stack
  canary.  Stack smashing detection fired at function exit, mid-suite.
  Resolution: `make clean && make test` — green again.  Follow-up
  opportunity (logged): add `-MMD -MP` dependency tracking to
  `test/host/Makefile`; the kernel `Makefile` already does this so
  this is a host-side gap.
- **Path-only renames are surprisingly clean.**  Slice 8.0a.1 touched
  41 files but produced zero behavioral surface area changes.  Verified
  by sampling four randomly-chosen `test/kernel/*.c` diffs — all four
  were single-line `#include` updates with no surrounding code touched.
- **The errcode `NX_EABORT = -8` was already landed.**  Session 81's
  spec proposed a new `NX_EABORT = -10` slot.  Reviewing
  `framework/registry.h` during this session showed line 40
  (`#define NX_EABORT -8`) — slice 3.x had already added it for the
  hook-chain ABORT path on `nx_ipc_send` and `nx_component_pause`.
  Slot `-10` was `NX_EPERM`.  Slice 8.0a reuses the existing `-8`;
  the SLOT-CALL-API.md reconciliation pruned the obsolete proposal.

## Decisions Made

- **Sub-slice 8.0a into 8 micro-slices.**  Reason: 8.0a's locked
  scope (~70 ktests, multi-component scaffolding, per-task slot
  lifecycle, reply pool, dispatcher integration) is too large to land
  in a single green-test checkpoint without losing the IDL-style "one
  slice = one commit, tests stay green" cadence we've kept since slice
  3.6.  How to apply: each sub-slice ends with `make test` strictly
  ≥ the prior sub-slice's count; no sub-slice partially implements a
  user-visible primitive (8.0a.3's skeleton returns `NX_ENOSYS` rather
  than a partial body so callers can compile).
- **Commit doc-side refinements before kernel-side code.**  Reason:
  the working-tree spec edits (DESIGN.md / SLOT-CALL-API.md /
  HANDOFF.md / IMPLEMENTATION-GUIDE.md) had non-trivial design content
  — new subsections, errcode reconciliation, signature change, R7
  invariant extension — that subsequent code needs to reference.
  Landing them as a separate commit gives the spec a stable git SHA
  that 8.0a.1 → 8.0a.8 commits can cite.  How to apply: when working-
  tree spec edits accumulate, land them before the next code-bearing
  slice.
- **Reuse `NX_EABORT = -8` rather than introducing a new errcode.**
  Reason: the existing slot is semantically identical (hook-chain
  ABORT, signed by both `nx_ipc_send` and `nx_component_pause`); a
  parallel errcode would split the ABORT-handling path across two
  values for no benefit.  Slot `-10` is already `NX_EPERM`; renumbering
  errcodes mid-phase ripples through every comparison site.

## Status at End of Session

**Working:**
- Doc spec for slice 8.0a is locked and committed.  All four design
  docs are in sync.
- Source repo has the rename out of the way (`components/libnxlibc/`)
  and the IPC + slot scaffolding in place (no consumers yet).
- All tests green: `make test` 495/495 (93 python + 283 host + 119
  kernel — same baseline as Session 85).  `make test-interactive` 7/7.
  `make verify-iface-fresh` 0 drift.  `make verify-registry` 0 findings.

**Not yet working / pending:**
- `nx_slot_call_blocking` is not yet declared (lands in 8.0a.3).
- Kernel `components/posix_shim/` does not yet exist (lands in 8.0a.4).
- `struct nx_task` does not yet have `caller_slot` (lands in 8.0a.5).
- The reply pool, dispatcher integration, and end-to-end blocking call
  flow are not yet wired (land in 8.0a.6).
- STATE_LOST handler not yet implemented (lands in 8.0a.7).
- Cross-cutting test infrastructure not yet landed (lands in 8.0a.8).

## Next Steps

1. **Slice 8.0a.3** (immediate next):  `framework/slot_call.{h,c}`
   skeleton.  Header declares `nx_slot_call_blocking(struct nx_slot
   *slot, struct nx_ipc_message *msg, void *reply_buf, size_t
   reply_buf_len)` with doc strings copied verbatim from
   SLOT-CALL-API.md §"API".  Body validates args and returns
   `NX_ENOSYS` (the dispatcher round-trip lands in 8.0a.6).  Hooks
   into the kernel build alongside other `framework/*.c` files.  No
   callers yet.  Trivial slice — sets up the call surface so 8.0a.4
   and 8.0a.5 can compile with `extern` references.
2. **Follow-up (non-blocking):**  Add `-MMD -MP` header-dep tracking
   to `test/host/Makefile` so future `struct nx_slot` size changes
   don't reproduce this session's stack-smash detection.  Logged as a
   minor opportunity, not a blocker.

---

**Files Changed:**

`proj_docs/nonux/` (commit `37a70f3` "8.0a spec refinement: post-Session-85 reconciliation"):

- `DESIGN.md` — 4 new subsections (§"Sync-mode caller must be on a dispatcher", §"Tasks as IPC Senders", §"Multi-slot binding (N:1)" + §"Slot-side pause + blocking-call metadata", §"Edge inheritance"); Invariant 1 extended; ABORT-on-blocking-call rule; `posix_shim → scheduler` mode flip; hook-table provenance fix.
- `SLOT-CALL-API.md` — `(reply_buf, reply_buf_len)` arg pair; `NX_MSG_FLAG_REPLY_REQUESTED` request flag; `NX_EABORT = -8` reconciliation; `in_flight_reply_buf_len`; truncation guard; recursive-call assertion; ABORT-path buf-clear.
- `HANDOFF.md` — slice 8.0a immediate-next-step + Session 81 footnote.
- `IMPLEMENTATION-GUIDE.md` — slice 8.0a row reconciled with SLOT-CALL-API refinements.

`sources/nonux/` (commit `5f0b3e0` "Phase 8.0a.1: rename posix_shim → libnxlibc"):

- 6 file renames: `components/posix_shim/{README.md,crt0.S,manifest.json,nxlibc.c,nxlibc.h,posix.h}` → `components/libnxlibc/...`.
- `components/libnxlibc/manifest.json` — name field updated.
- `components/libnxlibc/README.md` — rewritten with cross-ref to slice 8.0a.2.
- `components/libnxlibc/nxlibc.c` — include paths.
- 30 `test/kernel/*.c` — `#include` path updates.
- `Makefile` — ~92 lines of recipe-path updates.

`sources/nonux/` (commit `93aa7b5` "Phase 8.0a.2: IPC + slot scaffolding for blocking-call infra"):

- `framework/ipc.h` — `NX_MSG_FLAG_REPLY_REQUESTED (1u << 2)` request flag.
- `framework/registry.h` — `#include "core/sched/waitq.h"`; new `struct nx_waitq resume_waitq` + `_Atomic(uint32_t) in_flight_calls` fields on `struct nx_slot`.
- `framework/registry.c` — `nx_slot_register` zero-initializes both new fields via `nx_waitq_init` and `atomic_init`.

`proj_docs/nonux/` (this session log + handoff updates, to be committed alongside this writeup):

- `IMPLEMENTATION-GUIDE.md` — Group B table sub-sliced; "Last Updated" + Phase 8 status footers updated.
- `HANDOFF.md` — Current Status, Phase checklist, Next Actions, Session Logs section all updated for sub-slicing + 8.0a.1/.2 landed.
- `HANDOFF-ARCHIVE.md` — Session 81 archived (per "keep last 5" convention).
- `logs/session-86-8.0a-kickoff.md` — this file.
