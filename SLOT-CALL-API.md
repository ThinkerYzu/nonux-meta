# Slot-Call API: nonux Synchronous Cross-Component Call Contract

**Project:** nonux
**Created:** 2026-04-29 (Session 81)
**Status:** Draft — locked at Session 81; landing in slice 8.0a.

---

## Navigation

**Project Docs:** [README](README.md) | [SPEC](SPEC.md) | [DESIGN](DESIGN.md) | [IMPLEMENTATION-GUIDE](IMPLEMENTATION-GUIDE.md) | [HANDOFF](HANDOFF.md) | [IDL-SCHEMA](IDL-SCHEMA.md) | [SLOT-CALL-API](SLOT-CALL-API.md) *(you are here)*

**This Document:**
- [Purpose](#purpose)
- [Background and Constraints](#background-and-constraints)
- [Architecture](#architecture)
- [Per-Task `caller_slot`](#per-task-caller_slot)
- [The posix_shim Component](#the-posix_shim-component)
- [Public API: `nx_slot_call_blocking`](#public-api-nx_slot_call_blocking)
- [Reply Path (Option β)](#reply-path-option-β)
- [Generated Wrapper Template](#generated-wrapper-template)
- [Pause Protocol Interaction](#pause-protocol-interaction)
- [Error Codes](#error-codes)
- [Concurrency Modes](#concurrency-modes)
- [Test Plan](#test-plan)
- [Open Items / Future Extensions](#open-items--future-extensions)

---

## Purpose

Phase 8 routes every cross-component call through the IPC framework so hot-swap is structurally safe (Session 79). Slice 8.0a lands the **synchronous slot-call infrastructure** that the IDL-driven sender wrappers (Group A) target. This document specifies the API contract, the reply-path mechanism, and the per-task / posix_shim plumbing the API depends on.

The design honors DESIGN.md's **dispatcher-only entry** invariant directly: the syscall caller never reads `slot->active`. The caller hands a request message + reply-buffer pointer to the dispatcher; the dispatcher resolves the slot, runs the handler, dispatches a reply message back to the caller's per-task slot; posix_shim's `handle_msg` wakes the blocked task.

---

## Background and Constraints

### From DESIGN.md

| Constraint | Source |
|---|---|
| Component handlers execute only on a framework-owned dispatcher thread | DESIGN.md §"Invariants" — Dispatcher-only entry |
| Reading `slot->active` is permitted only on a dispatcher thread | DESIGN.md §"Invariants" — Slot-resolve locality (R8) |
| `src_slot` in `struct ipc_message` is well-defined for every sender | DESIGN.md §"Every Component Occupies a Slot" |
| posix_shim is the kernel slot at the userspace/kernel boundary | DESIGN.md §"Connection Configuration" + §"Every Component Occupies a Slot" |
| Slot refs travel as typed caps in `msg->caps[]`, not in payload | DESIGN.md §"Slot References in IPC Messages" (R3) |
| Hold queue is keyed on `(src, dst)` pairs | DESIGN.md §"Pause Implementation" |
| `src_slot` is **not** for reply routing | DESIGN.md §"Message Format" |

### From the existing implementation (today's gaps)

| Today | Slice 8.0a target |
|---|---|
| `framework/syscall.c` calls `vops->read(self, ...)` directly (~184 sites across 19 files) | All cross-component calls go through `nx_slot_call_blocking` |
| `posix_shim` is a userspace library only (crt0 + libnxlibc.a), not a kernel slot | Userspace lib renamed to `libnxlibc/`; new kernel `posix_shim` component lands |
| Tasks have no slot identity; `src_slot` would be NULL for syscalls | Per-task `caller_slot` lands; src is well-defined per DESIGN |
| Sync IPC (`mode == NX_CONN_SYNC`) invokes handler on caller's stack — violates R8 if caller isn't a dispatcher | Sync mode = "block on reply waitq, dispatcher runs handler" semantics |
| No reply-routing infrastructure | New: `msg->flags |= NX_MSG_FLAG_REPLY_REQUESTED`, dispatcher posts reply message back |

---

## Architecture

```
   ┌──────────────────────────────────────────────────────────────────────┐
   │                      Task X (syscall caller)                         │
   │                                                                      │
   │  framework/syscall.c                                                 │
   │    │                                                                 │
   │    ▼                                                                 │
   │  nx_vfs_read(slot, file, buf, cap)        ← generated wrapper       │
   │    │                                                                 │
   │    ▼                                                                 │
   │  nx_slot_call_blocking(vfs_slot, &msg)                               │
   │    │  (1) atomic load slot->pause_state; apply edge policy           │
   │    │  (2) enqueue msg → dispatcher                                   │
   │    │  (3) block on task->reply_waitq                                 │
   └────┼─────────────────────────────────────────────────────────────────┘
        │
        ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                        Dispatcher kthread                            │
   │                                                                      │
   │  (4) dequeue request msg; resolve vfs_slot->active = vfs_simple      │
   │  (5) preempt_disable; vfs_simple->handle_msg(impl, msg) → rc         │
   │  (6) if NX_MSG_FLAG_REPLY_REQUESTED:                                 │
   │        build reply msg { dst = task->caller_slot, payload = rc + … } │
   │        enqueue reply                                                 │
   │                                                                      │
   │  (7) dequeue reply msg; resolve task->caller_slot->active = posix_shim│
   │  (8) posix_shim_handle_msg(impl, reply):                             │
   │        write rc + outputs into task->reply_buf                       │
   │        nx_waitq_wake_one(&task->reply_waitq)                         │
   └──────────────────────────────────────────────────────────────────────┘
        │
        ▼
   ┌──────────────────────────────────────────────────────────────────────┐
   │                   Task X resumes (reply_waitq woken)                 │
   │  (9) read rc + outputs from kstack reply_buf; return to userspace    │
   └──────────────────────────────────────────────────────────────────────┘
```

The **only** code that reads `slot->active` is the dispatcher kthread (steps 4 + 7), running with `preempt_disable()` around the handler invocation. Invariant #7 is honored without any "dispatcher-equivalent context" carve-out.

### The cost trade

Every blocking slot-call costs **two voluntary task switches** (caller → dispatcher → caller) plus one extra dispatcher round-trip for the reply. On QEMU virt aarch64 single-CPU this is roughly 1–10 µs per call vs. today's direct `ops->call` which is ~100 ns. Hot-path syscalls (`read`, `write`, `open`) become measurably slower.

This is the price of architectural correctness. Slice 8.0c's perf checkpoint adjusts the gate accordingly:

> **Hot-path perf gate (slice 8.0c, revised):** `make test` 453/453 + `make test-interactive` 7/7 stay green; per-syscall round-trip ≤ 10 µs on QEMU virt aarch64; absolute latency revisited in Phase 10 benchmarks.

(Original ≤10% regression gate is unattainable with Resolution 1 and is dropped.)

---

## Per-Task `caller_slot`

### Why per-task slots

The IPC router uses `src_slot` for cap-scan, hold-queue keying (`(src, dst)`), hook context, and the registry's connection-edge tracking. Without a per-task identity:
- Multiple concurrent syscall callers collide in the hold queue under `(NULL, dst)`.
- Hooks lose per-caller observability.
- DESIGN.md's R3 cap-audit ("sender legitimately holds this slot ref") cannot be enforced.

Promoting tasks to graph entities — each task gets a slot — fixes all four. Tasks remain scheduling entities (still in `core/sched/task.h`); the slot is just an identity for the graph.

### Data structure

```c
/* core/sched/task.h — additions */
struct nx_task {
    /* ... existing fields ... */

    struct nx_slot   caller_slot;       /* embedded; lifetime == task lifetime */
    struct nx_waitq  reply_waitq;       /* per-task; one in-flight call at a time */
    void            *in_flight_reply_buf;   /* set by wrapper; cleared on wake */
    int              in_flight_reply_rc;    /* set by posix_shim_handle_msg */
};
```

### Lifecycle

- **`nx_task_create`:**
  1. Initialize `caller_slot` as `mutability = HOT, concurrency = SHARED, name = "task#<pid>:<tid>"`.
  2. `nx_slot_register(&task->caller_slot)`.
  3. Bind `task->caller_slot.active = &g_posix_shim_component` (all task slots share the same component instance).
  4. Walk posix_shim's outgoing edges; for each `(posix_shim → svc_slot)`, register a parallel `(task->caller_slot → svc_slot)` edge with the same mode/stateful/policy.
  5. `nx_waitq_init(&task->reply_waitq)`.
- **`nx_task_destroy`:**
  1. Walk `task->caller_slot`'s outgoing edges; unregister each.
  2. `nx_slot_unregister(&task->caller_slot)`.
  - **No drain check needed.** A task can only be destroyed via its own `sys_exit`, which runs on its own kstack, which means it cannot simultaneously be blocked in `nx_slot_call_blocking`. v1 has no path to externally destroy a blocked task.

### Edge count

Today's posix_shim has 4 deps (vfs, sched, mm, char_device.serial). With `NX_PROCESS_TABLE_CAPACITY = 128`, peak is `128 × 4 = 512` task→service edges. The registry stores connections as a heap-allocated linked list (`framework/registry.c:507` uses `calloc`), so no static capacity to tune; allocation is bounded by available kernel memory.

---

## The posix_shim Component

### Renaming

Today's `components/posix_shim/` is the **userspace** library (crt0 + libnxlibc.a) replacing musl's runtime. Slice 8.0a renames it:

```
components/posix_shim/   →   components/libnxlibc/
```

The new `components/posix_shim/` is the **kernel-side** component declared by DESIGN.md as the syscall-entry boundary.

### Files

```
components/posix_shim/
├── manifest.json          # declares deps {vfs, sched, mm, char_device.serial}
├── posix_shim.c           # init/enable/handle_msg/on_dep_swapped/destroy
└── README.md
```

`gen/posix_shim_deps.h` is auto-generated from `manifest.json` per the existing dependency-injection mechanism (DESIGN.md §"Dependency Injection Mechanism").

### Manifest (sketch)

```json
{
  "name": "posix_shim",
  "version": "0.1.0",
  "spawns_threads": false,
  "pause_hook": false,
  "concurrency": "shared",
  "mutability": "warm",
  "requires": {
    "vfs":      { "version": ">=0.1.0", "mode": "async", "stateful": true,  "policy": "queue" },
    "scheduler":{ "version": ">=0.1.0", "mode": "async", "stateful": false, "policy": "queue" },
    "mm":       { "version": ">=0.1.0", "mode": "async", "stateful": false, "policy": "queue" },
    "char_device.serial": { "version": ">=0.1.0", "mode": "async", "stateful": false, "policy": "queue" }
  }
}
```

All deps are `mode: async` because syscall callers (task `caller_slot` instances) are not on a dispatcher; the sync-mode same-stack shortcut doesn't apply.

> **Note on DESIGN.md:** the `kernel.json` example in DESIGN.md §"Kernel Configuration File" shows `posix_shim → scheduler` as `mode: sync`. Slice 8.0a's actual `kernel.json` deviates with `mode: async` for all four edges. DESIGN.md gets a one-section update with slice 8.0a explaining the constraint (sync-mode is dispatcher-to-dispatcher only).

### Component state

```c
struct posix_shim_state {
    struct nx_component   base;        /* must be first */
    struct posix_shim_deps deps;       /* generated; auto-resolved at boot */
};

extern struct posix_shim_state *g_posix_shim;   /* singleton, accessed by syscall.c */
```

`framework/syscall.c` reaches deps via `g_posix_shim->deps.vfs` (etc.) instead of `nx_slot_lookup("vfs")`, so the wrapper signature is `nx_vfs_read(g_posix_shim->deps.vfs, ...)`.

### Component ops

```c
static int posix_shim_init(void *self) {
    struct posix_shim_state *s = self;
    g_posix_shim = s;
    return 0;
}

static int posix_shim_enable(void *self) {
    /* Per-task slot wiring is done at task_create time, not here.
     * posix_shim's enable just marks the boundary as live. */
    return 0;
}

static int posix_shim_handle_msg(void *self, struct nx_ipc_message *msg) {
    /* Dispatcher delivers reply messages to per-task caller_slots, all
     * of which share this component instance.  Route by msg->dst_slot
     * which identifies the originating task. */
    if (!(msg->flags & NX_MSG_FLAG_REPLY)) {
        /* Unsolicited message to a task slot — should never happen; tasks
         * are senders, not receivers (except for replies). */
        return NX_EINVAL;
    }
    struct nx_task *task = task_from_caller_slot(msg->dst_slot);
    if (!task || !task->in_flight_reply_buf) return NX_EINVAL;

    /* Decode rc + outputs from msg->payload into task->in_flight_reply_buf.
     * Wrapper-emitted reply struct shape is per-op. */
    memcpy(task->in_flight_reply_buf, msg->payload, msg->payload_len);
    task->in_flight_reply_rc = ((struct reply_header *)msg->payload)->rc;
    nx_waitq_wake_one(&task->reply_waitq);
    return 0;
}

static int posix_shim_on_dep_swapped(void *self, struct nx_slot *dep_slot,
                                     struct nx_component *old, struct nx_component *new,
                                     uint32_t flags) {
    if (flags & SWAP_STATE_LOST) {
        /* Walk the kernel handle table; mark all entries pointing at
         * dep_slot as stale.  Subsequent userspace ops on those fds
         * return NX_EBADF. */
        nx_handle_table_invalidate_for_slot(dep_slot);
    }
    return 0;
}
```

`task_from_caller_slot()` is a small helper that maps a slot pointer back to its containing task — implementable as `container_of(slot, struct nx_task, caller_slot)` since the slot is embedded.

### Concurrency mode

posix_shim's component is `concurrency: shared`. Single instance, multiple per-task slots all bound to it. On single-CPU there's no cross-CPU concern; the dispatcher kthread serializes all `handle_msg` calls. SMP eventually wants `serialized` (framework spinlock) or per-CPU split — defer.

---

## Public API: `nx_slot_call_blocking`

### Signature

```c
/*
 * Issue a blocking call to a slot.  Caller's task blocks on its own
 * reply waitq until the dispatcher posts a reply.
 *
 * `slot` is the destination slot.  `msg` carries the request:
 *   msg->src_slot   = current task's caller_slot (set by wrapper)
 *   msg->dst_slot   = slot
 *   msg->msg_type   = op_id from generated `enum nx_<iface>_op_id`
 *   msg->flags     |= NX_MSG_FLAG_REPLY_REQUESTED
 *   msg->payload    = pointer to wrapper-built request struct (kstack)
 *   msg->reply_buf  = pointer to wrapper-allocated reply struct (kstack)
 *   msg->n_caps    + msg->caps  -- optional; per-iface
 *
 * Returns the handler's int rc (NX_OK or negative NX_E*) on successful
 * round-trip, or one of:
 *   NX_EBUSY    — slot paused with REJECT policy on the edge.
 *   NX_ELOOP    — REDIRECT depth cap exceeded (loop in fallback chain).
 *   NX_ENOENT   — slot has no active impl with handle_msg.
 *   NX_EABORT   — NX_HOOK_IPC_SEND chain returned ABORT.
 *   NX_EINVAL   — NULL/mismatched args.
 */
int nx_slot_call_blocking(struct nx_slot *slot, struct nx_ipc_message *msg);
```

### Body sequence

```
1. Validate args; assert msg->src_slot == nx_task_current()->caller_slot.
2. Read slot->pause_state (atomic acquire).
3. If pause_state != NX_SLOT_PAUSE_NONE:
     find_edge(msg->src_slot, msg->dst_slot)  -- per-task edge from task_create
     a. policy == QUEUE   → wait on slot->resume_waitq; recheck on wake.
     b. policy == REJECT  → return NX_EBUSY.
     c. policy == REDIRECT → retarget msg->dst_slot = slot->fallback;
                             re-enter with depth+1; cap at NX_IPC_REDIRECT_DEPTH_MAX.
4. Set up reply: task->in_flight_reply_buf = msg->reply_buf.
5. Fire NX_HOOK_IPC_SEND chain; ABORT → return NX_EABORT (skip enqueue).
6. nx_ipc_scan_send_caps(msg->src_slot, msg).  Forged caps → NX_EINVAL.
7. nx_dispatcher_enqueue(msg).
8. nx_waitq_wait_with_deadline(&task->reply_waitq, 0).
9. Clear task->in_flight_reply_buf.
10. Return task->in_flight_reply_rc.
```

### Constraints

- **Single in-flight call per task.** v1 enforces this via `task->in_flight_reply_buf` being a single pointer. Nested calls (a syscall handler recursively calling another syscall) are not supported in v1; assert if `in_flight_reply_buf` is non-NULL on entry.
- **Caller must be a task with a registered `caller_slot`.** ISRs and kthreads cannot call this — they don't have caller_slots. (Boot code uses framework_slot or NULL src; out of scope for v1 user-syscall path.)
- **Bounded handlers still apply.** A handler that takes 10 ms blocks the syscall caller for 10 ms. The framework's bounded-handler rule (DESIGN.md §"Bounded Handlers and Async Split") is more important than ever.

---

## Reply Path (Option β)

Replies are dispatched messages. The dispatcher kthread runs both the request handler AND the reply handler.

### Request handler (vfs_simple, etc.)

```c
/* dispatcher loop, after dequeuing request msg */
struct nx_component *active = slot_resolve_per_mode(msg->dst_slot);
preempt_disable();
slot->in_flight_calls++;             /* atomic, defends against swap during handler */
nx_hook_dispatch(NX_HOOK_IPC_SEND);
nx_ipc_scan_send_caps(...);
int rc = active->descriptor->ops->handle_msg(active->impl, msg);
nx_ipc_scan_recv_caps(...);
slot->in_flight_calls--;
preempt_enable();

if (msg->flags & NX_MSG_FLAG_REPLY_REQUESTED) {
    /* Build reply: dst = msg->src_slot (the task's caller_slot),
     * src = msg->dst_slot, payload = wrapper-defined reply struct
     * with rc as first field, flags = NX_MSG_FLAG_REPLY. */
    struct nx_ipc_message *reply = build_reply(msg, rc);
    nx_dispatcher_enqueue(reply);
}
```

`build_reply` allocates from a per-CPU pre-built message pool (matching the existing ISR-message-pool pattern from `framework/dispatcher.h`); the reply payload is a fixed-size struct generated per-op by the IDL generator.

### ABORT path

If `NX_HOOK_IPC_SEND` returns ABORT, the dispatcher synthesizes a reply with `rc = NX_EABORT` and posts it, ensuring the caller wakes cleanly:

```c
if (nx_hook_dispatch(NX_HOOK_IPC_SEND) == NX_HOOK_ABORT) {
    if (msg->flags & NX_MSG_FLAG_REPLY_REQUESTED) {
        post_synthetic_reply(msg, NX_EABORT);
    }
    return;  /* skip handler invocation */
}
```

Same on `NX_HOOK_IPC_RECV` ABORT after the handler runs (drop the reply but post NX_EABORT to wake caller).

### Reply handler (posix_shim)

```c
/* dispatcher loop, after dequeuing reply msg */
struct nx_component *active = slot_resolve_per_mode(msg->dst_slot);
/* msg->dst_slot is a task's caller_slot; active is g_posix_shim_component. */
posix_shim_handle_msg(active->impl, msg);
/* posix_shim writes payload into task's in_flight_reply_buf, wakes waitq. */
```

The reply path also fires `NX_HOOK_IPC_RECV` (so observability hooks see the reply leg). `nx_ipc_scan_recv_caps` runs on the reply too, in case future op return shapes carry caps.

### Per-CPU reply pool

Reply messages are allocated from a per-CPU pre-built pool to avoid `kmalloc` on the dispatcher hot path. Pool size is determined by max in-flight calls = max tasks (`NX_PROCESS_TABLE_CAPACITY = 128`); slice 8.0a sizes the pool at 256 (2× safety margin) and asserts on exhaustion.

---

## Generated Wrapper Template

Per-op wrapper, emitted by `tools/gen-iface.py` from the IDL:

```c
/* GENERATED — DO NOT EDIT.  See interfaces/idl/vfs.json. */

int64_t nx_vfs_read(struct nx_slot *slot, void *file,
                    void *buf, size_t cap)
{
    /* Request payload — kstack-allocated. */
    struct nx_vfs_msg_read req = {
        .file = (uint64_t)(uintptr_t)file,
        .cap  = cap,
    };

    /* Reply payload — kstack-allocated; dispatcher writes here. */
    struct nx_vfs_reply_read reply_buf = { 0 };

    struct nx_ipc_message msg = {
        .src_slot    = nx_task_current()->caller_slot_ptr,
        .dst_slot    = slot,
        .msg_type    = NX_VFS_OP_READ,
        .flags       = NX_MSG_FLAG_REPLY_REQUESTED,
        .payload     = &req,
        .payload_len = sizeof(req),
        .reply_buf   = &reply_buf,
        .n_caps      = 0,
        .caps        = NULL,
    };

    int rc = nx_slot_call_blocking(slot, &msg);
    if (rc != NX_OK) return rc;

    /* Copy bytes_out portion back into caller's buffer. */
    memcpy(buf, reply_buf.buf_data, reply_buf.bytes_read);

    return reply_buf.ret_value;
}
```

Wrapper sets up:
- request struct on kstack (input scalar fields, opaque handles, slot_refs as caps).
- reply struct on kstack (output scalar fields, output buffers up to a max size, return value).
- `msg->reply_buf` pointer threading to the dispatcher's reply-decoder.

Wrapper does NOT touch `slot->active`. R8 holds.

---

## Pause Protocol Interaction

### Pause-state read on caller side

`nx_slot_call_blocking` reads `slot->pause_state` (atomic acquire). The slot's pause state is mutated by the pause protocol; reading it without holding any lock is safe per the SMP-from-day-one design.

### Edge-driven policy

The per-task→service edge (registered at `nx_task_create`) carries the same policy as the corresponding posix_shim→service edge. Caller's behavior on pause:

- **QUEUE** — block on `slot->resume_waitq` (slice 7.8a primitive). On wake, re-read pause_state; if still non-NONE, repeat. The resume path's `nx_waitq_wake_all(&slot->resume_waitq)` releases all blocked callers atomically.
- **REJECT** — return `NX_EBUSY` immediately.
- **REDIRECT** — retarget to `slot->fallback`, recurse with depth cap (existing `NX_IPC_REDIRECT_DEPTH_MAX`).

### In-flight counter

Slice 8.0a adds `_Atomic(uint32_t) in_flight_calls` to `struct nx_slot`. The dispatcher kthread increments before invoking the handler, decrements after. The pause protocol's drain step waits for this counter to reach 0 before transitioning `DRAINING → DONE` — this is what guarantees no handler is running on the slot when the swap proceeds.

The increment/decrement happens in **dispatcher context** (with `preempt_disable()` already held), so no extra synchronization is needed beyond the atomic. The counter belongs on the slot rather than the component because handler-in-progress is the slot's invariant, not the component's (mid-swap, slot is between two components).

---

## Error Codes

Existing codes (`framework/registry.h`):

| Code | Meaning |
|---|---|
| `NX_OK = 0` | Success |
| `NX_EINVAL = -1` | Invalid argument |
| `NX_ENOENT = -2` | No such entry |
| `NX_ENOMEM = -3` | Out of memory |
| `NX_EBUSY = -7` | Slot paused, REJECT policy |
| `NX_EDEADLINE = -9` | Wait deadline exceeded |
| `NX_ELOOP = -11` | REDIRECT depth cap |

**New in slice 8.0a:**

| Code | Meaning |
|---|---|
| `NX_EABORT = -10` | Hook chain returned ABORT |

(`NX_EABORT = -10` slots between `NX_EDEADLINE = -9` and `NX_ELOOP = -11`; numbering preserves chronological allocation order.)

---

## Concurrency Modes

`nx_slot_call_blocking` honors the destination slot's concurrency mode. The branching lives in the dispatcher's `slot_resolve_per_mode(slot)` — the caller doesn't see it.

| Slot mode | Dispatcher behavior |
|---|---|
| `SHARED` | Direct: `slot->active` |
| `SERIALIZED` | Acquire `slot->u.serial_lock`; release after handler |
| `PER_CPU` | Resolve `slot->u.instances[smp_cpu_id()]` instead of `slot->active` |
| `DEDICATED` | Enqueue to `slot->u.dedicated.queue`; the dedicated thread runs handler — same reply-dispatch flow |

v1 has no `DEDICATED` components; the dispatch path is implemented but untested at runtime until one lands. `SERIALIZED` and `PER_CPU` similarly unused in v1; implemented for completeness.

---

## Test Plan

Slice 8.0a is **the foundation**; ~70 ktests land here (per Session 79's Phase 8 test plan). Cross-cutting test infrastructure also lands here so every later slice can lean on it.

### Cross-cutting infrastructure

- **Mock component** — configurable handle_msg that records args, returns scripted rc, can be made to take an arbitrary number of cycles. Used by all slices that need to exercise pause/swap behavior without a real component.
- **Hook-chain inspector** — installs N hooks at known priorities and asserts they fire in order with correct context shape.
- **Recompose event logger** — subscribes to `NX_EV_*` events and records them; used by Group C tests.
- **Pause-injector ktest fixture** — pause a slot at a chosen pause_state; useful for testing the pause-during-call paths.
- **Cap-forgery harness** — builds messages with caps the sender doesn't hold; asserts router rejects.
- **Equivalence-runner macro** — `ASSERT_EQUIVALENT(direct_call, wrapped_call)` — used by 8.0c for per-callsite migration tests.

### Slice 8.0a-specific ktests (~70)

| Group | Count | Coverage |
|---|---|---|
| Per-task slot lifecycle | ~10 | task_create registers caller_slot + 4 edges; task_destroy unregisters cleanly; lookup by container_of. |
| posix_shim component lifecycle | ~5 | init/enable/disable/destroy; deps auto-resolved; singleton accessor. |
| `nx_slot_call_blocking` happy path | ~10 | Single call returns rc; multi-call sequence on same task; per-op equivalence to direct call (mock component). |
| Pause-state transitions | ~12 | NONE/CUTTING/DRAINING/DONE on the dst slot during a call; QUEUE blocks, REJECT returns NX_EBUSY, REDIRECT routes; resume_waitq wake releases all. |
| In-flight counter | ~5 | Inc/dec across handler invocation; pause-drain waits; concurrent calls (host build) increment correctly. |
| Hook chain | ~10 | IPC_SEND fires with right ctx; IPC_RECV fires on reply; ABORT in SEND posts NX_EABORT reply; ABORT in RECV posts NX_EABORT reply; observe-only hooks don't change rc. |
| Cap-scan | ~6 | Forged slot_ref → NX_EINVAL; valid slot_ref via task→service edge passes; BORROW vs TRANSFER. |
| Reply path | ~8 | Reply dispatched to caller_slot; posix_shim_handle_msg writes reply_buf + wakes waitq; reply pool exhaustion asserts; per-CPU pool ownership. |
| ISR-context guard | ~2 | Calling `nx_slot_call_blocking` from ISR context panics (host-build assertion). |
| Identity property under PAUSE_NONE + no hooks + no caps | ~2 | Mock component with side-effect-counted handler; round-trip count equals direct-call count. |

### Negative tests

- Calling `nx_slot_call_blocking` with non-matching `src_slot` → KASSERT.
- Recursive call from same task (in_flight_reply_buf already set) → KASSERT.
- Cap-bearing call with NULL src (shouldn't happen post-Path-A but guard) → NX_EINVAL.

### Existing test baselines

- `make test` 453/453 stays green.
- `make test-interactive` 7/7 stays green.

Slice 8.0a should leave all production code paths unchanged — wrappers exist but no callers use them yet (8.0c migrates callsites). Existing tests run against the legacy direct-call paths; new ktests exercise the new infrastructure with a mock component.

---

## Open Items / Future Extensions

These are explicitly out of scope for slice 8.0a but the API is designed to accommodate them.

1. **Async-mode-per-edge** (slice 8.6). Today every blocking-call wrapper waits for a reply. A future fire-and-forget variant (`nx_slot_call_oneway`) returns immediately after enqueue; semantically matches DESIGN's `mode = ASYNC` for connections that don't need replies.

2. **Multi-thread tasks** (post-Phase-9). A future task struct may be one thread of many in a process; per-task slot registration extends naturally (one slot per thread). External destruction (e.g. `pthread_cancel`) of a blocked thread re-introduces the in-flight-counter discipline at task_destroy time, which slice 8.0a defers.

3. **Per-edge hook attachment** (Phase 8.6 or later). DESIGN.md §"Integration with Other Subsystems" envisions hooks attached to specific edges (e.g. `posix_shim → vfs` only), not just global per-point chains. Today's implementation (per-point chain + in-callback filter) is consistent in observable behavior; per-edge attachment is an optimization for high-frequency hooks.

4. **Multi-out_param replies.** If a future op needs multiple output-by-pointer params, the IDL gains `out_params: [...]` (already in IDL-SCHEMA.md's open items) and the reply struct grows multiple output fields. No API change.

5. **DEDICATED-mode destinations.** Slice 8.0a implements the dispatch branch for completeness; no production users in v1.

6. **Reply payload caps.** v1 reply structs carry only scalars (rc, byte counts, etc.). If a future op returns a `slot_ref` (e.g. config-handle API in slice 8.3 returning a slot reference for the live composition), the reply path's `nx_ipc_scan_recv_caps` already runs and the caller's wrapper handles cap retention.

7. **`NX_HOOK_SYSCALL_ENTER` / `_EXIT`** (slice 8.7). Distinct from IPC hooks; fires at SVC entry/exit. Not slice 8.0a's scope.

---

## References

- [Session 81 log](logs/session-81-slot-call-api.md) — the design discussion that produced this spec.
- [Session 80 log](logs/session-80-idl-schema.md) — the IDL schema discussion.
- [Session 79 log](logs/session-79-phase-8-plan.md) — the Phase 8 plan that motivated this work.
- [IDL-SCHEMA.md](IDL-SCHEMA.md) — the IDL spec; this doc covers the runtime side.
- [IMPLEMENTATION-GUIDE.md §Phase 8](IMPLEMENTATION-GUIDE.md#phase-8-runtime-recomposition-and-config-manager) — slice plan.
- [DESIGN.md](DESIGN.md) §"Invariants" — Dispatcher-only entry + slot-resolve locality.
- [DESIGN.md](DESIGN.md) §"Every Component Occupies a Slot" — the posix_shim mandate.
- [DESIGN.md](DESIGN.md) §"Pause Implementation" — pause-state state machine + (src, dst) hold-queue keying.
- [DESIGN.md](DESIGN.md) §"Slot References in IPC Messages" — R3 cap discipline.
- [framework/ipc.h](../../sources/nonux/framework/ipc.h) — existing message + cap struct.
- [framework/registry.h](../../sources/nonux/framework/registry.h) — slot/connection/component data structures.
- [core/sched/task.h](../../sources/nonux/core/sched/task.h) — task struct that gains `caller_slot`.
- [core/sched/waitq.h](../../sources/nonux/core/sched/waitq.h) — slice 7.8a wait-queue primitive that reply_waitq builds on.
