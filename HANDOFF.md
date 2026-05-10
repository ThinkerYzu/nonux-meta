# Handoff: nonux

**Project:** nonux
**Started:** 2026-04-17

---

## Navigation

**Project Docs:** [README](README.md) | [SPEC](SPEC.md) | [DESIGN](DESIGN.md) | [IMPLEMENTATION-GUIDE](IMPLEMENTATION-GUIDE.md) | [HANDOFF](HANDOFF.md) *(you are here)* | [IDL-SCHEMA](IDL-SCHEMA.md) | [SLOT-CALL-API](SLOT-CALL-API.md)

**Quick Jump:** [Current Status](#current-status) | [Next Actions](#next-actions) | [Session Logs](#session-logs)

---

## Current Status

**Phase:** Phases 1–7 complete.  **Phase 8 CLOSED** (Session 104).  **Phase 9b started** — slices 9b.1–9b.4 CLOSED (Sessions 105–108).  Plan landed Session 79; IDL schema + slot-call API locked Sessions 80–81; slices 8.0pre.1 → 8.0pre.4 landed Sessions 82–85 — **Group A (generator infrastructure) closed at Session 85**.  **Group B kicked off Session 86:** original slice 8.0a sub-sliced into 8.0a.1 → 8.0a.8; all sub-slices landed Sessions 86–92 — **Group B slice 8.0a closed Session 92**.  **Slice 8.0b landed Session 93** — activated generated `handle_msg` shims on all 5 production components + `uart_pl011`.  **Session 94 — slice 8.0c infrastructure + ktest payload fix.**  **Session 95 — slice 8.0c CLOSED:** root cause of the `syscall.c` migration hang found and fixed.  **Session 96 — slice 8.0d CLOSED:** migrated component-to-component calls (vfs_simple → ramfs/procfs) through new `nx_fs_*` sync-dispatcher wrappers (`framework/fs_call.c`).  Key findings: (1) kernel path uses direct `handle_msg` call (not `nx_slot_call_blocking`) since vfs_simple is dispatcher-context; (2) GCC -O2 strict-aliasing heisenbug — in-place reply write through type-punned pointer requires `__asm__ volatile(""::: "memory")` barrier before reading reply; (3) `resolve_fs()`/`resolve_for_path()` removed from vfs_simple.  **Session 97 — slice 8.0e CLOSED:** `verify-registry` R9 rule banning `->iface_ops` reads outside `framework/dispatcher.c` — exemptions: `bootstrap.c` by filename, `#if __STDC_HOSTED__` fast-paths, C comment lines; 0 findings on current codebase.  **Group B complete.**  **Session 98 — slice 8.1 CLOSED:** kernel-side pause/drain/resume.  Moved `in_flight_calls` bump from handler-invocation to enqueue in `nx_dispatcher_enqueue` (kernel build); `slot_drain_cb` on kernel now yields until `in_flight_calls == 0` instead of no-op `nx_ipc_dispatch`.  Added ABORT-path decrement.  3 new ktests.  QEMU timeout bumped 300 → 360 s.  **Session 99 — slice 8.2 CLOSED:** `framework/recompose.c` orchestrator.  `struct recomp_plan`, `nx_recompose()`, Kahn's topological pause order from registry edges, rollback on pause failure, connection rewiring, `slot_clear_pause` for replaced components, `timer_pause()`/`timer_resume()` bookends, `on_dep_swapped` notification.  6 host tests + 2 kernel tests.  Key finding: `nx_waitq_wake_all` on `resume_waitq` deferred — unsafe to call from `slot_resume_cb` after dispatcher drain yields (kernel crash reproduced and root-caused).  **Session 100 — slice 8.3 CLOSED:** `framework/config.c` runtime config manager.  `nx_config_open()`, `nx_config_query_snapshot()`, `nx_config_swap_component()`; three new syscalls `NX_SYS_CONFIG_OPEN/QUERY/SWAP` (42–44); `NX_HANDLE_CONFIG` handle type.  10 host tests + 4 kernel tests.  Key findings: (1) kernel Makefile has no header dep tracking — stale `handle.o` rejected new `NX_HANDLE_CONFIG=9`; fix: `make clean` when header enums change; (2) registry polluted with leaked `caller_slot` entries from non-reaped child tasks — ktest snapshot check uses `nx_slot_lookup` instead of snapshot iteration for name validation.  **Session 101 — slice 8.4 CLOSED:** `components/sched_priority/` second scheduler.  8 priority buckets (0=default, 7=highest); `set_priority` works.  `__weak` shim exports `sched_rr_purge_user_tasks` for ktest teardown compat.  `kernel.json` now boots with `sched_priority`.  16 host tests + ktest bootstrap/sched updated.  **Session 102 — slice 8.5 CLOSED:** headline live scheduler swap.  `kernel.json` `"alternatives": ["sched_rr"]` → both schedulers compiled in; `bootstrap.c` only enables bound components (alternatives stay READY); both enable hooks call `sched_init` so the global scheduler driver updates on live swap; `sched_rr_purge_user_tasks` dispatch routes to sched_priority when it is active; `test/kernel/ktest_live_swap.c`: 3 kthreads survive `sched_priority → sched_rr` swap, behaviour change observable (`set_priority` NX_OK → NX_EINVAL); pre-drain guard prevents `timer_pause + idle-WFI` deadlock from stale ppoll blocked tasks; QEMU timeout 360 → 450 s.  2 new kernel tests.  **Session 103 — slice 8.6 CLOSED:** runtime async↔sync mode switching.  `build_pause_order` extended to pause `to_slot` for `NX_CONN_REWIRE` changes; `nx_config_set_conn_mode()` convenience wrapper + `NX_SYS_CONFIG_REWIRE` (45) EL0 syscall; 6 host tests + 2 kernel tests.  Key findings: (1) pre-existing dangling slot-node bugs in ktest_pause/recompose (edge not unregistered before slot) fixed — documented in TESTING-GUIDE.md §"Kernel test fixture pattern" with static-slot design; (2) `sched_rr_enable` latent bug: g_idle_task was never enqueued into sched_rr on live swap — fixed by enqueuing in the enable hook; (3) Makefile `#` comment inside `KTEST_C` backslash-continued list silently drops subsequent entries — discovered and fixed; QEMU timeout 450 → 900 s.  **Session 104 — slice 8.7 CLOSED:** `NX_HOOK_SYSCALL_ENTER`/`_EXIT` hook points.  `sc` union arm in `nx_hook_context` exposes `num`, `a[6]`, `rc` pointer, `tf`; ENTER ABORT skips body; EXIT hook can override return value.  5 ktests.  Key finding: `NX_HOOK_POINT_COUNT` enum shift (8→10) caused stale `component_hook_test.o` to encode value 8 (now `NX_HOOK_SYSCALL_ENTER`) as "bad point" — `make clean` required, same pattern as Session 100.  **Phase 8 closed.**

**Tests:** `make test-tools` → **102/102 pass**; `make test-host` → **485/485 pass**; `make test-interactive` → **7/7 pass** (last run Session 115); `make test-kernel` → **152/152 pass** (3600 s).  `make verify-iface-fresh`: 0 drift.  `make verify-registry`: 0 findings (R2,R4,R9).  Sessions 124–128 are documentation-only (book chapters 2–6 + Part II/III swap); test counts unchanged since session 123.

**Latest session log:** [Session 128](logs/session-128-book-chapter-6.md) — **book chapter 6 — Virtual memory and the MMU**.  `book/06-virtual-memory-and-mmu.md` (1247 lines) walks `core/mmu/mmu.c` end-to-end in three layers.  *MMU bring-up:* the three-knob configuration (4 KiB granule, 39-bit VA via `T0SZ=25`, three-level walk); the descriptor bit-layout decode (V/Tbl/AttrIdx/AP/SH/AF/PA/PXN/UXN); the three nonux helpers `device_block` (PXN+UXN MMIO), `normal_block` (UXN-only kernel RAM), `user_block` (cleared UXN, `AP=0b01` user-RW); `MAIR_VALUE` indirection (Device-nGnRnE @ idx 0, Normal WB-WA @ idx 1) with side note on *why* the indirection exists; bit-by-bit `TCR_VALUE` decode (T0SZ/IRGN/ORGN/SH0/TG0/EPD1=1/IPS); the static L1+L2_mmio+L2_ram tables and the L1 wiring; a worked example walking VA `0x4009_0000` → PA `0x40090000` and VA `0x0900_1000` → PA `0x09001000` through the same machinery; the SCTLR enable sequence step-by-step (`dsb ish` → MAIR/TCR/TTBR0 → `isb` → `tlbi vmalle1` → `dsb ish` → `ic iallu` → `dsb ish` → `isb` → `SCTLR.{M,C,I}` → final `isb`) with side note on *why the kernel survives its own MMU enable* (identity-map invariant); the `CPACR_EL1.FPEN` tweak so musl FP-using `memset`/`memcpy` doesn't trap.  *User window:* slot-64 `user_block` upgrade in the kernel `l2_ram_table`, the `64 << 21` arithmetic giving window VA `0x4800_0000`, `drop_to_el0`'s `eret` as the demotion entry, and how AP+UXN bits enforce EL0 confinement.  *Per-process address spaces:* `mmu_create_address_space`'s three allocations (L1 page, L2_ram page, 8 MiB user backing); the "shared kernel mappings, private user window" invariant as a 2-table ASCII diagram; `mmu_switch_address_space`'s TTBR0 + `tlbi vmalle1` sequence with side note on ASIDs as a future optimisation; `mmu_copy_user_backing`'s TTBR0-save-to-kernel-then-restore trick around the fork memcpy with the user-window-aliasing rationale; closes the loop on chapter 5's `pmm_reserve_range(mmu_user_window_base(), …)` forward-pointer in §"A boot-time consequence: PMM reservation".  Full ASCII diagram tying physical RAM, kernel L1 identity map, and per-process L1s together.  Seven side notes (no high-half kernel yet, no ASIDs yet, no demand paging yet, page tables come from kheap and that's why PMM reservation matters, one MMU per core, MMU also handles unaligned access, MMU is also what makes `kprintf("%s", bad_ptr)` a clean abort).  Two cross-link patches in chapter 5 turn its "next chapter on the MMU" forward-pointers into real chapter-6 links.  `BOOK-OUTLINE.md` chapter-6 row `planned → shipped` with the "drafted pre-Phase-9" annotation; `book/README.md` chapter-6 entry now a real link.  **Part III complete (chapters 5–6).**  Documentation-only; test counts unchanged.  `book/05-physical-memory-and-pmm.md` (1123 lines) walks the path from "the kernel image ends at `__free_mem_start`, RAM ends at `RAM_END`" to "the buddy component bound to `memory.page_alloc`", in two halves.  PMM half: the bitmap layout + self-accounting init (bitmap pages mark themselves allocated); `bit_test` / `try_claim` / `release` with `__atomic_fetch_or` ACQUIRE / `__atomic_fetch_and` RELEASE; `pmm_alloc_page` first-fit-with-hint walked line-by-line; `pmm_free_pages` constant-time + the "set hint backwards on free" working-set rationale; `pmm_alloc_pages` two-pass (cheap scan, then atomic claim with rollback); `pmm_reserve_range` for the user-window aliasing case (idempotent via `try_claim`).  Side note: *why single-CPU still needs atomics* (interrupt-during-RMW failure mode anchors atomics in chapter 4's preemption model).  Buddy half: three-reason motivation (power-of-two requests, coalescing, swappable strategy); `nx_mm_ops` interface contract (forward-points to chapter 10 IDL toolchain); 16-page pool geometry (intentionally tiny); intrusive `free_node` in-place free-list trick; **alloc = split until you fit** with worked example showing free-list state through one order-2 alloc; **free = merge while the buddy is free** with the address-XOR insight (`addr ^ block_size`) and a worked example showing three allocs/three frees that round-trip back to the original order-4 block; lifecycle (`init` carves, `destroy` returns, `enable`/`disable` no-ops by design).  Annotation pass on the real boot-log lines so the reader can map `[pmm] total=… free=…` to the source code.  Full ASCII layered diagram closing the chapter (RAM → PMM → kheap + mm_buddy → consumers).  `BOOK-OUTLINE.md` chapter-5 row `planned → shipped`; `book/README.md` chapter-5 entry now a real link.  **Part III started (chapter 5 of 18 shipped).**  Documentation-only; test counts unchanged.

**Blockers:** None.

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
> - :ballot_box_with_check: Phase 8 (runtime recomposition + config manager) — closed Session 104, 15 slices in 3 groups:
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
>         - :ballot_box_with_check: 8.0c migrate framework production paths — `syscall.c` (14 `vops->` callsites + `sys_getdents64` kstack fix) — Session 95
>         - :ballot_box_with_check: 8.0d migrate component-to-component calls (vfs_simple → ramfs/procfs) — `framework/fs_call.c` sync-dispatcher wrappers + aliasing barrier — Session 96
>         - :ballot_box_with_check: 8.0e `verify-registry` R9 banning `->iface_ops` outside `framework/dispatcher.c` — Session 97
>     - **Group C — runtime recomposition**
>         - :ballot_box_with_check: 8.1 lift pause/drain/resume from host into kernel build + slice-3.9 rollback — Session 98
>         - :ballot_box_with_check: 8.2 `framework/recompose.c` orchestrator + topo pause order + leaf-slot demo — Session 99
>         - :ballot_box_with_check: 8.3 `framework/config.c` runtime config manager + handle API + EL0 syscall surface — Session 100
>         - :ballot_box_with_check: 8.4 second scheduler (`sched_priority`) — standalone validation — Session 101
>         - :ballot_box_with_check: 8.5 **headline** live scheduler swap with running tasks — Session 102
>         - :ballot_box_with_check: 8.6 runtime async↔sync mode switching — Session 103
>         - :ballot_box_with_check: 8.7 `NX_HOOK_SYSCALL_ENTER` / `_EXIT` hook points — closes Phase 8 — Session 104
> - :ballot_box_with_check: Phase 9b slice 9b.1 — component-owned object tables, fs/vfs/char_device IDL id:u32, ramfs/procfs/vfs_simple/uart_pl011 updated (Session 105)
> - :ballot_box_with_check: Phase 9b slice 9b.2 — `NX_HANDLE_RESOURCE`, union id/object, slot→target; sys_open/close/read/write/seek/ioctl/fork/dup3/fcntl/ppoll updated; console pre-install switched (Session 106)
> - :ballot_box_with_check: Phase 9b slice 9b.3 — `nx_char_device_write/read` via `nx_slot_call_blocking`; sys_read/sys_write RESOURCE arms route through slot calls; `char_device_call.c` created; 10 kernel tests (Session 107)
> - :ballot_box_with_check: Phase 9b slice 9b.4 — retired `NX_HANDLE_FILE` + `NX_HANDLE_CONSOLE`; dead FILE/CONSOLE arms removed from 9 syscall sites; `NX_HANDLE_DIR` deferred (Session 108)
> - :white_large_square: Phase 9 (per-process MM rework), Phases 10–11 (integration tests + benchmarks, docs + AI operability)

### What We Have

Build state and infra (per-slice detail is in the [Session Logs](#session-logs) and authoritative docs live in [IMPLEMENTATION-GUIDE.md](IMPLEMENTATION-GUIDE.md); architecture in [DESIGN.md](DESIGN.md)):

- **Source repo** — local `sources/nonux/` ↔ public https://github.com/ThinkerYzu/nonux (MIT, default branch `master`, pushed 2026-04-22)
- **Toolchain (verified 2026-04-20)** — `aarch64-linux-gnu-gcc` 15.2.0, `qemu-system-aarch64` 10.1.0
- **Build / run** — `make` → `kernel.bin`; `make test` → 590+/590+ (93 python + 397 host + 123 kernel); `tools/run-qemu.sh [-t SECONDS]` boots in QEMU; `make validate-config` / `make verify-registry` / `make verify-iface-fresh` gate the build
- **kernel.json** — declarative config currently wiring seven slots: `char_device.serial ← uart_pl011`, `scheduler ← sched_priority` (slice 8.4; was `sched_rr`), `memory.page_alloc ← mm_buddy`, `filesystem.root ← ramfs`, `filesystem.proc ← procfs` (slice 7.7b.2), `vfs ← vfs_simple`, `posix_shim ← posix_shim` (slice 8.0a.4). Booted kernel reports `composition (gen=N, 7 slots, 7 components)` with all seven ACTIVE
- **Directory structure** — `core/` (boot, cpu, pmm, irq, lib), `framework/`, `interfaces/`, `components/`, `test/` (host + conformance, kernel, bench), `tools/`, `docs/`
- **MIT License**

Key design decisions — see [DESIGN.md §Key Design Decisions](DESIGN.md#key-design-decisions) and the topical sections it cross-references (Component Graph Registry, Pause Implementation, Slot-Based Indirection, AI Verification, Dependency Injection Mechanism, Component-Spawned Threads).

---

## Next Actions

### Forward step

1. **Phase 9 — per-process MM rework.**  Phase 9b closed (Session 108), post-9b fixes landed Sessions 109–116 (el0_file, posix_musl, ktest failures, idle latency, reap-on-wait, POSIX dot entries, chdir/getcwd).  Sessions 117–128 covered repo hygiene + tutorial-style docs (libnxlibc relocation, subdirectory READMEs, generated-HTML cleanup, book chapters 1–6 — Parts I, II, III complete, book style guide + EL section, book outline).  All tests pass.  Next: either start Phase 9 — L3 4 KiB pages, per-process VMAs, demand paging, COW fork (see [IMPLEMENTATION-GUIDE.md §Phase 9](IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework)) — or continue the book with chapter 7 (Kernel threads and context switching, first chapter of Part IV per [BOOK-OUTLINE.md](BOOK-OUTLINE.md#chapter-list)).  Note: chapter 6 was drafted pre-Phase-9 per the per-chapter call recorded in [Session 128](logs/session-128-book-chapter-6.md); a revision pass on chapter 6's per-process / fork sections is scheduled when Phase 9 lands.

   **Tests at end of Session 128:** `make test-tools` **102/102 pass**; `make test-host` **485/485 pass**; `make test-interactive` **7/7 pass** (last run Session 115); `make test-kernel` **152/152** (3600 s).  `make verify-iface-fresh` 0 drift; `make verify-registry` 0 findings (R2,R4,R9).

### Deferred — actionable when a workload demands

3. **7.6d.N.final.c-full — full signal-handler dispatch trampoline.**  Promote `sys_rt_sigaction` from no-op stub to actually-installs-handler (per-process `sigaction[NSIG]` table); EL0-return path pushes a synthetic frame on the user stack, sets PC to the handler, sets x0 = signo, sets LR to an EL0 trampoline (or musl's `sa_restorer`) that issues `NX_SYS_RT_SIGRETURN` to restore.  Switches Ctrl-C from "kill everyone" to "post SIGINT, deliver via handler".  Estimated ~200-300 lines kernel + ktest using mock-RX 0x03 injection.  v1 minimal Ctrl-C / Ctrl-D side channels (slice 7.6d.N.final.c-minimal, Session 71) cover the scripted smoke tests; .c-full lands when interactive UX needs handler dispatch.

3. **7.6d.N.16 — Drop in-handle generation encoding + POSIX-fd-to-slot alignment.**  Originally a 30-line paydown for slice 7.6d.N.11's `sys_fcntl` band-aid (replace `(generation << 8) | (idx + 1)` with `idx + 1`); slice 7.6d.N.15's investigation showed the proper fix needs a non-zero `NX_HANDLE_INVALID` sentinel (e.g. `0xFFFFFFFF`) so encoded value 0 can name fd 0 — ripples through every `NX_HANDLE_INVALID` comparison and the slice-5.3 stale-handle invariants (which would then assert via the in-table generation field rather than encoded-value mismatch).  Lands when a workload demands `exec 3< file` or similar literal-fd-3+ idioms.  ~30 lines + slice-5.3 test churn.

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

1. **[Session 128](logs/session-128-book-chapter-6.md)** (2026-05-10) — **Book chapter 6 — Virtual memory and the MMU**.  `book/06-virtual-memory-and-mmu.md` (1247 lines) walks `core/mmu/mmu.c` end-to-end.  *MMU bring-up:* the three-knob configuration (4 KiB granule, 39-bit VA via `T0SZ=25`, three-level walk); descriptor bit-layout decode (V/Tbl/AttrIdx/AP/SH/AF/PA/PXN/UXN); the three nonux helpers `device_block` / `normal_block` / `user_block`; `MAIR_VALUE` indirection (Device-nGnRnE @ idx 0, Normal WB-WA @ idx 1) with side note on *why* the indirection exists; bit-by-bit `TCR_VALUE` decode; the static L1+L2 tables and L1 wiring; worked walks of VA `0x4009_0000` → PA `0x40090000` (kernel-RAM Normal) and VA `0x0900_1000` → PA `0x09001000` (PL011 Device); the SCTLR enable sequence step-by-step (`dsb ish` → MAIR/TCR/TTBR0 → `isb` → `tlbi vmalle1` → `dsb ish` → `ic iallu` → `dsb ish` → `isb` → `SCTLR.{M,C,I}` → final `isb`) with side note on *why the kernel survives its own MMU enable* (identity-map invariant); the `CPACR_EL1.FPEN` tweak.  *User window:* slot-64 `user_block` upgrade in the kernel `l2_ram_table` giving window VA `0x4800_0000`, `drop_to_el0`'s `eret` as the demotion entry, AP+UXN bits enforcing EL0 confinement.  *Per-process address spaces:* `mmu_create_address_space`'s three allocations (L1, L2_ram, 8 MiB user backing); shared-kernel/private-user-window invariant as a 2-table ASCII diagram; `mmu_switch_address_space`'s TTBR0 + `tlbi vmalle1` sequence with ASIDs side note; `mmu_copy_user_backing`'s TTBR0-save-to-kernel-then-restore trick around the fork memcpy.  Closes the loop on chapter 5's `pmm_reserve_range(mmu_user_window_base(), …)` forward-pointer.  Two cross-link patches in chapter 5 turn its "next chapter on the MMU" pointers into real chapter-6 links.  `BOOK-OUTLINE.md` chapter-6 row `planned → shipped` with "drafted pre-Phase-9" annotation; `book/README.md` chapter-6 entry now a real link.  **Part III complete (chapters 5–6).**  Documentation-only; test counts unchanged.
2. **[Session 127](logs/session-127-book-chapter-5.md)** (2026-05-09) — **Book chapter 5 — Physical memory and the page allocator (PMM)**.  `book/05-physical-memory-and-pmm.md` (1123 lines) walks the path from "the kernel image ends at `__free_mem_start`, RAM ends at `RAM_END`" to "the buddy component bound to `memory.page_alloc`", in two halves.  PMM half: bitmap + self-accounting init (the bitmap pages mark themselves allocated); `bit_test` / `try_claim` / `release` with `__atomic_fetch_or` ACQUIRE / `__atomic_fetch_and` RELEASE; `pmm_alloc_page` first-fit-with-hint; `pmm_free_pages` constant-time + "set hint backwards on free" working-set rationale; `pmm_alloc_pages` two-pass scan-then-claim with rollback; `pmm_reserve_range` for the user-window aliasing case.  Side note on *why single-CPU still needs atomics* (interrupt-during-RMW), anchoring the design in chapter 4's preemption model.  Buddy half: three-reason motivation (power-of-two requests, coalescing, swappable strategy); `nx_mm_ops` interface contract (forward-points to chapter 10); 16-page pool geometry; intrusive `free_node` in-place free-list trick; **alloc = split until you fit** with worked example through one order-2 alloc; **free = merge while the buddy is free** with the `addr ^ block_size` XOR insight + a 3-alloc/3-free worked example that round-trips to the original order-4 block; lifecycle (`init`/`destroy` carve/return; `enable`/`disable` no-ops by design).  Annotation pass on the real boot-log lines + full ASCII layered diagram (RAM → PMM → kheap + mm_buddy → consumers).  `BOOK-OUTLINE.md` chapter-5 row `planned → shipped`; `book/README.md` chapter-5 entry now a real link.  **Part III started.**  Documentation-only; test counts unchanged.
3. **[Session 126](logs/session-126-book-chapter-4.md)** (2026-05-09) — **Book chapter 4 — The timer and ticks**.  `book/04-timer-and-ticks.md` (760 lines) walks "how the kernel measures time" → "how a tick becomes a context switch", reusing chapter 3's IRQ pipeline as the substrate.  Covers: why a kernel needs a heartbeat (preemption / timeouts / bookkeeping); the ARM Generic Timer + the EL1 physical timer specifically; `CNTFRQ_EL0` / `CNTP_TVAL_EL0` / `CNTP_CTL_EL0` with the 62.5 MHz ÷ 10 Hz = 6.25M reload arithmetic; `timer_init` line-by-line; why the timer is a PPI; `on_tick`'s three jobs (rearm first → avoid drift, bump `g_ticks`, call `sched_tick`); the policy `tick` op via `sched_rr_tick` (quantum decrement → `need_resched`); the IRQ-stub's `bl sched_check_resched` as the flag's consumer with `cpu_switch_to` named but deferred to the threads chapter; full ASCII diagram of the timer-driven preemption pipeline; `timer_ticks()` as monotonic uptime; `timer_pause`/`timer_resume` for recomposition with the nest-counter + saturate-at-zero pattern.  Two minimal patches to chapter 3 turn its "covered in a later chapter" + boot-order placeholder into real cross-links.  **Part II complete (chapters 3–4).**  `BOOK-OUTLINE.md` chapter-4 row `planned → shipped`; `book/README.md` chapter-4 entry now a real link.  Documentation-only; test counts unchanged.
4. **[Session 125](logs/session-125-book-chapter-3.md)** (2026-05-09) — **Book chapter 3 — Exceptions, the GIC, and IRQs**.  `book/03-exceptions-gic-and-irqs.md` (1211 lines) closes the gap chapter 2's RX diagram glossed over: the four exception kinds (sync/IRQ/FIQ/SError), the 16-entry vector table (4 contexts × 4 kinds × 128 bytes), `vectors_install` writing `VBAR_EL1`, the trap-frame layout + SAVE/RESTORE macros, the four `on_*` C handlers (with `on_sync` walking `ESR.EC` classification + the EL0-fault-as-process-exit / EL1-fault-as-panic split), GICv2 distributor + CPU-interface split + SGI/PPI/SPI numbering + ack/EOI handshake, the per-IRQ dispatch table (release/acquire on the function-pointer slot), DAIF masking + `daifset/daifclr #2` immediate, and the `boot_main` bring-up order — `vectors_install` → `gic_init` → per-IRQ register/enable → `irq_enable_local`.  Two minimal patches turn chapter 2's "later chapter on exceptions and the GIC" forward references into real cross-links.  `BOOK-OUTLINE.md` chapter-3 row `planned → shipped`; `book/README.md` chapter-3 entry now a real link.  Documentation-only; test counts unchanged.
5. **[Session 124](logs/session-124-parts-2-3-swap.md)** (2026-05-09) — **Chapter 2 polish + Part II/III swap**.  Three small chapter-2 follow-up commits (MMIO framing fix; IRQ properly introduced on first body use; unintroduced function names glossed in body — `uart_rd`/`_wr`, `nx_console_write`, `nx_console_read_nonblocking`, `console_read_ready_pred`, pollset register/unregister); chapter 2 grew 1108 → 1163 lines.  Outline restructured in `BOOK-OUTLINE.md` and `book/README.md`: Part II is now "Time and interrupts" (chapters 3 · 4); Part III is now "Memory" (chapters 5 · 6).  Reading-dependencies + Phase 9 dependency cross-refs updated for the new numbering.  Rationale: chapter 2 leaves the reader hot on the PL011 IRQ; chapter 3 (Exceptions/GIC) keeps the thread live where PMM would have been a topic shift.  Documentation-only; test counts unchanged.
(Sessions 80–123 archived to [HANDOFF-ARCHIVE.md](HANDOFF-ARCHIVE.md) per the "keep last 5" convention.)
Older entries: see [HANDOFF-ARCHIVE.md](HANDOFF-ARCHIVE.md) (Sessions 1–123).

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
