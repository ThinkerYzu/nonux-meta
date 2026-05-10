# Session 123: Book chapter 2 — the console: kprintf and the UART

**Date:** 2026-05-09
**Phase:** Phases 1–9b complete; post-9b fixes through Session 119; book chapters 1–2 shipped; Phase 9 not yet started
**Branch:** master

---

## Goals

- Draft chapter 2 of the [nonux book](../../sources/nonux/book/README.md)
  per the [outline](../BOOK-OUTLINE.md): walk a `kprintf` call from
  C source down to a byte on the QEMU terminal, then walk the RX
  side from a keystroke up to `nx_console_read`.
- Update `book/README.md` (link the chapter) and
  `proj_docs/nonux/BOOK-OUTLINE.md` (mark `shipped`).

## What Was Done

### `sources/nonux/book/02-console-and-uart.md` written

1108 lines (within the 600–1500 band per
[BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md#length-and-pace);
chapter 1 is 968). Structure follows the chapter skeleton from
[BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md):

- **Intro** (3 paragraphs) — opens with the boot banner the reader
  saw at the end of chapter 1, then states the question the chapter
  answers.
- **File list** — `core/lib/lib.h`, `core/lib/printf.c`,
  `framework/console.{h,c}`, `components/uart_pl011/uart_pl011.c`.
- **Terms section** — 11 terms: console, UART, PL011, MMIO, device
  register, device driver, `volatile`, FIFO, ring buffer, IRQ, ISR,
  format string. Standard names with one-line plain-English
  definitions, per the style guide's terminology rules.
- **What we're building toward** — the two-paths framing (direct
  `kprintf` vs. component path) so the reader knows there is a
  later layer they're not seeing yet.
- **What's a UART?** — three-wire diagram, async meaning, where
  UARTs still show up today, why kernels start with one, then the
  PL011 specifically.
- **MMIO** — magic addresses, the QEMU `virt` memory map (table
  with GIC, UART, RAM, kernel image), why `volatile` matters, why
  the access width is 32 bits even though the pointer is `uint8_t *`.
- **TX path** — `uart_putc`, `uart_init` no-op, `\n` → `\r\n`
  translation, `kprintf`'s format-string parser including the
  base-N integer formatter, full TX flow diagram.
- **RX path** — interrupt-driven model, `nx_console_init` programming
  the chip + `irq_register` + `gic_enable`, `nx_console_rx_isr`'s
  drain loop with Ctrl-C / Ctrl-D side channels, the SPSC ring
  buffer (head/tail with C11 atomics, why it's lock-free, why the
  pattern doesn't generalize to multi-producer), `nx_console_read`
  with the wait-queue-and-pollset retry loop, full RX flow diagram.
- **Two paths to the same UART** — the component-side
  `uart_pl011_write` is six lines and forwards to `nx_console_write`,
  i.e. the same `uart_putc` we already saw. Three reasons the
  component layer exists (swappable slot, separates contract from
  device, links itself in via `NX_COMPONENT_REGISTER_NO_DEPS_IFACE`),
  with explicit "the framework chapter covers the rest later".
- **A few extra things to know** — single-CPU caveats, ISR
  safety, what TX FIFO smoothing buys us, no encoding awareness,
  why TX needs no ring but RX does.
- **Where to read more** — source links into the codebase, a
  back-reference to chapter 1's CPU-state section, and external
  links (PL011 TRM, QEMU `virt` source, GICv2 spec) reserved to
  this section per the style guide.

### `book/README.md` updated

Chapter 2 status flipped from plain text to a link. No other
changes.

### `BOOK-OUTLINE.md` updated

Chapter 2 row: status `planned` → `shipped`. No other changes.

## Key Findings

- **The chapter naturally hits two distinct hardware/software
  patterns.** TX is the simple synchronous "wait on a status
  flag, then write" pattern; RX is the producer-consumer
  ISR-plus-ring pattern. Putting them in one chapter works
  because they share a chip and the comparison ("why does RX
  need a ring but TX doesn't?") is the cleanest way to motivate
  the ring buffer.
- **Two-paths framing earns its keep.** The first instinct was to
  describe only `kprintf` and ignore `uart_pl011`. But the source
  has the component sitting right there in
  `components/uart_pl011/`, and a curious reader will look. Naming
  the component path early ("we'll cover this later, but here's
  what it is") avoids the awkward state where the chapter feels
  inconsistent with the source tree.
- **The component-path section had to be tight.** Slots,
  components, the registry, and the dispatcher each get their own
  later chapters. The chapter 2 treatment is one paragraph plus a
  six-line code snippet, no more — establishing existence and the
  fact that it routes to the same `uart_putc` we already saw, and
  forward-deferring everything else without using a broken link.

## Decisions Made

- **Chapter 2 covers TX *and* RX.** The two share too much hardware
  context (PL011 registers, MMIO conventions) to split into two
  chapters. The outline calls for one chapter; the draft confirmed
  one chapter is the right size (~700 lines, well within the
  600–1500 band).
- **No opening framing analogy.** Chapter 1 used the
  "opening-a-restaurant" framing for its first section. Chapter 2's
  topic is concrete enough — "what does this line of code do?" —
  that an analogy would just be in the way. Per the style guide,
  framing analogies are at most one per chapter and only if they
  pay off; not using one here is fine.
- **No forward links to later chapters.** Per the style guide
  ("don't reference future chapters as if they exist"), every
  forward reference is prose only — "we'll cover this in a later
  chapter on X" — never a markdown link.
- **Chapter 1 §"What the CPU is doing..." is the only back-link.**
  The `volatile` and EL1-MMIO discussion both naturally point at
  chapter 1's CPU-state table; a single anchor link makes the
  connection navigable without overlinking.

## Status at End of Session

- Chapter 2 drafted and committed to `sources/nonux/book/`.
- `book/README.md` and `BOOK-OUTLINE.md` reflect `shipped` status.
- No code changes; tests not re-run. Test counts unchanged from
  Session 118: 102/102 tools · 485/485 host · 152/152 kernel.

## Next Steps

- The user has two natural directions: keep writing the book
  (chapter 3 — Physical memory and the page allocator) or pick
  up Phase 9 (per-process MM rework — see
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework)).
- Chapter 4 (MMU) and chapter 15 (Processes) carry a Phase 9
  caveat in [BOOK-OUTLINE.md](../BOOK-OUTLINE.md#phase-9-dependency).
  Whichever way Phase 9 work lands first, that decision feeds
  back into when those two chapters get drafted.

---

**Files Changed:**

`sources/nonux`:
- `book/02-console-and-uart.md` — new (1108 lines, → 1137 after the
  amendment below).
- `book/README.md` — chapter 2 row turned into a link.

`proj_docs/nonux`:
- `BOOK-OUTLINE.md` — chapter 2 status flipped to `shipped`.

---

## Post-session amendment — Terms section gap fix

After the chapter shipped, an audit against
[BOOK-STYLE-GUIDE.md](../BOOK-STYLE-GUIDE.md) rule 2 ("Explain
every standard term on first use") found five
systems-programming standard terms used in the body without a
first-use definition: **POSIX**, **system call (syscall)**,
**file descriptor (fd)**, **kernel thread (kthread)**, and
**signal / SIGTERM**. None are invented terms (so rule 1 stays
clean), but the audience profile is "comfortable with C, never
looked inside a kernel before" — those five aren't core C and
need a one-line gloss.

Fix: append all five to the Terms section, and bold the first
body appearance of each so the reader's eye links text →
Terms. No prose-level rewrites elsewhere; the patch is +29
lines, contained.

The other potential gap flagged in audit ("EL0" / "EL1" reused
without re-definition) was kept as-is — chapter 1's Terms
section defines them, and rule 3 governs definitions
*per-chapter*, not across chapters; the book's intro tells
readers a missing term is "usually one or two earlier in the
book".
