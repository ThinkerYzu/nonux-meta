# Session 67: slice 7.6d.N.10 — bundled FILE-on-stdin workloads run end-to-end

**Date:** 2026-04-28
**Phase:** 7 slice 7.6d.N.10 (busybox-as-shell, FILE-on-stdin + command substitution)
**Branch:** master

---

## Goals

Slice 7.6d.N.9 (session 66) closed stdout-redirection-to-file
against an external command (`cat /banner > /tmp/copy`).  That
slice exercised a HANDLE_FILE at **slot 0** (encoded fd 1) — the
post-fork child opened the redirect file, dup3'd it onto fd 1,
and execve'd cat into the resulting layout.

This slice bundles the two natural next escalations that *each*
exercise a new kernel composition (rather than just rearranging
ash machinery — see session 66's bundling audit and lessons for
why we trimmed `&&` / `||` / 3-stage pipe out):

1. **`cat < /banner` — input redirection (workload a).**  First
   FILE handle at **slot 2** (encoded fd 0 = POSIX STDIN_FILENO).
   ash dup3s an O_RDONLY FILE handle onto fd 0 in the post-fork
   child, then execs cat with no path argument; cat reads from
   stdin via the FILE arm of `sys_read` through the slot-2
   redirection (slice 7.6d.N.6b's slot-2-routing was previously
   only tested with a CHANNEL replacement in `echo hello | cat`).
2. **`echo $(cat /banner)` — command substitution (workload b).**
   First time a non-exec'd parent shell process is the pipe
   consumer.  ash sets up a pipe, forks once for the inner cat,
   the child execs cat with stdout redirected to the pipe write
   end; the parent ash drains the pipe via `sys_read`'s CHANNEL
   arm, tokenises, then dispatches the outer echo builtin.
   Inverse of slice 7.6d.N.6b where the pipe consumer was an
   exec'd cat — validates the `NX_EAGAIN`-yield-loop +
   peer-closed-EOF machinery from a different process role.

Discovery-driven, but with the explicit prediction (per HANDOFF
N.10's plan) that both *should* compose cleanly from already-
tested mechanisms.  Plan: write both, run, split into N.10a/10b
only if either surfaces a real kernel gap.

## Implementation

### `test/kernel/posix_busybox_sh_stdin_prog.c` (workload a)

libnxlibc-linked program.  Forks; child does:

```c
nxlibc_execve("/bin/busybox", { "sh", "-c", "cat < /banner", NULL }, NULL);
```

Parent emits `[bbsh-stdin-parent]`, `[bbsh-stdin-status=NN]`,
and one of `[bbsh-stdin-ok]` / `[bbsh-stdin-failed]`.  Same shape
as `posix_busybox_sh_cat_prog` (slice N.7); only the embedded -c
string differs.

### `test/kernel/posix_busybox_sh_cmdsub_prog.c` (workload b)

Same shape; embedded -c string is `echo $(cat /banner)`.  Marker
prefixes are `[bbsh-cmdsub-…]`.

### `test/kernel/posix_busybox_sh_{stdin,cmdsub}_prog_blob.S`

`.incbin` of the linked ELF with
`__posix_busybox_sh_{stdin,cmdsub}_prog_blob_{start,end}` symbols.

### `test/kernel/ktest_posix_busybox_sh_{stdin,cmdsub}.c`

Drives each program through the standard kernel-test plumbing
(`nx_process_create` + `nx_elf_load_into_process` +
`sched_spawn_kthread` + `drop_to_el0`).  Asserts parent exits 0
and emits ≥3 markers.  No vfs read-back assertion — both
workloads write to CONSOLE, not to a file, so the live UART log
between markers is the load-bearing observation:

- workload a: `hello from initramfs\n` between
  `[bbsh-stdin-parent]` and `[bbsh-stdin-status=00]`
- workload b: `hello from initramfs\n` between
  `[bbsh-cmdsub-parent]` and `[bbsh-cmdsub-status=00]`

### `Makefile`

Three list additions per workload (KTEST_C, KTEST_S, cleanup
list) + one build-rule block per workload (compile .c, link .elf
with libnxlibc, blob.S depends on .elf).  Same recipe as
N.7/N.8/N.9; only the prefix differs.

## Outcome — clean, first try (both workloads)

```
posix_busybox_sh_stdin_parent_forks_and_execs_busybox_sh_stdin [bbsh-stdin-parent]hello from initramfs
[bbsh-stdin-status=00][bbsh-stdin-ok]PASS
posix_busybox_sh_cmdsub_parent_forks_and_execs_busybox_sh_cmdsub [bbsh-cmdsub-parent]hello from initramfs
[bbsh-cmdsub-status=00][bbsh-cmdsub-ok]PASS
```

Both return status 0 with the banner text on stdout.  No new
kernel-side gaps surfaced; no need to split into N.10a/10b.
The plan's prediction held: each workload composes cleanly from
already-tested kernel mechanisms.

End-to-end traces (inferred from prior-slice mechanics; not
re-instrumented this session):

**Workload a (`cat < /banner`).**  ash parsed `cat < /banner`,
walked PATH for cat, fstatat'd `/bin/cat`, forked; child opened
`/banner` with O_RDONLY (`sys_openat` → HANDLE_FILE at some
slot N), dup3'd that handle onto fd 0 (newfd=0 → slot 2 per
slice N.6b's POSIX-STDIN-FILENO routing; FILE retain bumped refs
to 2; the inherited STDIN CONSOLE was overwritten), closed slot
N (FILE refs back to 1), execve'd `/bin/cat`.  `sys_exec`
swapped address space + ELF image without touching the handle
table.  cat ran with no argv path arg → read loop on fd 0 →
`sys_read` resolved fd 0 to slot 2 → HANDLE_FILE arm fired →
read /banner contents → wrote bytes to fd 1 (CONSOLE) → saw EOF
on read → exited 0.  ash returned that status.

**Workload b (`echo $(cat /banner)`).**  ash parsed the
substitution, allocated a pipe (`sys_pipe` → CHANNEL endpoints
at slots 3/4), forked; child closed the read end, dup3'd the
write end onto fd 1, closed both originals, execve'd
`/bin/cat /banner`.  cat read /banner, wrote to fd 1 (CHANNEL
arm of `sys_write`), exited 0; the channel saw both endpoints
close on the writer side, queueing peer-closed → EOF for the
reader.  The parent ash closed its copy of the write end,
read-looped on the read end (`sys_read` CHANNEL arm wraps
`nx_channel_recv` in the `NX_EAGAIN`-yield-loop until peer-EOF
returns 0 — slice N.6b machinery), tokenised the captured bytes
(`hello from initramfs` with the trailing newline collapsed by
ash's word-splitting), dispatched `echo hello from initramfs`
which wrote `hello from initramfs\n` to fd 1 (CONSOLE), exited 0.

Both runs exercise the kernel paths claimed in N.10's plan:
workload a is the first FILE-at-slot-2 install in any test;
workload b is the first non-exec'd-parent-as-pipe-consumer in
any test.

## Verification

`make test` → **420/420 PASSED** (51 python + 275 host + 94
kernel; +2 kernel tests from this slice), 0 leaks, 0 errors,
exit 0.

Both new ktests pass on first run with no kernel changes.  All
prior busybox tests stay green (`--help`, `sh exit 42`,
`sh echo hello`, `sh echo a; echo b`, `sh ls /`,
`sh echo hello | cat`, `sh cat /banner`, `sh echo a > /tmp/foo`,
`sh cat /banner > /tmp/copy`).

## Files changed

Production: none.

Test:
- New: `test/kernel/posix_busybox_sh_stdin_prog.c` (~70 lines)
- New: `test/kernel/posix_busybox_sh_stdin_prog_blob.S` (15 lines)
- New: `test/kernel/ktest_posix_busybox_sh_stdin.c` (~100 lines)
- New: `test/kernel/posix_busybox_sh_cmdsub_prog.c` (~85 lines)
- New: `test/kernel/posix_busybox_sh_cmdsub_prog_blob.S` (15 lines)
- New: `test/kernel/ktest_posix_busybox_sh_cmdsub.c` (~100 lines)
- Modified: `Makefile` (3 list additions + 2 build-rule blocks +
  2 cleanup-list entries)

Total: 7 files net (6 new, 1 Makefile edit), ~390 lines.

## What this slice did and didn't close

**Closed:**
- FILE-handle install at slot 2 (POSIX STDIN_FILENO) via the
  `dup3(_, 0, 0)` newfd=0 special-case.  Previously only
  exercised with a CHANNEL replacement in slice 7.6d.N.6b.
- `sys_exec`'s "preserve handle table" semantic for a FILE handle
  at slot 2 (slice N.9 covered slot 0 / fd 1).
- `sys_read` FILE arm reached via fd 0 / slot 2.
- `sys_read` CHANNEL arm + `NX_EAGAIN`-yield-loop +
  peer-closed-EOF reached from a non-exec'd parent shell process
  (slice N.6b had an exec'd cat as the consumer).
- The fork-time CHANNEL inheritance from slice 7.6a (where the
  CHANNEL endpoint that the parent reads from must survive the
  fork that creates the child writer) composes with the pipe
  setup in command substitution.

**Not closed (still deferred):**
- Cross-process FILE-fd inheritance through fork (the original
  7.6a residual).  Workload a opens the FILE in the child after
  fork; workload b doesn't involve a FILE on the pipe path.
  Surfacing this still needs a workload where the parent holds
  a FILE handle open *before* forking — ash's dominant idiom
  ("fork + redirect-in-child + exec") doesn't naturally produce
  that shape.

## Lessons

### Bundling worked exactly as planned.

Two workloads in one slice, no kernel gaps, zero need to split.
Doc/commit overhead is a single session log + a single HANDOFF
delta.  Had either workload surfaced a gap, the contract was to
split into N.10a/10b — but the audit (session 66) correctly
identified both as "compose already-tested mechanisms in a new
shape."  This is the right shape for bundling: workloads that
each test a *distinct* new composition but where each
composition's component pieces have already been validated
individually.

### The audit was right to trim.

Recall the original draft of N.10 had five workloads:
`cat < /banner`, `cat /banner && echo done`,
`cat /missing || echo failed`, `echo $(cat /banner)`,
`ls / | wc -l`.  Session 66's audit cut three to keep only the
two with new kernel surface.  This slice would have spent ~150
extra test-infrastructure lines + 3 extra entries in HANDOFF on
tests that all return clean PASS but exercise no new kernel
behavior.  Bundling and audit-trimming together kept the slice
tight.

### Discovery-driven workflow has two modes.

The escalation-ladder pattern (N.6a → N.6b, N.8 → N.9, N.10 →
…) interleaves *discovery* slices (likely surfaces a kernel
gap; size of the patch is unpredictable; expect one ktest +
production code) with *validation* slices (likely composes
clean; size is predictable; expect one or more ktests + zero
production code).  Naming them differently or numbering them
distinctly might be worth thinking about.  For now the
ballot-box-status field carries that information implicitly
(a slice with `(none — test only)` in the IMPL-GUIDE column is
a validation slice).

## What's next

### 7.6d.N.11 — Append-redirect: `echo a >> /tmp/foo`

Per HANDOFF N.10's "next" pointer: ash opens the redirect file
with `O_WRONLY | O_APPEND`; we don't translate `O_APPEND` (vfs
only knows `NX_VFS_OPEN_{READ,WRITE,CREATE}`).  Likely surfaces
either a flag-translation gap in `sys_openat` (cleanly add
`NX_VFS_OPEN_APPEND` + a switch arm in vfs_simple + matching
ramfs implementation that seeks-to-end before each write) or a
missing append-write semantic in ramfs (write_at vs
write_appending).  First slice since N.6b that's expected to
need real kernel/vfs change.  Single workload, kept separate
from N.10 because it's a *discovery* slice rather than a
*validation* slice.

### Eventually: 7.6d.N.final — interactive `/init = busybox sh`

Once non-interactive `sh -c` of progressively richer commands
runs cleanly (we're a few escalations away), switch the
production-side `/init` to busybox and start the interactive
flow.  Needs: UART RX IRQ, per-process line buffer, `sigaction`
(ash always calls it during interactive startup), `ioctl(0,
TIOCGWINSZ)` (terminal size — can stub -ENOTTY), `tcsetattr`
stub.  Closes slice 7.6d.

## References

- [HANDOFF.md §Current Status](../HANDOFF.md#current-status)
- [HANDOFF.md §Next Actions](../HANDOFF.md#next-actions)
- [Session 66](session-66-busybox-sh-copy-runs.md) — slice 7.6d.N.9 (the trimming audit + bundling rationale)
- [Session 64](session-64-busybox-sh-cat-runs.md) — slice 7.6d.N.7 (the test pattern for FILE-input-via-path)
- [Session 63](session-63-busybox-sh-pipe-runs.md) — slice 7.6d.N.6b (CHANNEL machinery + slot 0/1/2 reservation + pipe EOF)
