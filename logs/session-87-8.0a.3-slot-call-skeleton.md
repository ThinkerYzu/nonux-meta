# Session 87: slice 8.0a.3 — `framework/slot_call.{h,c}` skeleton

**Date:** 2026-04-30
**Phase:** Phase 8 — Group B (slice 8.0a, sub-slice 3 of 8)
**Branch:** master (source-side commit `def01b5` in `sources/nonux/` happened end-of-Session-86 wallclock but is the 8.0a.3 deliverable; this session's checkpoint is the proj_docs paperwork that records it)

---

## Goals

- Land slice **8.0a.3** — the call-surface skeleton that subsequent
  sub-slices target with `extern` references:
  - 8.0a.4 (kernel `posix_shim` component) issues blocking calls into
    vfs / scheduler / mm / char_device.serial via this entry.
  - 8.0a.5 (per-task `caller_slot`) wires `msg->src_slot` to the task's
    embedded slot.
- Keep the slice trivial — header + stub body returning `NX_ENOSYS`,
  no callers, no behavior change.  The full body — pause-state handling,
  hook chain, cap-scan, dispatcher enqueue, reply-waitq block, ABORT-path
  reply synthesis — lands in slice 8.0a.6 alongside the per-CPU reply
  pool and dispatcher integration.

## What Was Done

### Slice 8.0a.3 (commit `def01b5`, `sources/nonux/`)

Pure additive scaffolding.  Two new files + two Makefile insertions.

**`framework/slot_call.h` (70 lines).**  The public API surface for
blocking calls.  Doc strings match SLOT-CALL-API.md §"API" verbatim:

```c
int nx_slot_call_blocking(struct nx_slot       *slot,
                          struct nx_ipc_message *msg,
                          void                  *reply_buf,
                          size_t                 reply_buf_len);
```

Header comments cover:
- Slice provenance (8.0a sub-sliced 8.0a.3 → 8.0a.8) so readers can map
  declared-but-unimplemented behavior to its landing slice.
- v1 single-in-flight-call constraint (asserted via
  `task->in_flight_reply_buf` non-NULL on entry — lands 8.0a.5).
- Caller-must-be-a-task constraint (ISRs and kthreads can't call this;
  they don't have caller_slots).
- Cross-references DESIGN.md §"Bounded Handlers and Async Split"
  (a 10 ms handler blocks the syscall caller for 10 ms).
- Error-code table — `NX_EBUSY` (REJECT), `NX_ELOOP` (REDIRECT depth
  cap), `NX_ENOENT` (no active impl with `handle_msg`),
  `NX_EABORT` (hook-chain ABORT), `NX_EINVAL` (NULL/mismatched args).
- Explicit "8.0a.3 stub: returns `NX_ENOSYS` for any well-formed call"
  callout, with the round-trip body landing in 8.0a.6.

**`framework/slot_call.c` (30 lines).**  Stub body:

```c
int nx_slot_call_blocking(struct nx_slot       *slot,
                          struct nx_ipc_message *msg,
                          void                  *reply_buf,
                          size_t                 reply_buf_len)
{
    if (!slot || !msg) return NX_EINVAL;
    (void)reply_buf;
    (void)reply_buf_len;
    return NX_ENOSYS;
}
```

NULL-arg check returns `NX_EINVAL` so partially-wired callers fail
cleanly with the "you passed bad args" code rather than the "this op
is not implemented yet" code.  All other well-formed calls return
`NX_ENOSYS` until 8.0a.6.

**Wiring.**  Two-line Makefile additions:
- `Makefile`: `framework/slot_call.c` added to `FW_C` after
  `framework/ipc.c` (close to its conceptual peer in the link order).
- `test/host/Makefile`: same insertion in `SRCS` so host tests can stub
  cross-component calls through the same entry once consumers land.

### Doc-side paperwork (this session's `proj_docs/nonux/` commit)

- This session log (`logs/session-87-8.0a.3-slot-call-skeleton.md`).
- HANDOFF.md "Current Status" + Phase checklist + Next Actions —
  ☑ slice 8.0a.3, advance forward step to slice 8.0a.4.
- HANDOFF.md Session Logs — Session 87 entry prepended; Session 82
  pruned to keep the "last 5" rolling window.
- IMPLEMENTATION-GUIDE.md Group B table — "✓ Landed Session 87" stamp
  on the slice 8.0a.3 row.

## Decisions Made

### Reply buffer as explicit `(buf, len)` arg pair vs. growing `nx_ipc_message`

Locked at Session 81; reaffirmed at Session 86 spec refinement; the
header now codifies it.  The IPC carrier stays generic (no reply-buffer
field bleeds into `struct nx_ipc_message`); the reply-buffer concept is
specific to the blocking-call wrapper protocol.  Async-mode-per-edge
(slice 8.6) and `nx_slot_call_oneway` (post-Phase-8) won't carry a
reply buffer at all — the carrier's payload-only shape supports both
patterns.

### Stub returns `NX_ENOSYS` not `NX_OK`

Any accidental caller (test fixture, partial wiring during 8.0a.4 or
8.0a.5 development) gets a hard-fail return code rather than silently
"succeeding" with no effect.  `NX_ENOSYS = -38` was added in the slice
3.x errcode bring-up (matches Linux ABI for "Function not implemented");
no new errcode added by this slice.

### NULL-arg branch returns `NX_EINVAL` distinct from `NX_ENOSYS`

Two reasons: (1) future ktest fixtures can distinguish "I passed garbage"
from "the op isn't wired yet"; (2) the NULL-arg check is genuine input
validation that the full 8.0a.6 body will also need — keeping it in the
stub means the validation logic is exercised even before the round-trip
body lands.

### No tests this slice

`tools/tests/` is for IDL/codegen pytest suites (slice 8.0pre.\* lineage).
`test/host/` and `test/kernel/` cover production paths.  Calling a stub
that just returns `NX_ENOSYS` doesn't justify a ktest entry — the next
substantive ktests are 8.0a.5 lifecycle tests for `caller_slot`, and the
real `nx_slot_call_blocking` round-trip ktests land in 8.0a.6 + 8.0a.8.

## Findings

- The header comment block is essentially a contract excerpt from
  SLOT-CALL-API.md.  Keeping the two in sync is the maintenance cost
  of having the API doc-per-spec.  Drift risk is bounded — the header
  is the public surface that callers see; the spec is the design
  rationale.  If the public surface changes the header changes; the
  spec follows.

- `framework/ipc.h` already defines `NX_MSG_FLAG_REPLY_REQUESTED` (slice
  8.0a.2); `framework/registry.h` already carries `resume_waitq` and
  `in_flight_calls` on `struct nx_slot` (slice 8.0a.2).  The 8.0a.3
  header includes `framework/ipc.h` which transitively pulls
  `framework/registry.h`, so all the types the API references are in
  scope at the use site.

- Link-order placement matters mildly: `slot_call.c` should land after
  `ipc.c` in `FW_C` because the body (when it lands in 8.0a.6) calls
  `nx_dispatcher_enqueue` and `nx_ipc_scan_send_caps`, both defined in
  earlier framework files.  Placing it next to `ipc.c` keeps the layer
  reading order consistent with the dependency direction.

## Tests

- `make test` → **495/495 pass** (93 python + 283 host + 119 kernel —
  same baseline as Session 86's 8.0a.2 close).  Pure scaffolding with no
  consumer means zero behavior delta.
- `make test-interactive` → **7/7 pass** (echo_cat, echo_hello,
  echo_pipe, ls_root, mkdir_tmp, ps_smoke, visible_prompt).
- `make verify-iface-fresh` → 0 drift.
- `make verify-registry` → 0 findings.

## Next Steps

Slice **8.0a.4** — kernel `components/posix_shim/` skeleton:
- New `components/posix_shim/` directory with `manifest.json`,
  `posix_shim.c` (init/enable/destroy stubs), `README.md`.
- `manifest.json` declares deps `{vfs, scheduler, mm, char_device.serial}`,
  all `mode: async, stateful: true (vfs)/false (others), policy: queue`.
- `kernel.json` gains a `posix_shim` slot and an entry binding
  `posix_shim ← posix_shim`.
- Auto-generated `gen/posix_shim_deps.h` plumbing follows the existing
  dependency-injection mechanism.
- New `posix_shim_handle_msg` is a stub returning `NX_EINVAL` — real
  reply-routing body lands in 8.0a.6 alongside the dispatcher integration.
- Booted kernel reports `composition (gen=N, 7 slots, 7 components)`
  with `posix_shim` ACTIVE alongside the existing six.
- No callers of the new slot yet — `framework/syscall.c` keeps its
  direct `vops->read(self, ...)` calls until slice 8.0c.
