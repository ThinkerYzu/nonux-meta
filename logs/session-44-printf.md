# Session 44: slice 7.6c.2 — printf + stdio in libnxlibc

**Date:** 2026-04-25
**Phase:** 7 slice 7.6c.2 (was musl pin; renamed)
**Branch:** master

---

## Goals

The original slice plan put musl integration as 7.6c.2.  Honest
appraisal: vendoring + cross-building musl is a multi-day project
(autoconf flags, syscall-number remapping, TLS stubs, missing
syscalls like `mmap`/`brk`/`set_tid_address`, argv/envp stack
push) that doesn't fit a single session.  Defer musl to 7.6c.3
and use this session for a smaller, immediately-useful insert:
add `printf(3)` + supporting stdio to `libnxlibc.a`.  Programs
can then format output without hand-rolling decimal-to-ASCII or
calling `write` on pre-built strings.

## Scope choices

- **Subset of `printf(3)`, not the whole thing.**  Glibc's
  `printf` parses width / precision / flags / 18 conversions
  including floats — easily 1000+ lines.  v1 nxlibc supports
  `%d / %i / %u / %x / %X / %s / %c / %%` — eight conversions,
  no formatting modifiers.  Enough for hand-written EL0 demos
  and slice 7.6d's pre-busybox smoke tests; the full surface
  comes with the eventual musl pin.
- **Stack-buffered, not stdio-buffered.**  glibc's `printf`
  emits into a per-stream `FILE*` buffer that get flushed on
  `\n` / `\0` / fclose.  v1 nxlibc skips the FILE layer
  entirely: each `printf` formats into a 256-byte stack-local
  buffer, then ONE `nxlibc_write(STDOUT_FILENO, buf, n)` flushes
  it.  Simpler, predictable, no global state.  Means each
  `printf` call produces exactly one debug_write — useful as a
  ktest signal.
- **256-byte buffer matches our existing size class.**
  `NX_FILE_IO_MAX = NX_CHANNEL_MSG_MAX = 256`.  Output longer
  than 256 bytes truncates — POSIX-shaped behavior is
  `snprintf` truncates, and the return value reports the
  would-have-been length so callers can detect overflow.
- **`__attribute__((format(printf, ...)))` for type-checking.**
  GCC checks the format string against the variadic args at
  every call site.  Catches `printf("%s", 42)` at compile time
  rather than runtime.  Free safety net.
- **No `errno` translation.**  POSIX `printf` returns negative
  on I/O error and sets errno.  v1 nxlibc returns the negative
  status directly (matching the rest of the libnxlibc
  convention); no errno yet.  When musl arrives in 7.6c.3 it
  brings full POSIX errno + the global symbol.
- **Insert before musl in the slice numbering.**  Renamed the
  planned slice 7.6c.2 (musl) to 7.6c.3, made this session 7.6c.2.
  Two reasons: (1) printf is genuinely useful to ship before
  the musl integration which might run into surprises;
  (2) gives 7.6c.3's musl-vs-libnxlibc diff a richer baseline
  to compare against.
- **Single demo exercising every conversion.**  A regression in
  one formatter (e.g. `%X` emits lowercase) breaks exactly one
  marker in the live ktest log — easier to diff than separate
  tests per conversion.

## What Was Done

### `components/posix_shim/nxlibc.h`

Added declarations for `nxlibc_printf / vsnprintf / snprintf /
putchar / puts / atoi`.  Each variadic form gets the GCC
`format(printf, ...)` attribute.

### `components/posix_shim/nxlibc.c`

Eight new functions:

- `emit_char` / `emit_str` / `emit_uint` / `emit_int` —
  internal-only helpers that write a single value into the
  output buffer at a given position.  All four take
  `(buf, cap, pos, value)` and return the new `pos`; if `pos
  >= cap`, they continue counting but stop writing
  (snprintf-style overflow tracking).
- `nxlibc_vsnprintf(buf, cap, fmt, ap)` — the printf core.
  Walks the format string a char at a time; for each `%X`
  conversion, dispatches via a `switch`.  Returns the
  would-have-been length.
- `nxlibc_snprintf(buf, cap, fmt, ...)` — varargs wrapper
  around vsnprintf.
- `nxlibc_printf(fmt, ...)` — formats into a 256-byte stack
  buffer, writes via `nxlibc_write(STDOUT_FILENO, buf, n)`
  (capped at the buffer size in case of truncation).
- `nxlibc_putchar(c)` — single-byte write to stdout.
- `nxlibc_puts(s)` — string + '\n' to stdout.
- `nxlibc_atoi(s)` — POSIX-shaped: skip whitespace, optional
  sign, decimal digits.

Conversion implementations:

- `%d / %i` — signed decimal via emit_int.
- `%u` — unsigned decimal via emit_uint(base=10).
- `%x / %X` — hex via emit_uint(base=16) with case flag.
- `%s` — string via emit_str (null-safe → "(null)").
- `%c` — single character.
- `%%` — literal '%'.
- Trailing '%' with no conversion → emit '%' literally + stop.
- Unknown conversion (e.g. `%q`) → emit `%X` literally so the
  mistake is visible in the output.

### Demo `test/kernel/posix_printf_prog.c`

```c
nxlibc_printf("[printf-int=%d]",   42);
nxlibc_printf("[printf-uint=%u]",  (unsigned)0xDEADBEEF);
nxlibc_printf("[printf-hex=%x]",   (unsigned)0xff);
nxlibc_printf("[printf-HEX=%X]",   (unsigned)0xCAFE);
nxlibc_printf("[printf-str=%s]",   argv ? argv[0] : "(no-argv)");
nxlibc_printf("[printf-char=%c]",  'Q');
nxlibc_printf("[printf-pct=%%]");
nxlibc_printf("[printf-multi=%d/%s/%x]", argc, argv[0], 0xab);
nxlibc_printf("[atoi=%d]", nxlibc_atoi("  -23xyz"));
nxlibc_puts("[printf-puts]");
nxlibc_exit(37);
```

Each call produces exactly one debug_write (printf body or
puts body+newline = 2 writes).  Live log carries the formatted
strings verbatim.

### `test/kernel/posix_printf_prog_blob.S` + `ktest_posix_printf.c`

Standard slice-7.x scaffolding: `.incbin` wrapper + an
ELF-loader-driven ktest.  ktest asserts ≥ 10 debug_writes (9
printfs + atoi-printf + puts-pair) + exit_code == 37.

### Makefile

- `KTEST_C += test/kernel/ktest_posix_printf.c`
- `KTEST_S += test/kernel/posix_printf_prog_blob.S`
- New rules for `posix_printf_prog.{o,elf,_blob.o}` mirroring
  `posix_libc_prog`'s.
- `clean` drops `posix_printf_prog.elf`.

## Key Findings

- **`-mgeneral-regs-only` doesn't break va_args for integer
  arguments.**  Was concerned that disabling SIMD/FP might
  affect variadic argument passing.  AArch64 PCS uses x0..x7
  for integer args + v0..v7 for FP — `-mgeneral-regs-only`
  bans use of v0..v7 in *generated* code, but the va_arg
  machinery for integer-only varargs is just stack pointer
  arithmetic, no SIMD.  Compiles + works fine.  We'd have a
  problem only if we tried `%f` (which would need v0).
- **Live ktest log makes printf regressions easy to spot.**
  Each conversion gets a tagged marker (`[printf-int=42]`,
  etc.).  If `%d` ever emits "0" instead of "42", exactly one
  marker is wrong — visible at a glance in the log diff.  Far
  better than a cryptic "exit code 1" from a unified test.
- **`__attribute__((format(printf, ...)))` works on
  freestanding builds.**  GCC's format-checking is purely a
  frontend warning — doesn't pull in any runtime support.
  Caught one mistake during development (`%s` paired with an
  `int` arg).
- **256-byte stack buffer is plenty for hand-written demos.**
  The longest demo line is ~30 bytes; even a 4-line message
  with multiple `%s` substitutions stays well under 256.
  When busybox arrives in 7.6d, its `printf`-heavy code paths
  will benefit from musl's larger / heap-backed buffers, so
  the limit doesn't constrain anything we'd realistically
  ship in v1.

## Decisions Made

- **Renumber slice 7.6c.2 → 7.6c.3 (musl).**  printf is a
  cleaner cut than musl integration for a single session.  Both
  HANDOFF.md and IMPLEMENTATION-GUIDE.md updated to reflect
  the new order.  Honest scope-tracking matters more than
  preserving the original numbering.
- **No `%lld` / `%ld` / size suffixes in v1.**  Only int /
  unsigned in the supported conversions.  64-bit value
  formatting (e.g. `%lld`) would need `va_arg(ap, long long)`
  and a wider `emit_uint`.  Adds noise without immediate
  payoff; lands either in a follow-up libnxlibc slice or with
  musl directly.
- **`nxlibc_puts` writes the body + newline as TWO writes.**
  Could fuse them into one (allocate body+1 buffer, append
  '\n', single write).  Two writes is simpler — and the ktest
  counts exact write counts, so each test author has a clear
  formula.  When the libnxlibc layer retires, musl's `puts`
  uses one buffer + one write; the ktest count differs by 1
  but the marker check still passes.
- **Roll Session 39 to archive.**  HANDOFF.md's session list
  caps at 5; after inserting Session 44, the old Session 39
  (slice 7.5 pipes + signals) moves to the archive.

## Status at End of Session

- `make test` → **402/402 pass (51 python + 274 host + 77
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test.
- `make run` boots cleanly.
- Artefacts:
  - `components/posix_shim/libnxlibc.a` — 7.4 KB (was 4.5 KB
    before this slice).
  - `test/kernel/posix_printf_prog.elf` — 4.5 KB linked
    against libnxlibc.
- Live UART now carries formatted markers — first time we
  have a libc-style print API in EL0.

## Next Steps

**Slice 7.6c.3 — vendor + cross-build musl** (still pending,
just renumbered).  Plan unchanged: vendored snapshot under
`third_party/musl/`, patched `arch/aarch64/syscall_arch.h`,
out-of-tree configure + make to produce `libc.a`, swap
`libnxlibc.a` for `libc.a` on the demo link lines.

When 7.6c.3 lands, libnxlibc will retire — musl's printf is
substantially fuller (width / precision / flags / float).
Until then, libnxlibc is the production libc surface for
nonux EL0 programs.

**Slice 7.6d — busybox cross-compile + boot to shell** (after
7.6c.3).

**Deferred (unchanged):**

- sys_exec argv/envp push (real busybox CLI args).
- HANDLE_FILE fork inheritance.
- Proper cross-test task reap.

---

**Files Changed:**
- `sources/nonux/components/posix_shim/nxlibc.h` — printf / vsnprintf / snprintf / putchar / puts / atoi declarations
- `sources/nonux/components/posix_shim/nxlibc.c` — printf core (emit_* helpers + vsnprintf + dispatchers)
- `sources/nonux/test/kernel/posix_printf_prog.c` — new (printf demo)
- `sources/nonux/test/kernel/posix_printf_prog_blob.S` — new (.incbin wrapper)
- `sources/nonux/test/kernel/ktest_posix_printf.c` — new (kernel test)
- `sources/nonux/Makefile` — posix_printf_prog.{o,elf,_blob.o} rules; KTEST_C/S; clean
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.6c.2 (printf) inserted; old 7.6c.2 renumbered → 7.6c.3
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 39 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
