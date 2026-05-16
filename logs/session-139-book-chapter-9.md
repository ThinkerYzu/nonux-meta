# Session 139: Book chapter 9 — Slots, components, and the registry

**Date:** 2026-05-16
**Phase:** Phases 1–9b complete; book chapters 1–9 shipped (Part I + Part II + Part III + Part IV complete; Part V chapter 9 of 2 shipped); Phase 9 not yet started
**Branch:** master

---

## Goals

- Draft the next book chapter, the first of Part V (Framework
  basics).  Per
  [BOOK-OUTLINE.md](../BOOK-OUTLINE.md#chapter-list) that is
  **chapter 9: Slots, components, and the registry**.  This
  chapter introduces the framework vocabulary that every
  remaining chapter leans on (chapter 10 IDL, chapter 11 IPC,
  chapter 12 filesystems, chapter 13 hooks, chapter 14
  recomposition).
- Cover slots, components, the descriptor / instance split,
  the six-state lifecycle, manifest-driven dependency
  injection, the `nx_components` linker section, the bootstrap
  walk that turns `kernel.json` into a live registry, and one
  worked end-to-end trace.

## What Was Done

### `sources/nonux/book/09-slots-components-and-registry.md` (new, 1587 lines)

Inside the 600–1500-line sweet spot per
[BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md#length-and-pace) —
slightly over (1587), well under the 2000-line split threshold;
this is the first framework chapter and introduces the bulk of
the framework vocabulary, so going long is the right trade.

Section structure:

1. **Intro** (3 paragraphs) — names the three questions the
   chapter answers (what is a slot, what is a component, what
   is the registry); back-references chapter 8's hint that
   round-robin and fixed-priority both fill the `scheduler`
   slot; forward-points to chapter 10 (IDL) and 11 (IPC) as
   the immediate consumers of this chapter's vocabulary.
2. **Files** — `framework/registry.{h,c}`,
   `framework/component.{h,c}`, `framework/bootstrap.{h,c}`,
   `kernel.json`, `gen/slot_table.c`, `core/boot/linker.ld`
   (for the `nx_components` section), plus `sched_rr` (as a
   simple component example) and `posix_shim` (as a component
   with non-trivial deps).
3. **Terms** (14 entries) — component, slot, interface,
   connection, manifest, kernel.json, component graph
   registry, `nx_components` section, component descriptor,
   component instance, lifecycle state, dependency injection,
   topological order, change event, snapshot.  Every term is
   the standard kernel-framework name; no invented metaphors
   per [BOOK-STYLE-GUIDE.md §Terminology rules](../BOOK-STYLE-GUIDE.md#terminology-rules).
4. **A quick analogy first** — motherboard slots + typed
   components.  Used per the style guide's "at most one
   whole-chapter framing analogy, only at the start" rule.
   Explicitly calls out where the analogy stops working
   (motherboard traces vs first-class connection edges).
5. **What's actually in `kernel.json`** — the boot
   composition file, simplified by removing the empty
   `connections`/`hooks` keys.  Calls out the qualified slot
   names (`char_device.serial`, `filesystem.root`) and the
   `alternatives` array (the dual-scheduler trick from
   chapter 8).  Sidebar: why JSON instead of a `.c` table
   (AI-operability argument).
6. **What a slot is — `struct nx_slot`** — field-by-field
   walk.  `name`, `iface`, `mutability`, `concurrency`,
   `active` (the load-bearing field — the indirection point
   that makes hot-swap transparent), `fallback`,
   `pause_state`, `resume_waitq`, `in_flight_calls`.  Sub-section
   on where slot storage comes from (caller-owned: gen-config,
   test fixtures, or component-local).
7. **What a component is — `struct nx_component` and the
   descriptor** — the descriptor/instance split, the seven
   fields of `struct nx_component_descriptor`, the
   five fields of `struct nx_component`, why `state_size` and
   `deps_offset` exist (so the framework can allocate state
   without knowing the component's struct shape), and the
   distinction between `ops` (framework-facing lifecycle) and
   `iface_ops` (interface-specific operations).
8. **The `NX_COMPONENT_REGISTER` macro** — quoted verbatim
   from `framework/component.h`, then walked: the static deps
   table, the const descriptor, the
   `__attribute__((section("nx_components"), used))`
   annotation, why `KEEP` and `used` matter, the
   `__start_/_stop_` markers that GCC/ld auto-generate
   because the section name is a valid C identifier.  Side
   note: why a linker section instead of a constructor-based
   registry (browsable list at link time, explicit topo order
   instead of link order).
9. **Manifests and dependency injection** — `posix_shim`'s
   manifest as the example (four required deps with mixed
   stateful/policy attributes), the generated
   `posix_shim_deps.h` (typed struct + `offsetof`-based
   macro), how the component author embeds the deps in their
   state, the macro plumbing that joins container-offsetof
   with field-offsetof.
10. **`nx_resolve_deps`: how slot pointers get installed** —
    the three-step loop: lookup → write typed pointer →
    register edge.  Calls out the "talks-but-forgets-edge"
    bug the framework can't detect at runtime but
    `verify-registry` flags.
11. **The component lifecycle** — the ASCII state diagram
    pasted from `framework/component.h`, the six states one
    by one, the six lifecycle verbs, the five-step pattern
    every verb shares (check state → fire hook chain → call
    ops callback → transition registry state → return OK).
12. **The Component Graph Registry** — three linked lists,
    per-slot edge indexes, the generation counter, the
    mutation API for slots/components/connections, what
    `struct nx_connection` carries (from/to/mode/stateful/
    policy/installed_gen), the three pause policies (queue/
    reject/redirect), lookup, traversal, change events, the
    bounded change log, and snapshots with reference
    counting + JSON serialisation.  Side note: why a
    separate snapshot type (decoupled readers/writers,
    stable view).
13. **Bootstrap: how the registry is filled at boot** — four
    steps: slot register, component register/bind,
    topological `init`+`enable`, scheduler/dispatcher
    publish.  Kahn's algorithm explained with the actual
    `deps_ready` check; the two failure modes (`NX_ENOENT`
    on missing required dep, `NX_ELOOP` on cycle).
14. **A worked example: tracing `posix_shim` from manifest
    to ACTIVE** — end-to-end narrative from link time through
    every event, generation bump, and state transition for
    one component.  Concrete: descriptor in `nx_components`
    section, slot register, allocation, bind via
    `nx_slot_swap`, dependencies-not-ready wait, then
    `nx_resolve_deps` writes four typed pointers + emits four
    `CONNECTION_ADDED` events, `init` runs, `enable` runs,
    component lands `ACTIVE`.
15. **What the registry buys us** — five bullets: single
    point of truth, decoupled introspection, hot-swap basis,
    AI-readable kernel image, test fixtures with full
    control.  Plus a one-paragraph "the cost is the
    bookkeeping discipline" close.
16. **A few extra things to know** (7 side notes) —
    caller-owned slot/component storage, `nx_slot_unregister`
    refuses with `NX_EBUSY` when edges still reference,
    `nx_component_destroy` refuses when bound, generations
    bump only after commit, change log capacity 256,
    subscriber limit 16, no GC, single-writer registry
    model (atomics in IPC-read paths only).
17. **Where to read more** — eight source links (all five
    framework files + kernel.json + gen/slot_table.c +
    gen/posix_shim_deps.h), two reference-doc links
    (`docs/framework-registry.md`,
    `docs/framework-bootstrap.md`), one chapter-internal
    back-reference (chapter 8 for the
    two-components-one-slot insight), two external
    comparisons (seL4 capDL system descriptions, Fuchsia
    Component Framework's uses/exposes), plus a one-paragraph
    forward to chapter 10.

### `sources/nonux/book/README.md`

Part V row 9 promoted from `9. Slots, components, and the
registry` (no link, planned) to `9. [Slots, components, and
the registry](09-slots-components-and-registry.md)` (real
link, shipped).

### `proj_docs/nonux/BOOK-OUTLINE.md`

Chapter 9 row updated:

- Status: `planned` → `shipped`.
- "Reader sees" column rewritten from the one-line planning
  blurb to a real summary of what landed: registry.{h,c}
  (slots, components, connections, change events,
  snapshots, change log), component.{h,c} (six-state
  lifecycle, ops table, resolve_deps, REGISTER macros),
  bootstrap.c end-to-end walk, the `nx_components` linker
  section + `__start_/_stop_` markers, kernel.json +
  gen/slot_table.c + per-component gen/<name>_deps.h,
  manifest-driven dep injection (typed struct + offsetof
  macros), and the posix_shim worked trace.

## Key Findings

- **The descriptor/instance split is the chapter's central
  framing.**  Once a reader sees that `struct
  nx_component_descriptor` is a const, static, link-time
  artifact (one per source file, emitted into the
  `nx_components` section) and `struct nx_component` is a
  dynamic per-instance struct allocated at boot, the
  `NX_COMPONENT_REGISTER` macro, the linker section, and
  the bootstrap walk all fall into place.  Structuring the
  chapter to introduce the descriptor first (in §7) and
  *then* the instance lifecycle (in §11) — rather than
  jumping straight to the lifecycle — kept the
  vocabulary load manageable.
- **Indirection through the slot is the load-bearing
  property.**  Calling components hold pointers to slots,
  not to components.  The slot's `active` field is the
  single mutation point.  Once that is internalised, the
  hot-swap promise from chapter 8 stops being magic — it is
  just "change one pointer, atomically, after pausing
  traffic".  The chapter foregrounds this by walking
  `struct nx_slot` field-by-field with `active` as the
  centrepiece and the others as scaffolding around it.
- **Manifest-driven dep injection is the AI-operability
  proof point.**  Component authors never write the
  dependency table by hand.  They write
  `manifest.json`; `tools/gen-config.py` emits the typed
  struct and the offsetof macro; `NX_COMPONENT_REGISTER`
  joins them.  At runtime, `nx_resolve_deps` writes typed
  slot pointers and registers connection edges in one
  pass.  No name-based lookups happen in steady-state
  component code.  The chapter walks `posix_shim` end-to-end
  (manifest → gen header → register call → resolve →
  active) as the canonical example because it is the only
  shipped component with four required deps across mixed
  stateful/policy settings.
- **The chapter is a vocabulary chapter.**  It introduces
  more new terms than any other chapter so far (14 in the
  Terms section, vs 12 in chapter 8 and 8-10 in chapters
  1-7).  Forward chapters reuse all of them: chapter 10
  needs slot/iface/descriptor/component; chapter 11 needs
  connection/mode/stateful/policy; chapter 13 needs hook
  context with lifecycle (from, to) pairs; chapter 14 needs
  pause states, snapshots, the change log, and topological
  order.  The compactness target was "introduce each term
  exactly once, in the right order, with no forward
  references that can't be deferred".

## Decisions Made

- **Walk posix_shim, not vfs_simple or one of the
  schedulers, as the end-to-end dep-resolution example.**
  posix_shim is the only shipped component with four
  required deps with mixed stateful/policy attributes.  The
  schedulers have zero deps (use
  `NX_COMPONENT_REGISTER_NO_DEPS_IFACE`); vfs_simple has two
  but both are stateless.  posix_shim shows the full range
  of dep-descriptor options (`stateful: true` on vfs,
  `stateful: false` everywhere else) without needing
  multiple examples.
- **Defer the pause protocol to chapter 14.**  Chapter 9
  mentions `pause_state` and `resume_waitq` as fields on
  `struct nx_slot` and lists `PAUSED` as a lifecycle
  state, but does not walk the pause/drain/resume sequence.
  Reasons: (1) the protocol is interesting enough to
  deserve its own chapter (chapter 14 already has it as
  "Recomposition and runtime config"); (2) walking it here
  would force a forward reference to IPC (which is chapter
  11); (3) keeping chapter 9 vocabulary-focused makes
  chapter 14 stand alone as the "where the swap actually
  happens" chapter.
- **No worked end-to-end *message* trace — that is chapter
  11's job.**  Chapter 9 traces a *bootstrap* end-to-end
  (posix_shim from manifest to ACTIVE), not a *message*
  end-to-end (a syscall arriving at posix_shim, calling
  through to vfs_simple, etc.).  The slot/component/registry
  vocabulary is enough to understand boot; the IPC
  vocabulary (channels, message-flag bits, the dispatcher
  kthread, slot-call wrappers) needs its own chapter.
- **Show the `NX_COMPONENT_REGISTER_IFACE` expansion in
  full.**  The macro is the linchpin: every component in
  the source tree ends in one expansion of it, and
  understanding it ties together the descriptor section,
  the offsetof plumbing, and the deps-table generation.
  Quoting the full macro (not a simplified pseudocode
  version) was worth ~20 lines because every reader who
  goes back to the source will see exactly the same text
  in `component.h`.
- **The motherboard analogy stays.**  Per the style
  guide's "at most one whole-chapter framing analogy, only
  at the start" rule.  Motherboard slots are a standard
  textbook example, the analogy is true (typed connection
  points; swap a card; the board doesn't know which card
  is in), and it provides a concrete picture before the
  abstract definitions land.  The "where the analogy stops"
  paragraph at the end of the analogy section keeps the
  reader from drawing wrong inferences (no, edges are not
  PCB traces).

## Status at End of Session

- **Book chapters shipped:** 1, 2, 3, 4, 5, 6, 7, 8, 9.
  Part I + Part II + Part III + Part IV all complete;
  Part V (Framework basics) is half done — chapter 9
  shipped, chapter 10 (IDL) planned.
- **Documentation web updated:** book/README.md row 9
  promoted to a real link; BOOK-OUTLINE.md row 9 marked
  `shipped` with reader-sees rewritten.
- **Tests:** unchanged from session 138 (which was
  unchanged from session 137).  `make test-tools`
  **102/102 pass**; `make test-host` **485/485 pass**;
  `make test-interactive` **7/7 pass** (last run session
  115); `make test-kernel` **152/152** (3600 s).  This
  session is documentation-only; no code changed.  Session
  136's one-line `mmu_init` `l2_mmio_table[0] = 0` change
  is still waiting for a kernel-test re-run on the next
  code-bearing session.
- **`make verify-iface-fresh`:** 0 drift.
- **`make verify-registry`:** 0 findings (R2, R4, R9).

## Next Steps

- **Either start Phase 9** — per-process MM rework: L3 4 KiB
  pages, VMAs, demand paging, COW fork (per
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework))
  — and bundle a kernel-test re-run to validate session
  136's `mmu_init` change at the same time.  When this
  lands, the chapter-6 revision pass scheduled in session
  128 also becomes available.
- **Or continue the book with chapter 10** — Interface
  Definition Language (IDL), the second and final chapter
  of Part V.  Walks `interfaces/idl/*.json`,
  `tools/gen-iface.py`, and the generated
  `*_call.h` / `*_dispatch.h` / `*_msg.h` shims.  Natural
  follow-on now that chapter 9's vocabulary is in place
  (chapter 10 explains *what's at the other end* of the
  slot-binding the registry tracks).

---

**Files Changed:**
- `sources/nonux/book/09-slots-components-and-registry.md` — new, 1587 lines.  The full chapter 9 draft.
- `sources/nonux/book/README.md` — Part V row 9 promoted from "no link, planned" to a real link.
- `proj_docs/nonux/BOOK-OUTLINE.md` — chapter 9 row status `planned` → `shipped`, "Reader sees" column rewritten to reflect what actually landed.
