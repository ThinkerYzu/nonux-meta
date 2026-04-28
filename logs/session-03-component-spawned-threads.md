# Session 3: Component-Spawned Threads and Thread-Local Slot Dereferences

**Date:** 2026-04-18
**Phase:** Design extension (ahead of Phase 3 implementation)
**Branch:** master

---

## Goals

- Permit components to spawn their own worker threads for long CPU-bound work that fits neither the bounded-handler rule nor `dedicated` mode's single-threaded queue.
- Close a subtler version of the "slot leaked outside a dispatcher" loophole: a dispatcher handler resolving a slot and then handing the `impl*` to a component-spawned worker thread.
- Decide how to enforce the new rule without adding a new static checker — leverage the AI review rubric already in the design.

## What Was Done

### SPEC R18 — new sub-bullet

Added a "Component-spawned threads and thread-local dereferences" sub-bullet to functional requirement #18 (Component Graph Registry). Two rules:

1. **Dereferenced slot pointers are thread-local.** A dispatcher handler must not pass a resolved `impl*` to a worker thread. If a worker needs to reach another component, the component carries the **slot reference** into the worker (registered via `slot_ref_retain`, listed in the manifest's `retains` array), and the worker enqueues a message to the slot — a dispatcher resolves it. Dispatcher threads remain the only category permitted to dereference slots.
2. **Component drains its own threads.** Any component that spawns its own threads must declare a `pause_hook` in its manifest. Framework invokes it during the pause protocol's cutoff step; component is responsible for reaching worker quiescence within the pause deadline. Missing `pause_hook` on a thread-spawning component is a manifest validation error.

The sub-bullet also notes the preference order: try `dedicated` mode first (framework-managed thread + queue, no `pause_hook` needed). Use component-spawned threads only when the workload genuinely doesn't fit a simple message-queue dispatch pattern.

### DESIGN — new "Component-Spawned Threads" subsection under Execution Model

Inserted between "Bounded Handlers and Async Split" and "Concurrency Modes". Contents:

- **Scope.** Async split with completion messages handles long operations driven by external events (device I/O, timer, peer IPC). Component-spawned threads are for CPU-bound work with no external driver — rebuilding a large index, batch crypto with too-small units for per-op async split.
- **Rule 1 with good/bad code example.** The `bad_start_job` case hands `hasher = slot_resolve(...)` directly to `spawn_worker`; the `good_start_job` case hands `self` to the worker, and the worker uses `ipc_send(self->deps.hasher, req)` — a dispatcher resolves.
- **Rule 2 with hook signature.** `pause_hook` runs after the dispatcher cutoff barrier is visible but before the drain completion check; component signals workers, joins/parks, returns within the configured deadline (default 1 ms, per-slot configurable).
- **Enforcement split.** R8 static check catches the syntactic case (worker function dereferences a slot). The subtler dataflow case (handler copies resolved `impl*` into a struct passed to `spawn_worker`, crossing a `void*` thread-entry boundary) is delegated to the AI review rubric. Reviewer agent is the authority for this class.
- **Relationship to `dedicated`.** Clarified that `dedicated` is the framework-managed version of this pattern. Component-spawned threads are for pool-of-workers or blocks-on-non-framework-primitive shapes. Use `dedicated` first.

### DESIGN — Invariant #7 extended

Previously: slot dereferences must occur on a framework-owned dispatcher thread; ISRs and non-dispatcher kthreads enqueue messages.

Now also covers: **component-spawned worker threads enqueue messages but never dereference slots — including pointers handed to them by a dispatcher handler.** Points to the new Component-Spawned Threads subsection for the rationale.

### DESIGN — AI Review Rubric expanded

Added two checklist items to the registry review rubric:

- Cross-thread slot-dereference rule: no dispatcher handler passes a resolved `impl*` into a worker through arguments, captured state, or a shared struct. The rubric text explicitly names the reviewer agent as the authority for this item because the dataflow crosses user-supplied thread-entry boundaries that deterministic analysis cannot always resolve.
- `pause_hook` presence-and-quality: if the component spawns workers, the manifest declares a `pause_hook` and the hook reaches quiescence within the pause deadline.

### DESIGN — "Design Evolution" dated entry

Added a "2026-04-18 — Component-Spawned Threads" section summarizing the design shift, the enforcement split, and the tie-back to the project's AI-verified philosophy.

### HANDOFF

Added a new row to the Key Design Decisions table; session log entry.

## Key Findings

- The `dedicated` mode covers only a narrow slice of "I need long work." Pool-of-workers shapes or workers that block on non-framework primitives (e.g., waiting on a hardware doorbell that isn't wired through the IPC router) need something else — and the honest answer is "let the component spawn a thread, and hold it to two rules."
- The slot-leak loophole for component-spawned threads is a strict **superset** of the IPC-message loophole closed in Session 2. IPC leaks are fixed with typed caps + `slot_ref_retain`. Thread leaks can't be fixed the same way because the handoff is a plain `void *` to a user-supplied thread entry point — the framework has no hook to inspect it. This is why the AI review rubric is load-bearing here rather than just supplementary.
- Splitting enforcement (R8 for syntactic cases, AI rubric for dataflow cases) is the correct posture for this project. It's consistent with the Session 2 stance that AI reviewers are part of the build gate, not an optional quality pass. Adding a new R-check with a "framework-only thread spawn API" would have constrained every component author for one enforcement case that the rubric already handles.

## Decisions Made

- **Component-spawned threads are allowed** alongside dispatcher-served handlers, for CPU-bound work that fits neither async split nor `dedicated`. They are an escape hatch: dedicated first, component threads second, never as a default.
- **Dereferenced slot pointers are thread-local.** The rule extends slot-resolve locality to "the thread that resolved the slot" rather than just "a dispatcher thread category." Cross-thread pointer handoff is disallowed even if both threads belong to the same component.
- **`pause_hook` is a manifest-declared, component-owned drain mechanism.** The framework cannot drain threads it does not own; making the component responsible — and making `pause_hook` a validation error if it's missing when threads are spawned — is the minimum surface that preserves the pause protocol's guarantees.
- **AI review rubric is the enforcement mechanism for rule (1).** No new static checker, no framework-imposed thread-spawn API, no dataflow-analysis pass. The reviewer agent walks worker entry points and verifies no `impl*` is reachable from arguments or captured state. This is consistent with Session 2's commitment that compliance is AI-verified, not human-review-discipline.
- **R8 retains its original scope.** R8 is still "no slot dereference reachable from an ISR or non-dispatcher kthread call graph." It catches the syntactic case; it does not gain a new dataflow-analysis requirement for cross-thread pointer leaks.
- **Manifest gains a new optional field (`pause_hook`) and existing `retains` array carries slot refs worker threads will use.** No new manifest concept beyond the hook function pointer; `retains` already exists for runtime-acquired slot refs.

## Status at End of Session

- SPEC.md R18 extended with the cross-thread rule.
- DESIGN.md gained a "Component-Spawned Threads" subsection, extended Invariant #7, and new rubric items. Design Evolution dated entry added.
- HANDOFF.md Key Design Decisions updated; Session 3 entry added.
- No code changes this session — design-only.
- Phase 1 code still written-but-unbuilt (blocker unchanged: cross-compiler not installed).
- Implementation of `pause_hook` plumbing and rubric item enforcement folds into Phase 3 framework work; no new phase added.

## Next Steps

- Install cross-compiler and boot Phase 1 in QEMU (unchanged from Sessions 1 and 2).
- In Phase 3, wire `pause_hook` into the component descriptor emitted by `COMPONENT_REGISTER` and into `gen-config.py`'s manifest schema validation. Validation error if a manifest marks `spawns_threads: true` without declaring a `pause_hook`.
- When drafting the AI review checklist template (deferred from Session 2), include the two new rubric items so every new component gets them at generation time.

---

**Files Changed:**
- `SPEC.md` — new sub-bullet under requirement #18 ("Component-spawned threads and thread-local dereferences") with both rules, enforcement split, and preference-order note.
- `DESIGN.md` — new "Component-Spawned Threads" subsection under Execution Model (rule 1 with good/bad code example, rule 2 with `pause_hook` signature, enforcement split, relationship to `dedicated`); Invariant #7 extended to name component-spawned threads explicitly; Review Rubric gained two items; Design Evolution dated entry for 2026-04-18; header Last Updated bumped.
- `HANDOFF.md` — new Key Design Decisions row; Session 3 log entry at top; status stamp updated.
- `logs/session-03-component-spawned-threads.md` — this session log.
