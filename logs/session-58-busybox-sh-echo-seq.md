# Session 58: slice 7.6d.N.3 — `busybox sh -c "echo a; echo b"` runs end-to-end

**Date:** 2026-04-27
**Phase:** 7 slice 7.6d.N.3 (busybox-as-shell, multi-statement script)
**Branch:** master

---

## Goals

Slice 7.6d.N.2 (session 57) made `busybox sh -c "echo hello"`
work end-to-end via the `echo` builtin: ash parses the -c
string, dispatches `echo`, writes `hello\n` via musl stdio
(`__stdio_write` → SYS_writev → magic-fd-handle → UART), runs
end-of-input shutdown, exits 0.  Single-statement script.

This sub-slice escalates one rung further to the smallest
MULTI-statement script: `echo a; echo b`.  Differences from
the single-statement path:

1. ash's parser has to build a list (or sequence) node with
   two child commands instead of a single command tree.
   Pulls more from mallocng — exercises whatever ash code
   path handles the `;` separator.
2. The shell's main eval loop has to actually iterate across
   statements: dispatch `echo a`, return, see the next
   statement, dispatch `echo b`, then end-of-input.  Slice
   7.6d.N.2 only ran one statement.
3. Two consecutive stdio writes from the shell (one per
   `echo`).  Confirms stdio buffering across calls.
4. Total output is `a\nb\n` instead of `hello\n` — six
   bytes split across two writev calls (or possibly
   coalesced by stdio buffering into one).

Discovery-driven, just like the prior 7.6d.N.x sub-slices.
Write the test, observe, plan the next sub-slice from what
surfaces.

## Implementation

### `test/kernel/posix_busybox_sh_echo_seq_prog.c`

libnxlibc-linked program.  Forks; child does:
```c
nxlibc_execve("/bin/busybox", { "sh", "-c", "echo a; echo b", NULL }, NULL);
```
Parent waits + emits `[bbsh-seq-parent]`, `[bbsh-seq-status=NN]`,
and one of `[bbsh-seq-ok]` (status==0) / `[bbsh-seq-failed]`
(anything else).  Same shape as `posix_busybox_sh_echo_prog`;
only the embedded -c string differs.

### `test/kernel/posix_busybox_sh_echo_seq_prog_blob.S`

`.incbin` of the linked ELF with
`__posix_busybox_sh_echo_seq_prog_blob_{start,end}` symbols.

### `test/kernel/ktest_posix_busybox_sh_echo_seq.c`

Drives the program through the existing kernel-test
infrastructure (process_create, elf_load_into_process,
sched_spawn_kthread + drop_to_el0).  Same yield-cap as the
prior busybox tests (32768 ticks).  Asserts only parent-side
liveness (parent reached the marker print, emitted ≥3
debug_writes).

### `Makefile`

Three sources added to `KTEST_C` / `KTEST_S` / build rules
+ the cleanup list.  Same recipe as `posix_busybox_sh_echo_prog`;
only the embedded -c string differs.

## Outcome — clean, first try

```
posix_busybox_sh_echo_seq_parent_forks_and_execs_busybox_sh_echo_seq [bbsh-seq-parent]a
b
[bbsh-seq-status=00][bbsh-seq-ok]PASS
```

`a\nb\n` — emitted by ash's two `echo` builtin invocations
`[bbsh-seq-status=00]` — child exited 0
`[bbsh-seq-ok]` — success-path marker

End-to-end: ash parsed `-c "echo a; echo b"` (sequence node
with two echo statements), iterated the sequence dispatching
each `echo` builtin, each wrote its argument + newline via
musl's stdio, ash ran end-of-input shutdown, exited cleanly
with status 0.  No new kernel-side gaps — no cap exhaustion
this time (after slice 7.6d.N.2's bump to 64), no new
syscalls reached, no faults.

The session-57 plan listed three failure-mode candidates for
this sub-slice (parser-path syscalls, stdio flush ordering
surprises, mallocng exhaustion).  None surfaced.  ash's
sequence parser + statement dispatch are evidently inside
the working set established by slice 7.6d.N.2.

## Verification

`make test` → **413/413 PASSED** (51 python + 275 host + 87
kernel; +1 kernel test from this slice), 0 leaks, 0 errors,
exit 0.

`posix_busybox_sh_echo_seq_parent_forks_and_execs_busybox_sh_echo_seq`
captures `[bbsh-seq-parent]a\nb\n[bbsh-seq-status=00][bbsh-seq-ok]`.

All prior busybox tests stay green (`--help`, `sh exit 42`,
`sh echo hello`).

## Files changed

Production: none.

Test:
- New: `test/kernel/posix_busybox_sh_echo_seq_prog.c` (~95
  lines)
- New: `test/kernel/posix_busybox_sh_echo_seq_prog_blob.S`
  (15 lines)
- New: `test/kernel/ktest_posix_busybox_sh_echo_seq.c` (~100
  lines)
- Modified: `Makefile` (3 list additions + 1 build rule
  block + 1 cleanup-list entry)

Total: 4 files net (3 new, 1 Makefile edit), ~215 lines.

## What's next (7.6d.N.4 candidates)

Builtins-only escalation has now exercised:
- single-statement script with no I/O (`exit 42`, slice 7.6d.N.1)
- single-statement script with one stdio write (`echo hello`, slice 7.6d.N.2)
- multi-statement sequence with two stdio writes (`echo a; echo b`, this slice)

Natural next escalation: **`busybox sh -c "ls /"`** — first
NON-builtin.  ash parses `ls` as an external command (not a
builtin in the standard sense; it IS a busybox applet but
ash dispatches via fork+exec lookup, not in-process applet
dispatch).  Will surface multiple gaps at once:

1. **PATH lookup.**  ash's default PATH is
   `/sbin:/usr/sbin:/bin:/usr/bin`.  Our initramfs has only
   `/bin/busybox`, `/init`, `/banner` — no `/bin/ls`.  Three
   ways to make it work:
   - Add `/bin/ls -> /bin/busybox` symlink to initramfs.
     Requires `tools/pack-initramfs.py` to learn cpio symlink
     entries (currently emits regular files only) AND ash's
     execve must follow symlinks (probably yes — Linux
     execve does, our `sys_exec` would need to as well).
   - Add a duplicate copy of `/bin/busybox` at `/bin/ls`.
     1.22 MB extra in initramfs — wasteful but no kernel
     changes needed.
   - Make `tools/pack-initramfs.py` busybox-aware: emit
     applet-name files that are all the same busybox blob,
     deduplicating at cpio-pack time only.  (`pack-initramfs.py`
     gains a `--busybox-applets` flag.)

   Lowest-friction discovery path: pick the duplicate-copy
   approach for first attempt; surface other gaps before
   investing in symlink machinery.

2. **fork-then-exec from ash.**  Different code path from
   our test parents' fork-then-exec — ash's fork is part of
   its own job-management code, may pull in different
   syscalls (sigprocmask around the fork to mask SIGCHLD?
   tcsetpgrp for foreground process group?).

3. **process-table churn.**  ash forks a child for `ls`,
   then waits on it.  After `ls` exits, ash continues +
   exits.  Our `sched_rr_purge_user_tasks` still doesn't
   reap, so this leaks the `ls` process state on top of the
   ash-parent + ash-child + test-parent leaks.  Cumulative
   leak per `sh -c "ls /"` test ≈ 3 processes × 8 MiB =
   24 MiB.  64-slot cap can absorb this for now but the
   waterline is rising fast.

4. **`ls` itself.**  Once the applet runs, it needs:
   - `opendir / readdir` — we have `NX_SYS_READDIR` since
     slice 6.4.  Translation layer would need `__NR_getdents64`
     mapped (currently -ENOSYS).
   - `stat / fstatat` — completely unmapped.  `ls` calls
     stat on every entry to get type/perms.  Without it `ls`
     might either bail early (unlikely — it usually degrades
     gracefully) or print unstyled names only.

That's a multi-discovery sub-slice and worth its own arc.

Alternative incremental step: **`busybox sh -c "true && echo
ok"`** (logical-and operator).  Still all builtins (`true` +
`echo`), still single-process, but exercises ash's
short-circuit evaluator.  Even smaller jump than this slice,
arguably too small to warrant a sub-slice.  Skip.

Another option: **`busybox sh -c "echo $?"`** (variable
expansion + `$?` magic variable).  Exercises ash's parameter
expansion and the exit-status capture machinery (must
remember the previous command's status).  Single-process,
single statement.  Mildly interesting but doesn't move us
toward the slice-7.6d.N.final goal of an interactive shell.

**Recommended next:** `busybox sh -c "ls /"` (option above) —
the first sub-slice that exercises fork-into-applet and
external-command lookup.  Discovery-driven; expect to capture
a failure and add scaffolding accordingly.

## Build-side gotcha (still unresolved)

When musl's `syscall_arch.h` / `syscall_cp.s` change,
busybox's incremental build doesn't reliably rebuild against
new musl.  Same workaround as sessions 56 + 57: `rm -f
test/kernel/initramfs.cpio test/kernel/initramfs_blob.o
kernel-test.bin kernel-test.elf && make test`.

This session didn't touch musl, so the issue didn't bite.
The two-line fix (`touch $BUSYBOX_BIN` after the inner
`make` in `tools/build-busybox.sh`) is still on the
opportunistic list.

---

**Last Updated:** 2026-04-27
