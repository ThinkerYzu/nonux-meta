# Session 43: slice 7.6c.1 — libnxlibc.a (minimal POSIX-named C archive)

**Date:** 2026-04-25
**Phase:** 7 slice 7.6c.1
**Branch:** master

---

## Goals

Slice 7.6c's canonical plan (from IMPLEMENTATION-GUIDE) was: vendor
musl, patch its syscall header, produce `libc.a`.  That's a
substantial multi-session project: network-bound submodule + autoconf
+ cross-compile + syscall-number remapping.  Rather than spend this
session half-integrating musl (with a coin-flip chance of a
still-broken build at the end), split 7.6c into two sub-slices and
nail the *smaller* one cleanly:

- **7.6c.1 (this session):** build a minimal `libnxlibc.a` — a static
  archive exposing POSIX-named wrappers (`write`, `_exit`, etc.)
  over our existing `nx_posix_*` static inlines.  Offline-clean,
  self-contained, same link-line shape as musl's `libc.a`.
- **7.6c.2 (future):** vendor musl, swap it in where libnxlibc
  currently sits; same link-line change, source unchanged.

Gets us the POSIX-named C surface now + proves the archive-based
build pattern works + leaves slice 7.6c.2's scope compact and
well-defined.

## Scope choices

- **Minimal archive, not a full libc.**  libnxlibc covers only the
  functions busybox-shaped code is likely to touch in the near term:
  stdio I/O via file descriptors, process lifecycle, signals, and
  four string/mem helpers.  No printf, no stdio FILE* layer, no
  malloc beyond raw syscall returns, no environment.  When slice
  7.6c.2's musl lands, the full libc surface comes with it;
  libnxlibc retires.
- **Magic fd routing in `write` / `read` / `close`.**  POSIX
  programs call `write(1, ...)` to talk to stdout — matching that
  expectation means the demo runs without `open("/dev/tty", ...)`
  preamble we don't have yet.  `nxlibc_write` special-cases fds
  0..2:
  - fd 1 (stdout) and fd 2 (stderr) → `NX_SYS_DEBUG_WRITE` (UART).
  - fd 0 (stdin) read returns 0 (EOF) — no console input source
    in v1.
  - fd 0..2 close is a no-op (no real handle-table entry).
  - fd > 2 dispatches through the real `NX_SYS_READ`/`WRITE`
    handlers, which slice 7.5 already made handle-type-polymorphic
    (CHANNEL ↔ FILE).
- **`nxlibc_` prefix, not bare POSIX names.**  Using bare
  `write`/`read`/etc. in nxlibc would collide with musl when slice
  7.6c.2 lands them both in the same link step (temporarily).  The
  `nxlibc_` prefix makes it obvious which surface is which during
  the transition; slice 7.6c.2 will either `#define write
  nxlibc_write` in a compat header, or the demo switches to pure
  POSIX names directly against musl.  Either way, the source-level
  churn is small.
- **`.c` + `.o` + `ar rcs`, not CMake / autoconf / whatever.**
  The top-level Makefile is already the canonical build system;
  one more `cc` + `ar` invocation adds no new tooling dependency.
  A future multi-file libc (e.g. when we start adding `printf` +
  stdio helpers in 7.6c.2) will naturally want to expand the
  Makefile rule, not ditch make for something else.
- **Keep `posix.h` static-inline headers in place alongside
  `nxlibc.h`.**  posix.h's `nx_posix_*` inlines are still useful
  for `.c` code that doesn't want to go through libnxlibc (the
  kernel-side ktest scaffolding — nothing linked against
  libnxlibc.a).  Both headers can be included in the same
  translation unit without symbol collision (inline `nx_posix_*`
  and extern `nxlibc_*` don't overlap).
- **Archive carries crt0 as the first member.**  Standard archive
  behaviour: the linker pulls in archive members on demand as
  unresolved symbols surface.  `_start` (crt0's symbol) is the ELF
  entry, so it's always unresolved at link time for a -T-scripted
  program — guaranteeing crt0 gets pulled in first.  Same effect
  as the slice 7.6c.0 `crt0.o + main.o` explicit-order link, but
  packaged as a single-archive argument.

## What Was Done

### `components/posix_shim/nxlibc.h` — new

Declarations for 15 POSIX-named functions + 6 constants:

- Process: `nxlibc_exit`, `nxlibc_fork`, `nxlibc_execve`,
  `nxlibc_waitpid`, `nxlibc_kill`.
- File I/O: `nxlibc_open`, `nxlibc_close`, `nxlibc_read`,
  `nxlibc_write`, `nxlibc_lseek`.
- IPC: `nxlibc_pipe`.
- libc: `nxlibc_strlen`, `nxlibc_strcmp`, `nxlibc_memcpy`,
  `nxlibc_memset`.
- Constants: `NXLIBC_STDIN_FILENO` (0), `NXLIBC_STDOUT_FILENO`
  (1), `NXLIBC_STDERR_FILENO` (2); `NXLIBC_O_*` flags;
  `NXLIBC_SEEK_*` whence; `NXLIBC_SIG{KILL,TERM}`.

All values bit-compatible with POSIX + slice 7.5's constants.

### `components/posix_shim/nxlibc.c` — new

Implementations.  Each function is a 1-3 line shim; the non-trivial
logic is `nxlibc_write`'s stdout/stderr routing:

```c
nxlibc_ssize_t nxlibc_write(nxlibc_fd_t fd, const void *buf, size_t count)
{
    if (fd == NXLIBC_STDOUT_FILENO || fd == NXLIBC_STDERR_FILENO)
        return nx_posix_debug_write(buf, count);
    return nx_posix_write(fd, buf, count);
}
```

### Makefile

- Added `AR := $(CROSS)ar` alongside the existing CC/LD/OBJCOPY.
- New rules:
  - `components/posix_shim/nxlibc.o` — compiled with
    `POSIX_PROG_CFLAGS`.
  - `components/posix_shim/libnxlibc.a` — `ar rcs` of
    `crt0.o + nxlibc.o`.
  - `test/kernel/posix_libc_prog.o/.elf` — compiled + linked
    with `-Lcomponents/posix_shim -lnxlibc`.
  - `test/kernel/posix_libc_prog_blob.o` — `.incbin` wrapper.
- `KTEST_C += test/kernel/ktest_posix_libc.c`.
- `KTEST_S += test/kernel/posix_libc_prog_blob.S`.
- `clean` drops `libnxlibc.a` + `posix_libc_prog.elf`.

### Demo `test/kernel/posix_libc_prog.c` — new

```c
int main(int argc, char **argv, char **envp)
{
    static const char marker[] = "[libc-ok]";
    char buf[16];
    size_t n = nxlibc_strlen(marker);
    nxlibc_memcpy(buf, marker, n);
    nxlibc_write(NXLIBC_STDOUT_FILENO, buf, n);
    if (argc != 1) nxlibc_exit(1);
    if (!argv || !argv[0]) nxlibc_exit(2);
    if (nxlibc_strcmp(argv[0], "nonux") != 0) nxlibc_exit(3);
    nxlibc_exit(53);
}
```

### Test `test/kernel/ktest_posix_libc.c` — new

KTEST `posix_libc_write_routes_stdout_through_debug_and_exit_53`:
loads the blob, spawns kthread, drops to EL0, asserts the
`[libc-ok]` marker fires and the host process exits with code 53.

## Key Findings

- **GNU ld archive-search pulls crt0 in from `-lnxlibc` without
  naming it.**  Before this slice, `posix_main_prog.elf` listed
  `crt0.o` explicitly on the link line.  With `libnxlibc.a`
  containing crt0 as a member, the linker resolves the
  unresolved `_start` symbol from the archive automatically —
  `-lnxlibc` alone suffices.  Cleaner, matches how real libc
  archives work.
- **Slice-7.x ktest linker line survives archive linkage.**  Was
  worried that `init_prog.ld`'s `*(.text*)` glob might pick up
  archive members in a surprising order (crt0 not first,
  `_start` not at offset 0).  In practice, ld pulls archive
  members right-to-left on the `.o ... -lnxlibc` command and
  places each at the next `.text` boundary — crt0 lands first
  because it's the one resolving `_start`, which is the ELF
  entry symbol.
- **AAPCS register `x9` is safe to clobber at program entry.**
  The crt0 already uses `x9` for the sp-alignment round-trip,
  and I re-read the AAPCS to double-check: x9..x15 are
  caller-saved temporaries.  At `_start` there's no caller's
  context to preserve — the kernel's `drop_to_el0` zeroed every
  reg — so touching x9 is unconditionally fine.

## Decisions Made

- **Skip printf, malloc, stdio FILE* in this slice.**  Each
  would warrant its own slice's worth of design work (printf's
  format string parser is ~400 lines; stdio needs a line-buffer
  policy; malloc needs an allocator).  Busybox will bring them
  in as part of musl; anyone hand-writing an EL0 C program in
  the meantime can use nxlibc's direct `write` + string helpers.
- **Magic fd routing in nxlibc, not in the kernel.**  Could have
  added fd=1/2 → debug_write routing to `NX_SYS_WRITE` itself,
  making `posix_write(1, ...)` from any caller hit the UART.
  Chose to keep the magic in the userspace libc instead — the
  kernel stays fd-agnostic (a fd is just a handle), and the
  "stdout goes to UART" convention is explicitly a POSIX
  userspace affordance.  When slice 7.6d wires a real console
  component with per-fd sink routing, the kernel still doesn't
  need to learn about stdio.
- **Use `nxlibc_` prefix not bare POSIX names.**  Collision
  avoidance with the eventual musl.  Explicit documented choice
  in `nxlibc.h`'s header comment; slice 7.6c.2 will layer a
  compat mapping or migrate demos directly to musl's names.
- **Roll Session 38 to archive.**  HANDOFF.md's session list
  is capped at 5; after inserting Session 43, the old Session
  38 (slice 7.4d posix_shim header) moves to the archive.

## Status at End of Session

- `make test` → **401/401 pass (51 python + 274 host + 76
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test.
- `make run` boots cleanly.
- Artefacts:
  - `components/posix_shim/libnxlibc.a` — 4.5 KB archive (2
    members: crt0.o, nxlibc.o).
  - `test/kernel/posix_libc_prog.elf` — 2.3 KB EL0 binary,
    linked against `-lnxlibc`.
- EL0 marker catalog (now 23 milestones): + `[libc-ok]`.
- The POSIX-named C surface is live; slice 7.6c.2's musl swap
  is now a finite-scope follow-up rather than a blocker.

## Next Steps

**Slice 7.6c.2 — vendor musl + swap libnxlibc for libc.**

- Vendor a musl snapshot (1.2.x) under `third_party/musl/`.
- Patch `arch/aarch64/syscall_arch.h` to emit our `NX_SYS_*`
  numbers via `svc #0` (the Linux-AArch64-ABI calling
  convention + register assignments already match; only the
  syscall-number table differs).
- Build musl's `libc.a` via its out-of-tree build.
- Swap the demo's link line from `-lnxlibc` to `-lc`, rename
  the `nxlibc_*` symbols in the demo source to pure POSIX
  names.  Same ktest, same exit code.
- Retire `libnxlibc.a` once musl is in use across all demos.

**Slice 7.6d — busybox cross-compile + boot to shell** (after
7.6c.2).  Depends on musl for its libc; initramfs slurp from
slice 7.6b gives us the blob-embed path for the busybox
binary.

**Deferred (unchanged):**

- sys_exec argv/envp push (real busybox CLI args).
- HANDLE_FILE fork inheritance.
- Proper cross-test task reap.

---

**Files Changed:**
- `sources/nonux/components/posix_shim/nxlibc.h` — new (POSIX-named surface)
- `sources/nonux/components/posix_shim/nxlibc.c` — new (implementation + fd-routing)
- `sources/nonux/test/kernel/posix_libc_prog.c` — new (C demo using POSIX names)
- `sources/nonux/test/kernel/posix_libc_prog_blob.S` — new (.incbin wrapper)
- `sources/nonux/test/kernel/ktest_posix_libc.c` — new (kernel test)
- `sources/nonux/Makefile` — AR variable; nxlibc.o + libnxlibc.a rules; posix_libc_prog build; KTEST_C/S updates; clean
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.6c.1 filled in; 7.6c.2 renumbered
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 38 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
