# Handoff: nonux

**Project:** nonux
**Started:** 2026-04-17

---

## Navigation

**Project Docs:** [README](README.md) | [SPEC](SPEC.md) | [DESIGN](DESIGN.md) | [IMPLEMENTATION-GUIDE](IMPLEMENTATION-GUIDE.md) | [HANDOFF](HANDOFF.md) *(you are here)* | [IDL-SCHEMA](IDL-SCHEMA.md) | [SLOT-CALL-API](SLOT-CALL-API.md)

**Quick Jump:** [Current Status](#current-status) | [Next Actions](#next-actions) | [Session Logs](#session-logs)

---

## Current Status

**Phase:** Phases 1–6 complete.  **Phase 7 closed** (Session 78).  **Phase 8 in progress.**  Plan landed Session 79; IDL schema + slot-call API locked Sessions 80–81; slices 8.0pre.1 → 8.0pre.4 landed Sessions 82–85 — **Group A (generator infrastructure) closed at Session 85**.  **Group B kicked off Session 86:** original slice 8.0a sub-sliced into 8.0a.1 → 8.0a.8 (each sub-slice a green checkpoint, same overall scope and ~70-ktest budget locked Session 81).  Sub-slices **8.0a.1** (rename `posix_shim` → `libnxlibc`) and **8.0a.2** (IPC + slot scaffolding: `NX_MSG_FLAG_REPLY_REQUESTED` request flag + `resume_waitq` and `_Atomic in_flight_calls` fields on `struct nx_slot`) landed Session 86; **8.0a.3** (`framework/slot_call.{h,c}` skeleton — public API surface + stub body returning `NX_ENOSYS` until 8.0a.6 wires the dispatcher round-trip) landed Session 87.  Doc-side spec refinement was committed Session 86 — DESIGN.md gained 4 new subsections (§"Sync-mode caller must be on a dispatcher", §"Tasks as IPC Senders", §"Multi-slot binding (N:1)", §"Edge inheritance"); SLOT-CALL-API.md reconciled (`(reply_buf, reply_buf_len)` arg pair, `NX_EABORT = -8` reused — no new errcode, in-flight buf-len + truncation guard).  **Next forward step: slice 8.0a.4** (kernel `components/posix_shim/` skeleton + manifest + `kernel.json` wiring — booted kernel reports 7 slots / 7 components with `posix_shim` ACTIVE alongside the existing six).

**Tests:** `make test` → **495/495 pass** (93 python + 283 host + 119 kernel — same baseline as Session 85; 8.0a.1 → 8.0a.3 are all pure scaffolding with zero production consumers); `make test-interactive` → **7/7 pass** (echo_cat, echo_hello, echo_pipe, ls_root, mkdir_tmp, ps_smoke, visible_prompt).  `make verify-iface-fresh`: 0 drift.  `make verify-registry`: 0 findings.

**Latest session log:** [Session 87](logs/session-87-8.0a.3-slot-call-skeleton.md) — slice 8.0a.3 paperwork (source commit `def01b5` records the `framework/slot_call.{h,c}` skeleton landing).

**Blockers:** None.  Source-side commit `def01b5` adds the call surface; this session's `proj_docs/nonux/` commit records it.  Next forward step is slice **8.0a.4** — see Next Actions below.

> **Phase checklist** (slice-level detail in [IMPLEMENTATION-GUIDE.md](IMPLEMENTATION-GUIDE.md) and the [Session Logs](#session-logs) / [HANDOFF-ARCHIVE.md](HANDOFF-ARCHIVE.md)):
> - :ballot_box_with_check: Spec, Design, Implementation guide
> - :ballot_box_with_check: Phase 1 (boot to serial), Phase 2 (test harness + PMM + GIC + timer)
> - :ballot_box_with_check: Phase 3 (component framework: 3.1–3.9b.2)
> - :ballot_box_with_check: Phase 4 (scheduler: 4.1–4.4)
> - :ballot_box_with_check: Phase 5 (VM + handles: 5.1–5.6)
> - :ballot_box_with_check: Phase 6 (VFS + ramfs: 6.1–6.4)
> - :ballot_box_with_check: Phase 7 (process model + POSIX shim) — closed at Session 78:
>     - :ballot_box_with_check: 7.1, 7.2, 7.3, 7.3.5, 7.4 (a/b/c/d), 7.5
>     - :ballot_box_with_check: 7.6a, 7.6b, 7.6c.0, 7.6c.1, 7.6c.2, 7.6c.3 (a/b/c), 7.6c.4
>     - :ballot_box_with_check: 7.6d.1, 7.6d.2 (a/b/c), 7.6d.3a, 7.6d.3b, 7.6d.3c
>     - :ballot_box_with_check: 7.6d.N.0 – 7.6d.N.15
>     - :ballot_box_with_check: 7.6d.N.final.a, .b, .c-minimal, .d, .e
>     - :white_large_square: 7.6d.N.16 (deferred — POSIX-fd-to-slot alignment)
>     - :white_large_square: 7.6d.N.final.c-full (deferred — signal-handler trampoline)
>     - :white_large_square: 7.6d.3.x (later — only if busybox `sh` needs more syscalls)
>     - :ballot_box_with_check: 7.7a Interactive smoke tests — `ls /` + `echo \| cat` (Session 73)
>     - :ballot_box_with_check: 7.7b.1 `mkdir` + hierarchical paths in ramfs (Session 74)
>     - :ballot_box_with_check: 7.7b.2 `ps` via procfs (Session 75) — closes Phase 7's slice-7.7 exit criteria
>     - :ballot_box_with_check: 7.8a Wait-queue primitive + `NX_TASK_BLOCKED` activation (Session 76)
>     - :ballot_box_with_check: 7.8b `NX_SYS_PPOLL` built on waitqs (Session 77)
>     - :ballot_box_with_check: 7.8c Migrate yield-loops + revert busybox `fflush(stdout)` patch (Session 78) — closes Phase 7
> - :white_large_square: Phase 8 (runtime recomposition + config manager) — plan landed Session 79, 15 slices in 3 groups:
>     - **Group A — generator infrastructure** *(closed Session 85)*
>         - :ballot_box_with_check: 8.0pre.1 IDL schema + `tools/gen-iface.py` + first interface (vfs) — Session 82
>         - :ballot_box_with_check: 8.0pre.2 fs interface + `fs_types.h` via `includes:` — Session 83
>         - :ballot_box_with_check: 8.0pre.3 sched / mm / char_device interfaces + IRQ-entry codegen — Session 84
>         - :ballot_box_with_check: 8.0pre.4 cutover paperwork — generator output canonical, DESIGN.md §"Interface Definition Language", R7 parallel-rule note — Session 85
>     - **Group B — IPC migration** *(8.0a sub-sliced into 8.0a.1 → 8.0a.8 at Session 86)*
>         - :ballot_box_with_check: 8.0a.1 rename `posix_shim` → `libnxlibc` (frees the name for the kernel-side boundary component) — Session 86
>         - :ballot_box_with_check: 8.0a.2 `NX_MSG_FLAG_REPLY_REQUESTED` + `struct nx_slot` `resume_waitq` + `_Atomic in_flight_calls` — Session 86
>         - :ballot_box_with_check: 8.0a.3 `framework/slot_call.{h,c}` skeleton (decl + stub body returning `NX_ENOSYS`) — Session 87
>         - :white_large_square: 8.0a.4 kernel `components/posix_shim/` skeleton + manifest + `kernel.json` wiring
>         - :white_large_square: 8.0a.5 per-task `caller_slot` in `struct nx_task` + lifecycle in `nx_task_create`/`_destroy` + edge cloning
>         - :white_large_square: 8.0a.6 reply-message pool + dispatcher integration + `nx_slot_call_blocking` body end-to-end
>         - :white_large_square: 8.0a.7 `posix_shim_on_dep_swapped` STATE_LOST handler + `nx_handle_table_invalidate_for_slot()`
>         - :white_large_square: 8.0a.8 cross-cutting test infrastructure (mock component, hook-chain inspector, recompose event logger, pause-injector fixture, cap-forgery harness, equivalence-runner macro) — closes 8.0a
>         - :white_large_square: 8.0b activate generated `handle_msg` shims on every component
>         - :white_large_square: 8.0c migrate framework production paths (~30 syscall.c sites + dispatcher/bootstrap/component/process)
>         - :white_large_square: 8.0d migrate component-to-component calls (vfs_simple → ramfs/procfs)
>         - :white_large_square: 8.0e `verify-registry` rule banning direct ops access outside `framework/slot_call.c`
>     - **Group C — runtime recomposition**
>         - :white_large_square: 8.1 lift pause/drain/resume from host into kernel build + slice-3.9 rollback
>         - :white_large_square: 8.2 `framework/recompose.c` orchestrator + topo pause order + leaf-slot demo
>         - :white_large_square: 8.3 `framework/config.c` runtime config manager + handle API + EL0 syscall surface
>         - :white_large_square: 8.4 second scheduler (`sched_priority`) — standalone validation
>         - :white_large_square: 8.5 **headline** live scheduler swap with running tasks
>         - :white_large_square: 8.6 runtime async↔sync mode switching
>         - :white_large_square: 8.7 `NX_HOOK_SYSCALL_ENTER` / `_EXIT` hook points — closes Phase 8 (added Session 81)
> - :white_large_square: Phases 9–11 (per-process MM rework, integration tests + benchmarks, docs + AI operability)

### What We Have

Build state and infra (per-slice detail is in the [Session Logs](#session-logs) and authoritative docs live in [IMPLEMENTATION-GUIDE.md](IMPLEMENTATION-GUIDE.md); architecture in [DESIGN.md](DESIGN.md)):

- **Source repo** — local `sources/nonux/` ↔ public https://github.com/ThinkerYzu/nonux (MIT, default branch `master`, pushed 2026-04-22)
- **Toolchain (verified 2026-04-20)** — `aarch64-linux-gnu-gcc` 15.2.0, `qemu-system-aarch64` 10.1.0
- **Build / run** — `make` → `kernel.bin`; `make test` → 495/495 (93 python + 283 host + 119 kernel); `tools/run-qemu.sh [-t SECONDS]` boots in QEMU; `make validate-config` / `make verify-registry` / `make verify-iface-fresh` gate the build
- **kernel.json** — declarative config currently wiring six slots: `char_device.serial ← uart_pl011`, `scheduler ← sched_rr`, `memory.page_alloc ← mm_buddy`, `filesystem.root ← ramfs`, `filesystem.proc ← procfs` (slice 7.7b.2), `vfs ← vfs_simple`. Booted kernel reports `composition (gen=30, 6 slots, 6 components)` with all six ACTIVE
- **Directory structure** — `core/` (boot, cpu, pmm, irq, lib), `framework/`, `interfaces/`, `components/`, `test/` (host + conformance, kernel, bench), `tools/`, `docs/`
- **MIT License**

Key design decisions — see [DESIGN.md §Key Design Decisions](DESIGN.md#key-design-decisions) and the topical sections it cross-references (Component Graph Registry, Pause Implementation, Slot-Based Indirection, AI Verification, Dependency Injection Mechanism, Component-Spawned Threads).

---

## Next Actions

### Forward step

1. **Phase 8 — Runtime Recomposition + Config Manager.**  Plan landed Session 79; expanded to 16 slices Session 81 (full breakdown in [IMPLEMENTATION-GUIDE.md §Phase 8](IMPLEMENTATION-GUIDE.md#phase-8-runtime-recomposition-and-config-manager); session logs at [Session 79](logs/session-79-phase-8-plan.md) + [Session 80](logs/session-80-idl-schema.md) + [Session 81](logs/session-81-slot-call-api.md)).  Today's production paths bypass the slice-3.8 IPC router — ~184 direct `slot->active->descriptor->iface_ops` callsites across 19 files, all five production components have `handle_msg = NULL`.  A live recompose against this surface is a use-after-free.  **Group A (8.0pre.1–4) closed at Session 85** — JSON-IDL-driven code generator emits typedef + msg + call + dispatch (+ optional ISR) headers per interface; all five production interfaces declared via IDL; generator output canonical and machine-checked via `make verify-iface-fresh`.  **Group B (8.0a → 8.0e)** routes every cross-component call through `nx_slot_call_blocking`, activates the generated shims on every component, migrates framework + component-to-component callsites, and locks down with a `verify-registry` rule — slot-call API + posix_shim promotion + per-task slot model locked Session 81 in [SLOT-CALL-API.md](SLOT-CALL-API.md).  **Session 86 sub-sliced 8.0a into 8.0a.1 → 8.0a.8** so each landing is a green checkpoint; **8.0a.1 (rename) and 8.0a.2 (IPC + slot scaffolding) landed Session 86**.  **Group C (8.1–8.7)** ships the original Phase 8 deliverables plus syscall-boundary hooks: kernel-side pause/drain/resume, `framework/recompose.c` orchestrator, config handle API + EL0 syscall, second scheduler, live-swap demo, per-edge async↔sync mode switching, `NX_HOOK_SYSCALL_ENTER`/`_EXIT`.  ~270+ new ktests across the migration; cross-cutting test infrastructure lands in slice **8.0a.8**; per-callsite equivalence tests land *before* migration in slice 8.0c.  Hot-path perf gate revised Session 81 — original ≤10% regression unattainable with Resolution 1; replaced with ≤10 µs round-trip on QEMU virt aarch64 + revisit absolute perf in Phase 10.  Estimated ~11–21 sessions remaining (sub-slicing of 8.0a is structural, not scope-expanding).

   **Immediate next step:** Slice **8.0a.4** — kernel `components/posix_shim/` skeleton.  New `components/posix_shim/` directory with `manifest.json` + `posix_shim.c` (init/enable/destroy stubs; `handle_msg` returns `NX_EINVAL` until 8.0a.6 fills in the reply-routing body) + `README.md`.  Manifest declares deps `{vfs, scheduler, mm, char_device.serial}`, all `mode: async`, with `policy: queue` (per SLOT-CALL-API.md §"The posix_shim Component" — sync-mode is dispatcher-to-dispatcher only).  `kernel.json` gains a `posix_shim` slot and an entry binding `posix_shim ← posix_shim`.  Auto-generated `gen/posix_shim_deps.h` plumbing follows the existing dependency-injection mechanism (DESIGN.md §"Dependency Injection Mechanism").  No callers yet — `framework/syscall.c` keeps its direct `vops->read(self, ...)` calls until slice 8.0c.  Booted kernel reports `composition (gen=N, 7 slots, 7 components)` with `posix_shim` ACTIVE alongside the existing six.

   **Tests at end of Session 87:** `make test` **495/495 pass** (93 python + 283 host + 119 kernel; same baseline as Session 86 — 8.0a.3 is pure scaffolding with zero consumers, NULL-arg branch returns `NX_EINVAL`, well-formed calls return `NX_ENOSYS`); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.

### Deferred — actionable when a workload demands

2. **7.6d.N.final.c-full — full signal-handler dispatch trampoline.**  Promote `sys_rt_sigaction` from no-op stub to actually-installs-handler (per-process `sigaction[NSIG]` table); EL0-return path pushes a synthetic frame on the user stack, sets PC to the handler, sets x0 = signo, sets LR to an EL0 trampoline (or musl's `sa_restorer`) that issues `NX_SYS_RT_SIGRETURN` to restore.  Switches Ctrl-C from "kill everyone" to "post SIGINT, deliver via handler".  Estimated ~200-300 lines kernel + ktest using mock-RX 0x03 injection.  v1 minimal Ctrl-C / Ctrl-D side channels (slice 7.6d.N.final.c-minimal, Session 71) cover the scripted smoke tests; .c-full lands when interactive UX needs handler dispatch.

3. **7.6d.N.16 — Drop in-handle generation encoding + POSIX-fd-to-slot alignment.**  Originally a 30-line paydown for slice 7.6d.N.11's `sys_fcntl` band-aid (replace `(generation << 8) | (idx + 1)` with `idx + 1`); slice 7.6d.N.15's investigation showed the proper fix needs a non-zero `NX_HANDLE_INVALID` sentinel (e.g. `0xFFFFFFFF`) so encoded value 0 can name fd 0 — ripples through every `NX_HANDLE_INVALID` comparison and the slice-5.3 stale-handle invariants (which would then assert via the in-table generation field rather than encoded-value mismatch).  Lands when a workload demands `exec 3< file` or similar literal-fd-3+ idioms.  ~30 lines + slice-5.3 test churn.

4. **Reap-on-wait.**  Have `sys_wait` call `nx_process_destroy(child)` after collecting exit status, freeing the process struct + handle table + MMU address space + 8 MiB user backing.  Today every test leaks the full child state: `sched_rr_purge_user_tasks` only unlinks tasks from the runqueue, never frees them.  Slice 7.6c.4 bumped `NX_PROCESS_TABLE_CAPACITY` + `MMU_MAX_ADDRESS_SPACES` 16 → 32 to paper over the leak; slice 7.6d.N.2 bumped 32 → 64; slice 7.6d.N.15 bumped 64 → 128.  Lands when the next round of escalations refills the 128-slot waterline (cap is plenty for now).

### Non-blocking opportunistic follow-ups

5. **Vector I/O syscalls beyond `writev` / `readv`.**  Slice 7.6c.3c + 7.6d.N.6a wired what musl's `__stdio_write` / `__stdio_read` need; the natural follow-on family returns `-ENOSYS`:
   - **`pwritev` / `preadv`** (`__NR_pwritev = 70` / `__NR_preadv = 69`) — positional vector I/O.  busybox doesn't use them; add when a real demo forces it.
   - **`pwritev2` / `preadv2`** (`__NR_pwritev2 = 287` / `__NR_preadv2 = 286`) — Linux-specific RWF_* flag extensions.  Almost no userspace uses them; defer indefinitely.
   - **`process_vm_writev` / `process_vm_readv`** (`__NR_process_vm_writev = 271` / `__NR_process_vm_readv = 270`) — cross-process memory access.  Big departure from our user-window-per-process model; defer until a multi-process debugger/tracer lands.

   Implementation pattern is already established by `sys_writev` (~30 lines kernel + 1 line each in `syscall_arch.h` and `syscall_cp.s` per addition).

6. **Phase 5 follow-ups still open:**
   - Per-task TTBR0 page table (kernel moves to high half via TTBR1).  Needed before multiple concurrent EL0 processes can share a single L1 root cleanly.
   - Type-aware handle duplicate so `dup` of a CHANNEL bumps the underlying refcount (today's plain duplicate breaks the invariant).
   - `copy_from_user` with page-fault fixup — current bounds-check is correct but doesn't handle faults gracefully.

7. **Test-harness follow-ups:**
   - Per-test quantum override.  `SCHED_RR_DEFAULT_QUANTUM_TICKS = 2` is a compromise; a `kernel.json` knob (once gen-config emits per-component config macros) lets tests tune per-build.
   - C-compiled EL0 crt0 for `main(argc, argv)` entry — needed when busybox expects POSIX main semantics directly.

8. **`verify-registry.py` extensions:**
   - pycparser-backed R1 / R6 (deep-dataflow, gated behind `--deep` so default `make test` stays fast).
   - R4 control-flow extension so "retain in enable, release only in error path" is flagged.
   - R8 call-graph once ISR / kthread entry points have a tagging convention.

9. **Infrastructure polish:**
    - Proper AArch64 Linux Image header so `-kernel` self-describes the load offset instead of hardcoding `0x40080000` in linker.ld.
    - `make test-host COMPONENT=X` filter (SPEC documents it; trivial once a future composition makes it useful).
    - Fix the `LOAD segment with RWX permissions` linker warning — unlocks with split-permission kernel pages (different PXN/AP on kernel text vs data blocks).

> The per-process memory management rework — formerly tracked here as "long-term, post-Phase-7" — is now [IMPLEMENTATION-GUIDE.md §Phase 9](IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework).  It owns the L3 4 KiB / VMA / demand-paging / COW work.

### Deferred indefinitely

- GUI support — long-term goal, not near-term.

---

## Session Logs

Each log captures that session's goals, decisions, findings, and next steps — the canonical narrative lives in the linked file.

1. **[Session 87](logs/session-87-8.0a.3-slot-call-skeleton.md)** (2026-04-30) — slice **8.0a.3** paperwork.  Source-side commit `def01b5` (in `sources/nonux/`) records the call-surface skeleton: `framework/slot_call.h` declares `nx_slot_call_blocking(struct nx_slot *, struct nx_ipc_message *, void *reply_buf, size_t reply_buf_len)` with doc strings matching SLOT-CALL-API.md §"API" verbatim; `framework/slot_call.c` returns `NX_EINVAL` on NULL args and `NX_ENOSYS` for any well-formed call (full body — pause-state, hook chain, cap-scan, dispatcher enqueue, reply-waitq block, ABORT-path reply synthesis — lands in slice 8.0a.6).  Wiring: `framework/slot_call.c` added to `FW_C` after `framework/ipc.c` in both `Makefile` and `test/host/Makefile` `SRCS`.  Two reasons for `NX_EINVAL`-vs-`NX_ENOSYS` branching: (1) future ktest fixtures distinguish "garbage args" from "op not wired"; (2) the NULL-arg validation is exercise even before the round-trip body lands.  No callers; no behavior delta.  This session's `proj_docs/nonux/` commit records the landing — session log + IMPLEMENTATION-GUIDE slice row + HANDOFF Phase checklist + Next Actions advance to slice 8.0a.4.  Tests: `make test` **495/495 pass** (same baseline as Session 86); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.  Next: slice **8.0a.4** (kernel `components/posix_shim/` skeleton + manifest + `kernel.json` wiring).
2. **[Session 86](logs/session-86-8.0a-kickoff.md)** (2026-04-30) — **Group B kicked off; slice 8.0a sub-sliced into 8.0a.1 → 8.0a.8**; **8.0a.1 (rename) and 8.0a.2 (IPC + slot scaffolding) landed**.  Three commits this session — proj_docs spec refinement (`37a70f3`) + source-repo `5f0b3e0` (8.0a.1 rename) + source-repo `93aa7b5` (8.0a.2 scaffolding).  **Spec refinement:** DESIGN.md gained 4 new subsections — §"Sync-mode caller must be on a dispatcher" (bounds `mode: sync` to dispatcher-thread callers; rules out syscall-entry edges); §"Tasks as IPC Senders" (formalizes per-task `caller_slot` as a graph entity); §"Multi-slot binding (N:1)" + §"Slot-side pause + blocking-call metadata" (documents `resume_waitq` + `in_flight_calls` slot fields with slice provenance); §"Edge inheritance" (per-task slots clone the boundary component's outgoing edges so cap-scan + pause hold-queue work uniformly).  Plus Invariant 1 extended to cover task-embedded `caller_slot`; ABORT-on-blocking-call synthesizes a reply (otherwise reply-waitq hangs); `NX_HOOK_SYSCALL_ENTER`/`_EXIT` retagged Phase 7 → Slice 8.7; `posix_shim → scheduler` flips `mode: sync → async` in 2 example sites.  SLOT-CALL-API.md reconciled — `nx_slot_call_blocking` signature gained explicit `(reply_buf, reply_buf_len)` arg pair instead of widening `nx_ipc_message`; new `NX_MSG_FLAG_REPLY_REQUESTED` request flag (existing `NX_MSG_FLAG_REPLY` is set on the reply itself); `NX_EABORT = -10` proposal dropped (slot `-10` is `NX_EPERM`; slice 8.0a reuses existing `NX_EABORT = -8` from slice 3.x); added `in_flight_reply_buf_len` field, truncation guard in `posix_shim_handle_msg`, recursive-call assertion + ABORT-path buf-clear in body sequence.  **Slice 8.0a.1 (rename):** path-only — `components/posix_shim/` → `components/libnxlibc/` (6 file renames staged from prior session), manifest name field, README cross-ref to slice 8.0a.2, 30 `test/kernel/*.c` `#include` updates, ~92 lines of Makefile recipe-path updates.  **Slice 8.0a.2 (scaffolding):** pure additive — `NX_MSG_FLAG_REPLY_REQUESTED (1u<<2)` request flag in `framework/ipc.h`; `struct nx_waitq resume_waitq` + `_Atomic(uint32_t) in_flight_calls` fields on `struct nx_slot`; `nx_slot_register` zero-initializes both via `nx_waitq_init` + `atomic_init`.  Required `make clean` mid-slice — `test/host/Makefile` lacks header-dep tracking, so bumping `sizeof(struct nx_slot)` invalidated stale host `.o` files (manifested as stack-smash detection in `generation_monotonically_increases`); follow-up opportunity: add `-MMD -MP` dependency tracking to host Makefile.  **Sub-slicing decision:** original slice 8.0a's scope (~70 ktests, posix_shim promotion + slot_call infra + per-task caller_slot + reply pool + STATE_LOST handler + cross-cutting test infrastructure) is too large for one commit; sub-sliced into 8 micro-slices so each landing is a green checkpoint.  Total scope and ~70-ktest budget unchanged.  IMPLEMENTATION-GUIDE.md §"Group B" table grew to show 8 sub-rows for 8.0a; HANDOFF.md Phase checklist tracks each.  **Tests:** `make test` **495/495 pass** (93 python + 283 host + 119 kernel — same baseline as Session 85; both 8.0a.1 and 8.0a.2 are zero-consumer changes); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.  Next: slice **8.0a.3** (`framework/slot_call.{h,c}` skeleton — header + body returning `NX_ENOSYS` until 8.0a.6 wires the dispatcher round-trip).
2. **[Session 85](logs/session-85-8.0pre.4-cutover.md)** (2026-04-30) — slice 8.0pre.4 landed; **Group A (generator infrastructure) closed**.  Doc-only slice.  No code-shape changes; no in-tree generated header was modified.  DESIGN.md gained §"Interface Definition Language" between "Build Flow" and "Runtime Config Manager" — formalizes the IDL pattern, lists the five generator artefacts per IDL (`<iface>.h`, `<iface>_msg.h`, `<iface>_call.h`, `<iface>_dispatch.h`, optional `<iface>_isr.h`), states explicitly that generator output is canonical and must not be hand-edited, documents `make verify-iface-fresh` as the machine check (wired since slice 8.0pre.1 as a `make all` / `make test` prerequisite).  "Build Flow" diagram extended with the IDL pipeline (interfaces/idl/*.json → gen-iface → verify-iface-fresh) alongside the existing kernel.json → validate-config → make pipeline.  R7's table row in DESIGN.md and R7's section in AI-RULES.md gained a parallel-rule note for IDL artefacts: machine-checked today via `verify-iface-fresh` (since the artefacts are committed); manifest-derived `gen/<name>_deps.h` keeps its existing AI-verified disposition (since `gen/` remains gitignored).  Latent dual-maintenance language pruned: `tools/idl-meta-schema.json`'s `constants` description rewrote ("when the hand-written header is replaced" → "carries the API surface that the IDL declares"); `includes` description expanded with the post-8.0pre.2 reality (suppresses auto-detected forward decls when non-empty).  `tools/README.md` dropped "post-cutover" hedge and the "match today's hand-written headers" type-system caveat; added the optional-`ctype` mention to `opaque_self_handle`.  `tools/gen-iface.py` source comments aligned ("matches today's hand-written conventions" → "the codified rule" / direct fact).  Makefile: `gen-iface` and `verify-iface-fresh` target comments updated to reference DESIGN.md §"Interface Definition Language" + record slice-8.0pre.1 → 8.0pre.4 lineage.  IDL-SCHEMA.md status line + cutover-scope subsection rewritten in past tense; explicit decision-record on **keeping the in-tree generated headers committed** (deletion would force every fresh clone + every CI run to invoke `make gen-iface` before any useful build, with no compensating safety gain since `verify-iface-fresh` already enforces no-hand-edit).  IMPLEMENTATION-GUIDE.md slice 8.0pre.4 row marked ✓ with one-paragraph summary; Phase 8 status + Last Updated footers (both copies) updated; Group A flagged complete.  Tests: `make test` **495/495 pass** (same baseline as Session 84), `make test-interactive` **7/7 pass**, `make verify-iface-fresh` 0 drift, `make verify-registry` 0 findings.  No new tests this slice (the existing byte-equal regeneration / banner-presence / verify-exit tests in `tools/tests/test_gen_iface.py` already cover the cutover invariants).  Next: slice 8.0a (`framework/slot_call.{h,c}` runtime infra + posix_shim component + per-task `caller_slot` + cross-cutting test infrastructure) — kicks off Group B.
3. **[Session 84](logs/session-84-8.0pre.3-sched-mm-char-device.md)** (2026-04-30) — slice 8.0pre.3 landed.  Three remaining IDLs (`scheduler`, `mm`, `char_device`) + IRQ-entry codegen.  All five production interfaces are now IDL-driven.  Schema extensions (three orthogonal, all backward-compatible): optional `ctype` on `opaque_self_handle` params (typed kernel-object pointers like `struct nx_task *task`); optional `ctype` on `void_ptr` returns (typed pointer returns like `pick_next` → `struct nx_task *`); new `u32` / `u64` return types for scalar unsigned-int ops (e.g. `mm.max_order`).  Generator extension: `framework/<iface>_isr.h` is now emitted for any IDL with at least one `context: "irq"` op; carries `NX_<IFACE>_ISR_POOL_SIZE` define (default 32 — IDL-SCHEMA documented value) + per-IRQ-op `nx_<iface>_<op>_from_irq(struct nx_slot *slot, ...)` declarations.  Bodies + pool storage land with slice 8.0a alongside `nx_slot_call_blocking`.  `interfaces/idl/scheduler.json` (~50 lines, 6 ops, `iface_id = 4100`) declares `core/sched/task.h` via `includes:` and uses typed `opaque_self_handle.ctype = "struct nx_task"` in 3 sites + typed `void_ptr.ctype = "struct nx_task"` for `pick_next`'s return.  `interfaces/idl/mm.json` (~50 lines, 4 ops, `iface_id = 4099`) uses `u32` return for `max_order` + `void_ptr` for `alloc_pages`; no `includes:`, no `constants:`.  `interfaces/idl/char_device.json` (~50 lines, 2 ops, `iface_id = 4101`) is the first IDL authored fresh with no hand-written predecessor; `rx_byte` op exercises `context: "irq"` end-to-end.  `interfaces/scheduler.h` and `interfaces/mm.h` regenerated to canonical generator output (banner + stdint added; `unsigned` → `uint32_t` canonicalization in mm.h's 3 sites; cosmetic spacing fixes).  C-level shape unchanged at the linker level — `uint32_t` is `typedef unsigned int` on aarch64-linux-gnu so `mm_buddy_alloc_pages`'s assignment to the function-pointer slot remains type-compatible without component-side changes.  **Decision history**: `struct nx_task *` representation prompted three candidate paths (new `kobject_ref` IDL type / extend `opaque_self_handle` with optional ctype / hijack `struct_*` types).  Took the extension path because `opaque_self_handle` already had the right wire shape (u64 in payload, opaque to sender's introspection); adding optional `ctype` adds C-level type information without inventing new wire semantics.  Symmetric extension to `void_ptr` returns falls out naturally.  Tests (`tools/tests/test_gen_iface.py`): +17 across 6 new classes — `TestSchedulerByteForByte` (2), `TestMmByteForByte` (1), `TestCharDeviceByteForByte` (2), `TestIrqEntryArtifact` (3), `TestTypedOpaqueSelfHandle` (4), `TestTypedVoidPtrReturn` (3), `TestU32Return` (2).  `make test` 478 → **495/495 pass** (93 python + 283 host + 119 kernel); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.  Generated artifacts NOT yet consumed by production code (verified by grep) so kernel build path is unaffected; activation lands with slice 8.0a (runtime infra) + slice 8.0b (component shim activation).  Next: slice 8.0pre.4 (cutover) or slice 8.0a (runtime infra) in either order or in parallel.
4. **[Session 83](logs/session-83-8.0pre.2-fs-iface.md)** (2026-04-30) — slice 8.0pre.2 landed.  Second IDL after vfs.  Data shapes factored into hand-written `interfaces/fs_types.h` (carrying `struct nx_fs_dirent`, `struct nx_fs_stat`, `NX_FS_DIRENT_NAME_MAX`, `NX_FS_KIND_*`); both `fs.json` and `vfs.json` declare it via the existing `includes:` schema array.  Generator (`tools/gen-iface.py`) extended: author-declared `includes:` now emit in **both** typedef header (`<iface>.h`) and msg header (`<iface>_msg.h`) — the msg header embeds struct values by sizeof in per-op request/reply structs, needs the full def in scope; auto-detected forward declarations suppressed when `includes:` is non-empty (the includes' definitions serve as declarations, redundant forward decls eliminated).  **No new schema fields** — IDL describes operations only; data layout stays in C.  `interfaces/idl/fs.json` (~110 lines): top-level doc preserves the four prose blocks (Path convention / Ownership / Concurrency / Error convention) verbatim from `fs.h`; 7 constants in 2 groups (NX_FS_OPEN_*, NX_FS_SEEK_* — type-coupled constants live in `fs_types.h`); 9 ops (op-IDs 1–9 mirror vfs by deliberate shape parity); `iface_id = 4098`.  `interfaces/idl/vfs.json` modified: gained `includes: [{path: "interfaces/fs_types.h"}]`; lost `forward_decls_doc` (unused now that includes-driven path skips auto-detected forward decls).  **Canonicalization decisions** (per DESIGN.md R7 — IDL is source of truth post-cutover): per-value `NX_FS_SEEK_*` trailing comments folded into the SEEK group's leading doc rather than adding a `trailing_doc` schema field; `int (*open)(...)` un-wrapped because it fits one line at 79 chars after greedy pack; `int (*readdir)(...)` rewrapped after `cookie,` (was after `dir_path,` in source); `/* ---------- Section ----- */` divider comments deleted (each section's prose lives on the leading constant or struct doc).  C-level shape unchanged — same `#define`s, same struct field types + names, same op signatures — so callers compile identically.  **Decision history**: initial draft prototyped a `structs:` schema extension for inline struct definitions (verbatim-string `body` field per struct); user redirected to the types-header path on the principle that IDL describes operations and C describes data layout.  The `structs:` work was reverted; final landing uses only the existing `includes:` field plus a small generator extension.  See [Session 83 log](logs/session-83-8.0pre.2-fs-iface.md) §Decisions Made for the trade-off analysis.  **Side benefit**: latent `_msg.h` compile gap noted at the 8.0pre.1 boundary (referenced struct types inline without including their definitions) is now closed — both `fs_msg.h` and `vfs_msg.h` `#include "interfaces/fs_types.h"` automatically.  **8.0pre.4 cutover scope narrowed**: hand-written *types* headers survive the cutover; only ops headers get deleted.  Tests (`tools/tests/test_gen_iface.py`): +7 across 2 new classes — `TestFsByteForByte` (in-tree fs.h byte-equal to fresh regeneration; IDL declares fs_types.h via `includes:`) and `TestIncludesAuthorDeclared` (`includes:` emitted in both header types; forward-decl auto-detection suppressed when `includes:` present; legacy behavior preserved when `includes:` absent; both headers carry same include list).  `make test` 471 → **478/478 pass** (+7 new tests); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.  Next: slice 8.0pre.3 (sched, mm, char_device — char_device exercises IRQ-entry ops + the unimplemented `<iface>_isr.h` generator template) or slice 8.0a (runtime infra) in either order or in parallel.
(Sessions 80, 81, 82 archived to [HANDOFF-ARCHIVE.md](HANDOFF-ARCHIVE.md) per the "keep last 5" convention.)
Older entries: see [HANDOFF-ARCHIVE.md](HANDOFF-ARCHIVE.md) (Sessions 1–82).

---

## Document Web

**Related Documents:**
- [SPEC.md](SPEC.md) - Requirements and constraints
- [DESIGN.md](DESIGN.md) - Architecture and design decisions
- [IMPLEMENTATION-GUIDE.md](IMPLEMENTATION-GUIDE.md) - Code-level details
- [README.md](README.md) - Project overview

**When adding new entries:**
- Create new session log in `logs/` directory
- Update [Session Logs](#session-logs) section with link (keep last 5; move older to archive)
- Update [Current Status](#current-status) with progress
