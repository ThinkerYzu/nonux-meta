# Spec: nonux

**Project:** nonux
**Created:** 2026-04-17
**Last Updated:** 2026-04-17
**Status:** Approved

---

## Navigation

**Project Docs:** [README](README.md) | [SPEC](SPEC.md) *(you are here)* | [DESIGN](DESIGN.md) | [IMPLEMENTATION-GUIDE](IMPLEMENTATION-GUIDE.md) | [HANDOFF](HANDOFF.md)

**This Document:**
- [Problem Statement](#problem-statement)
- [Goals](#goals)
- [Non-Goals](#non-goals)
- [Requirements](#requirements)
- [Constraints](#constraints)
- [Success Criteria](#success-criteria)
- [Open Questions](#open-questions)

---

## Problem Statement

Linux is a monolithic kernel with a GPL license that makes it difficult to experiment with alternative kernel designs, swap components freely, or use kernel code in permissively-licensed projects. Its architecture has calcified over decades — changing a subsystem means navigating millions of lines of tightly coupled code. There is no practical way to swap out a scheduler, memory manager, or filesystem implementation and benchmark the difference without deep kernel surgery.

Researchers, OS hobbyists, and engineers who want to explore kernel design trade-offs need a kernel that is:
- **Composable** — components snap together like Lego bricks
- **Instrumentable** — easy to intercept, hook, and inject code at any boundary
- **Testable** — every component can be tested in isolation and in combination
- **Permissively licensed** — BSD/MIT, no copyleft restrictions

### Background

Existing alternatives (seL4, Zircon, Redox, xv6) each solve part of this but not all. seL4 is formally verified but not designed for rapid experimentation. Zircon is tied to Fuchsia. Redox is Rust-only. xv6 is educational but too minimal for real workloads. None of them prioritize the "swap any component and benchmark it" workflow.

nonux fills this gap: a microkernel designed from day one for composability and experimentation, with enough POSIX compatibility to run real Unix tools (busybox, bash) as validation.

---

## Goals

What this project will accomplish:

1. **Composable microkernel** — A minimal kernel core with well-defined component interfaces. Subsystems (scheduler, memory manager, VFS, IPC, device drivers) are independent modules that can be swapped, replaced, or reconfigured without modifying the core.
2. **Privileged-mode services** — Unlike traditional microkernels that force services into userspace, nonux allows services to run in privileged mode for performance while maintaining clean module boundaries. The isolation is architectural (interfaces), not just ring-based.
3. **POSIX compatibility layer** — Provide enough POSIX API coverage to run busybox and standard Unix tools. This is a compatibility shim, not a design goal — the native kernel API may differ significantly from POSIX.
4. **Comprehensive test infrastructure** — Unit tests for individual components, integration tests for component combinations, and performance benchmarks. Users can intercept, inject, and hook any component or communication channel for testing.
5. **ARM64 on QEMU first** — Start with ARM64 as the primary target, running in QEMU for rapid development iteration. x86_64 support comes later.
6. **Permissive license** — All code under MIT or BSD license.
7. **AI-enabled composability** — The kernel is designed to be operated by AI agents, not just humans. An AI should be able to understand the architecture, select components, configure them, swap implementations, connect modules, and build a working kernel — all through machine-readable metadata and structured interfaces. This means: declarative configuration (not hidden in code), structured component manifests, predictable naming conventions, and documentation that doubles as a machine-readable spec.

## Non-Goals

What this project will explicitly **not** do (to prevent scope creep):

1. **Not a Linux replacement for production** — This is an experimentation platform, not a production OS. Stability and driver coverage are secondary to composability and testability.
2. **Not POSIX-first** — POSIX is a compatibility layer, not the native API. The kernel's internal interfaces should be clean and modern, not constrained by 1970s design.
3. **Not Rust** — C/C++ is the implementation language. AI tooling handles memory safety analysis. We don't pay the Rust complexity tax.
4. **Not SMP from day one** — Start single-core. Multi-core support is a future phase.
5. **Not GUI in v1** — GUI support is a long-term goal, not a near-term deliverable.

---

## Requirements

### Functional Requirements

1. **Minimal kernel core** — The core provides: CPU abstraction (context switching, interrupt handling), physical/virtual memory management primitives, and IPC. Everything else is a service.
2. **Component interface standard** — Every service/component exposes a well-defined interface (C headers + message protocol). Components communicate through a uniform IPC mechanism. Each component must be interchangeable with any other component that implements the same interface.
3. **Object-oriented handle-based syscall API** — The native syscall interface is handle-based, not numbered. Processes interact with kernel objects (processes, channels, memory regions, files, devices) through typed handles. Each handle carries a set of permitted operations. Different processes can hold different handle types with different operation sets — there is no single global syscall table. This enables per-process or per-task API surfaces: a sandboxed process might only have handles that support read/send, while a privileged service holds handles with full control. The POSIX compatibility layer maps traditional syscalls (open, read, write, fork) onto handle operations underneath.
4. **Async-first IPC** — IPC is asynchronous by default. Messages are sent without blocking the sender. However, a connection between two components can be configured as **synchronous** via a composition parameter — the framework then blocks the sender until the reply arrives. This is a per-connection setting declared in the kernel config, not a per-call decision in code. The sync/async mode can also be changed at runtime without restarting either component — the framework drains in-flight messages, switches the mode, and resumes. This enables live performance tuning and A/B testing of IPC strategies. Components are written against the async interface; the sync shortcut is transparent (the component doesn't know or care whether the caller blocks). This keeps component code simple and uniform while allowing performance tuning at composition time.
5. **Boot to shell** — The system boots on QEMU ARM64 (virt machine), loads an init process, mounts a root filesystem, and drops into a busybox shell.
6. **Process model** — fork/exec (or equivalent), signals, process groups. Enough to run a shell with job control.
7. **Virtual filesystem (VFS)** — Pluggable filesystem layer. At minimum: ramfs/tmpfs for initial rootfs, and a simple on-disk filesystem.
8. **POSIX shim** — A userspace or kernel-space library that translates POSIX syscalls to native kernel calls. Must support enough of POSIX to run busybox.
9. **Hooking/interception framework** — Any IPC message, syscall, or component boundary can be intercepted by test code or instrumentation. This is a first-class feature, not an afterthought.
10. **Test harness** — Components can be compiled and tested outside the kernel (in a host userspace test environment), in QEMU with the full kernel, or with mock dependencies.
    - **Independent component testing** — Each component has its own test suite in its `test/` directory. Tests run in isolation on the host machine with mock dependencies injected — no kernel boot required. A component is not considered complete until its tests pass.
    - **Memory correctness** — Every component test runs under a memory tracking layer that verifies: no leaks (every allocation freed), no use-after-free, no double-free, no buffer overflows. The test harness wraps `malloc`/`free` with tracking and reports violations as test failures. This proves the component follows its documented ownership semantics (transfer/borrow/shared).
    - **Interface conformance** — A generic interface conformance test suite exists for each interface type (scheduler, vfs, block_device, etc.). Any component implementing that interface must pass the conformance suite, which verifies: all required operations are implemented, return values match the spec, error conditions are handled as documented, and ownership contracts are respected.
    - **Lifecycle health** — Every component test exercises the full lifecycle: `init` → `enable` → (operate) → `disable` → `destroy`. Tests verify: clean state after init (no resource leaks from previous runs), resources acquired during active phase are fully released after disable, destroy leaves zero allocations, and the component can be cycled (init→enable→disable→destroy) repeatedly without leaks or state corruption.
11. **Component documentation standard** — Every component must include structured documentation covering:
   - **Usage** — What the component does, how to use it, configuration options
   - **Interface** — Complete API/message specification with types, semantics, and error conditions
   - **Relationships** — Which other components it depends on, which depend on it, and what interfaces it implements
   - **Interchangeability** — What interface contract an alternative implementation must satisfy to replace this component
   - **Resource ownership** — Defined in the manifest's `interface` section: every pointer parameter and return value annotated with ownership type (transfer/borrow/shared/static). Machine-parseable by AI and validated by the config tooling. Human-readable resource lifecycle notes (e.g., "allocates run queue on enable, freed on disable") go in the component's README.md.
12. **Architecture documentation** — The kernel maintains a living architecture document that describes:
    - **Minimum required modules** — The smallest set of components needed for a bootable system
    - **Module types** — Categories of components (scheduler, memory manager, VFS, driver, etc.) and what interface each type must implement
    - **Composition rules** — How to connect modules together, which combinations are valid, and dependency constraints
    - **Configuration guide** — How to select, swap, and configure components for a custom kernel build
13. **Component lifecycle and hot-swap** — Every component implements a standard lifecycle: `init`, `enable`, `disable`, `destroy`. Resource management rules:
    - **Clean state at boundaries** — After `init`, the component holds no resources beyond its own static state. After `disable`, all resources acquired during the enabled phase are released or handed off. After `destroy`, zero resources remain.
    - **Ownership protocol** — Every interface function documents ownership transfer semantics per-parameter and per-return-value in the manifest's `interface` section, using one of: **transfer** (caller gives up ownership, callee must free), **borrow** (caller retains ownership, pointer valid only for the call duration), **shared** (reference-counted, last holder frees), **static** (kernel-lifetime object, never freed). Plain value types use **value** (no ownership tracking). Missing annotation on a pointer parameter is a manifest validation error.
    - **Reference counting** — Objects with `shared` ownership use two refcount mechanisms depending on context:
      - **Userspace objects** — The handle table is the refcount. Each handle is a reference. When the last handle to an object is closed (`nx_handle_close`), the object is destroyed. Duplicating a handle (`nx_handle_duplicate`) increments the refcount. No separate counter needed — the handle system already tracks this.
      - **Kernel-internal objects** — Objects shared between components embed a refcount field (`struct nx_ref`) directly in the struct. The framework provides `nx_ref_get()` / `nx_ref_put()` operations. When the count reaches zero, a destroy callback is invoked. Every kernel struct that can be shared must include `struct nx_ref` and a destroy function. The test harness verifies refcounts reach zero during lifecycle tests — a leaked reference is a test failure.
    - **Drain on disable** — When a component is disabled, it must drain in-flight operations, release all held resources (buffers, handles, locks), and reach a quiescent state. The framework provides a drain protocol so dependent components can coordinate.
    - **Hot-swap sequence** — To replace a running component: disable old → verify quiescent → enable new. The framework ensures no messages are routed to the component during the swap window. State migration between old and new implementations is explicit via a `migrate` callback (optional — not all components need it).
    - **Slot-based indirection** — Components never hold direct pointers to each other. All inter-component communication goes through **slots** or the IPC router. A component receives a slot reference to its dependency, and the framework resolves the current active implementation at call time. This makes hot-swap transparent — callers don't hold stale pointers when a target is replaced.
    - **Stateful vs stateless connections** — Each connection between components is declared as stateful or stateless in the manifest:
      - **stateless** — No accumulated state between calls (e.g., scheduler `pick_next`, timer `get_ticks`). Hot-swap is safe at any time — the new implementation works immediately.
      - **stateful** — The caller has built up state against the target (e.g., open file handles on VFS, established sessions on a network stack). Hot-swapping the target means the caller's accumulated state may become invalid.
    - **Swap notification for stateful connections** — When a stateful dependency is swapped, the framework notifies all callers via an `on_dep_swapped(slot, old_impl, new_impl)` callback. The caller can:
      - **Re-establish** — Reopen files, reconnect sessions against the new implementation
      - **Fail gracefully** — Return errors for in-progress operations that depended on the old state
      - **Ignore** — If it can tolerate the state reset (e.g., best-effort caching)
    - **State transfer for stateful swaps** — Beyond the target's internal state (`migrate_export`/`migrate_import`), stateful connections may need to transfer **client-visible state** (open handles, sessions, cursors). The old implementation exports this state, and the new implementation imports it. This is optional — not all implementations support it. If a stateful swap is requested without a migration plan, the framework:
      - Issues a warning to the caller via `on_dep_swapped` with a `STATE_LOST` flag
      - Requires a `force` flag on the swap request — without it, the swap is rejected with `NX_ERR_STATE_LOSS`
      - Logs the event for diagnostics
    - **Runtime recomposition** — The full composition (adding, removing, replacing components and rewiring connections) can be performed at runtime, not just at boot. However, components declare a **mutability level** in their manifest:
      - **hot-swappable** — Can be replaced/reconfigured at runtime (default for most services)
      - **warm-swappable** — Can be replaced, but requires a brief pause of dependent components (e.g., a filesystem switch may pause I/O briefly)
      - **frozen** — Cannot be changed at runtime. Reserved for core primitives that everything else depends on (e.g., the IPC transport itself, the core memory allocator, interrupt dispatch). Attempting to swap a frozen component is rejected by the framework.
    The manifest declares the mutability level; the framework enforces it. This lets AI and users freely experiment with recomposition while preventing changes that would pull the rug out from under the running system.
    - **Pause and reconnect protocol** — Recomposition (rewiring connections, replacing multiple components) requires coordinated pausing across affected components. The framework provides:
      - **Pause interface** — Every component implements `pause()` and `resume()` callbacks. `pause()` tells the component to finish its current operation and enter a quiescent state — it stops accepting new work but completes in-flight operations. `resume()` returns it to active processing. Unlike `disable()`, a paused component retains all its state and resources.
      - **Drain-with-cutoff** — Pausing a component follows a two-step protocol to handle messages already queued:
        1. **Cutoff** — Framework atomically marks the component as PAUSING and activates the message policy (queue/reject/redirect) for all new messages. After this point, no new messages enter the component's inbox.
        2. **Drain** — The component processes all messages already in its queue (a finite, bounded set). Sync-mode callers blocked waiting for replies get their responses during this phase. Once the queue is empty, the component enters PAUSED.
        This ensures no messages are lost, drain time is bounded, and the boundary between "processed" and "held for later" is clean.
      - **Coordinated pause** — When recomposing a subgraph, the framework pauses components top-down (dependents first, then dependencies). Each component must acknowledge the pause (reach quiescent state) before the framework proceeds. The framework enforces a timeout — if a component fails to pause within the deadline, the recomposition is aborted and all components are resumed.
      - **Message policy for paused components** — Messages arriving after the cutoff point are handled according to a policy declared in the manifest:
        - **queue** (default) — Messages are buffered by the framework. Delivered in order when the component resumes. Buffer has a configurable max size; overflow returns an error to the sender.
        - **reject** — Messages are immediately rejected with `NX_ERR_PAUSED`. The sender is responsible for retrying. Appropriate for components where stale messages are worse than failed sends.
        - **redirect** — Messages are forwarded to a designated fallback component (if configured). Useful for graceful degradation during warm-swap.
      - **Reconnect** — While components are paused, the framework rewires connections (changes endpoints, inserts/removes components in the graph). The new wiring is validated before any component resumes. Resume is bottom-up (dependencies first, then dependents), the reverse of pause order.
      - **Atomic recomposition** — The entire pause→rewire→resume sequence is atomic from the perspective of components outside the affected subgraph. External senders see either the old configuration or the new one, never a partially-rewired state.
      - **SMP considerations (future)** — The single-core implementation is a simplification of the full multi-core protocol. The design is SMP-aware from day one:
        - **Pause barrier** — The PAUSING flag must be atomically visible to all CPUs. On SMP, this requires memory barriers (ARM64 `dmb`/`dsb`) so every CPU applies the message policy before any drain begins.
        - **Per-CPU queue drain** — In SMP, components may have per-CPU local queues to avoid lock contention. After the barrier, each CPU's local queue contains a finite set of messages. All per-CPU queues must be drained before PAUSED is reached.
        - **IPI synchronization** — For `thread-per-instance` components, the framework sends inter-processor interrupts to ensure all CPU-local instances see the pause and complete their drain.
        - **Memory ordering** — The pause flag, queue state, and message policy changes must be correctly ordered with respect to all CPUs. The v1 single-core implementation uses the same flag/state protocol but without the cross-CPU synchronization — making the SMP upgrade a matter of adding barriers, not redesigning the protocol.
    - **Execution model** — nonux v1 commits to a specific execution discipline:
      - **Per-CPU dispatcher threads** — Each CPU runs a single kernel dispatcher thread that is pinned to that CPU and is the only thread on that CPU which executes component code. The dispatcher loop is: dequeue a message → resolve target slot → invoke handler → repeat. A component is "running on a CPU" only while that CPU's dispatcher is inside its handler.
      - **Slot-resolve locality** — Dereferencing a slot (reading `slot->active`, calling `slot_resolve()`, or invoking `slot->active->ops->...`) is permitted only on a framework-owned dispatcher thread — either a per-CPU dispatcher or a `dedicated` component's private thread. Interrupt handlers and arbitrary kernel threads MUST NOT dereference slots; they may only enqueue messages. This is what makes hot-swap's drain step complete: once every dispatcher's queue is drained and no handler is executing, no live slot pointer exists anywhere in the kernel, so the old component can be destroyed without RCU or generation fences. Violations of this rule would reintroduce use-after-free windows during swap.
      - **Non-preemptible handlers** — While a component handler is executing on the dispatcher, kernel preemption is disabled on that CPU. Interrupts remain enabled, but interrupt handlers are forbidden from calling into components *and* from dereferencing slots — they enqueue a message and return. User tasks (EL0) remain fully preemptible under the scheduler; only kernel-side component handlers run to completion.
      - **Bounded handlers** — Every handler must complete in bounded time. Work that cannot be bounded (device I/O, sleep, long computation) must be split: "start op" returns immediately; the completion arrives later as a message. This is a contract every component is written against, checked by code review and by latency bounds in the test harness.
    - **Concurrency mode** — Each component declares in its manifest how it is instantiated and entered:
      - **shared** (default) — Single instance. Any CPU's dispatcher may enter its handler; component is responsible for cross-CPU synchronization using C11 atomics or framework-provided locks when it holds state shared across CPUs.
      - **serialized** — Single instance, framework holds a per-slot lock across every handler invocation. No two CPUs can be inside this component at once. Provided for simple components that don't want to manage their own synchronization; the cost is lock contention on hot paths.
      - **per-cpu** — N instances, one per CPU. Each CPU's dispatcher only routes to its local instance. No cross-CPU entry; no shared state between instances. Ideal for per-CPU caches, run queues, statistics collectors.
      - **dedicated** — Escape hatch for the rare component that genuinely cannot fit the bounded-handler rule (e.g., long synchronous cryptographic work). The component runs on its own private thread with its own message queue; it does not share the per-CPU dispatcher. Use sparingly — every `dedicated` component pays the cost of an extra kernel thread and context switches on every call. v1 minimum config uses zero `dedicated` components.
    The framework enforces concurrency contracts via the config validator and the registry: retaining a `shared` slot from a caller that promised single-entry is rejected; per-cpu slots cannot be reached from a CPU whose local instance is paused during recomposition; `dedicated` components go through a different hot-swap path (stop-thread / swap / start-thread) than dispatcher-served components.
14. **Machine-readable component manifests** — Each component includes a structured manifest file (YAML or similar) that an AI agent can parse to understand:
    - Component name, type, and version
    - Provided interfaces and required interfaces (dependencies)
    - Configuration parameters with types, defaults, valid ranges, and descriptions
    - Concurrency mode (shared, serialized, per-cpu, or dedicated)
    - Compatibility constraints (which other components/versions it works with)
    - Build integration (source files, compile flags, link requirements)
15. **Declarative kernel configuration** — The kernel build is driven by a single declarative config file that specifies which component implementation to use for each module slot, their configuration parameters, and composition. An AI agent can generate a valid config by reading the manifests and composition rules — no implicit knowledge required.
16. **Validation tooling** — A config validator that checks a kernel configuration for completeness (all required module types filled), compatibility (no conflicting components), and dependency satisfaction. Runnable as `make validate-config` before building.
17. **Dependency management** — Components declare their dependencies explicitly in the manifest. The framework manages dependencies at both build time and runtime:
    - **Dependency declaration** — Each manifest lists required interfaces (e.g., a VFS requires a block_device) and optional dependencies (e.g., a scheduler may optionally use a stats collector). Version constraints are supported (e.g., `"requires": {"timer": ">=0.2.0"}`).
    - **Dependency resolution** — The config validator and build system resolve the full dependency graph from `kernel.json`. Missing dependencies are errors. Circular dependencies are errors. The resolver reports a clear error message naming the unsatisfied dependency and which component needs it.
    - **Initialization ordering** — At boot, the framework topologically sorts components by their dependency graph and initializes them in dependency order (dependencies first). Shutdown/disable is in reverse order.
    - **Hot-swap dependency awareness** — When swapping a component at runtime, the framework checks that the replacement satisfies the same interface and that all dependents are compatible with the new version. If a swap would break a dependent, it is rejected with a diagnostic.
    - **Dependency visualization** — `make deps` prints or generates the component dependency graph (text or dot format) for inspection by humans or AI agents.
18. **Component Graph Registry** — The framework owns an explicit, live data structure representing the running composition. It is a first-class v1 feature because the hooking/interception framework (requirement #9) targets composition boundaries, and those boundaries must be enumerable and observable at runtime.
    - **Universal registration** — Every slot is registered with the registry at creation. Every slot reference held by a component (dependency edge) is registered at component init. Every connection (caller-slot → callee-slot with mode and stateful attributes) is registered when established. Unregistered slots, slot references, or connections are a framework invariant violation — the build and runtime should refuse to proceed.
    - **Registry contents** — For each slot: type, name, mutability, concurrency mode, current active component, inbox/pause state. For each component instance: manifest identity, lifecycle state, list of held slot refs, list of slots where it is the active impl. For each connection: endpoints, mode (sync/async), stateful flag, pause policy, installed hooks.
    - **Introspection API** — The registry exposes traversal primitives (enumerate slots, list a slot's dependents and dependencies, walk a subgraph from a root) and a snapshot operation that returns a consistent view of the graph for logging, debugging, or serialization. Snapshots must be obtainable without blocking the kernel.
    - **Change events for instrumentation** — The registry emits events for every graph mutation: `slot_created`, `slot_destroyed`, `slot_swapped`, `component_state_changed`, `connection_added`, `connection_removed`, `connection_retuned` (mode or stateful flag change). Hooks and test harnesses subscribe to these events — this is how instrumentation attaches to composition changes without polling.
    - **Change history (registrator)** — The registry maintains an append-only log of graph mutations with timestamp, event type, affected nodes/edges, and a compact diff. The log has a bounded size with a configurable retention policy. It is queryable from userspace for debugging reproducibility ("what was the composition at time T?") and is serializable for offline analysis.
    - **Concurrency** — Readers (instrumentation, introspection) use a snapshot mechanism (RCU-style or seqlock) and never block. Writers (slot creation, hot-swap, recomposition) serialize through the framework's existing recomposition lock — the registry does not introduce a second synchronization domain.
    - **Slot refs in messages** — Slot references can be carried across IPC as typed capabilities (not as raw pointers buried in payload bytes). Each slot-ref capability in a message declares its ownership mode (`borrow` or `transfer`) matching the existing ownership vocabulary:
      - **borrow** (default) — The reference is valid only for the duration of the receiver's message handler. The IPC framework drops it on handler return.
      - **transfer** — The sender gives up the reference; the receiver must either release it or retain it before the handler returns. Unclaimed transfer capabilities are a protocol error and are dropped with an event logged.
      If the receiver wants to hold a slot ref beyond the handler, it must call `slot_ref_retain()`, which registers a new connection edge in the Component Graph Registry rooted at the receiver. Holding a slot pointer past handler return without registration is a framework invariant violation — detectable because the registry knows every connection edge each component legitimately holds. The IPC router scans each outgoing and incoming message for slot-ref capabilities; messages with unregistered or forged slot refs are rejected. This closes the loophole where a component could receive a slot via IPC and stash it outside the registry's view.
    - **Component-spawned threads and thread-local dereferences** — A component may spawn its own worker threads alongside its dispatcher-served handlers for long CPU-bound work that cannot fit the bounded-handler rule and whose driver is not an external event (so async split via completion message is not applicable). Two rules apply:
      - **Dereferenced slot pointers are thread-local.** A dispatcher handler must not pass a resolved component pointer (the `impl*` obtained by resolving a slot ref) to a worker thread. If the worker needs to call another component, the component carries the **slot reference** into the worker (registered via `slot_ref_retain`, listed in the manifest's `retains` array) and the worker reaches that component by enqueueing a message to the slot — a dispatcher resolves. Dispatcher threads remain the only category permitted to dereference slots. Rule enforcement is partly static (R8 catches the case where a worker function syntactically dereferences a slot) and partly delegated to the AI review rubric (the case where a handler copies an already-resolved pointer into a struct handed to the worker is a data-flow pattern beyond the deterministic checker).
      - **Component drains its own threads.** The framework's pause protocol drains dispatcher queues it owns; it has no visibility into component-spawned threads. Any component that spawns threads must declare a `pause_hook` in its manifest. The framework invokes `pause_hook` during the pause protocol's cutoff step; the component signals its workers to stop, joins or parks them, and reaches quiescence within the pause deadline. A missing `pause_hook` on a component that spawns threads is a manifest validation error. A `pause_hook` that exceeds the deadline aborts the recomposition.
      Prefer `dedicated` mode first — it is the framework-managed version of this pattern (private thread + framework-owned queue, no `pause_hook` needed). Component-spawned threads are for the rare cases where the workload genuinely doesn't fit a simple message-queue dispatch model.
    - **AI-enforceable compliance** — nonux is an AI-driven project; every rule in this requirement must be both *followed* by AI agents writing components and *verified* by AI agents reviewing them. Compliance is not a human-only review discipline. Concretely:
      - **Machine-checkable rules** — Each rule above maps to a predicate an AI (or a deterministic tool it invokes) can evaluate over source and manifests: no `struct slot *` field in a component struct without a matching registered connection, no IPC message schema with a slot-ref field outside the typed cap array, no `slot_ref_retain` without a paired release in `disable`/`destroy`, no slot pointer obtained outside framework-provided APIs.
      - **Static checker** — `make verify-registry` runs a structural analysis on every component and reports registry-rule violations with file:line citations. AI agents must run this before marking a component complete, and CI must run it on every change.
      - **Runtime assertions** — The test harness enforces the registry invariants (Design §Component Graph Registry, "Invariants") after every lifecycle transition, every `slot_swap`, and every message handler return. Violations fail the test suite.
      - **Review rubric** — Each component has an AI review checklist that a reviewer agent walks through before the component is merged, covering registration, retention, and release for slots, components, and connections. The rubric lives in the component's `README.md` template so the reviewing AI has a single, stable entry point.
      - **Manifest-derived claims** — Each manifest's `provides`, `requires`, `interface`, and message schema declarations are the authoritative source of truth an AI agent uses to generate and verify registration code. Code that disagrees with the manifest is a verification error.

### Non-Functional Requirements

1. **Performance** — IPC latency for in-kernel (privileged-mode) services should be comparable to function calls. The architecture must not force unnecessary context switches.
2. **Configurability** — A configuration system (like Kconfig or simpler) to select which components to include, which implementations to use, and which hooks to enable.
3. **Code clarity** — The codebase should be readable. Prefer straightforward C over clever macros. Each component should be understandable in isolation.

---

## Constraints

1. **C/C++** — Primary implementation language. C for kernel core and low-level components, C++ where it genuinely helps (e.g., component frameworks, test harness). No Rust.
2. **ARM64 first** — Initial target is AArch64 on QEMU virt machine. No x86 code in the first phase.
3. **QEMU development** — All initial development and testing happens in QEMU. No real hardware required.
4. **Cross-compilation** — Must build on a Linux host using standard ARM64 cross-compiler toolchain (aarch64-linux-gnu-gcc or aarch64-none-elf-gcc).
5. **MIT license** — All original code under MIT license. Third-party code (busybox, etc.) keeps its own license but must be compatible.
6. **No large external dependencies** — The kernel itself has zero runtime dependencies. Build tools: cross-compiler, QEMU, make/cmake, and standard host tools only.

---

## Success Criteria

How we know the project is done. Each criterion should be objectively verifiable:

1. **Boot to shell** — `make run` boots the kernel in QEMU ARM64, displays boot messages, and drops into a busybox shell within 5 seconds.
2. **Run basic commands** — `ls`, `cat`, `echo`, `ps`, `kill`, `mkdir`, `rm`, `cp`, `mv`, `pipe` all work correctly in the busybox shell.
3. **Component swap demo** — Swap the scheduler implementation (e.g., round-robin vs priority) by changing a config option, rebuild, and show different scheduling behavior with a test program.
4. **Hook demo** — Register a hook that logs all IPC messages between two services, run a workload, and show the captured trace.
5. **Test suite passes** — `make test` runs unit tests for all core components and integration tests, all passing. Coverage of core kernel components > 80%.
6. **Benchmark baseline** — `make bench` runs IPC latency, context switch, and syscall benchmarks and produces a report. Results are reproducible across runs.
7. **Documentation completeness** — Every component in the kernel has a doc file covering usage, interface, relationships, and interchangeability. The architecture doc accurately describes the minimum module set, module types, composition rules, and how to build a custom kernel configuration.
8. **AI agent can build a kernel** — Given only the component manifests and architecture doc, an AI agent (e.g., Claude) can: pick components for each required module type, generate a valid kernel config, build it, and boot it in QEMU — without any human guidance beyond "build me a minimal kernel" or "build me a kernel with priority scheduling and ext2".

---

## Open Questions

Unresolved decisions that affect the spec. Resolve before implementation starts:

1. ~~**IPC mechanism**~~ — **RESOLVED**: Async by default, with per-connection sync shortcut configurable at composition time. See Functional Requirement #3.
2. ~~**Native syscall API**~~ — **RESOLVED**: Object-oriented handle-based API. Typed handles with per-handle operation sets, enabling different syscall surfaces per process. See Functional Requirement #3.
3. ~~**Build system**~~ — **RESOLVED**: Make. Simple, universal, no extra dependencies. Component selection driven by the declarative config, wired into the Makefile.
4. ~~**Component binary interface**~~ — **RESOLVED**: Statically linked initially. Runtime dynamic loading is a future goal but not required for v1. Components are compiled into the kernel image for now; the interfaces are designed so that dynamic loading can be added later without changing component code.
5. ~~**Memory model for privileged services**~~ — **RESOLVED**: Software discipline + testing. No hardware isolation between privileged services. Clean interfaces, documented ownership, and comprehensive testing enforce correctness. This keeps the design simple and avoids the performance cost of separate page tables for in-kernel services.
6. ~~**Manifest format**~~ — **RESOLVED**: JSON. Machine-readable by default (AI-friendly), well-supported everywhere, no extra parser dependencies.

---

**Last Updated:** 2026-04-17
