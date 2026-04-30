# Session 81: Slot-Call API Design — Slice 8.0a Spec Locked

**Date:** 2026-04-29
**Phase:** Phase 8 prep (no code changes; design only)
**Branch:** master

---

## Goals

- Lock the slice 8.0a `nx_slot_call_blocking` API contract before generator wrapper-emission templates depend on it (per IMPLEMENTATION-GUIDE.md risk note).
- Reconcile the proposed sync-call mechanics with DESIGN.md's invariants (dispatcher-only entry, slot-resolve locality, R3 cap discipline).
- Settle nine open questions that surfaced during the design discussion, including caller identity for syscall context, reply-routing mechanism, naming, and per-task graph membership.

## What Was Done

### Initial proposal — Resolution 2 (rejected)

Proposed an `nx_slot_call_sync` that runs handlers directly on the caller's task stack with `preempt_disable()` + in-flight counter as a "dispatcher-equivalent context" — analogous to the IRQ-return reschedule shim's framing. Cheap (<1 µs per call), but required a DESIGN.md carve-out weakening Invariant #7 (dispatcher-only entry).

User pushed back: *"the api of sending message give target slot. This slot should be resolved by the dispatcher. the system call doesn't need to do that by itself."*

### Resolution 1 locked — dispatcher resolves the slot

Caller never reads `slot->active`. The wrapper builds a request msg + kstack-allocated reply struct, hands it to the dispatcher via `nx_dispatcher_enqueue`, blocks on a per-task `reply_waitq`. The dispatcher kthread:

1. Dequeues the request, resolves `slot->active`, runs the handler under `preempt_disable()`.
2. If `NX_MSG_FLAG_REPLY_REQUESTED`, builds a reply message (`dst = caller's task slot, payload = rc + outputs`), enqueues the reply.
3. Dequeues the reply, calls `posix_shim_handle_msg` which writes payload into `task->in_flight_reply_buf` and wakes `task->reply_waitq`.

Two voluntary task switches per call (caller → dispatcher → caller); the original ≤10% perf gate is unattainable. Replaced with: ≤10 µs per round-trip on QEMU virt aarch64, absolute latency revisited in Phase 10.

This honors DESIGN.md's Invariant #7 directly without any "dispatcher-equivalent context" carve-out.

### Eight design questions resolved (Q1–Q9)

The user resolved the open questions in order:

| Q | Decision | Rationale |
|---|---|---|
| Q1 | Rename today's userspace lib `components/posix_shim/` → `components/libnxlibc/`; new `components/posix_shim/` is the kernel boundary component. | DESIGN.md uses "posix_shim" specifically for the boundary component. |
| Q2 | All four posix_shim → service edges = `mode: async` (deviating from DESIGN.md's `posix_shim → scheduler = sync` example). | Sync-mode same-stack shortcut requires caller on a dispatcher; syscall callers are not. DESIGN.md gets a one-section note with slice 8.0a. |
| Q3 | ABORT path posts a synthetic reply with `rc = NX_EABORT` so the caller wakes. | Caller is blocked on reply waitq; ABORT-without-reply hangs the caller. |
| Q4 | Wire `posix_shim_on_dep_swapped` to call `nx_handle_table_invalidate_for_slot()` on `STATE_LOST`. | Hot-swap correctness; ~30+50 lines. |
| Q5 | Add slice 8.7 for `NX_HOOK_SYSCALL_ENTER` / `_EXIT`. | DESIGN.md lists these as Phase 7 hooks but they were never implemented; slot-call API doesn't need them, but they belong in Phase 8's scope. |
| Q6 | Rename `nx_slot_call_sync` → **`nx_slot_call_blocking`**. | "sync" overloads with the connection-mode "sync"; "blocking" describes what it does to the caller. |
| Q7 | Reply slot lives on caller's kstack (not heap-allocated). | Caller blocks before returning, so kstack lifetime covers reply delivery. |
| Q8 | Each task gets a slot. | Per-task identity for hooks, hold-queue keying, cap-scan; collapses earlier "documentation only" mitigation. |
| Q9 | **Option β** — replies flow through the dispatcher as messages, not a side-channel waitq wake. | Architecturally uniform with the IPC model (hooks/cap-scan see replies). User accepted the extra dispatcher round-trip cost. |

### Per-task slot architecture

Per Q8, `struct nx_task` grows an embedded `struct nx_slot caller_slot` and a per-task `reply_waitq`. Lifecycle:

- `nx_task_create` registers `caller_slot`, binds `active = posix_shim_component`, walks posix_shim's outgoing edges, and registers parallel `(caller_slot → svc_slot)` edges with the same mode/stateful/policy.
- `nx_task_destroy` plain-unregisters the edges and the slot. **No drain check** because v1 has no path to externally destroy a blocked task: `sys_exit` runs on the exiting task's own kstack, which means the task isn't blocked. User-corrected my initial proposal of an in-flight counter discipline at task_destroy time.

All task `caller_slot`s share the same `active = posix_shim_component` instance — the registry already supports N→1 (`nx_component_foreach_bound_slot`).

Edge count check: `NX_PROCESS_TABLE_CAPACITY = 128`, posix_shim has 4 deps → peak 512 task→service edges. Initially I worried this might overflow a fixed `NX_GRAPH_CONNECTION_CAPACITY`; checked the source — `nx_connection_register` uses `calloc` per call (`framework/registry.c:507`), no static capacity. Bump not needed.

### posix_shim component definition

Per the dependency-injection mechanism (DESIGN.md §1553), the new `components/posix_shim/` provides:

- `manifest.json` with `requires: { vfs, sched, mm, char_device.serial }`, all `mode: async stateful: true policy: queue` (vfs alone is `stateful: true`; rest are `stateful: false`).
- `posix_shim.c` with `init` (sets `g_posix_shim` singleton), `enable`, `handle_msg` (routes replies via `task_from_caller_slot()` → `container_of` → writes payload to `task->in_flight_reply_buf` → wakes `task->reply_waitq`), `on_dep_swapped` (Q4 invalidation).
- Auto-generated `gen/posix_shim_deps.h` per existing dependency pipeline.
- `framework/syscall.c` reaches deps via `g_posix_shim->deps.vfs` (etc.) instead of `nx_slot_lookup`.

posix_shim's `concurrency: shared`. Single instance bound to N task caller_slots; dispatcher kthread serializes all reply-handler invocations.

### Spec document landed

Wrote `proj_docs/nonux/SLOT-CALL-API.md`. Sections:
- Purpose + DESIGN.md constraints + today's gaps.
- Architecture diagram showing the full request → handler → reply → wake flow.
- Per-task `caller_slot` data structure + lifecycle + edge count discussion.
- posix_shim component spec — manifest, files, ops, on_dep_swapped, concurrency mode.
- Public API: `nx_slot_call_blocking` signature + body sequence + constraints.
- Reply path (Option β) including ABORT handling and per-CPU reply pool sizing.
- Generated wrapper template (vfs.read example).
- Pause protocol interaction — pause-state read on caller side, edge-driven policy, in-flight counter on slot.
- Error codes including new `NX_EABORT = -10`.
- Concurrency-mode dispatch table.
- Test plan — ~70 ktests, cross-cutting infrastructure (mock component, hook inspector, recompose event logger, pause injector, cap-forgery harness, equivalence-runner macro).
- Open items / future extensions.

### Doc-web wiring

- `SLOT-CALL-API.md` added to navigation rows in HANDOFF.md, IMPLEMENTATION-GUIDE.md, IDL-SCHEMA.md.
- IMPLEMENTATION-GUIDE.md slice 8.0a entry expanded to reference the new spec.
- IMPLEMENTATION-GUIDE.md slice 8.7 added (SYSCALL_ENTER/EXIT hooks per Q5).
- HANDOFF.md current status updated; Session 81 added to session list.

## Key Findings

- **DESIGN.md's Invariant #7 is load-bearing.** I was about to weaken it with a "dispatcher-equivalent context" carve-out for the syscall path. User push-back forced the cleaner architecture (Resolution 1) which honors the invariant directly.
- **posix_shim's promotion is bigger than a sentinel.** It's a full component with manifest-declared deps that auto-resolve at boot. All four kernel-facing service edges register automatically through the dependency-injection pipeline; no manual `nx_connection_register` calls in bootstrap.
- **Per-task slots make src_slot meaningful for syscalls.** Without them, all syscall callers collide as `src = NULL` — breaks hold-queue keying, hooks lose per-caller identity, R3 cap-audit can't enforce. With them, the graph has a real node for each running task; hooks see who called what.
- **The reply-via-dispatched-message model (Option β) is uniform but expensive.** Two extra dispatcher round-trips per call (request handler + reply handler). On single-CPU QEMU this is 1–10 µs per syscall vs. ~100 ns today. Hot syscalls become measurably slower; we accept this for architectural uniformity (hooks/cap-scan see replies, future caps-in-replies works naturally) and revisit absolute perf in Phase 10.
- **Connection registry has no static capacity.** Today's `framework/registry.c:507` uses `calloc` per connection. Earlier worry about ~600 worst-case edges was unfounded; allocation is bounded by available kernel memory only.
- **`task_destroy` doesn't need an in-flight counter discipline.** A task can only be destroyed via its own `sys_exit`, which runs on its own kstack — meaning the task isn't blocked. v1 has no external-kill path. User's correction; defer the counter to whenever multi-thread + external-kill scenarios land.
- **DESIGN.md has a stale example.** `posix_shim → scheduler = mode: sync` won't work with Resolution 1; updated to `mode: async` in our actual `kernel.json` and noted the divergence in DESIGN.md.

## Decisions Made

All nine open questions resolved (table above).

Plus three decisions not in the original Q1–Q9:

- **Per-task `reply_waitq` and single in-flight call per task.** Simpler than per-call waitq; v1 syscalls don't issue concurrent IPCs from the same task. Recursive calls from a syscall handler are not supported in v1 (asserts on entry if `in_flight_reply_buf` is already set).
- **Per-CPU reply message pool sized at 256.** 2× safety margin over `NX_PROCESS_TABLE_CAPACITY = 128`; assertion on exhaustion (matches existing ISR-message-pool discipline from `framework/dispatcher.h`).
- **`NX_EABORT = -10` slots between `NX_EDEADLINE = -9` and `NX_ELOOP = -11`.** Numbering preserves chronological allocation order.

## Status at End of Session

- `proj_docs/nonux/SLOT-CALL-API.md` written (~700 lines).
- `proj_docs/nonux/logs/session-81-slot-call-api.md` (this file).
- IMPLEMENTATION-GUIDE.md updated — slice 8.0a deliverables expanded; slice 8.7 added.
- HANDOFF.md updated — current status, next actions, session list.
- No source code changes.
- Tests unchanged: `make test` 453/453; `make test-interactive` 7/7.

## Next Steps

- **Session 82+:** Slice 8.0pre.1 implementation kickoff (or reorder — see below).
  1. Write `tools/idl-meta-schema.json` (JSON Schema draft-07 form of IDL-SCHEMA.md's rules).
  2. Write `tools/gen-iface.py` skeleton.
  3. Write `interfaces/idl/vfs.json`.
  4. Wire `make gen-iface` + `make verify-iface-fresh`.
  5. Compatibility test: generated `interfaces/vfs.h` matches today's hand-written file byte-for-byte.

- **Possible reorder:** slice 8.0a is now a substantial body of work (per-task slots, posix_shim promotion, libnxlibc rename, reply infrastructure, DESIGN.md update). The IDL generator (8.0pre.1) just needs the wrapper template's signature, which is locked at this point in `nx_vfs_read(struct nx_slot *slot, ...) → int64_t`. Generator can land before the runtime infra; the wrappers will reference `nx_slot_call_blocking` as an extern function until 8.0a fills in the implementation.

  This reorder would let 8.0pre.1–4 (generator + per-iface IDL) land first, then 8.0a (runtime infra) once the wrapper template is in production use. Or land in parallel.

- **DESIGN.md update with slice 8.0a:** clarify §IPC §"Sync Shortcut" — sync-mode same-stack invocation requires caller on a dispatcher; syscall callers are not, so all syscall edges are `mode: async`.

---

**Files Changed:**

- `proj_docs/nonux/SLOT-CALL-API.md` — new doc; slice 8.0a's full API spec.
- `proj_docs/nonux/HANDOFF.md` — Current Status + Next Actions updated; Session 81 added to session list; Session 76 rolled to archive; doc-web row gains SLOT-CALL-API link.
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Phase 8 slice 8.0a deliverables expanded; new slice 8.7 row added; nav row gains SLOT-CALL-API link.
- `proj_docs/nonux/IDL-SCHEMA.md` — nav row gains SLOT-CALL-API link.
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 76 entry rolled in.
- `proj_docs/nonux/logs/session-81-slot-call-api.md` — this file.
