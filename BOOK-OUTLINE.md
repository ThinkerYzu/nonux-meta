# Book Outline

The full chapter plan for the [nonux book](../../sources/nonux/book/README.md).
This document is the canonical reference for the book's scope and
reading order. The book's own `README.md` (in the source repo)
shows readers what's shipped; **this** file shows what's planned.

Style and authoring rules: [BOOK-STYLE-GUIDE.md](BOOK-STYLE-GUIDE.md).

---

## Status legend

- **shipped** — chapter is committed in `sources/nonux/book/`
- **draft**   — drafting in progress, not yet committed
- **planned** — outline only, no draft yet

---

## Chapter list

### Part I — Coming up

| # | Status | Title | Reader sees |
|---|---|-------|-------------|
| 1 | shipped | Boot and linker | `core/boot/` (start.S, linker.ld, boot.c). Bootloader, ARM64 boot, exception levels, linker script, ELF vs raw binary, full boot timeline. |
| 2 | planned | The console: kprintf and the UART | `core/lib/lib.h`, `components/uart_pl011/`, the PL011 RX ISR. How a `kprintf` call lands as a byte on the screen. |

### Part II — Memory

| # | Status | Title | Reader sees |
|---|---|-------|-------------|
| 3 | planned | Physical memory and the page allocator (PMM) | `core/pmm/`, the bitmap, `__free_mem_start` → `RAM_END`, page-level allocation, `components/mm_buddy/`. |
| 4 | planned | Virtual memory and the MMU | `core/mmu/`, page table walks (L0–L3), identity vs user mapping, MAIR / TCR, kernel/user split, TTBR0 / TTBR1. *(See "Phase 9 dependency" below.)* |

### Part III — Time and interrupts

| # | Status | Title | Reader sees |
|---|---|-------|-------------|
| 5 | planned | Exceptions, the GIC, and IRQs | `core/cpu/exception.S`, the vector table, four exception classes, `core/irq/` (GICv2 distributor + CPU interface), masking. |
| 6 | planned | The timer and ticks | `core/timer/`, ARM Generic Timer at 10 Hz, the tick handler, the dispatch chain into the scheduler. |

### Part IV — Tasks

| # | Status | Title | Reader sees |
|---|---|-------|-------------|
| 7 | planned | Kernel threads and context switching | `struct nx_task`, kstack layout, `nx_task_switch_to`, kthread spawn, the saved-register frame, what TPIDR_EL1 holds. |
| 8 | planned | The scheduler | `core/sched/`, runqueue, idle task, preemption, yield, plus the two real schedulers: `components/sched_rr/` and `components/sched_priority/`. |

### Part V — Framework basics

| # | Status | Title | Reader sees |
|---|---|-------|-------------|
| 9 | planned | Slots, components, and the registry | `framework/registry.c`, `framework/component.c`, `framework/bootstrap.c`, `kernel.json`, the `NX_COMPONENT_REGISTER` macro and the `nx_components` linker section. |
| 10 | planned | Interface Definition Language (IDL) | `interfaces/idl/*.json`, `tools/gen-iface.py`, the generated `*_call.h` / `*_dispatch.h` / `*_msg.h` shims. |

### Part VI — IPC and storage

| # | Status | Title | Reader sees |
|---|---|-------|-------------|
| 11 | planned | Channels and IPC | `framework/ipc.c`, the dispatcher kthread, async vs sync, `framework/slot_call.c`, `framework/channel.c`, capabilities, the `slot_ref_retain` / `release` ownership model. |
| 12 | planned | Filesystems (VFS, ramfs, procfs) | `components/vfs_simple/`, `components/ramfs/`, `components/procfs/`, `framework/fs_call.c`, walking a `sys_open(...)` from EL0 down to `ramfs_op_lookup`. |

### Part VII — Framework, advanced

| # | Status | Title | Reader sees |
|---|---|-------|-------------|
| 13 | planned | Hooks: intercepting and overriding | `framework/hook.c`, hook points, the chain, abort/continue, the syscall-enter / -exit hooks added in Session 104. |
| 14 | planned | Recomposition and runtime config | `framework/recompose.c`, pause/drain/resume, the live scheduler swap (Session 102), `framework/config.c`, `NX_SYS_CONFIG_*` syscalls. |

### Part VIII — Userspace

| # | Status | Title | Reader sees |
|---|---|-------|-------------|
| 15 | planned | Processes and the user/kernel boundary | `framework/process.c`, `struct nx_process`, the EL0 transition (`drop_to_el0`), per-process address space, the user window. *(See "Phase 9 dependency" below.)* |
| 16 | planned | System calls: EL0 ↔ EL1 | the `svc` instruction, the sync-exception arm of the vector table, `framework/syscall.c` dispatch, the call/return register dance. |

### Part IX — Userspace runtime and build

| # | Status | Title | Reader sees |
|---|---|-------|-------------|
| 17 | planned | POSIX shim, libnxlibc, and busybox | `components/posix_shim/`, `lib/libnxlibc/`, `third_party/musl/` (vendored), `third_party/busybox/`, the `/init` boot path, the syscall-translation layer. |
| 18 | planned | Build, test, AI operability | `Makefile`, `tools/gen-config.py`, `tools/validate-config.py`, `tools/verify-registry.py`, the four test tiers (tools / host / kernel / interactive), the manifest-driven build promise. |

### Appendices

| | Status | Title | Why it exists |
|---|---|-------|--------------|
| A | planned | ARM64 cheat sheet | Instruction quick-ref (`bl`, `eret`, `wfi`/`wfe`, `svc`, `dsb`/`isb`), register naming, exception-level register tables — pulled together for cross-chapter reference. |
| B | planned | Glossary | Cumulative list of every term defined in any chapter, alphabetical. |
| C | planned | Source map | One-page table of every directory under `sources/nonux/`: what's there, which chapter covers it. |
| D | planned | Where to read more | Curated external links: ARM ARM relevant chapters, Linux `Documentation/arm64/`, xv6, seL4, OSDev wiki. |

---

## Reading dependencies

Every chapter assumes the chapters before it. The structure was
designed so this is true with no forward references. A few
dependencies worth calling out explicitly:

- **Chapters 11–12 (IPC, FS) depend on chapters 9–10 (slots, IDL).**
  This is why framework basics come *before* IPC. nonux's IPC *is*
  slot calls, and the FS is structured as components — both are
  hard to explain without slot/component already defined.
- **Chapters 13–14 (hooks, recomposition) depend on chapters 11–12.**
  This is why "framework, advanced" comes *after* IPC. Hooks attach
  to slot-call boundaries; recomposition acts on running
  compositions. Both want concrete examples to lean on.
- **Chapter 15 (Processes) depends on chapter 4 (MMU) and chapter 7
  (Threads).** A process is "an address space + a task running
  inside it"; both pieces need to be in hand first.
- **Chapter 16 (Syscalls) depends on chapter 5 (Exceptions/GIC) and
  chapter 15 (Processes).** Syscalls *are* an exception class on
  ARM64, and they exist to serve userspace processes.
- **Chapter 17 (POSIX shim) depends on chapter 16 (Syscalls).** The
  shim is essentially a syscall translator with userspace plumbing
  around it.

---

## Phase 9 dependency

Chapter 4 (Virtual memory and the MMU) and parts of chapter 15
(Processes and the user/kernel boundary) describe behavior that is
currently scheduled to change in [Phase 9](IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework)
(per-process MM rework — L3 4 KiB pages, VMAs, demand paging, COW
fork). Two options for those chapters:

1. **Draft after Phase 9 lands.** Cleaner — the chapter describes
   the post-Phase-9 reality directly, no rework.
2. **Draft pre-Phase-9, refine after.** Faster to ship the early
   chapters, but requires a follow-up revision pass.

The decision is per chapter. Whichever is chosen goes in that
chapter's session log.

---

## Length and pace

Per the [style guide](BOOK-STYLE-GUIDE.md): aim for 600–1500 lines
per chapter. Chapter 1 came in at 968 lines, which felt about
right. If a chapter routinely overshoots, split it; if it
undershoots, look hard at whether two chapters should merge.

No fixed schedule. Chapters land when they're written.

---

## Per-chapter workflow

For every new chapter:

1. Outline first (intro / Terms / main content / extras / further
   reading), per the [style guide](BOOK-STYLE-GUIDE.md).
2. Identify the standard terms the chapter introduces.
3. Write the chapter.
4. Update [`book/README.md`](../../sources/nonux/book/README.md) —
   change the chapter's status from `planned` to a real link.
5. Update this outline — change the chapter's status to `shipped`.
6. Write a session log under [`logs/`](logs/SESSION-LOG-TEMPLATE.md)
   per the project-wide workflow described in
   [`README.md`](README.md).

When the chapter list changes (renaming, reordering, adding,
removing): update both this file and `book/README.md`, plus any
cross-references from other chapters.

---

## See also

- [BOOK-STYLE-GUIDE.md](BOOK-STYLE-GUIDE.md) — voice, terminology,
  formatting rules
- [Book README](../../sources/nonux/book/README.md) — what's
  actually shipped
- [HANDOFF.md](HANDOFF.md) — current project status and session
  history
