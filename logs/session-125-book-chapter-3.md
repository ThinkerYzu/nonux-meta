# Session 125: Book chapter 3 — Exceptions, the GIC, and IRQs

**Date:** 2026-05-09
**Phase:** Phases 1–9b complete; book chapters 1–2 shipped; chapter 3 drafted this session; Phase 9 not yet started
**Branch:** master

---

## Goals

- Draft the next book chapter, picking up where chapter 2 left
  off (PL011 RX IRQ delivery).
- Per the post-session-124 outline, that chapter is **chapter 3:
  Exceptions, the GIC, and IRQs** — the first chapter of Part II
  (Time and interrupts).

## What Was Done

### `sources/nonux/book/03-exceptions-gic-and-irqs.md` (new, 1211 lines)

Walks the full path from "the PL011 raises a wire" to "the right
C function runs", filling the gap chapter 2's RX diagram glossed
over ("GIC delivers IRQ 33 to the CPU → CPU jumps to EL1 IRQ
vector → `irq_dispatch()` → `nx_console_rx_isr`").

Section structure (per
[BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md#chapter-structure)):

1. Intro (3 paragraphs, opens by quoting chapter 2's
   hand-waved diagram and naming the gap).
2. Files (`core/cpu/vectors.S`, `core/cpu/exception.{c,h}`,
   `core/irq/irq.{c,h}`, `core/irq/gic.c`, `core/boot/boot.c`).
3. **Terms** — 21 entries covering: exception, sync vs async,
   IRQ / FIQ / SError, vector / vector table, `VBAR_EL1`, DAIF,
   trap frame, `ELR_EL1` / `SPSR_EL1` / `ESR_EL1` / `FAR_EL1`,
   `eret` (already met ch.1), GIC, distributor / CPU interface,
   SPI / PPI / SGI, ack / EOI.
4. Quick recap of where chapter 2 left off.
5. **Exceptions: the four kinds** — sync vs IRQ vs FIQ vs SError,
   plus the four `on_*` C functions.
6. **The vector table** — 16 entries × 128 bytes, four contexts ×
   four kinds, the `.balign 0x80` / `.balign 2048` mechanics, why
   each entry is just one `b` to an out-of-line stub,
   `vectors_install` writing `VBAR_EL1`.
7. **What gets saved on an exception** — what the CPU does for
   us, then `SAVE_TRAPFRAME` walked instruction-by-instruction
   with an ASCII stack diagram, `RESTORE_TRAPFRAME`'s inverse,
   and the C-side `struct trap_frame`.
8. **The four C handlers** — `_sync_stub`/`_irq_stub` assembly,
   `mov x0, sp; bl on_*` pattern, `on_irq` (trivial dispatch),
   `on_sync` (busy — reads ESR.EC, splits syscalls / EL0 faults
   / kernel bugs), `on_fiq` / `on_serror` (panic stubs).
9. **Reading `ESR_EL1`** — the EC field, the table of EC values
   we care about (0x15 SVC64, 0x20/0x21 inst-abort, 0x24/0x25
   data-abort, 0x00 unknown), why EL0 faults become process
   exits and EL1 faults panic.
10. **The GIC: from a wire to a CPU** — distributor + CPU
    interface split, why the split exists (multi-core), the
    device-to-CPU diagram, SGI/PPI/SPI numbering with the
    PL011=SPI#1=IRQ 33 example.
11. **Programming the GIC** — `gic_init`'s three writes
    (distributor enable, PMR, CPU-interface enable), `gic_enable`
    (priority + ISENABLER), `gic_disable` (ICENABLER), with a
    note on the set/clear-pair pattern.
12. **Acknowledging and ending** — `IAR` / `EOIR`, the bottom-10
    -bits mask, why mismatched ack/EOI hangs the system.
13. **The dispatcher and the per-IRQ table** — `g_table[128]`,
    the release/acquire ordering on the function-pointer slot,
    `irq_dispatch`'s five steps (ack → spurious filter →
    handler-lookup → call/disable → EOI), the unhandled-IRQ
    self-defense (mask at GIC).
14. **Masking IRQs with DAIF** — what each letter masks, the
    `daifset/daifclr #2` immediate, the `"memory"` clobber, when
    the CPU masks IRQs automatically (every exception entry),
    when the kernel masks them by hand (boot, IRQ-shared
    critical sections).
15. **The boot order** — the `boot_main` snippet showing
    `vectors_install` → `gic_init` → per-IRQ `irq_register +
    gic_enable` → `irq_enable_local`, and why the order
    matters.
16. **End-to-end: IRQ 33 once more** — the full path with no
    hand-waving, redoing chapter 2's diagram now that every
    arrow is concrete.
17. **A few extra things to know** — six side notes (EL1t mode
    is dead code, sync-from-EL1 is a kernel bug, no
    "vector-table-size register", the unhandled-IRQ
    self-defense, GICv2 vs GICv3 portability, spurious
    1022/1023, why `mov x0, sp` is more than convenience).
18. **Where to read more** — relative source links + back-refs
    to chapter 1 (privilege levels) and chapter 2 (RX IRQ
    setup); ARM ARM (DDI0487), GICv2 spec (IHI0048), QEMU
    `virt` source.

Length: 1211 lines after softening two forward-references to
chapter 4 (per style-guide rule "don't reference future chapters
as if they exist"). Comfortably within the 600–1500 band.

### Chapter 2 cross-link patches (minimal)

Two small edits on `book/02-console-and-uart.md` to replace its
"a later chapter" placeholders with real links to chapter 3
(both in the body text under "Setting up the IRQ" and in the
"Where to read more" entry for the GICv2 spec). Per
[BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md#revising) — minimal
patches once a chapter is committed; just turn the dangling
"later chapter" mentions into real cross-references.

### Index / outline updates

- **`book/README.md`** — chapter 3 entry under Part II changed
  from plain text to `[Exceptions, the GIC, and IRQs](03-...md)`.
- **`BOOK-OUTLINE.md`** — chapter 3's status changed `planned →
  shipped`. The "Reader sees" column expanded with the actual
  list of files and topics covered (vectors.S + trap frame +
  on_sync EC classification + GICv2 + DAIF + boot order).

## Key Findings

- **Chapter 2's "a later chapter on exceptions and the GIC"
  forward reference made chapter 3 unusually constrained on
  scope.** Chapter 2 explicitly told the reader that chapter 3
  would explain `irq_register` and `gic_enable` "under the
  covers". That's a contract — chapter 3 had to fully cover both,
  not just one or the other. End-to-end IRQ 33 path got its own
  section ("End-to-end: IRQ 33 once more") to honor that.
- **The hardest decision was where to put `on_sync`.** Chapter 3
  is "Exceptions, the GIC, and IRQs", and `on_sync` covers
  syscalls and faults — both of which are deferred to later
  chapters (syscalls to ch.16, faults won't have a dedicated
  chapter). Decided to walk `on_sync` thoroughly for completeness
  (the function is right there in the source), but only deeply
  enough to cover what makes the exception class machinery work
  — the EC field, the split between EL0-origin and EL1-origin,
  why each is handled differently. The actual syscall dispatch
  and fault-to-signal delivery get short forward pointers.
- **The "unintroduced function names" rule from session 124
  applies here too.** Every function the reader sees in a code
  block is introduced in surrounding prose: `vectors_install`,
  `SAVE_TRAPFRAME`/`RESTORE_TRAPFRAME` (macros), all four
  `on_*` C handlers, `gic_init`/`gic_enable`/`gic_disable`,
  `gic_ack`/`gic_eoi`, `irq_register`/`irq_unregister`/
  `irq_dispatch`, `irq_enable_local`/`irq_disable_local`. No
  function name appears in code without a prose introduction
  earlier in the same chapter.

## Decisions Made

- **Cover GICv2 only; GICv3 only as a forward-portability note.**
  The QEMU `virt` machine ships GICv2; that's what the source
  drives. A side-note explains GICv3 differs in *register
  surface* but not in *conceptual model* — distributor + CPU
  interface, ack/EOI, SGI/PPI/SPI numbering all remain.
- **Walk every register the source touches.** ESR_EL1 / FAR_EL1
  / ELR_EL1 / SPSR_EL1 / VBAR_EL1 / DAIF / GICD_CTLR /
  GICD_IPRIORITYR / GICD_ISENABLER / GICD_ICENABLER /
  GICC_CTLR / GICC_PMR / GICC_IAR / GICC_EOIR. The chapter is
  long-ish (1211 lines) partly because of this, but the
  alternative — gloss over registers and have the reader
  cross-reference the ARM ARM mid-chapter — would defeat the
  tutorial-style goal.
- **No framing analogy at the top.** Style guide allows one;
  none felt natural and non-strained for the topic. The
  recap-of-chapter-2-diagram opening serves the same "motivate
  the chapter" role without inviting a flawed metaphor (cf.
  the chapter-1 freezer incident).
- **No content changes to chapter 1.** Chapter 1's exception-
  level material is the only forward dependency chapter 3 reads
  back into. Chapter 1 is unchanged.

## Status at End of Session

- `book/03-exceptions-gic-and-irqs.md` drafted, 1211 lines,
  staged in `sources/nonux`.
- `book/README.md` chapter-3 link wired up, staged.
- `book/02-console-and-uart.md` two minimal cross-link patches,
  staged.
- `BOOK-OUTLINE.md` chapter-3 row updated `planned → shipped`,
  staged in `proj_docs/nonux`.
- `HANDOFF.md` Current Status, Latest session log, Next
  Actions, Session Logs list — to be updated in this commit.
- No code changes. Tests not re-run; test counts unchanged from
  session 124.

## Next Steps

- Either start Phase 9 (per-process MM rework — L3 4 KiB pages,
  VMAs, demand paging, COW fork; see
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework))
  or continue the book with chapter 4 (the timer and ticks),
  the second half of Part II.

---

**Files Changed:**

`sources/nonux` (this commit):
- `book/03-exceptions-gic-and-irqs.md` — new, 1211 lines.
- `book/README.md` — chapter-3 entry now a real link.
- `book/02-console-and-uart.md` — two minimal cross-link patches
  (forward references to chapter 3 made concrete).

`proj_docs/nonux` (this commit):
- `BOOK-OUTLINE.md` — chapter 3 status `planned → shipped`,
  "Reader sees" column expanded.
- `HANDOFF.md` — Current Status, Latest session log line, Next
  Actions, Session Logs list updated.
- `logs/session-125-book-chapter-3.md` — new (this file).
