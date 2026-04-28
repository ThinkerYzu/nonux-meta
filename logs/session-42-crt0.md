# Session 42: slice 7.6c.0 — EL0 C-runtime bootstrap (musl prereq)

**Date:** 2026-04-25
**Phase:** 7 slice 7.6c.0
**Branch:** master

---

## Goals

Open slice 7.6c (musl integration) by landing the C-runtime
infrastructure musl will plug into — crt0, linker conventions, and a
tiny libc surface.  Splitting the work this way means slice 7.6c.1
(actual musl pin + cross-build) is a clean drop-in: replace our
crt0 with musl's, replace the static-inline libc helpers with musl's
real symbols, change linker flags.  No structural surprises waiting
once the musl tarball lands.

## Scope choices

- **crt0 in posix_shim, not in test/kernel/.**  posix_shim is the
  userspace-facing component; crt0 belongs there alongside the
  POSIX wrappers.  When 7.6c.1 vendors musl, crt0.S becomes the
  fallback for "I don't want to link against musl" — header-only
  builds still work, just lose `int main()` form.
- **Synthesise argv in `.rodata`, don't expect sys_exec to push
  it.**  POSIX programs receive argv/envp from a kernel-prepared
  user stack; the loader-side push-argv-onto-sp_el0 dance is a
  proper sys_exec extension that lands in slice 7.6d when busybox
  needs real CLI args.  For 7.6c.0, crt0 fakes a single-element
  `{ "nonux", NULL }` vector — programs that need real argv (none
  in v1) will work once 7.6d adds the kernel-side push.  The
  sub-slice scope is tight: prove the C-runtime SHAPE works,
  defer real argv to when something needs it.
- **Don't go through the C wrapper for the final exit.**  crt0
  could `bl nx_posix_exit` after main returns; it's only one
  static-inline call.  But that requires routing through C
  semantics for what's really a one-shot syscall right at
  program shutdown.  Inline `mov x8, #11; svc #0` is shorter,
  matches what musl's crt does, and means the assembler-only
  view of the program lifecycle is `_start → main → svc #11`.
- **`static inline` libc helpers, not separate .c implementations.**
  Same reasoning as posix_shim's existing wrappers (slice 7.4d):
  EL0 C programs `#include` the header and the compiler inlines
  every call.  No separate libc.a to link against.  When 7.6c.1
  pulls in musl, these `static inline` definitions become
  shadowed by musl's `extern` declarations — the C standard says
  `static inline` doesn't externally collide, so the swap is
  source-compatible.  Programs that explicitly use `nx_*` names
  keep getting our inline; programs using POSIX-named
  `strlen/memcpy/etc.` get musl's after 7.6c.1.
- **Demo exercises EVERY helper.**  The libc helpers are tiny
  (3-5 lines each), but we want a CI-style regression catch if
  any of them generates a libgcc memcpy call or similar.
  `posix_main_prog.c` strlen + memcpy + strcmp the marker before
  emitting it — any compiler-introduced external dependency
  fails the link.

## What Was Done

### `components/posix_shim/crt0.S` — new

Tiny crt0:

```asm
_start:
    mov     x9, sp                  # defensive 16-byte align
    and     x9, x9, #~15
    mov     sp, x9
    mov     x0, #1                  # argc
    adrp    x1, _argv_storage       # argv via PIC arithmetic
    add     x1, x1, :lo12:_argv_storage
    mov     x2, #0                  # envp = NULL
    bl      main
    mov     x8, #11                 # NX_SYS_EXIT
    svc     #0
    1: wfe; b 1b                    # unreachable
```

`.rodata` holds the argv backing:

```asm
_argv_progname: .asciz "nonux"
_argv_storage:  .quad _argv_progname, 0
```

### `components/posix_shim/posix.h` — libc helpers

Four `static inline` additions:

- `size_t nx_strlen(const char *)`
- `int    nx_strcmp(const char *, const char *)`
- `void  *nx_memcpy(void *, const void *, size_t)`
- `void  *nx_memset(void *, int, size_t)`

Each is 3-6 lines.  Straight loops; no SIMD, no alignment
tricks.  Compiler inlines them at every call site.

### `test/kernel/posix_main_prog.c` — new

```c
int main(int argc, char **argv, char **envp)
{
    (void)envp;
    static const char marker[] = "[main-ok]";
    char buf[16];
    size_t n = nx_strlen(marker);
    nx_memcpy(buf, marker, n);
    nx_posix_debug_write(buf, (size_t)n);

    if (argc != 1) return 1;
    if (!argv || !argv[0]) return 2;
    if (nx_strcmp(argv[0], "nonux") != 0) return 3;
    return argc + 46;   // → exit(47)
}
```

Six distinct exit codes (1..3 for specific failures, 47 for
success) so the ktest can pinpoint regressions.

### `test/kernel/posix_main_prog_blob.S` — new

`.incbin` wrapper, same shape as the other slice-7.x EL0 demos.

### `test/kernel/ktest_posix_main.c` — new

KTEST `posix_main_entry_invokes_main_with_argv_and_returns_
exit_code`:

- Loads the embedded blob via the slice-7.3 ELF loader.
- Spawns kthread + drops to EL0.
- Yields until debug_write counter ≥ 1 (proves crt0 → main
  → debug_write reached).
- Walks the process table looking for the host process EXITED
  with `exit_code == 47` (proves crt0 → main return →
  nx_posix_exit reached).

### Makefile

- `components/posix_shim/crt0.o` — its own translation unit.
- `test/kernel/posix_main_prog.o` — same flag set as
  posix_prog.o (POSIX_PROG_CFLAGS).
- `test/kernel/posix_main_prog.elf` — links `crt0.o` first,
  then `posix_main_prog.o`.  Linker order matters: crt0 owns
  the ELF entry symbol so it must be at the start of `.text`.
- `clean` drops `posix_main_prog.elf`.

## Key Findings

- **ARM64 has no direct `bic sp, sp, #imm`.**  My first crt0
  draft tried `bic sp, sp, #15` for the alignment.  The
  assembler emits "expected an integer or zero register at
  operand 2" — `sp` isn't a valid `bic` destination because
  `bic` requires a GPR register slot, and `sp` is encoded
  differently from `x0..x30` (it's `xzr` aliased to context).
  Fix: round-trip via `x9` (caller-saved scratch, AAPCS lets us
  clobber freely at program entry):
  ```
  mov  x9, sp
  and  x9, x9, #~15      # (or the bic variant — and-with-mask is fine)
  mov  sp, x9
  ```
  Standard idiom; musl's own crt0 uses the same trick.
- **`adrp` + `:lo12:` is the canonical ARM64 PIC argv load.**
  The argv_storage symbol is in `.rodata`; addressing it as
  ```
  adrp x1, _argv_storage
  add  x1, x1, :lo12:_argv_storage
  ```
  resolves at link time because `init_prog.ld` pins everything
  at `0x48000000`.  No GOT, no dynamic relocations — we're
  building a position-dependent statically-linked ELF.
- **Linker order is load-bearing for crt0.**  `init_prog.ld`'s
  `*(.text*)` glob picks up sections in command-line order.
  Putting `crt0.o` BEFORE `posix_main_prog.o` ensures `_start`
  lands at offset 0 of `.text`, which puts it at the
  user-window base after `init_prog.ld` pins `. = 0x48000000`.
  Reverse order works at link time but the entry symbol ends
  up after main's code — `drop_to_el0` jumps to the wrong
  place.

## Decisions Made

- **No real `int main()` argc/argv from sys_exec yet.**  Lands
  in slice 7.6d when busybox needs real CLI parsing.  Until
  then, crt0 fakes argc=1 + argv={ "nonux", NULL } so programs
  can `(void)argc; (void)argv;` and move on, while a forward-
  thinking program (`posix_main_prog.c`) can already test for
  expected values.
- **Helpers prefixed `nx_` not `posix_`.**  The slice 7.4d
  posix_shim convention is `nx_posix_*` for POSIX-named
  syscalls; the libc helpers are nx_-prefixed (no `posix_`
  middle word) because they're not POSIX-specific (memcpy
  isn't a POSIX call — it's libc).  When musl arrives,
  POSIX-named `memcpy` becomes available as a real symbol;
  `nx_memcpy` stays as an explicit nonux-namespaced helper
  for code that doesn't want to link musl.
- **Don't refactor existing programs.**  Every existing slice-
  7.x EL0 program uses `_start` directly.  Switching them to
  `int main()` form would force linking crt0.o into 7+ build
  rules for no real benefit — the existing programs work
  fine.  New EL0 programs (slice 7.6d's busybox, primarily)
  use the main-form by default; we can migrate older
  programs opportunistically if their layout changes.
- **`bic` vs `and` doesn't matter for the alignment.**
  `and x9, x9, #~15` and `bic x9, x9, #15` produce identical
  code.  Picked `and` for explicit clarity — the mask is
  visible in the source, no mental "bic = AND NOT" inversion
  required.

## Status at End of Session

- `make test` → **400/400 pass (51 python + 274 host + 75
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test.
- `make run` boots cleanly.
- 22 distinct EL0 markers across the test suite (now adds
  `[main-ok]`).
- The C-runtime story is good enough for slice 7.6d's busybox
  to plug into; only sys_exec's argv push is still missing,
  and that's a kernel-side change rather than a userspace
  one.

## Next Steps

**Slice 7.6c.1 — vendor + cross-build musl.**

- Vendor a musl snapshot (e.g., 1.2.5) under
  `third_party/musl/` rather than a git submodule — keeps the
  build offline-clean, lets us pin a specific version, and
  any future bumps go through a deliberate vendor refresh.
- Patch `arch/aarch64/syscall_arch.h` to map musl's
  `__syscall*` family to our `NX_SYS_*` numbers via `svc #0`.
  Linux-AArch64-ABI compatibility means most of musl's syscall
  helpers work unchanged; only the syscall-number table differs.
- Run musl's configure + make to produce
  `third_party/musl/lib/libc.a` (built by an out-of-tree
  build directory so a `make clean` leaves musl's checkout
  intact).
- Validate: relink `posix_main_prog` against `libc.a` instead
  of posix_shim's headers; replace `nx_posix_debug_write` with
  `write(STDOUT_FILENO, ...)`.  Same `[main-ok]` marker, this
  time through musl's libc.

**Slice 7.6d — busybox cross-compile + boot to shell** (after
7.6c.1).  Cross-compile busybox against the musl from 7.6c.1,
drop into the initramfs, set `/init` to busybox's `sh`.

**Deferred:**

- sys_exec argv/envp push (needed for real busybox CLI args).
- HANDLE_FILE fork inheritance.
- Proper cross-test task reap.

---

**Files Changed:**
- `sources/nonux/components/posix_shim/crt0.S` — new (`_start` → main → exit)
- `sources/nonux/components/posix_shim/posix.h` — `nx_strlen / strcmp / memcpy / memset` static inlines
- `sources/nonux/test/kernel/posix_main_prog.c` — new (main-entry C demo)
- `sources/nonux/test/kernel/posix_main_prog_blob.S` — new (.incbin wrapper)
- `sources/nonux/test/kernel/ktest_posix_main.c` — new (kernel test)
- `sources/nonux/Makefile` — crt0.o + posix_main_prog.{o,elf} + blob rules; KTEST_C/S updates; clean
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.6c sub-sliced into 7.6c.0 (done) + 7.6c.1 (pending)
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 37 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
