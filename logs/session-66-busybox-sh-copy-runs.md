# Session 66: slice 7.6d.N.9 — `busybox sh -c "cat /banner > /tmp/copy"` runs end-to-end

**Date:** 2026-04-28
**Phase:** 7 slice 7.6d.N.9 (busybox-as-shell, redirection on an external command)
**Branch:** master

---

## Goals

Slice 7.6d.N.8 (session 65) covered stdout-redirection-to-file in
the **builtin** form: `echo a > /tmp/foo`.  echo runs in-process
in ash, so the redirection setup, the write, and the restore all
happen in the same address space — no exec in the middle.  That
slice landed `NX_SYS_FCNTL`, the `sys_dup3` FILE retain branch,
and the refcounted vfs per-open.

This sub-slice flips the redirection target onto an **external**
command: `cat /banner > /tmp/copy`.  The structural difference is
that ash forks before the redirect setup runs — the FILE handle
the redirection installs at fd 1 has to survive `sys_exec` into
`/bin/cat`.  Concretely:

```
parent ash         fork
child  ash         openat("/tmp/copy", O_CREAT|O_WRONLY|O_TRUNC) -> fdN
child  ash         dup3(fdN, 1, 0)     (FILE retain; slot 0 was CONSOLE)
child  ash         close(fdN)          (FILE refs back to 1)
child  ash         execve("/bin/cat", { "cat", "/banner" })
child  cat         openat("/banner") -> HANDLE_FILE (a different one)
child  cat         read loop -> EOF
child  cat         write(1, ...)       (HANDLE_FILE write into /tmp/copy)
child  cat         exit
parent ash         wait + exit 0
```

Three things this slice cared about going in:

1. **`sys_exec` preserves the handle table.**  Documented in
   slice 7.6d.N.6b's session 63 log; never explicitly verified
   for `HANDLE_FILE` because the only FILE-fd workloads so far
   either ran in the post-fork child (slice 7.6a `xpipe` —
   CHANNEL not FILE) or in-process in ash (7.6d.N.8 echo
   builtin).  This slice is the first to install a FILE handle
   in a process and then exec out of it.
2. **Cross-process FILE-fd inheritance through fork.**  The
   stretch goal called out in HANDOFF.md.  As it turns out, this
   workload doesn't exercise that path — the FILE handle is
   opened in the child *after* fork, so `sys_fork`'s handle-
   table walk sees no FILE entries to dup.  That gap is still
   open; surfacing it requires a workload where the parent has
   a FILE handle open *before* it forks (e.g. ash with input
   redirection running a multi-command pipeline that fans out).
3. **The dup3+retain refcount machinery actually works for
   FILE.**  Slice 7.6d.N.8 tested it inside one process (open,
   dup3, close, write, restore, exit).  This slice tests it
   across an exec boundary — open in process A, dup3 in process
   A, close in process A, exec to process B (same struct,
   different image), write in process B, exit in process B.

Discovery-driven: write the test, observe what surfaces, fix
the gaps that block.

## Implementation

### `test/kernel/posix_busybox_sh_copy_prog.c`

libnxlibc-linked program.  Forks; child does:

```c
nxlibc_execve("/bin/busybox", { "sh", "-c", "cat /banner > /tmp/copy", NULL }, NULL);
```

Parent waits and emits `[bbsh-copy-parent]`,
`[bbsh-copy-status=NN]`, and one of `[bbsh-copy-ok]` (status==0)
/ `[bbsh-copy-failed]` (anything else).  Same shape as
`posix_busybox_sh_cat_prog` from the prior slice; only the
embedded -c string differs.

### `test/kernel/posix_busybox_sh_copy_prog_blob.S`

`.incbin` of the linked ELF with
`__posix_busybox_sh_copy_prog_blob_{start,end}` symbols.

### `test/kernel/ktest_posix_busybox_sh_copy.c`

Drives the program through the same kernel-test plumbing as the
other busybox tests (`nx_process_create` +
`nx_elf_load_into_process` + `sched_spawn_kthread` +
`drop_to_el0`).  After the parent exits, the test re-opens
`/tmp/copy` through `vfs_simple` and KASSERTs the contents are
exactly `hello from initramfs\n` (21 bytes — the full banner).
That assertion is the load-bearing part: it confirms cat actually
wrote the data to the FILE handle inherited through exec, not
that the redirection was silently dropped.

### `Makefile`

Three sources added to `KTEST_C` / `KTEST_S` / build rules + the
cleanup list.  Same recipe as the cat / redir tests; only the
embedded -c string and blob symbol prefix differ.

## Outcome — clean, first try

```
posix_busybox_sh_copy_parent_forks_and_execs_busybox_sh_copy [bbsh-copy-parent][bbsh-copy-status=00][bbsh-copy-ok]PASS
```

`[bbsh-copy-status=00]` — child ash exited 0
`[bbsh-copy-ok]` — success-path marker
The vfs_simple read-back KASSERT fired silently (it's part of the
ktest, not the program-side output); had the file been missing,
short, or wrong, the ktest would have aborted before reaching
PASS.

End-to-end: ash parsed `-c "cat /banner > /tmp/copy"`, walked
PATH for `cat`, fstatat'd `/bin/cat`, forked, the child opened
`/tmp/copy` with O_CREAT|O_WRONLY|O_TRUNC (`sys_openat`), dup3'd
the new FILE handle onto fd 1 (replacing the CONSOLE handle the
child inherited at process_create; the dup3 FILE retain bumped
refs to 2), closed the original FILE fd (refs back to 1),
execve'd `/bin/cat /banner`.  `sys_exec` swapped the address
space + ELF image but left the handle table alone, so cat ran
into a process where slot 0 was the FILE pointing at /tmp/copy
and slots 1/2 were still the inherited CONSOLE entries.  cat
opened /banner (HANDLE_FILE at slot 3+), read it, wrote it to
fd 1 (FILE arm of `sys_write` — slice 6.3), saw EOF on read,
closed /banner, exited 0.  ash returned that status as its own.

No new kernel-side gaps surfaced.  The dup3+retain machinery
from slice 7.6d.N.8 carried straight across the exec boundary
without modification.  `sys_exec`'s "doesn't touch the handle
table" semantic worked as advertised: the per-open struct in
kheap and the underlying ramfs file in the ramfs static array
are both stable across the address-space swap, so the handle
slot keeps its `(type, rights, object)` triple meaningful.

## Verification

`make test` → **418/418 PASSED** (51 python + 275 host + 92
kernel; +1 kernel test from this slice), 0 leaks, 0 errors,
exit 0.

`posix_busybox_sh_copy_parent_forks_and_execs_busybox_sh_copy`
captures `[bbsh-copy-parent][bbsh-copy-status=00][bbsh-copy-ok]`
and the post-parent vfs_simple read-back of `/tmp/copy` returns
exactly `hello from initramfs\n`.

All prior busybox tests stay green (`--help`, `sh exit 42`, `sh
echo hello`, `sh echo a; echo b`, `sh ls /`, `sh echo hello |
cat`, `sh cat /banner`, `sh echo a > /tmp/foo`).

Defensive `rm -f test/kernel/initramfs.cpio` before the run
(per session 65's lessons) — busybox source itself wasn't
touched this slice, so the cpio rebuild was likely
unnecessary, but the cost is one initramfs repack on the
first run.

## Files changed

Production: none.

Test:
- New: `test/kernel/posix_busybox_sh_copy_prog.c` (~70 lines)
- New: `test/kernel/posix_busybox_sh_copy_prog_blob.S` (15 lines)
- New: `test/kernel/ktest_posix_busybox_sh_copy.c` (~115 lines)
- Modified: `Makefile` (3 list additions + 1 build rule block +
  1 cleanup-list entry)

Total: 4 files net (3 new, 1 Makefile edit), ~210 lines.

## What this slice did and didn't close

**Closed:** the question of whether `sys_exec` preserves a
HANDLE_FILE entry across the address-space swap.  Yes — handles
point at kheap-allocated objects (ramfs_open structs) that aren't
tied to user memory, and `sys_exec` deliberately doesn't reset
the table (per slice 7.6d.N.6b's session 63 design note).  The
refcounted vfs per-open from slice 7.6d.N.8 carries through
without further changes.

**Not closed:** cross-process FILE-fd inheritance through *fork*
itself.  `sys_fork`'s handle-table walk still skips FILE entries
(`framework/syscall.c:802`, comment "HANDLE_FILE is intentionally
NOT duplicated (per-cursor state)").  This workload doesn't
exercise that path because ash opens the redirect file in the
child *after* fork.  The original HANDOFF.md prediction —
"forces fork to dup-and-retain a FILE handle into the child's
table" — was structurally wrong; for stdout-redirection on an
external command, the open lives on the child's side of the fork.

A workload that *does* force fork-time FILE inheritance would be
something like `cat /banner | wc` where ash opens the input FILE
in the parent (less common) or a multi-stage pipeline whose
outer shell holds a redirected fd open across multiple forks.
ash's "fork + redirect-in-child + exec" model rarely produces
this shape; busybox-driven workloads can sometimes hit it via
`>>` append on a long-running shell loop.  Worth deferring until
a real workload forces the issue, since the design fix (a `dup`
op on the vfs driver that clones the per-open with its own
cursor) is non-trivial — vfs_simple's current per-open struct
holds a single cursor that two slots would race over.

## What's next (7.6d.N.10+ — slice bundling decision)

Escalation ladder so far:

| # | Workload | What it added |
|---|---|---|
| 7.6d.N.1 | `exit 42` | `mmap` |
| 7.6d.N.2 | `echo hello` | (none, capacity bump) |
| 7.6d.N.3 | `echo a; echo b` | (none) |
| 7.6d.N.4 | `ls /` (failed) | `fstatat` |
| 7.6d.N.5 | `ls /` (works) | `getdents64`, `openat`, HANDLE_DIR |
| 7.6d.N.6b | `echo hello \| cat` | CONSOLE, slot 0/1/2 reservation, channel EOF, `waitpid(-1)` |
| 7.6d.N.7 | `cat /banner` | (none — test only) |
| 7.6d.N.8 | `echo a > /tmp/foo` | `fcntl`, `dup3` FILE retain, vfs `retain` op |
| 7.6d.N.9 | `cat /banner > /tmp/copy` | (none — test only) |

Several of the natural next escalations are structurally similar
to N.7/N.9 (test-only, composing already-tested mechanisms),
which means writing them one-per-slice doubles the doc/commit
overhead without surfacing useful information.  Bundle the
"likely clean" ones into a single slice; keep the one
"likely surfaces a gap" workload as its own slice.

### 7.6d.N.10 (NEXT) — Bundled FILE-on-stdin workloads

**Audit-trimmed list.**  Initial draft of this slice had five
candidate workloads (input redirection + `&&` + `||` + `$()` +
3-stage pipe).  Three of those — `cat /banner && echo done`,
`cat /missing || echo failed`, `ls / | wc -l` — turned out on
audit to test ash's parser/dispatcher and not the kernel.  The
discovery-driven sliced workflow trades doc/commit overhead for
*kernel discovery*; tests that exercise no new kernel paths
spend that overhead without buying anything.  See **Lessons** for
the audit reasoning.

The two that survived are kept because each exercises a *new
kernel composition*:

1. **`cat < /banner` — input redirection.**  ash forks, child
   opens `/banner` with O_RDONLY, dup3s onto fd 0 (replacing the
   inherited STDIN CONSOLE), execs cat with no path argument;
   cat reads from stdin.  **New kernel-side composition:** a
   FILE handle at slot 2 (POSIX STDIN_FILENO; `dup3(_, 0, 0)`'s
   newfd=0 special-case routes there because encoded fd 0
   collides with NX_HANDLE_INVALID).  First time a FILE replaces
   the inherited STDIN CONSOLE, and first time cat reads through
   the FILE arm of `sys_read` via fd 0 (slot 2) rather than
   fd 3+.  Slice 7.6d.N.6b's slot-2-redirect logic was tested
   with a CHANNEL replacement (`echo hello | cat`); this
   exercises the FILE replacement.

2. **`echo $(cat /banner)` — command substitution.**  ash forks
   + opens a pipe + the child execs `cat /banner` with stdout
   redirected to the pipe write end + the *parent shell* reads
   from the pipe read end + tokenises the output as arguments
   to the outer echo + dispatches the echo builtin.  **New
   kernel-side composition:** the pipe consumer is the parent
   (a non-exec'd shell process), inverse of N.6b where the
   consumer was an exec'd cat.  Validates `nx_channel_recv` +
   the yield-loop-on-NX_EAGAIN POSIX-blocking behavior from a
   different process role.

Implementation pattern: write both test programs + blobs +
ktests + Makefile rules in one pass (~300 lines of test
infrastructure), run `make test`, fix whichever surfaces a real
gap, rinse + repeat.  If either needs real kernel work, split
into 7.6d.N.10a/10b.

### Skipped (no new kernel surface)

These tests *would* validate that the existing kernel composition
holds, but they don't surface kernel-side anything to study.
Reconsider in Phase 9 (test & benchmarks) when regression
coverage becomes the goal.

- **`cat /banner && echo done` — `&&` short-circuit.**  Tests
  ash's exit-status capture + conditional dispatcher.  Same
  kernel paths as N.7.
- **`cat /missing || echo failed` — `||` short-circuit.**
  Symmetric to above; sys_openat's ENOENT propagation was
  already exercised by N.4's PATH walk.
- **`ls / | wc -l` — 3-stage pipeline.**  Pure scale-up of
  N.6b's 2-stage pipe.  Adds the wc cpio duplicate but no new
  kernel paths.  Pressure on the 64-slot v1 caps is a v1-hack
  concern, not a kernel-correctness one — only worth doing if
  a real workload tips the caps and forces the deferred
  reap-on-wait fix.

### 7.6d.N.11 — `echo a >> /tmp/foo` (append-redirect)

Single-workload slice, kept separate because it's the first
7.6d.N escalation since N.6b that's expected to need a real
kernel/vfs change.  ash opens with `O_WRONLY | O_APPEND`; we
don't translate `O_APPEND` (vfs only knows `NX_VFS_OPEN_{READ,
WRITE,CREATE}` per `interfaces/{fs,vfs}.h`).  Likely surfaces
either a flag-translation gap in `sys_openat` (cleanly add
`NX_VFS_OPEN_APPEND` with one bit in the mask + one switch arm
in `vfs_simple` + matching ramfs implementation that seeks-to-
end before each write) or a missing append-write semantic in
ramfs.  Likely ~30 lines of kernel + ~30 lines of ramfs + 1 vfs
`seek` invocation.

### 7.6d.N.final — interactive `busybox sh` over UART RX

The final form of slice 7.6d.  Needs RX IRQ + per-process line
buffer + `sigaction` for SIGINT + `ioctl(TIOCGWINSZ)` stub +
`tcsetattr` stub.  Big slice — defer until the discovery ladder
runs out of meaningful single-command escalations (i.e., until
the bundled 7.6d.N.10 + 7.6d.N.11 land cleanly).

## Lessons / Notes

- **The "deferred FILE inheritance through fork" gap doesn't
  block real workloads as much as expected.**  The HANDOFF.md
  framing implied that an external-command redirect would
  force the issue; in practice ash's fork-then-redirect-in-
  child idiom moves the FILE-handle creation onto the child
  side, where no fork inheritance is needed.  The gap stands
  but isn't on the critical path for slice 7.6d.

- **Mirror tests are still useful.**  Slices 7.6d.N.7 and
  7.6d.N.9 are both test-only — no production code changed.
  They exist because each is a minimal proof that two
  previously-tested mechanisms (FILE arm of sys_read +
  CONSOLE arm of sys_write; FILE retain in dup3 + sys_exec
  preservation) actually compose end-to-end.  Skipping them
  on the grounds that "the parts work, so the whole must
  work" leaves silent regressions hidden until a much larger
  workload trips them — which is then much harder to pin to
  the right slice.

- **The ramfs read-back assertion is the load-bearing
  check.**  Without it, the test would still pass on a
  silently-broken redirect (cat writes to CONSOLE, /tmp/copy
  stays empty / nonexistent, parent exits 0).  Anyone
  copy-pasting the test recipe for a future redirection
  slice should keep that pattern: `vops->open` +
  `vops->read` + KASSERT on the bytes, not just on the
  child's exit code.

- **Audit candidate workloads for kernel surface before
  bundling them into a slice.**  The first draft of slice
  7.6d.N.10 bundled five "simple" workloads (input redirect,
  `&&`, `||`, `$()`, 3-stage pipe).  Audit revealed that
  three of them — `&&`, `||`, `ls | wc -l` — exercise zero
  new kernel paths; they're entirely tests of busybox's ash
  parser and dispatcher.  The discovery-driven sliced
  workflow buys *kernel discovery* with doc/commit overhead;
  spending that overhead on tests that won't surface kernel
  behavior is a bad trade.  Trim to workloads that exercise
  a *new kernel composition* (a path or combination not yet
  tested), and defer pure busybox-machinery tests to a
  Phase-9 regression sweep.  Concrete heuristic: for each
  candidate, ask "what kernel codepath does this exercise
  that wasn't already exercised by an earlier slice?"  If
  the answer is "none" or "same as slice X", drop it from
  the discovery slice.  N.10 went from 5 candidates to 2
  after applying this — only `cat < /banner` (FILE-as-stdin
  via dup3 special-case) and `echo $(cat /banner)` (parent
  shell reads from pipe, inverse of N.6b) survived.

---

**Last Updated:** 2026-04-28
