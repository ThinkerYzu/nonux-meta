# Book Style Guide

Rules for writing chapters of [the nonux book](../../sources/nonux/book/README.md).
The book is a tutorial-style walkthrough aimed at people studying
how kernels are built. This guide captures the conventions every
chapter must follow.

When the rules here conflict with whatever feels natural to write,
**the rules win**. They exist because the audience is non-obvious
and small inconsistencies compound across a book-length work.

---

## Audience

Write for people who:

- Are comfortable reading C.
- Have never looked inside a kernel before.
- May be CS students, hobbyists, or working programmers from
  application-domain backgrounds.
- Read English fluently but are not necessarily native speakers.

Do **not** assume the reader knows:

- ARM assembly, the ELF format, or how a linker works.
- OS-course terms (mode, ring, supervisor, segment, page table) —
  these need to be introduced.
- Hardware-specific abbreviations (MMU, GIC, UART, IRQ, ISR, …).

The target reading level is roughly **8th grade** in
sentence/vocabulary complexity. The *concepts* will often be
graduate-level, but the *prose* shouldn't be.

---

## Voice and tone

- **Plain English.** Short sentences. Active voice. Common words.
  Aim for an average sentence length around 15–20 words; cap at
  ~30. Break long sentences into shorter ones.
- **Conversational but precise.** "We" and "you" are fine. "Let's"
  is fine. Casual phrasing is fine. Imprecise statements are not —
  if a fact has caveats, state them.
- **No filler.** Don't say "it's important to note that". Just
  state the thing. Don't pad sections with throat-clearing
  paragraphs.
- **No emoji.** No exclamation points (except in real
  exclamations, which are rare).
- **Bold for first use of a term, monospace for code.** See
  Formatting below.

---

## Terminology rules

These are the strictest rules in this guide. Style and tone are
flexible; terminology is not.

### 1. Use standard terms, not invented metaphors.

When the industry has a name for something, use it. Then explain
it in plain English. Don't invent a folksy substitute.

| Use this                  | Not this                              |
|---------------------------|---------------------------------------|
| ELF header                | "ELF packing list" / "moving box"     |
| Raw binary / flat binary  | "the bytes dumped on the floor"       |
| Boot ROM                  | "sticky note inside the front door"   |
| Interrupt / IRQ           | "tap on the shoulder"                 |
| CPU                       | "the brain of the computer"           |
| Bootloader                | "the morning-shift program"           |

Reasons:

- Standard terms are searchable. A reader who later wants to learn
  more can type the term into a search engine.
- Standard terms compose. Once a reader knows "loader", they can
  read about "dynamic loader" or "PIE loader" without a fresh
  glossary.
- Invented terms create lock-in to *this* book.

### 2. Explain every standard term on first use.

A standard term *and* a plain-English explanation is the goal —
not one or the other. Pattern:

> The **linker** (a tool called `ld`) takes a bunch of object
> files and combines them into a single executable.

Bold the term. Give the one-sentence explanation in the same
sentence or the next one. Don't make the reader hunt.

### 3. Define each term once per chapter.

After the first definition, use the term freely. The reader is
allowed to learn things and remember them. If a chapter is long
enough that the reader might have forgotten, link back to the
defining paragraph rather than re-defining inline.

### 4. Maintain a Terms section near the start of each chapter.

Quick-reference list of every term the chapter introduces, with
one-line definitions. Goes after the chapter intro, before the
content sections. Lets readers scroll back when they get lost.

The Terms list is not a replacement for in-text definitions — it's
a safety net.

---

## When analogies work — and when they don't

Analogies are allowed but constrained.

**OK to use:**

- **Standard textbook analogies.** "A stack is like a pile of
  plates" is fine — it's the standard analogy used in every CS
  course, the reader will encounter it again, and it doesn't
  conflict with the standard term.
- **Illustrative comparisons.** "An address is like a house number
  on a long street" doesn't introduce a non-standard term; it just
  helps the reader picture sequential bytes.
- **Whole-chapter framing analogies.** A chapter can open with an
  extended analogy (e.g., "starting a kernel is like opening a
  restaurant") to motivate the topic. Use sparingly — at most one
  per chapter, only at the start.

**Not OK:**

- **Invented terms.** See the table above. If you find yourself
  saying "I'll call this an X for now", stop and use the real
  name.
- **Analogies that aren't true.** A framing analogy still has to
  match real-world experience. The first draft of chapter 1 said
  the bootloader is like turning on the freezer in the morning —
  but freezers run 24/7, so the analogy invited the reader to
  doubt everything else. **Fact-check every analogy.**
- **Strained analogies.** If you're explaining the analogy more
  than the thing, drop the analogy.

When in doubt: **just say the thing**. Plain technical English at
8th-grade level is almost always clearer than an analogy.

---

## Formatting conventions

### Bold

Use **bold** for the first appearance of a term that's defined
right there. After the first appearance, plain text.

> The **compiler** reads your `.c` source files and produces
> **object files** (`.o`).

### Monospace

Use `monospace` for:

- Code (function names, variable names, type names)
- File paths and filenames
- Register names (`x0`, `sp`, `CurrentEL`)
- Addresses (`0x40080000`)
- Command-line arguments and shell commands
- Section names (`.text`, `.rodata`, `nx_components`)
- Macros (`NX_COMPONENT_REGISTER`)

### Code blocks

Code blocks should be:

- Real code, not pseudocode, when possible.
- Long enough to stand on its own (don't quote a single-line
  fragment with no context).
- Followed (or preceded) by prose that explains what to look at.
  Don't drop a 30-line block on the reader and walk away.

For shell commands, use `$` for the prompt only when showing
input/output together. Plain commands (no prompt) are fine when
explaining what to run.

### Tables

Use tables when comparing 3+ items along the same dimensions
(permissions of `.text` vs `.rodata`, registers at boot, …).
Don't force a list into a table; if there's only one column of
information per row, it's a list.

### Block quotes

Block quotes (`>`) are for side notes — interesting context that
isn't on the main thread of the chapter. The reader can skip
them and not lose the plot. Use sparingly; if every page has one,
they stop standing out.

### Diagrams

ASCII diagrams are encouraged for boot timelines, layout,
boot-stage chains, etc. They render in any editor and survive in
plain-text contexts (terminals, code review).

Real diagrams (`.png` / `.svg`) are allowed for things ASCII can't
express (timing diagrams, complex state machines). Place them
under `book/figures/<chapter>/` and reference with relative
links.

### Cross-references

Always use **relative links**. Examples:

- Within the book: `[chapter 2](02-...md)`,
  `[the linker section above](#the-linker-script)`.
- To source code: `[\`core/boot/start.S\`](../core/boot/start.S)`.
- To reference docs: `[\`docs/framework-bootstrap.md\`](../docs/framework-bootstrap.md)`.
- To the project README: `[\`../README.md\`](../README.md)`.

Never link to absolute paths or external URLs that could move.
External URLs (e.g., to `kernel.org` or ARM ARM) are OK in the
"Where to read more" section at the end of a chapter, but not
inline in the prose.

---

## Chapter structure

Every chapter follows this skeleton (deviate only with reason):

```
# <Chapter title>

<1–3 paragraph intro>: what the chapter is about, what the reader
will know by the end, who it's for if that varies from the book's
default audience.

The relevant files in this repo:
- <list of source files this chapter walks through>

---

## A quick analogy first  (optional)

Whole-chapter framing analogy if there is one. One short section.

---

## Terms you'll see

Quick-reference list of every term introduced in the chapter.

---

## <Foundational background>  (if needed)

If the chapter assumes background that earlier chapters didn't
cover, introduce it here before getting to the chapter's main
content.

---

## <Main content sections>

The actual walkthrough. Each section is one focused topic.

---

## A few extra things to know  (optional)

Loose ends, side notes, related material that doesn't fit the
main flow.

---

## Where to read more

Links into the codebase, the reference docs (`docs/`), and 1–2
external authoritative resources (ARM ARM, kernel.org, …).
```

Length guidance: 600–1500 lines per chapter is the comfortable
zone. Below 400 lines, ask whether two short chapters should be
merged. Above 2000 lines, split it.

---

## Chapter naming and numbering

- **Filename:** `NN-topic-slug.md`. Two-digit prefix, kebab-case
  topic, `.md` extension. Example: `01-boot-and-linker.md`.
- **Numbers are reading order**, not creation order. If a new
  chapter belongs between existing ones, renumber and update the
  TOC.
- **One chapter per topic.** Don't merge unrelated topics into one
  file just because each is short.
- **No duplicate slugs even at different numbers.** Pick a unique,
  descriptive slug.

---

## Workflow

### Writing a new chapter

1. **Sketch the outline first.** What's the chapter's main claim?
   What does the reader need to know going in? What will they
   know coming out?
2. **Identify the standard terms** the chapter introduces. Build
   the Terms section from this list.
3. **Write the walkthrough** following the chapter skeleton above.
4. **Read it aloud.** Sentences that are awkward to say aloud are
   awkward to read.
5. **Fact-check every claim**, especially analogies. The freezer
   incident in chapter 1 is the canonical reminder.
6. **Update `book/README.md`** to add the chapter to the TOC.
7. **Update cross-references** from earlier chapters that should
   forward-reference the new one.

### Revising

- Save aggressive revisions for *before* the chapter is committed
  upstream. Once it's out, prefer minimal patches.
- When replacing a metaphor with a standard term (or vice versa),
  check every page — don't leave the old phrasing lurking.

### Session logs

Every session that lands chapter changes — drafting, revising,
restructuring — gets a session log under
[`logs/`](logs/SESSION-LOG-TEMPLATE.md), per the
project-wide doc-driven workflow described in
[`README.md`](README.md). The log records what changed and why,
test-count updates (usually unchanged for book-only sessions), and
the next steps.

---

## What not to do

A short list of patterns that have already been tried and rejected.

- **Don't pile on metaphors.** First-draft chapter 1 used four
  invented metaphors in the first three paragraphs ("moving box",
  "packing list", "sticky note", "tap on the shoulder"). Each one
  was an extra thing the reader had to track. They were all
  removed in revision.
- **Don't over-explain background.** If the reader already needed
  to know C to be reading the book, don't define `int`. Trust the
  audience description.
- **Don't reference future chapters as if they exist.** Until
  chapter N is written, don't link to it. Forward-references that
  go nowhere read as broken.
- **Don't bury the lede.** State the chapter's main claim in the
  first three paragraphs. The reader should know what they're
  signing up for before scrolling.
- **Don't write inside-baseball prose.** Avoid "as we'll see in
  slice 8.0a.6" — that's session-history language that means
  nothing to a reader. The book describes the *kernel*, not the
  *commit history*.

---

## See also

- [Book index / TOC](../../sources/nonux/book/README.md) — the
  book itself.
- [`README.md`](README.md) — proj_docs project overview.
- [`HANDOFF.md`](HANDOFF.md) — current status / session logs.
