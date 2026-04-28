# nonux

> **Source code:** [github.com/ThinkerYzu/nonux](https://github.com/ThinkerYzu/nonux) — companion repo (MIT-licensed).  This repo (`nonux-meta`) holds the spec, design, implementation guide, handoff package, and per-session work logs.  Current status, phase progress, and test counts live in [HANDOFF.md](HANDOFF.md).

**Created:** 2026-04-17
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
