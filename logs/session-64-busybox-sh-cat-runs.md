# Session 64: slice 7.6d.N.7 — `busybox sh -c "cat /banner"` runs end-to-end

**Date:** 2026-04-27
**Phase:** 7 slice 7.6d.N.7 (busybox-as-shell, cat reads a real file)
**Branch:** master

---

## Goals

Slice 7.6d.N.6b (session 63) closed the first multi-process
pipeline (`echo hello | cat`) by introducing the CONSOLE
handle, slot 0/1/2 reservation, channel EOF, and
`waitpid(-1)`.  In that slice cat's stdin was a CHANNEL
handle (the read end of the pipe) — a non-seekable byte
stream with explicit peer-close → EOF semantics.

This sub-slice escalates one rung sideways: drive cat
against a **real ramfs file** (`/banner`, seeded by
initramfs as `hello from initramfs\n`).  Differences from
the pipe slice:

1. cat's input is a `HANDLE_FILE` (slice 6.3 vfs reads), not
   a `HANDLE_CHANNEL` — exercises the FILE arm of `sys_read`
   and EOF-by-end-of-file rather than EOF-by-peer-closed.
2. ash forks once (one stage) instead of twice — closes a
   different control-flow path through ash's eval loop.
3. cat may take a different fast path: `bb_copyfd_eof` can
   choose sendfile (regular-file optimisation) before
   falling back to the read-write loop.  Slice 7.6d.N.6a
   noted that `__NR_sendfile (71)` is unmapped, so cat
   should land on the read-loop path; this slice confirms
   the fall-through actually works.
4. PATH walk for `/bin/cat` was already exercised by slice
   7.6d.N.4 (cpio-duplicate added then), so this sub-slice
   inherits a working PATH lookup for cat verbatim.

Discovery-driven, just like the prior 7.6d.N.x sub-slices.
Write the test, observe, plan the next sub-slice from what
surfaces.

## Implementation

### `test/kernel/posix_busybox_sh_cat_prog.c`

libnxlibc-linked program.  Forks; child does:
```c
nxlibc_execve("/bin/busybox", { "sh", "-c", "cat /banner", NULL }, NULL);
```
Parent waits + emits `[bbsh-cat-parent]`,
`[bbsh-cat-status=NN]`, and one of `[bbsh-cat-ok]`
(status==0) / `[bbsh-cat-failed]` (anything else).  Same
shape as `posix_busybox_sh_pipe_prog`; only the embedded
-c string differs.

### `test/kernel/posix_busybox_sh_cat_prog_blob.S`

`.incbin` of the linked ELF with
`__posix_busybox_sh_cat_prog_blob_{start,end}` symbols.

### `test/kernel/ktest_posix_busybox_sh_cat.c`

Drives the program through the existing kernel-test
infrastructure (process_create, elf_load_into_process,
sched_spawn_kthread + drop_to_el0).  Same yield-cap as the
prior busybox tests (32768 ticks).  Asserts only parent-side
liveness (parent reached the marker print, emitted ≥3
debug_writes; nxlibc_write to fd 1/2 still routes through
`NX_SYS_DEBUG_WRITE` for libnxlibc-linked callers).

### `Makefile`

Three sources added to `KTEST_C` / `KTEST_S` / build rules
+ the cleanup list.  Same recipe as the pipe test; only
the embedded -c string differs.

## Outcome — clean, first try

```
posix_busybox_sh_cat_parent_forks_and_execs_busybox_sh_cat [bbsh-cat-parent]hello from initramfs
[bbsh-cat-status=00][bbsh-cat-ok]PASS
```

`hello from initramfs\n` — cat read the full /banner content
through the FILE handle and wrote it to stdout (CONSOLE).
`[bbsh-cat-status=00]` — child exited 0
`[bbsh-cat-ok]` — success-path marker

End-to-end: ash parsed `-c "cat /banner"`, walked PATH,
fstatat found `/bin/cat`, forked, child execve'd
`/bin/cat /banner`, busybox dispatched to the cat applet,
cat opened `/banner` (HANDLE_FILE at slot 3+), looped
`read(fd, buf, N)` against the FILE arm of `sys_read`,
saw EOF when the read returned 0 bytes, wrote each chunk
through the CONSOLE arm of `sys_write` (slot 1 →
`g_nx_console`), closed the file, exited 0; ash returned
that status as its own exit code.

No new kernel-side gaps surfaced.  cat fell off the
sendfile fast-path correctly (the `__NR_sendfile (71)`
ENOSYS noted in slice 7.6d.N.6a), the read-write loop
worked, and FILE → CONSOLE byte-shuffling worked
identically to the established CHANNEL → CONSOLE path
from the pipe slice.

## Verification

`make test` → **416/416 PASSED** (51 python + 275 host + 90
kernel; +1 kernel test from this slice), 0 leaks, 0 errors,
exit 0.

`posix_busybox_sh_cat_parent_forks_and_execs_busybox_sh_cat`
captures `[bbsh-cat-parent]hello from initramfs\n[bbsh-cat-status=00][bbsh-cat-ok]`.

All prior busybox tests stay green (`--help`, `sh exit 42`,
`sh echo hello`, `sh echo a; echo b`, `sh ls /`,
`sh echo hello | cat`).

## Files changed

Production: none.

Test:
- New: `test/kernel/posix_busybox_sh_cat_prog.c` (~60 lines)
- New: `test/kernel/posix_busybox_sh_cat_prog_blob.S` (15
  lines)
- New: `test/kernel/ktest_posix_busybox_sh_cat.c` (~95
  lines)
- Modified: `Makefile` (3 list additions + 1 build rule
  block + 1 cleanup-list entry)

Total: 4 files net (3 new, 1 Makefile edit), ~180 lines.

## What's next (7.6d.N.8 candidates)

The escalation ladder so far:
- single statement, builtin, no I/O (`exit 42`, 7.6d.N.1)
- single statement, builtin, stdio write (`echo hello`, 7.6d.N.2)
- multi-statement sequence (`echo a; echo b`, 7.6d.N.3)
- non-builtin via PATH walk (`ls /` failed, 7.6d.N.4)
- directory listing (`ls /` works, 7.6d.N.5)
- first multi-process pipeline (`echo hello | cat`, 7.6d.N.6b)
- non-builtin reading a real file to stdout (`cat /banner`,
  this slice)

Natural next escalations, each likely to surface a real
kernel gap:

1. **3-stage pipeline.**  `ls / | cat | wc -l` or
   `echo hello | cat | cat`.  Forks 3 processes wired via
   2 pipes.  Slice 7.6d.N.6b only tested 2 stages; chained
   pipes exercise the same machinery but multiply the
   handle-table / process-table pressure.  64-slot caps
   start feeling tight; reap-on-wait deferred since slice
   7.4 may finally need to land.  `wc -l` itself is
   another applet — exercises a different read pattern
   (counts newlines).

2. **Output redirection to a file.**  `echo a > /tmp/foo`
   (or `> /banner.copy`).  Tests:
   - `O_CREAT | O_WRONLY | O_TRUNC` open path (slice 6.3's
     `sys_open` gates O_CREAT, but the ramfs `create` op
     covers it; needs verification end-to-end through ash).
   - ash performing `dup3(fd_of_foo, 1)` before
     `execve("echo", ...)` — exercises FILE-handle dup
     (slice 7.6d.N.6a's `sys_dup3` only special-cased
     CHANNEL semantics; FILE branch is in code but
     unexercised at this layer).
   - Fork inheritance for HANDLE_FILE — slice 7.6a left
     this deferred because vfs_simple's per-open cursor is
     owned by the per-handle object pointer.  Lands here
     if ash forks after `dup3`'ing the file fd.

3. **Input redirection from a file.**  `cat < /banner`.
   Same as `cat /banner` from cat's POV but exercises the
   shell-side redirection plumbing: ash opens /banner,
   dup3's onto fd 0, execs cat with no argv.  Slightly
   different from `cat /banner` because cat reads from
   stdin (whatever-fd-0-is) rather than opening its own
   argv-named file.

4. **Variable expansion + command substitution.**
   `echo $(cat /banner)`.  Exercises ash's `$()` machinery
   — fork+pipe+wait, then re-tokenise the output as
   arguments to the next command.  Stretches the
   pipeline machinery but doesn't surface much new
   kernel-side.

5. **Conditional execution.**  `cat /banner && echo done`
   (`&&`) or `cat /missing || echo failed` (`||`) —
   exercises ash's short-circuit evaluator + observed
   exit-status capture.  Single-process per command.
   Light kernel exercise, more shell-machinery exercise.

**Recommended next:** option 2, **`echo a > /tmp/foo`**
(or `> /tmp.txt` — picking a path that doesn't already
exist in the ramfs).  Most likely to surface a real
kernel gap (FILE-fd inheritance through fork + dup3; ramfs
`O_CREAT` exercised end-to-end; potentially a missing
write-side syscall in the ramfs/vfs layer).  Plus it
unblocks the slice 7.6a deferred FILE-inheritance work.

The 3-stage pipeline (option 1) is also tempting but is
mostly a stress-test of existing infrastructure rather than
a discovery escalation; defer until a third user actually
hits the 64-slot cap.

## Build-side gotcha (still unresolved)

When musl's `syscall_arch.h` / `syscall_cp.s` change,
busybox's incremental build doesn't reliably rebuild against
new musl.  Workaround: `rm -f test/kernel/initramfs.cpio
test/kernel/initramfs_blob.o kernel-test.bin kernel-test.elf
&& make test`.

This session didn't touch musl, so the issue didn't bite.
The two-line fix (`touch $BUSYBOX_BIN` after the inner
`make` in `tools/build-busybox.sh`) is still on the
opportunistic list.

---

**Last Updated:** 2026-04-27
