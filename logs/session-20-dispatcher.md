# Session 20: slice 3.9b.1 — dispatcher kthread + MPSC inbox

**Date:** 2026-04-22
**Phase:** 3 slice 3.9b.1 (runtime dispatcher upgrade — mechanics)
**Branch:** master

---

## Goals

Move `nx_ipc_dispatch` off the caller's thread onto a framework-owned
dispatcher kthread. Replace the per-slot FIFO inbox with a global MPSC
lock-free queue (on the kernel build). Keep every existing host IPC /
pause / hook test green by retaining the per-slot inbox as the host
build's backing store.

Net effect: DESIGN.md §Execution Model — Per-CPU Dispatcher Loop goes
from sketched-only to wired.

## What Was Done

### `core/lib/mpsc.h` — new, header-only

Vyukov-style multi-producer single-consumer intrusive queue. Producer
push is a single `atomic_exchange` on the tail + a release store of the
previous node's `next` pointer. Consumer pop walks head-forward, with
the classic tail-catch (push-the-stub) dance when head == tail.

Makes `nx_ipc_enqueue_from_irq` trivially R8-safe: an ISR that holds
a pre-built `struct nx_ipc_message` enqueues in constant instructions,
no `slot->active` deref, no allocation, no blocking.

### `framework/dispatcher.{h,c}` — new

Single global MPSC (v1 single-CPU; SMP gets per-CPU queues later). API:

- `nx_dispatcher_init()` — spawns the dispatcher kthread on kernel via
  `sched_spawn_kthread`; on host it just initialises the queue.
- `nx_dispatcher_enqueue(msg)` — ISR-safe push.
- `nx_dispatcher_pump_once()` — pops once and runs IPC_RECV hook +
  handler. The kthread body is `while (pump_once()) ; yield();`.
  Host tests call it directly as their pump.
- `nx_dispatcher_reset()` — test-only queue drain.

`disp_node` (a `struct nx_mpsc_node`) is embedded in `struct nx_ipc_message`
alongside the existing `_next` so a message can be on at most one of
{per-slot inbox on host, dispatcher MPSC on kernel, per-edge hold
queue on either}. They never overlap — the pointer field and the
mpsc node live in different storage.

### `framework/ipc.c` — host vs kernel split

- The per-slot-inbox apparatus (`g_queues`, `ipc_slot_queue`, `enqueue`,
  `dequeue`, `queue_for`, the original `nx_ipc_inbox_depth`) is now
  guarded `#if __STDC_HOSTED__`. Kernel build drops all of it.
- `do_send`'s async branch: host enqueues to per-slot inbox as before;
  kernel calls `nx_dispatcher_enqueue`.
- `nx_ipc_dispatch(slot, max)` is host-only logic; kernel returns 0.
- `nx_ipc_enqueue_from_irq` added: host returns `NX_EINVAL` (no
  dispatcher thread); kernel alias for `nx_dispatcher_enqueue`.

Existing host tests (100+ cases in `component_ipc_test.c` /
`component_pause_test.c` / `component_hook_test.c`) keep working
because the host path is identical to slice 3.8 behaviour.

### `framework/bootstrap.c`

After `sched_init` succeeds, call `nx_dispatcher_init()`. Runs before
`irq_enable_local`, so the first tick finds a ready MPSC.

### Test counts

- Python: **51** (unchanged).
- Host:   **163** (153 → +10). `dispatcher_test.c`: 4 raw MPSC cases
  (empty-pop, single roundtrip, FIFO-N, interleaved push/pop), 6
  dispatcher cases (empty-pump, single handler invocation, N-FIFO,
  NULL-args rejection, reset drops pending, IPC_RECV hook fires
  before handler).
- Kernel: **25** (22 → +3). `ktest_dispatcher.c`:
  `dispatcher_kthread_is_spawned_by_bootstrap` (enqueue then yield;
  handler observably runs — proves the kthread is alive, not a stub),
  `dispatcher_enqueue_from_irq_reaches_handler` (same path via
  `nx_ipc_enqueue_from_irq`), `dispatcher_drains_multiple_messages_
  in_order` (4-message FIFO).
- Total:  **239 / 239 pass**, 0 leaks, 0 errors.
- `make verify-registry`: 0 findings.
- Non-ktest `make && qemu` boots cleanly — idle loop runs with the
  dispatcher kthread present on the runqueue.

## Decisions

**Host / kernel split, not uniform MPSC.** The alternative was to
replace the host per-slot inbox with the same MPSC and rewrite every
test's `nx_ipc_dispatch(slot, max)` call. That's 16 call-sites across
2 test files — doable but creates churn without real correctness
payoff. Following the established precedent (`kheap.c`,
`cpu_switch_to`, timestamp handling in `registry.c`), host and
kernel genuinely diverge under `#if __STDC_HOSTED__`. The `nx_ipc_send`
contract is identical either way.

**Dispatcher kthread yields when empty, not blocks.** The full
DESIGN.md text calls for `msg_queue_dequeue_wait` (block until a
producer wakes). Slice 3.9b.1 uses cooperative `nx_task_yield` after
draining — when no messages are pending, the dispatcher gives up the
CPU until the scheduler picks it again. A proper wait-queue is a
later optimization (and requires a wake-up primitive that doesn't
exist yet). The current shape is correct, just not as efficient as
it could be.

**`disp_node` is a separate field, not a union with `_next`.** A union
would save 8 bytes per message but aliases the storage — a bug where
a message ended up on two queues simultaneously would silently corrupt
both. Clean separation makes the "on exactly one list" invariant
visible.

**`nx_ipc_enqueue_from_irq` is NX_EINVAL on host.** Host tests that
want to simulate an ISR-enqueue call `nx_dispatcher_enqueue` directly.
Making the ISR-specific entry function a no-op on host means any test
that accidentally relies on ISR-specific behaviour fails loudly, not
silently passes.

**Dispatcher reset uses `nx_mpsc_init` to drop pending.** `nx_mpsc_pop`
doesn't free nodes (they're caller-owned), so after pop-loop the queue
is in a valid but non-canonical "post-drain" state. Re-init gets back
to the canonical stub-at-head state so the next push/pop sequence
starts clean. Acceptable because `reset` is test-only and not called
concurrently with producers.

## What's Next

**Slice 3.9b.2 — correctness polish.** Pause-failure rollback (closes
the slice-3.8 DRAINING-on-failure bug). `pause_hook` 1 ms wall-clock
deadline enforced against `cntpct_el0`. Runtime `NX_HOOK_SLOT_SWAPPED`
dispatch from the registry's swap path.

That closes 3.9b and Phase 3 entirely. Phase 5 (virtual memory + handle
system + first userspace) comes after.

## Files Touched

- `core/lib/mpsc.h` — new.
- `framework/dispatcher.{h,c}` — new.
- `framework/ipc.h` — `disp_node` field; `nx_ipc_enqueue_from_irq`
  declaration; nx_ipc_dispatch doc updated.
- `framework/ipc.c` — per-slot inbox guarded under `__STDC_HOSTED__`;
  async path + dispatch + enqueue_from_irq split by build.
- `framework/bootstrap.c` — `nx_dispatcher_init()` after `sched_init`.
- `Makefile` — `FW_C` gains `framework/dispatcher.c`; `KTEST_C` gains
  `test/kernel/ktest_dispatcher.c`.
- `test/host/Makefile` — `SRCS` gains dispatcher_test.c + the
  framework/dispatcher.c cross-compiled source.
- `test/host/dispatcher_test.c` — new (10 tests).
- `test/kernel/ktest_dispatcher.c` — new (3 tests).
- `proj_docs/nonux/HANDOFF.md` / `README.md` — status + slice 3.9b.1
  check-off.
- `proj_docs/nonux/logs/session-20-dispatcher.md` — this log.
