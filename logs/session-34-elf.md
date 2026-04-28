# Session 34: slice 7.3 — ELF loader + first standalone-built EL0 program

**Date:** 2026-04-24
**Phase:** 7 slice 7.3
**Branch:** master

---

## Goals

Stop baking EL0 programs into the kernel-test source code.  Slice
5.5 (hello), 5.6 (channel), 6.3 (file), and 6.4 (readdir) all ship
their EL0 programs as hand-written `.S` files that get compiled
into `kernel-test.bin`'s `.rodata` and then memcpy'd into the user
window byte-for-byte.  That works but it scales badly — every
program has to be PC-relative, no separately-linked toolchain
support, no way for a future slice to load a program that wasn't
built with the kernel.

Slice 7.3 introduces the ELF loader: parse a PT_LOAD-bearing
AArch64 executable, copy segments into a target process's address
space, return the entry point.  Now any EL0 program can be built
as its OWN artifact — `.text` at whatever VA the ELF specifies,
entry point where the ELF says, all the normal linker-script
knobs available.

## Scope choices

- **In-memory blob, not vfs path.**  `nx_elf_load_into_process`
  takes `(blob, len)` — a raw byte buffer.  Reading from a ramfs
  path via vfs_simple is a natural follow-up but requires
  bumping `RAMFS_FILE_CAP` (currently 256 bytes, way too small for
  even a minimal ELF — ours is ~66 KiB due to phdr alignment).
  That bump lands opportunistically or with slice 7.4 where
  fork/exec's exec() needs path-based loading anyway.  For 7.3 the
  blob is embedded into `kernel-test.bin` via `.incbin`, giving us
  the benefit (EL0 program separately linked) without the ramfs
  rework.
- **ET_EXEC only.**  Static executables with absolute VAs.
  ET_DYN (position-independent) would require relocation handling
  — deferred.  Our `init_prog.ld` links at `0x48000000` (the
  user-window base), which matches what slice 7.2's
  `mmu_create_address_space` lays out per process.
- **No TTBR0 flip during the copy.**  The loader writes via the
  kernel-visible identity-map alias of the process's user
  backing, not via the per-process TTBR0.  Requires a new
  `mmu_address_space_user_backing(root)` helper to expose that
  pointer.  Benefit: kernel context stays coherent; no worry
  about TLB stale entries during multi-segment copies.
- **I-cache flush after copy, not before drop-to-EL0.**  The
  loader emits `dsb ish ; ic iallu ; dsb ish ; isb` after all
  segments are in place.  Data cache doesn't need an explicit
  clean-to-PoU because `ic iallu` implies enough ordering with
  the `dsb ish` beforehand (the stores hit coherent caches which
  the `ic iallu` invalidates).  Without this, a freshly-written
  page might be fetched as stale instruction bytes from the
  I-cache.

## What Was Done

### `framework/elf.{h,c}` — new

- `struct nx_elf_info {entry, segment_count}` — the compact
  header summary.  `struct nx_elf_segment {file_offset, vaddr,
  file_size, mem_size, flags}` — per-PT_LOAD shape.
- `nx_elf_parse(blob, len, *out)` — validates ELF magic
  (`0x7F 'E' 'L' 'F'`), `ELFCLASS64`, `ELFDATA2LSB`, `ET_EXEC`,
  `EM_AARCH64`, and that the phdr table fits in the blob.
  Populates `out` if given.  Returns `NX_OK` or `NX_EINVAL`.
- `nx_elf_segment(blob, len, idx, *out)` — dense iterator:
  skips non-LOAD phdrs internally so the caller's `idx` matches
  `info.segment_count`.  Returns one segment's raw shape.
- `nx_elf_load_into_process(target, blob, len, *out_entry)` —
  the end-to-end loader.  Walks every PT_LOAD, computes the
  kernel-visible destination via `mmu_address_space_user_backing
  + (vaddr - user_window_base)`, memcopies `file_size` bytes,
  zeros `mem_size - file_size`, emits the dcache/icache barrier
  sequence, writes the entry to `*out_entry`.  Host stub
  validates the blob and reflects the entry point without
  actually copying (no MMU).

### `core/mmu/mmu.{h,c}` — user-backing accessor

- `mmu_address_space_user_backing(root)` looks up the
  bookkeeping entry keyed by `root` and returns the
  per-process 2 MiB backing's kernel-visible pointer (identity-
  map alias, VA==PA).  Returns NULL for root==0, for the kernel
  root (no per-process backing), or an unrecognised root.  Host
  returns NULL.
- No ABI/test impact on existing code: the previously-added
  `mmu_*` entry points are unchanged.

### Standalone EL0 ELF artifact

- `test/kernel/init_prog.S` — `_start` calls
  `NX_SYS_DEBUG_WRITE("[el0-elf-ok]", 13)` and parks in `wfe`.
  No PC-relative gymnastics needed since this program is a real
  ELF with absolute VAs linked at the user-window base.
- `test/kernel/init_prog.ld` — linker script: `ENTRY(_start); . =
  0x48000000; .text : { *(.text*) *(.rodata*) } .data : { ... }
  .bss (NOLOAD) : { ... }`.  Discards `.comment`, `.note*`,
  `.ARM.attributes`, `.eh_frame` — the loader doesn't consume them
  and keeping them would bloat the blob.
- `test/kernel/init_prog_blob.S` — `.incbin
  "test/kernel/init_prog.elf"` wrapped between
  `__init_prog_blob_{start,end}` labels for C code to reference.

### Makefile rules

- `test/kernel/init_prog.o`: standard `.S → .o`.
- `test/kernel/init_prog.elf`: linked with `-T init_prog.ld`.
- `test/kernel/init_prog_blob.o`: explicit prereq on
  `init_prog.elf` so the assembler picks up regenerated bytes
  (standard `.S` rule doesn't track `.incbin` deps).
- `KTEST_S += init_prog_blob.S` pulls the blob into
  `kernel-test.bin`.
- `clean` target removes `init_prog.elf`.

### Tests

- `test/kernel/ktest_elf.c` — three tests:
  - `elf_parse_header_reports_entry_and_one_pt_load` — parser
    summary matches `init_prog.ld`.
  - `elf_parse_rejects_malformed_magic` — negative case: flip
    byte 0 in a copy of the blob, expect `NX_EINVAL`.
  - `elf_load_and_drop_to_el0_runs_marker_syscall` — the
    headline round-trip.  Creates a process, loads the blob,
    spawns a kthread, reassigns `task->process = target`, yields
    until `debug_write_calls > 0` (proof EL0 ran), dequeues the
    stranded task, restores kernel TTBR0, destroys the process.
    `[el0-elf-ok]` joins the other five EL0 markers in the
    live ktest log.
- `test/host/elf_test.c` — 8 tests on synthetic blobs:
  - `accepts_valid_single_load` — happy path.
  - `skips_non_load_phdrs_in_count` — one PT_NULL pad + 3
    PT_LOADs → parser reports 3.
  - `iterates_by_dense_index` — `idx=0..2` each returns the
    expected vaddr + sizes.
  - `rejects_bad_magic` / `wrong_class_or_encoding` /
    `wrong_type_or_machine` / `truncated_blob` — validation
    failure paths.
  - `host_stub_reflects_entry` — the host loader returns the
    entry even though it can't actually copy anything.
- Synthetic-blob builder in the host test file writes bytes
  field-by-field into a static buffer.  Readable, reviewable,
  and tied to the spec values (`ELFCLASS64 = 2`, `ET_EXEC = 2`,
  etc.) directly.

### Wiring

- Top Makefile: `FW_C += framework/elf.c`; `KTEST_C +=
  ktest_elf.c`; `KTEST_S += init_prog_blob.S`; Makefile rules
  for building `init_prog.elf` and its `.o`.
- `test/host/Makefile`: `SRCS += elf_test.c` + `elf.c`.

## Key Findings

- **`.incbin` beats manual bin2c.**  Our first design used a
  Python `tools/bin2c.py` to emit a C byte array from the ELF.
  GAS's `.incbin` directive does the same thing in one line,
  requires no new tooling, and keeps the blob out of the
  preprocessor.  The only gotcha is that `.incbin` doesn't
  auto-track dependencies — Makefile needs an explicit prereq
  on the ELF file for the `.S` wrapper's `.o`.
- **Kernel loader without TTBR0 flip is cleaner than with.**
  Alternative: switch TTBR0 to the target during copy, switch
  back after.  That works but adds TLB-flush cost per load +
  complicates the kernel-only invariant ("we never run kernel
  code under a user TTBR0 unless we're about to context-switch
  into that task").  The user-backing-alias approach keeps the
  kernel firmly in its own TTBR0 during the copy.  Requires
  `mmu_address_space_user_backing` but that's a small addition.
- **I-cache flush is mandatory, not merely defensive.**  First
  draft of `ktest_elf_*` occasionally ran garbage from the
  user window — turns out the program bytes were in the D-cache
  post-copy, but the CPU was fetching from I-cache via the old
  mapping (from the slice-7.2 "two-processes-different-bytes"
  test earlier in the run).  Adding `dsb ish ; ic iallu ; dsb
  ish ; isb` after the copy makes every subsequent load run on
  freshly-fetched instruction bytes.

## Decisions Made

- **Host loader is a validation stub.**  Alternative: make the
  loader a no-op on host that returns NX_ENOTSUP.  Chose
  validate-and-reflect because host tests want to verify the
  PARSER side works even though they can't exercise the MMU
  side — running `nx_elf_parse` through the loader path is the
  simplest way to get that coverage without a third entry
  point.  How to apply: whenever a kernel-only function has a
  data-validation sub-step, reflect that sub-step in the host
  stub instead of returning NX_ENOTSUP.
- **Loader takes `struct nx_process *`, not a TTBR0 root.**
  Alternative: `nx_elf_load_into_ttbr0(uint64_t root, ...)`.
  Chose the process handle because (a) callers already have
  it, (b) it lets a future slice plug in per-process quota
  checks / perms without changing the signature, (c) it matches
  the vocabulary of everything else in `framework/`.  How to
  apply: any future loader-ish function takes the process, not
  a lower-level primitive, so extra per-process context can
  be consulted without API churn.
- **Don't migrate existing EL0 ktests to the loader.**  The
  baked-in `user_prog_*.S` files still work; migrating them
  would mean assembling four more standalone ELFs + maintaining
  their build rules.  Worth it when there's a pay-off — e.g.,
  slice 7.4 where fork/exec will need the ELF loader anyway
  and the migration rides along.  How to apply: "migrate when
  there's a reason," not "migrate because the new thing is
  prettier."

## Status at End of Session

- `make test` → **390/390 pass (51 python + 274 host + 65 kernel),
  0 leaks, 0 errors, exit 0**.  +11 from slice 7.2 (+8 host,
  +3 kernel).
- `make run` boots cleanly under QEMU.  The ELF loader is only
  exercised by `ktest_elf`; normal boot doesn't touch it.
- Live ktest log now shows six complete EL0 round-trips:
  `[el0] hello` · `[el0-chan-ok]` · `[el0-file-ok]` ·
  `[el0-rdr-ok]` · `[el0-elf-ok]` · (plus the various `ktest`
  internals) — each a distinct syscall family or loading
  mechanism.
- `init_prog.elf` is 66 KiB on disk (phdr alignment is 0x10000)
  but only ~36 bytes of actual program code.  First slice
  where build-system-emitted artefact size becomes worth
  noting; if it becomes a space concern, a custom linker
  script that packs segments more tightly would help.

## Next Steps

Slice 7.3.5 (opportunistic) or roll into 7.4:

- Bump `RAMFS_FILE_CAP` from 256 bytes to something that can
  hold a minimal static ELF (at least 1 KiB; probably 64 KiB
  to match our build output).  Every pre-existing ramfs test
  still passes because nothing depends on the 256 cap today.
- Add `nx_process_create_from_elf(path)` — opens the path via
  the `vfs` slot, reads bytes into a kernel buffer, delegates
  to `nx_elf_load_into_process`.  Probably introduces a
  `framework/process.c` helper or a utility in `elf.c`.
- Seed ramfs with `/init` (the same `init_prog.elf` blob) at
  bootstrap time via a new `ramfs_seed_from_blob` helper, so
  the loader has something to load without every test
  populating it manually.

Slice 7.4 proper (after the ramfs-path loader):

- `components/posix_shim/` — map POSIX calls onto the existing
  nonux syscalls.
- `NX_SYS_FORK` — duplicate process's address space (eager
  copy via `mmu_create_address_space` + byte-copy of the user
  backing; slice 8's hot-swap can revisit COW later) + handle
  table; child resumes from fork with x0 = 0.
- `NX_SYS_EXEC(path)` — load a new ELF into the caller's
  process (allocate a fresh address space, swap in, free the
  old; handle table carries over).
- `NX_SYS_WAIT(pid, *status)` — polls with yield until target
  is EXITED, returns exit code.

---

**Files Changed:**
- `sources/nonux/framework/elf.h` — new (parser + loader API)
- `sources/nonux/framework/elf.c` — new (implementation, kernel + host variants)
- `sources/nonux/core/mmu/mmu.h` — `mmu_address_space_user_backing` prototype
- `sources/nonux/core/mmu/mmu.c` — implementation
- `sources/nonux/test/kernel/init_prog.S` — new (standalone EL0 program)
- `sources/nonux/test/kernel/init_prog.ld` — new (linker script)
- `sources/nonux/test/kernel/init_prog_blob.S` — new (`.incbin` wrapper)
- `sources/nonux/test/kernel/ktest_elf.c` — new (3 tests)
- `sources/nonux/test/host/elf_test.c` — new (8 tests on synthetic blobs)
- `sources/nonux/Makefile` — FW_C += elf.c; KTEST_C += ktest_elf.c; KTEST_S += init_prog_blob.S; rules for init_prog.elf + blob prereq; clean adds init_prog.elf
- `sources/nonux/test/host/Makefile` — SRCS += elf_test.c + elf.c
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.3 rewritten with as-built details
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 29 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
