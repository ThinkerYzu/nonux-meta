# Session 2: Component Graph Registry + AI-Verifiable Composition

**Date:** 2026-04-17
**Phase:** Design extension (ahead of Phase 3 implementation)
**Branch:** master

---

## Goals

- Introduce an explicit, first-class registry for the running composition so that instrumentation, hot-swap, and AI review have a single source of truth.
- Close the loophole where a component could receive a slot reference via IPC and hold it outside the registry's view.
- Make every registry rule both *followable* and *verifiable* by AI agents, not just by human review.

## What Was Done

### Component Graph Registry (SPEC + DESIGN)

Added functional requirement #18 to `SPEC.md` and a new top-level section to `DESIGN.md`. Universal registration: every slot, every slot reference held by a component, and every connection between caller-slot and callee-slot is registered with the framework. The registry owns `struct graph_registry` containing hash-indexed `slot_node` and `component_node` collections plus a list of `connection` edges.

Mutation goes through a fixed API surface (`slot_register`, `component_register`, `connection_register`, and their unregister/state-change siblings). Every mutation emits a `graph_event` and appends to a bounded append-only change log. Readers use a seqlock / RCU-style snapshot mechanism so instrumentation never blocks the kernel.

Writes are serialized through the existing recomposition lock — the registry does not introduce a new synchronization domain.

### Slot References in IPC Messages

Extended the message format with a typed `caps` array separate from payload bytes. Each `ipc_cap` has a `kind` (slot ref, handle, VMO chunk) and an `ownership` mode (`CAP_BORROW`, `CAP_TRANSFER`). Borrowed refs are dropped automatically when the handler returns; transferred refs must be claimed (via `slot_ref_retain`) or released before return. The IPC router scans the cap array on send (reject forged refs) and on receive (drop borrows, flag unclaimed transfers, verify retention calls registered new edges).

`slot_ref_retain` can fail when retention would violate concurrency contracts, mutability, cycle invariants, or policy. This is the sole path for a component to hold a received slot reference beyond the handler's scope.

### AI-Enforceable Compliance

The registry rules matter only if they're mechanically enforced. Added an "AI-enforceable compliance" clause to SPEC #18 and an "AI Verification" subsection to DESIGN, naming seven static checks (R1–R7) for a `make verify-registry` analyzer:

1. R1 — no fabricated slot pointers
2. R2 — no `struct slot *` field outside manifest declarations
3. R3 — slot refs only in caps, never in payload fields
4. R4 — retain/release pairing reachable from `disable`/`destroy`
5. R5 — sender owns every slot it passes in a cap
6. R6 — handlers don't stash borrowed caps
7. R7 — generated registration code matches manifest (regenerate + diff)

Runtime assertions are added to the test harness: pointer audit against registry, generation fence after swaps, event-trace matching. Every component's `README.md` will carry a versioned AI review checklist regenerated when framework rules evolve. `make verify-registry` becomes a build prerequisite — failure to pass means the build fails, like `-Werror`.

### Documentation Updates

Beyond SPEC and DESIGN, updated:

- `IMPLEMENTATION-GUIDE.md` — added `framework/registry.c/.h`, `tools/verify-registry.py`, `test/host/registry_assert.c/.h` to the file tree. Rewrote Phase 3 steps to name the registry, cap scanning, and retain/release explicitly. Added `make verify-registry` to the build/test section.
- `HANDOFF.md` — new rows in the Key Design Decisions table (Registry, Change history, Slot refs in messages, AI verification). Session 2 log entry.
- `README.md` — new "Component Graph Registry" Core Concept section; promoted "AI-Operable" to "AI-Operable and AI-Verified" noting the build-gate enforcement.

### Dependency Injection Mechanism (follow-up)

A gap surfaced during Q&A after the initial registry work was documented: the docs named `resolve_deps()` but never showed how it knows the offset of each dep field inside a component's state struct. SPEC #18 R7 asserts "generated registration code matches the manifest" — the *rule* was there but the *mechanism* was not.

Closed the gap by adding a "Dependency Injection Mechanism" subsection to DESIGN.md under Dependency Management. Key pieces:

- **`resolve_deps()` body** — a fixed, generic loop that walks a `const struct component_descriptor *` and writes slot pointers into the component using descriptor-supplied offsets. Simultaneously calls `connection_register()` so the edge exists in the registry before `init()` runs.
- **Generated `gen/<component>_deps.h`** — `tools/gen-config.py` emits the typed `<component>_deps` struct (one `struct slot *` per manifest dep) plus a `_DEPS_TABLE(CONTAINER, FIELD)` macro that expands to `dep_descriptor` initializers with `offsetof()` into a caller-supplied container type.
- **`COMPONENT_REGISTER` macro** — single-line author-side expansion that builds a static `struct component_descriptor` in a dedicated `.components` linker section. The macro is where `offsetof()` gets evaluated, because only the component's translation unit sees the container type's full layout.
- **Framework types** — `struct dep_descriptor` (name, offset, required, version, mode, stateful, policy) and `struct component_descriptor` (name, state_size, deps_offset, deps array, ops).
- **R7 enforcement** — `verify-registry.py` regenerates the deps header on every build and byte-diffs it against the committed copy. Drift fails the build.
- **Runtime-acquired refs** — slot refs obtained via `slot_ref_retain` or `slot_lookup` at runtime are not in the descriptor; the component must list them in its manifest's `retains` array so the pointer audit covers both static and dynamic paths.

Also updated `IMPLEMENTATION-GUIDE.md` Phase 3 step 6: `gen-config.py` output list now includes `gen/<component>_deps.h` with a diff-check note. File tree under `gen/` lists the same.

### v1 Execution Model (follow-up 2)

Earlier sessions had named three concurrency modes (`single-threaded`, `thread-safe`, `thread-per-instance`) but never committed to *how components actually run*. A user question — "has each processor a dedicated thread to consume message queue? So, slot can be dereferenced only in these thread. It is impossible doing hot-swap while other threads are using the component from a slot" — surfaced that the design tacitly assumed an execution model without spelling it out.

Followed the thread through:

1. **Per-CPU dispatcher** is indeed the target model. Each CPU runs one pinned kernel thread whose loop is `dequeue → resolve slot → handler → repeat`. Only the dispatcher ever executes component code on that CPU.
2. **Preemption needs a clear rule.** Handlers run non-preemptibly (preempt_disable across the handler body). Interrupts still fire, but ISRs can only enqueue messages, never call into a component. User tasks (EL0) stay preemptible — the scheduler runs on the dispatcher in its own handler and preempts EL0 normally.
3. **Bounded-handler discipline.** Long operations must split into "start op" → later "completion" message. This is the project's answer to the question "what if a handler blocks?": it isn't allowed to.
4. **Thread-per-component rejected.** The user asked whether giving each component its own thread would solve the blocking concern more cleanly. It would, but at the cost of 30× more kernel threads, ~3× more context switches on every sync chain, and ~240 KB of stacks sitting idle — all to solve a case that bounded-handler + async split already solves. Accepted `dedicated` as an explicit escape hatch for the rare component that genuinely cannot bound its handlers; v1 minimum config uses zero.

New concurrency mode vocabulary replacing the old three:

- **`shared`** (default) — single instance, any CPU's dispatcher may enter concurrently. Component owns its cross-CPU sync (atomics preferred).
- **`serialized`** — single instance, framework holds a per-slot spinlock across each handler call. Used by components that don't want to manage their own sync.
- **`per-cpu`** — N instances, each CPU's dispatcher only reaches its local instance. No cross-CPU races.
- **`dedicated`** — own private thread and queue (escape hatch).

Code changes:

- `SPEC.md` #13 — "Concurrency mode" bullet rewritten; new "Execution model" bullet added above it with the dispatcher / non-preemptible-handler / bounded-handler rules. Manifest field list updated.
- `DESIGN.md` — replaced the "Concurrency Enforcement" subsection with a full "Execution Model (v1)" subsection: dispatcher loop in code, preemption rule, bounded-handler rule with before/after code example, concurrency-mode table, `slot_dispatch` switch covering all four modes, framework-enforcement notes, "Why this shape" rationale. Stale references to old mode names elsewhere in DESIGN.md updated. TOC gained the Execution Model entry.
- `HANDOFF.md` — replaced the "Concurrency" row in Key Design Decisions with two rows (Execution model + Concurrency modes).

## Key Findings

- The slot registry already needed to exist for hot-swap to work (it's what `component_swap` mutates), but it was implicit — documented as "framework maintains a slot → active component mapping" with no API surface for instrumentation or history. Promoting it to a first-class entity clarified that most of the existing design already assumed it.
- Slot references carried in IPC are a real leak in the original design. Without typed caps, a receiver could obtain a slot pointer through `memcpy`ing a payload byte range — the framework would have no visibility into the resulting edge. Typed caps make the scan target explicit.
- AI verification is qualitatively different from human review: the rubric has to be enumerable and deterministic, so rules like "no fabricated slot pointers" must be expressible as a static check, not a judgment call.
- The original DESIGN.md described the *rule* for dep injection (R7 manifest-as-source-of-truth) and the *interface* (`resolve_deps()` is called) but skipped the *mechanism* (how offsets flow from manifest to framework). This is the kind of gap that breaks AI-driven development: an AI writing `gen-config.py` or a new component has to invent a convention, and different invocations will pick different ones. Every design doc needs to go down to the offset-level mechanism when AI is a primary implementer.

## Decisions Made

- **Registry is v1, not deferred** — The hooking framework (SPEC #9) is v1, and hooks target composition boundaries. Those boundaries must be enumerable at runtime, which requires an explicit registry. Deferring the registry would mean deferring meaningful instrumentation.
- **Slot refs travel as typed capabilities, not payload pointers** — Scannability for the router and localized translation for future dynamic loading both argue for typed caps.
- **Retention is explicit, not inferred** — Keeping a received slot ref past handler return requires `slot_ref_retain`. The framework does not guess intent based on whether the pointer was written to `self`.
- **AI compliance is a build gate** — `make verify-registry` must pass before `make` / `make test` succeed. Parallels `-Werror` — framework rules are not optional suggestions.
- **Change log is bounded and in-kernel** — A ring buffer keyed by generation, readable from userspace via a `kstat`-like file. Offline analysis uses the JSON snapshot of the log.
- **Offset comes from the component's TU, not the generator** — `offsetof()` is evaluated inside the component's compilation unit via the `COMPONENT_REGISTER` macro because only that TU sees the full container layout. The generator emits the deps struct and the `_DEPS_TABLE(CONTAINER, FIELD)` macro; it cannot emit `offsetof(struct sched_component, ...)` because it does not know the container type. This keeps the generator component-agnostic and the component's container type a local concern.
- **Descriptor covers static deps only; `retains` array covers dynamic** — Slot refs acquired at runtime via `slot_ref_retain` or `slot_lookup` are declared in the manifest's `retains` array, not the `requires`/`optional` lists. The test harness's pointer audit consults both so nothing escapes registration.
- **Per-CPU dispatcher with non-preemptible handlers is the v1 execution model** — Component code only runs on a pinned per-CPU dispatcher thread. The handler body is non-preemptible (interrupts enabled but forbidden from calling into components). User tasks (EL0) remain fully preemptible. This is what makes hot-swap tractable, the registry's pointer-audit invariant enforceable, and the no-re-entrancy guarantee hold.
- **Bounded-handler rule + async split is the v1 blocking answer** — Handlers must complete in bounded time; anything long splits into "start op" → "later completion message." Rejected thread-per-component as the answer to the blocking concern: it is over-engineered for v1 at substantial cost (many threads, many context switches, no real payoff when handlers are naturally short). `dedicated` mode is the documented escape hatch; zero v1 components use it.
- **Concurrency modes rewritten** — `shared` (default) / `serialized` (framework lock) / `per-cpu` / `dedicated` replaces the prior `single-threaded` / `thread-safe` / `thread-per-instance` vocabulary. Reason: the old names conflated "how the component is instantiated" with "who provides the lock", and `thread-safe` suggested general thread safety while really meaning "skip framework lock."

## Status at End of Session

- SPEC.md, DESIGN.md, IMPLEMENTATION-GUIDE.md, HANDOFF.md, README.md all updated.
- No code changes this session — design-only.
- Phase 1 code still written-but-unbuilt (blocker unchanged: cross-compiler not installed).
- Registry implementation work is folded into Phase 3 steps; no new phase added.

## Next Steps

- Install cross-compiler and boot Phase 1 in QEMU (unchanged from Session 1).
- When Phase 3 begins, follow the expanded step list in `IMPLEMENTATION-GUIDE.md`: write `framework/registry.c` first (it's the substrate for lifecycle, IPC, and hooks), then the rest of the framework on top of it.
- Draft the AI review checklist template for `README.md` in new components (will be generated from the registry rule set).

---

**Files Changed:**
- `SPEC.md` — requirement #18 (Component Graph Registry) added, including AI-enforceable compliance clause. Requirement #13 concurrency rewritten: new "Execution model" bullet (per-CPU dispatcher, non-preemptible handlers, bounded handlers) and new four-mode concurrency vocabulary (`shared` / `serialized` / `per-cpu` / `dedicated`); manifest field list updated.
- `DESIGN.md` — new top-level section "Component Graph Registry" with structs, APIs, invariants, slot-refs-in-messages subsection, AI Verification subsection; TOC entry added. Follow-up: new "Dependency Injection Mechanism" subsection under Dependency Management covering `resolve_deps()` body, generated `gen/<component>_deps.h`, `COMPONENT_REGISTER` macro, `struct dep_descriptor` / `struct component_descriptor`, and `.components` linker section. Follow-up 2: replaced "Concurrency Enforcement" subsection with a full "Execution Model (v1)" subsection (dispatcher loop, non-preemptible-handler rule, bounded-handler rule with code example, four-mode table, `slot_dispatch` switch, framework-enforcement notes, "Why this shape" rationale); stale references to old mode names updated; TOC entry added.
- `IMPLEMENTATION-GUIDE.md` — Phase 3 expanded; file tree updated with registry, verify-registry, registry_assert; `make verify-registry` added to build pipeline. Follow-up: Phase 3 step 6 now names `gen/<component>_deps.h` as a generator output; `gen/` tree entry lists the per-component deps header.
- `HANDOFF.md` — Key Design Decisions table gained four rows for registry work; "Concurrency" row replaced by two rows (Execution model + Concurrency modes); Session 2 log entry inserted at top with three follow-up paragraphs; status stamp updated.
- `README.md` — new Component Graph Registry concept section; AI-Operable section renamed to AI-Operable and AI-Verified.
- `logs/session-02-registry-and-ai-verification.md` — this session log.
