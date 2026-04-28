# nonux

**Status:** Phases 3 + 4 + 5 + 6 complete. **Phase 7 in progress (slices 7.1 + 7.2 + 7.3 + 7.4 + 7.5 + 7.6a + 7.6b + 7.6c.0 + 7.6c.1 + 7.6c.2 + 7.6c.3 + 7.6c.4 + 7.6d.1 + 7.6d.2 + 7.6d.3a + 7.6d.3b + 7.6d.3c + 7.6d.N.0 + 7.6d.N.1 + 7.6d.N.2 + 7.6d.N.3 + 7.6d.N.4 + 7.6d.N.5 + 7.6d.N.6a + 7.6d.N.6b + 7.6d.N.7 + 7.6d.N.8 + 7.6d.N.9 + 7.6d.N.10 + 7.6d.N.11 + 7.6d.N.12 + 7.6d.N.13 + 7.6d.N.14 + 7.6d.N.15 + 7.6d.N.final.a + 7.6d.N.final.b + 7.6d.N.final.c (minimal) + 7.6d.N.final.d + 7.6d.N.final.e + 7.7a done + 7.6d.N.16 deferred + 7.6d.N.final.c-full deferred + opportunistic ENOSYS hygiene; 7.7b pending)**. **`busybox sh -c "ls /"` runs end-to-end** — ash forks + execve's `/bin/ls`, busybox dispatches to the ls applet, ls calls `opendir("/")` + `getdents64` + sorts + prints all 9 ramfs entries on stdout, exits 0.  Slice 7.6d.N.6a (this session) escalates toward `echo hello | cat` (first pipe): mapped `__NR_pipe2 (59) → NX_SYS_PIPE`, `__NR_clone (220) → NX_SYS_FORK`, added `NX_SYS_DUP3 (24)` + `NX_SYS_READV (25)`, made `sys_read` h=0 resolve slot-0 directly so `dup3(_, 0, 0)` round-trips through stdin reads — but the pipe ktest stays held out of `KTEST_C` until 7.6d.N.6b reserves slots 0/1/2 in `nx_handle_alloc` (pipe currently allocates at slots 0/1, colliding with POSIX STDOUT/STDERR).  **Slice 7.6d.2 morally closed (Session 54).** Build-script-only change that closes 7.6d.2's address-space-mismatch precondition: busybox 1.36.1's kbuild default puts text at VA `0x400000` (`-static -no-pie` aarch64 default) but our user window sits at `0x48000000`; the ELF loader (`framework/elf.c`) honours `PT_LOAD.p_vaddr` verbatim so loading busybox-as-built would write into MMIO space and fault.  Fix: `tools/build-busybox.sh` gains `-Wl,-Ttext-segment=0x48000000` in `EXTRA_LDFLAGS` (one-line change with a 14-line comment cross-referencing the other two places `0x48000000` is hardcoded — `init_prog.ld` and `core/mmu/mmu.c`'s `USER_WINDOW_INDEX = 64` arithmetic).  After re-link, `readelf -h` shows entry `0x480005c0`; `readelf -l` shows segment 1 at `VirtAddr 0x48000000`, segment 2 at `0x481d9770`.  Both segments now sit inside the user-window VA range (`0x48000000` – `0x48200000`).  Binary is the same 2,292,464 bytes — only segment headers + a few absolute addresses in text differ.  busybox still doesn't *fit* the 2 MiB window: segment 2 ends at `0x481e8930` (~1.91 MB into window), leaving only ~84 KiB for stack+heap — that's 7.6d.2b's job (grow window 2 MiB → 8 MiB).  `make test` runs 407/407 (51 python + 275 host + 81 kernel) green — unchanged.  Prior arc: slice 7.6d.1 vendored busybox 1.36.1 + committed minimal `nonux_defconfig` (1223 lines, 659 enabled symbols, with networking/modutils/mailutils/printutils/runit/sysklogd/loginutils/selinux + PAM/FEATURE_USE_INITTAB/*RPC*/MOUNT_NFS force-disabled) + wrote `tools/build-busybox.sh` + wired Makefile `busybox / busybox-clean` targets.  Slice 7.6c.3 closed musl integration with three kernel pieces — magic-fd-handle stdio in `sys_write` / `sys_read`, `NX_SYS_BRK = 17` with per-process `brk_addr`, `NX_SYS_WRITEV = 18` with iovec walker — bringing musl-linked printf + AUXV-consumption-via-sys_exec live end-to-end. Prior working arc: EL0 userspace creates a channel via SVC, sends bytes on one endpoint, receives them on the peer, and forwards the payload back to UART (`[el0-chan-ok]`) — full path from first instruction through MMU, syscall dispatcher, handle framework, rights attenuation, and IPC-style message passing is live. Kernel boots under QEMU with the MMU on at EL1 — 3-level 4 KiB-granule identity map (MMIO 0–1 GiB as Device-nGnRnE, RAM 1–2 GiB as Normal WB-WA Inner-Shareable), D-cache + I-cache on, `-mstrict-align` dropped. Brings up `uart_pl011` + `sched_rr` + `mm_buddy` to `NX_LC_ACTIVE`; mm_buddy is the first real `memory.page_alloc` component — classic power-of-two buddy on a 64 KiB per-instance pool, consumed via `interfaces/mm.h`. Boot context is promoted into an idle task; preemption drives through timer tick → `sched_tick` → IRQ-return reschedule shim → `cpu_switch_to`. A framework dispatcher kthread spawned at bootstrap drains a Vyukov-style MPSC inbox that async IPC senders (and `nx_ipc_enqueue_from_irq`) push to. `NX_HOOK_CONTEXT_SWITCH` + `NX_HOOK_IPC_RECV` + `NX_HOOK_SLOT_SWAPPED` fire on the right edges. Pause protocol rolls back cleanly on `pause_hook`/`ops->pause` failure and enforces a 1 ms wall-clock deadline on `pause_hook` via the `core/cpu/monotonic.h` deadline primitive. `timer_pause`/`timer_resume` land as the recomposition timer-quiescence primitive for Phase 8.
**Created:** 2026-04-17
**Last Updated:** 2026-04-28 (Session 73 — slice 7.7a: trivial interactive smoke tests.  Two new files in `test/interactive/` — `ls_root.{script,expected}` and `echo_cat.{script,expected}` — promote slice 7.6d.N.5's `ls /` and slice 7.6d.N.6b's `echo MARKER | cat` workloads from ktest-only into the live UART harness shipped in slice 7.6d.N.final.d.  No kernel changes.  First-iteration `ls_root.expected` had `/init`/`/bin/busybox` as expected substrings; updated to `argv_child`/`musl_prog`/`banner` after tracing through actual captured output (root cause: `sys_getdents64` strips one leading `/` per `framework/syscall.c:1925-1926`, then busybox ls treats embedded `/` as a path separator; recorded as cleanup target for slice 7.7b's hierarchical-path rework).  Slice 7.7's `mkdir`+`ps` exit criteria sub-sliced into 7.7b (next session) — both need new kernel surface (vfs/ramfs `mkdir` + `NX_SYS_MKDIRAT` + hierarchical paths; procfs OR `NX_SYS_GETPROCS` + busybox patch).  `make test-interactive` → **5/5 pass** (was 3/3); `make test` unchanged at **432/432 pass (51 python + 277 host + 104 kernel)**.  See [Session 73](logs/session-73-7.7a-trivial-interactive-scripts.md).)
**Goal:** A composable, Lego-like microkernel for experimenting with OS design trade-offs, under a permissive license.

---

## Project Documentation

### Core Documentation

- **README.md** (This document) - Project overview and getting started
- **[SPEC.md](SPEC.md)** — Problem statement, requirements, constraints, and success criteria
- **[DESIGN.md](DESIGN.md)** — Architecture, design decisions, and technical approach
- **[IMPLEMENTATION-GUIDE.md](IMPLEMENTATION-GUIDE.md)** — Detailed implementation guide with code examples
- **[HANDOFF.md](HANDOFF.md)** — Current status, next actions, session logs (the handoff package)

### Testing Documentation

- **[TESTING-GUIDE.md](TESTING-GUIDE.md)** — Testing procedures, expected output, troubleshooting

**Quick Links:**
- [Overview](#overview) | [Design & Architecture](DESIGN.md) | [Current Status](HANDOFF.md#current-status) | [Testing](TESTING-GUIDE.md)

---

## Overview

**nonux** ("No Linux" — a joke, and a statement) is a microkernel written in C/C++ from scratch, designed to be the Lego set of OS kernels. Every subsystem — scheduler, memory manager, filesystem, IPC — is a swappable component with a clean interface. You can snap different implementations together, benchmark them, and iterate on designs without fighting a monolithic codebase.

Unlike traditional microkernels, nonux doesn't force all services into userspace. Services can run in privileged mode for performance while still maintaining strict architectural boundaries. The isolation comes from well-defined interfaces and a comprehensive hooking/interception framework, not just hardware rings.

The project targets ARM64 on QEMU first, with POSIX compatibility provided as a shim layer so standard Unix tools (busybox) work out of the box. Everything is MIT-licensed.

## Recent Updates

See [HANDOFF.md](HANDOFF.md) for the full changelog, session history, and next actions.

---

## Core Concepts

### Components and Slots

The kernel is built from **components** — self-contained modules that implement a typed interface (scheduler, memory manager, VFS, etc.). Each component lives in its own directory with source code, a JSON manifest, and documentation. Components plug into **slots** — the kernel config declares which implementation fills each slot. Swap an implementation by changing one line in `kernel.json`.

### Handles

Userspace interacts with kernel objects through **typed handles** with per-handle permissions. There's no global syscall table — each process gets only the handles (and rights) it was given. A sandboxed process might have read-only file handles; a privileged service gets full control handles. The POSIX layer maps traditional calls onto handles underneath.

### Async-First IPC

Components communicate through async messages by default. A composition parameter can make any connection synchronous — and this can be changed at runtime without restarting anything. Components don't know or care which mode they're in.

### Lifecycle and Hot-Swap

Every component has a clean lifecycle: `init` → `enable` → `disable` → `destroy`. Resources are fully released at each boundary. Components can be swapped at runtime: drain messages, disable old, enable new. A mutability level (hot/warm/frozen) controls what can be changed live.

### Component Graph Registry

The framework maintains an explicit registry of the running composition — every slot, every slot reference, every connection. It's the single source of truth for "what is the kernel built out of right now?" Instrumentation, hot-swap, test harnesses, and AI reviewers all consume the same graph. A bounded append-only change log ("registrator") records every mutation, so the composition at any past moment is recoverable. Slot references crossing IPC travel as typed capabilities with explicit `borrow`/`transfer` ownership — stashing a received slot pointer past the handler requires an explicit `slot_ref_retain` call that registers the new edge.

### AI-Operable and AI-Verified

JSON manifests describe every component's interface, config options, dependencies, and resource ownership. A declarative `kernel.json` drives the build. An AI agent can read the manifests, generate a valid config, build, and boot — no tribal knowledge required. Going further: the rules that govern composition are **machine-checkable**, and `make verify-registry` enforces them on every build like `-Werror`. Components are written *and* reviewed by AI against the same deterministic rubric, so compliance is not a matter of human discipline.

---

## Implementation Phases

| Phase | Goal | Key Deliverable |
|---|---|---|
| 1 | Boot to serial | ARM64 boots in QEMU, prints to UART |
| 2 | Test harness & memory | Memory tracking layer, PMM, GIC, timer ticks |
| 3 | Component framework | Lifecycle, IPC, hooks, config tooling |
| 4 | Scheduler | Preemptive multitasking, swappable schedulers |
| 5 | Virtual memory & handles | Per-process address spaces, handle-based syscalls |
| 6 | VFS & ramfs | File operations on in-memory filesystem |
| 7 | Process model & POSIX | fork/exec, pipes, busybox shell |
| 8 | Runtime recomposition | Live hot-swap, runtime config changes |
| 9 | Test & benchmarks | Full test suite, IPC/context-switch benchmarks |
| 10 | Docs & AI operability | AI agent can build a kernel from manifests alone |

See [IMPLEMENTATION-GUIDE.md](IMPLEMENTATION-GUIDE.md) for detailed steps per phase.

---

## Development Repository

- **Public repo:** https://github.com/ThinkerYzu/nonux (MIT)
- **Local working directory:** `sources/nonux/`
- **Git branch:** master

Clone the public repo with:

```
git clone git@github.com:ThinkerYzu/nonux.git       # ssh
git clone https://github.com/ThinkerYzu/nonux.git   # https
```

---

## Related Projects

- [xv6](https://github.com/mit-pdos/xv6-riscv) — MIT educational Unix kernel (inspiration for simplicity)
- [seL4](https://sel4.systems/) — Formally verified microkernel (inspiration for IPC design)
- [Zircon](https://fuchsia.dev/fuchsia-src/concepts/kernel) — Fuchsia's kernel (inspiration for capability-based objects)

---

## Documentation Maintenance

### Documentation as a Web

**Core Principle:** This project directory is maintained as a **web of interconnected documents**, not isolated files.

- Documents are connected through **hyperlinks** (both markdown and HTML)
- Every document includes **navigation sections** at the top
- Cross-references point to **specific sections** using anchor links (#section-name)

### Agent Responsibilities

The agent (Claude) must actively maintain both **content and connections** in this documentation web:

**Content updates:**
- **Progress tracking**: Update status and milestone achievements
- **Implementation details**: Document design decisions, code structure, and technical approaches
- **Technical findings**: Record measurements, results, and key learnings
- **Architecture evolution**: Update design documents as the design evolves
- **Log maintenance**: Keep HANDOFF.md current with development activities

**Link maintenance:**
- **Add cross-references**: When creating new content, link to related existing content
- **Update navigation**: Add new documents to navigation bars on all pages
- **Verify links**: Ensure links remain valid as documents evolve
- **HTML generation**: Run `./scripts/convert_kb.sh` after updating markdown files

### Handoff as Complete Package

**Philosophy:** HANDOFF.md should serve as a **complete handoff package** that enables anyone to pick up the task and push forward without asking questions.

**Handoff quality test:** "Could a new team member read this and implement the next phase without asking clarifying questions?"

---

**Last Updated:** 2026-04-28 (Session 73 — slice 7.7a: trivial interactive smoke tests.  Two new files in `test/interactive/` (`ls_root.{script,expected}` and `echo_cat.{script,expected}`) push `make test-interactive` 3/3 → 5/5.  No kernel changes.  Slice 7.7's `mkdir`+`ps` exit criteria sub-sliced into 7.7b — both need new kernel surface.  `make test` unchanged at 432/432.  See [Session 73](logs/session-73-7.7a-trivial-interactive-scripts.md).)

**Previous:** 2026-04-28 (Session 72 — slice 7.6d.N.final.e: visible interactive busybox prompt.  `make run-busybox` now shows `# ` before the user types.  One-line vendored-busybox patch in `libbb/lineedit.c:put_prompt_custom` (`fflush(stdout);` after `fputs_stdout(prompt)`) — root cause was musl's line-buffered stdout holding ash's no-newline prompt until the next newline retroactively flushed it.  Side fix: `test/interactive/run.sh` switched to per-byte trickle (50 ms gap) — QEMU's `-serial stdio` does not backpressure on PL011 RX FIFO overflow.  +1 regression script; `make test-interactive` 3/3 stable; `make test` unchanged at 432/432.  See [Session 72](logs/session-72-N.final.e-visible-prompt.md).)

**Previous:** 2026-04-28 (Session 71 — slices 7.6d.N.final.b + .c (minimal) + .d.  `make run-busybox` boots an interactive busybox sh over QEMU UART.  Sub-slice b ships `pack-initramfs.py --busybox-init`, an EL0 init stub that `execve`s `/init` with `argv = {"sh", NULL}`, a build-flag-gated `boot.c` init runner (`-DNX_INIT_BUSYBOX`), and the `kernel-busybox.bin` build target.  Sub-slice c (minimal): Ctrl-C posts SIGTERM to all ACTIVE non-kernel processes via the existing polled path; Ctrl-D arms a one-shot EOF flag — full signal-handler trampoline deferred to `7.6d.N.final.c-full`.  Sub-slice d ships `tools/qemu-stdin-feed.sh` + `make test-interactive` (2 canned scripts).  +2 ktests; `make test` → 432/432 pass.  See [Session 71](logs/session-71-N.final.b-c-d.md).)

**Previous:** 2026-04-28 (Session 69 — four slices in one session: 7.6d.N.12 (sigaction/sigprocmask stubs) + 7.6d.N.13 (tolerable-syscall stubs sweep: getuid/euid/gid/egid + setuid/setgid + getpid/getppid + uname + set_tid_address) + 7.6d.N.14 (first 3-stage pipe `echo hello | tr a-z A-Z | wc -c` — surfaced + closed slot-position-preserving CHANNEL fork inheritance gap deferred since slice 7.6a) + 7.6d.N.15 (cross-process FILE-fd-through-fork via subshell `(cat) < /banner` — extends slot-position-preserving inheritance to FILE handles).  +4 ktests; `make test` → 425/425 pass.  N.16 deferred — proper POSIX-fd-to-slot alignment is bigger than the originally-scoped 30-line paydown, and skippable for slice 7.6d.N.final.)
