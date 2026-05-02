# Handoff: nonux

**Project:** nonux
**Started:** 2026-04-17

---

## Navigation

**Project Docs:** [README](README.md) | [SPEC](SPEC.md) | [DESIGN](DESIGN.md) | [IMPLEMENTATION-GUIDE](IMPLEMENTATION-GUIDE.md) | [HANDOFF](HANDOFF.md) *(you are here)* | [IDL-SCHEMA](IDL-SCHEMA.md) | [SLOT-CALL-API](SLOT-CALL-API.md)

**Quick Jump:** [Current Status](#current-status) | [Next Actions](#next-actions) | [Session Logs](#session-logs)

---

## Current Status

**Phase:** Phases 1–6 complete.  **Phase 7 closed** (Session 78).  **Phase 8 in progress.**  Plan landed Session 79; IDL schema + slot-call API locked Sessions 80–81; slices 8.0pre.1 → 8.0pre.4 landed Sessions 82–85 — **Group A (generator infrastructure) closed at Session 85**.  **Group B kicked off Session 86:** original slice 8.0a sub-sliced into 8.0a.1 → 8.0a.8; all sub-slices landed Sessions 86–92 — **Group B slice 8.0a closed Session 92**.  **Slice 8.0b landed Session 93** — activated generated `handle_msg` shims on all 5 production components + `uart_pl011`; `reply_payload_len` added to `nx_ipc_message`; `build_reply` copies per-op reply struct in-place; `bytes_in`/`bytes_out` pointer-encoded as `uint64_t`; +24 host tests.  **Next forward step: slice 8.0c** (migrate framework production paths to wrappers).

**Tests:** `make test` → **590/590 pass** (93 python + 374 host + 123 kernel — +24 tests vs. the Session 92 baseline of 566); `make test-interactive` → **7/7 pass** (echo_cat, echo_hello, echo_pipe, ls_root, mkdir_tmp, ps_smoke, visible_prompt).  `make verify-iface-fresh`: 0 drift.  `make verify-registry`: 0 findings.

**Latest session log:** [Session 93](logs/session-93-8.0b-handle-msg-shims.md) — slice 8.0b landed (activates generated `handle_msg` shims on all production components).

**Blockers:** None.  Next forward step is slice **8.0c** — see Next Actions below.

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
>         - :ballot_box_with_check: 8.0a.4 kernel `components/posix_shim/` skeleton + manifest + `kernel.json` wiring (composition: 7 slots / 7 components) — Session 88
>         - :ballot_box_with_check: 8.0a.5 per-task `caller_slot` in `struct nx_task` + `reply_waitq` + `in_flight_reply_buf{,_len,_rc}` + lifecycle in `nx_task_create`/`_destroy` + edge cloning of posix_shim's outgoing deps — Session 89
>         - :ballot_box_with_check: 8.0a.6 reply-message pool + dispatcher integration + `nx_slot_call_blocking` body end-to-end + `posix_shim_handle_msg` reply routing (first end-to-end blocking call against uart_pl011 in a ktest) — Session 90
>         - :ballot_box_with_check: 8.0a.7 `posix_shim_on_dep_swapped` STATE_LOST handler + `nx_handle_table_invalidate_for_slot()` — Session 91
>         - :ballot_box_with_check: 8.0a.8 cross-cutting test infrastructure (mock component, hook-chain inspector, recompose event logger, pause-injector fixture, cap-forgery harness, equivalence-runner macro) — closes 8.0a — Session 92
>         - :ballot_box_with_check: 8.0b activate generated `handle_msg` shims on every component — Session 93
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
- **Build / run** — `make` → `kernel.bin`; `make test` → 519/519 (93 python + 305 host + 121 kernel); `tools/run-qemu.sh [-t SECONDS]` boots in QEMU; `make validate-config` / `make verify-registry` / `make verify-iface-fresh` gate the build
- **kernel.json** — declarative config currently wiring seven slots: `char_device.serial ← uart_pl011`, `scheduler ← sched_rr`, `memory.page_alloc ← mm_buddy`, `filesystem.root ← ramfs`, `filesystem.proc ← procfs` (slice 7.7b.2), `vfs ← vfs_simple`, `posix_shim ← posix_shim` (slice 8.0a.4). Booted kernel reports `composition (gen=N, 7 slots, 7 components)` with all seven ACTIVE
- **Directory structure** — `core/` (boot, cpu, pmm, irq, lib), `framework/`, `interfaces/`, `components/`, `test/` (host + conformance, kernel, bench), `tools/`, `docs/`
- **MIT License**

Key design decisions — see [DESIGN.md §Key Design Decisions](DESIGN.md#key-design-decisions) and the topical sections it cross-references (Component Graph Registry, Pause Implementation, Slot-Based Indirection, AI Verification, Dependency Injection Mechanism, Component-Spawned Threads).

---

## Next Actions

### Forward step

1. **Phase 8 — Runtime Recomposition + Config Manager.**  Plan landed Session 79; expanded to 16 slices Session 81 (full breakdown in [IMPLEMENTATION-GUIDE.md §Phase 8](IMPLEMENTATION-GUIDE.md#phase-8-runtime-recomposition-and-config-manager); session logs at [Session 79](logs/session-79-phase-8-plan.md) + [Session 80](logs/session-80-idl-schema.md) + [Session 81](logs/session-81-slot-call-api.md)).  Today's production paths bypass the slice-3.8 IPC router — ~184 direct `slot->active->descriptor->iface_ops` callsites across 19 files, all five production components have `handle_msg = NULL`.  A live recompose against this surface is a use-after-free.  **Group A (8.0pre.1–4) closed at Session 85** — JSON-IDL-driven code generator emits typedef + msg + call + dispatch (+ optional ISR) headers per interface; all five production interfaces declared via IDL; generator output canonical and machine-checked via `make verify-iface-fresh`.  **Group B (8.0a → 8.0e)** routes every cross-component call through `nx_slot_call_blocking`, activates the generated shims on every component, migrates framework + component-to-component callsites, and locks down with a `verify-registry` rule — slot-call API + posix_shim promotion + per-task slot model locked Session 81 in [SLOT-CALL-API.md](SLOT-CALL-API.md).  **Session 86 sub-sliced 8.0a into 8.0a.1 → 8.0a.8** so each landing is a green checkpoint; **8.0a.1–8.0a.6 landed Sessions 86–90** — the first end-to-end blocking call now round-trips through the dispatcher in a kernel ktest.  **Group C (8.1–8.7)** ships the original Phase 8 deliverables plus syscall-boundary hooks: kernel-side pause/drain/resume, `framework/recompose.c` orchestrator, config handle API + EL0 syscall, second scheduler, live-swap demo, per-edge async↔sync mode switching, `NX_HOOK_SYSCALL_ENTER`/`_EXIT`.  ~270+ new ktests across the migration; cross-cutting test infrastructure lands in slice **8.0a.8**; per-callsite equivalence tests land *before* migration in slice 8.0c.  Hot-path perf gate revised Session 81 — original ≤10% regression unattainable with Resolution 1; replaced with ≤10 µs round-trip on QEMU virt aarch64 + revisit absolute perf in Phase 10.  Estimated ~9–18 sessions remaining (sub-slicing of 8.0a is structural, not scope-expanding).

   **Immediate next step:** Slice **8.0c** — migrate framework production paths to wrappers.  `syscall.c` (~30 direct `vops->read/write/...` callsites), `dispatcher.c`, `bootstrap.c`, `component.c`, `process.c` all reach `slot->active->descriptor->iface_ops` directly; 8.0c replaces each site with `nx_slot_call_blocking`.  Per-callsite equivalence tests land *before* the migration (pinning existing behavior), wrappers swap in, equivalence asserted again.  Hot-path perf checkpoint: `read`/`write`/`open` round-trip cycles measured before and after, gated to ≤10 µs on QEMU virt aarch64.  `make test` baseline + interactive 7/7 must stay green throughout.

   **Tests at end of Session 93:** `make test` **590/590 pass** (93 python + 374 host + 123 kernel; +24 vs. the Session 92 baseline of 566); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.

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

1. **[Session 92](logs/session-92-8.0a.8-test-infra.md)** (2026-05-01) — slice **8.0a.8** landed — **closes 8.0a**.  Source-side commit `d983e7a` (in `sources/nonux/`) lands six reusable test infrastructure pieces: `test/host/mock_component.h` (header-only programmable mock — `struct mock_handle` with embedded `mock_state` holding `handler_rc`, `call_count`, `last_msg`, optional `custom_fn`; `comp.impl = &state` pattern lets two independent mocks coexist without globals); `test/host/hook_inspector.h` (header-only observe-only hook — records up to 32 firings with `ipc_src/dst/msg`, always `NX_HOOK_CONTINUE`); `test/host/slice_8_0a8_test.c` (36 host tests across all 6 infra pieces — mock tests, hook-chain inspector tests, recompose event logger via `nx_graph_subscribe` + `nx_change_log_read`, pause-injector tests for REJECT/REDIRECT/state-transition coverage, cap-forgery harness for `nx_ipc_scan_send_caps`/`_recv_caps` R3 enforcement, `ASSERT_CALL_EQUIVALENT` macro validated against direct-vs-dispatcher paths); +2 kernel ktests in `ktest_bootstrap.c` (`hook_inspector_observe_only_does_not_alter_blocking_call_result` — IPC_SEND observer fires ≥1, result 0; `cap_scan_rejects_forged_slot_ref_cap_during_blocking_call` — `filesystem.root` cap forged by task's caller_slot → NX_EINVAL, in_flight_reply_buf cleared).  Snapshot buffer bumped 4096 → 8192 (three un-destroyed kthreads from slices 8.0a.6/.7/.8 each add ~250 bytes of JSON via registered `caller_slot` + 4 edges, pushing JSON past 4 KiB; ktest linker ordering puts new tests before the snapshot test).  Discovered that ktest linker section places tests in object-link order (not source order) — new tests appended at file end ran BEFORE the snapshot test.  `make test` 528 → **566/566 pass** (93 python + 350 host + 123 kernel; +38); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.  Next: slice **8.0b** (activate generated `handle_msg` shims on every component).
2. **[Session 91](logs/session-91-8.0a.7-handle-invalidation.md)** (2026-05-01) — slice **8.0a.7** landed.  Source-side commit `68b4718` (in `sources/nonux/`) closes handle-table invalidation on dep swap.  `struct nx_handle_entry` gains `struct nx_slot *slot` (NULL = immune); `nx_handle_alloc` / `nx_handle_table_init` / `nx_handle_close` zero it; `nx_handle_duplicate` propagates it.  New `nx_handle_alloc_with_slot()` + `nx_handle_set_slot()` helpers wired in `sys_open`'s FILE and DIR paths (the only two syscall.c callers in v1).  `nx_handle_table_invalidate_for_slot(dep_slot)` walks every process's handle table via `nx_process_for_each`, bumps generation + clears entries whose `slot == dep_slot`.  `posix_shim_on_dep_swapped` calls it when `NX_SWAP_STATE_LOST` is set.  `framework/component.h` gains `NX_SWAP_STATE_LOST (1u << 0)` and the `on_dep_swapped` callback slot in `nx_component_ops`.  **Tests:** +9 host (`handle_test.c`).  `make test` 519 → **528/528 pass** (93 python + 314 host + 121 kernel); `make test-interactive` **7/7 pass**; `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings.  Next: slice **8.0a.8**.
3. **[Session 90](logs/session-90-8.0a.6-reply-path.md)** (2026-05-01) — slice **8.0a.6** landed.  Source-side commit `34dd1cb` (in `sources/nonux/`) closes the blocking-call round-trip end to end.  `framework/dispatcher.c` gains 256-entry reply pool; `nx_slot_call_blocking` body complete (validate → pause-state walk → IPC_SEND hook → cap-scan → enqueue → block on reply_waitq); `posix_shim_handle_msg` routes reply payload + wakes waitq.  **Tests:** +16 host + 1 ktest (first end-to-end blocking call against uart_pl011).  `make test` 502 → **519/519 pass**.  Next: slice **8.0a.7**.
4. **[Session 89](logs/session-89-8.0a.5-caller-slot.md)** (2026-05-01) — slice **8.0a.5** landed.  Source-side commit `0917023` — per-task `caller_slot` + `reply_waitq` + `in_flight_reply_buf{,_len,_rc}` + lifecycle in `nx_task_create`/`_destroy` + edge cloning of posix_shim's outgoing deps.  **Tests:** +6 host + 1 ktest.  `make test` 495 → **502/502 pass**.  Next: slice **8.0a.6**.
5. **[Session 88](logs/session-88-8.0a.4-posix-shim-skeleton.md)** (2026-04-30) — slice **8.0a.4** landed.  Source-side commit `d0d30f0` — kernel-side `components/posix_shim/` skeleton + manifest + `kernel.json` wiring (7 slots / 7 components); gen-config dotted-dep-name extension; R2 skip for gen-config DI; snapshot buffer 2048 → 4096.  `make test` **495/495 pass**.  Next: slice **8.0a.5**.
(Sessions 80–87 archived to [HANDOFF-ARCHIVE.md](HANDOFF-ARCHIVE.md) per the "keep last 5" convention.)
Older entries: see [HANDOFF-ARCHIVE.md](HANDOFF-ARCHIVE.md) (Sessions 1–87).

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
