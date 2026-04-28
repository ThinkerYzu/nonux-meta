# Session 69: slices 7.6d.N.12 through 7.6d.N.15 — sigaction/procmask + tolerable stubs + 3-stage pipe + cross-process FILE-fd

**Date:** 2026-04-28
**Phase:** 7 slices 7.6d.N.12 / N.13 / N.14 / N.15 (four slices in one session)
**Branch:** master

---

## Goals

User asked the assistant to push through the queued sub-slices of
slice 7.6d.N up to (and ideally including) 7.6d.N.final, stopping
only when blocked.  Four slices closed in this session, each with
production code + a busybox-`sh` ktest that exercises a real
kernel composition gap (or, in N.13's case, a sweep of trivial
syscall stubs that ash needs to not bail at startup).  N.16 was
then declared skippable (no current workload demands strict
POSIX-fd ordering, and N.15's investigation showed the originally-
scoped 30-line paydown is incomplete — see "N.16 deferral" below).

## Slice 7.6d.N.12 — sigaction + rt_sigprocmask stubs

Workload: `busybox sh -c "trap 'echo bye' EXIT; echo body"`.
Output (live in QEMU log):

```
body
bye
```

ash's `setsignal` walks its trap table during startup and,
depending on the trap target, calls `sigprocmask` (to learn the
inherited mask) and `sigaction` (to install the kernel-side
handler).  Without these returning success ash bails before
parsing the script body.  The EXIT pseudo-signal trap fires
inside ash on the implicit-exit code path — no kernel-→user
trampoline needed for this workload (that lives in slice
7.6d.N.final).

Production:

- `framework/syscall.h` — `NX_SYS_RT_SIGACTION = 27`,
  `NX_SYS_RT_SIGPROCMASK = 28`, with a frank docstring
  acknowledging both are no-op stubs.
- `framework/syscall.c` — two one-line stub bodies returning
  `NX_OK`, plus the matching dispatch-table entries.
- `third_party/musl/arch/aarch64/syscall_arch.h` —
  `__NR_rt_sigaction (134) → 27`, `__NR_rt_sigprocmask (135) → 28`.
- `third_party/musl/src/thread/aarch64/syscall_cp.s` — same two
  entries on the asm-level cancellation-point path.

Test:

- `posix_busybox_sh_trap_prog.{c,_blob.S}` (libnxlibc-linked EL0
  parent forks + execs busybox sh).
- `ktest_posix_busybox_sh_trap.c` asserts parent exit_code == 0.

## Slice 7.6d.N.13 — Tolerable-syscall stubs sweep

Workload: `busybox sh -c "id; uname -a"`.
Output:

```
uid=0 gid=0
nonux nonux 0.1 v1 aarch64 GNU/Linux
```

(busybox's `id` also prints `id: can't get groups` because we
don't have `getgroups`; harmless for this slice — `id` still
exits 0.)

Ten new syscalls, every body a one-liner:

| NX_SYS_*              | Linux # | Body                                         |
|-----------------------|---------|----------------------------------------------|
| `GETUID = 29`         | 174     | `return 0;`                                  |
| `GETEUID = 30`        | 175     | `return 0;`                                  |
| `GETGID = 31`         | 176     | `return 0;`                                  |
| `GETEGID = 32`        | 177     | `return 0;`                                  |
| `SETUID = 33`         | 146     | `return NX_OK;` (no uid concept to set)      |
| `SETGID = 34`         | 144     | `return NX_OK;`                              |
| `GETPID = 35`         | 172     | `return nx_process_current()->pid;`          |
| `GETPPID = 36`        | 173     | `return nx_process_current()->parent_pid;`   |
| `UNAME = 37`          | 160     | `copy_to_user` a 6×65-byte hardcoded struct  |
| `SET_TID_ADDRESS = 38`| 96      | `return nx_process_current()->pid;` (musl)   |

`uname` returns sysname=nodename=`nonux`, release=`0.1`,
version=`v1`, machine=`aarch64`, domainname=`(none)`.  Hardcoded;
a future slice with a real hostname concept would store these on
`nx_process` or a kernel global.

Initramfs `Makefile` extended to bundle five new applet aliases:
`/bin/{id,uname,tr,wc,head}`.  All five point at the same
`busybox` blob (busybox dispatches to the right applet via
`basename(argv[0])`); pack-initramfs.py doesn't dedupe by
content, so each alias is a fresh cpio entry.

Side-effect of the new initramfs entries: `RAMFS_MAX_FILES`
bumped 16 → 24 (8 initramfs entries + 5 new aliases + 6 test-
created files = 19, headroom to 24).  Cost: 32 → 96 MiB static
.bss for the per-instance file table.  Two host tests
(`ramfs_create_at_capacity_returns_enomem` and
`ramfs_open_slot_exhaustion_returns_enomem`) had their
`ATTEMPTS` constants bumped 24 → 32 and 96 → 128 respectively to
keep proving "we hit the cap" past the new geometry.

Test:
`posix_busybox_sh_id_uname_prog.{c,_blob.S}` +
`ktest_posix_busybox_sh_id_uname.c`.

## Slice 7.6d.N.14 — First 3-stage pipe + slot-position-preserving CHANNEL fork inheritance

Workload: `busybox sh -c "echo hello | tr a-z A-Z | wc -c"`.
Output: `6`.

This was originally framed in HANDOFF.md as "quality-of-coverage,
likely clean first try, no production code".  It wasn't — the
test surfaced a real kernel composition gap and the slice closed
the gap with a real production fix.

**The bug.**  Slice 7.6a's `sys_fork` CHANNEL inheritance loop
used `nx_handle_alloc(child_tbl, ...)` to install each parent
CHANNEL into the child.  `nx_handle_alloc` picks the first
INVALID slot.  For 2-stage pipes that worked: parent's CHANNELs
sit in dense low slots (3, 4) and the child re-densifies starting
at its own first INVALID (slot 3 — slots 0/1/2 are pre-installed
CONSOLE).  But ash closes its own pipe ends right after each
fork in a multi-stage pipeline:

```
ash creates pipe1 (slots 3, 4).
ash forks echo. echo inherits slots 3, 4. ash closes its slot 4.
ash creates pipe2 (slots 5, 6).        — pipe2 lands at 5/6 because
                                          slot 4 is already INVALID
                                          but pipe1's slot 3 is still
                                          alive — wait, no, ash
                                          closes its prevfd reads later.
ash forks middle cat.                   — child loop sees parent
  parent_tbl: slot 3 = c1.r,              slots 3 (c1.r), 5 (c2.r),
              slot 4 = INVALID,           6 (c2.w).  nx_handle_alloc
              slot 5 = c2.r,              re-densifies them into child
              slot 6 = c2.w.              slots 3, 4, 5 — **NOT** the
                                          parent's 3, 5, 6.  Encoded
                                          handles drift.
ash forks wc.                           — same dense-rewrite drift.
```

Middle cat's user code (a forked copy of ash) dup3's by encoded
handle value (e.g. `dup3(7, 1, 0)` for "make pipe2.write into
my stdout").  In ash that handle pointed at slot 5; in middle
cat it now points at slot 6 (or further) — a different (wrong)
endpoint.  Result: middle cat's stdout isn't connected to wc's
stdin.  wc reads from a CHANNEL whose peer never writes; eventually
some other endpoint closes and wc sees EOF with 0 bytes
read.  Output: `0\n` instead of `6\n`.

Triangulated by changing the middle stage to `cat | cat` and
`echo hello | wc -c`: 2-stage `echo hello | wc -c` produces `6`,
2-stage `echo hello | tr a-z A-Z` produces `HELLO`, but any
3-stage variant produces `0` (or empty) because the second pipe
never carries data.

**The fix.**  Rewrite `sys_fork`'s inheritance loop to iterate
parent slots and write CHANNEL (and FILE — see N.15) entries at
the **same** child slot index, overwriting the pre-installed
CONSOLE if the parent has a CHANNEL/FILE in slot 0/1/2.  Encoded
handles now stay stable across fork.

Test: `posix_busybox_sh_pipe3_prog.{c,_blob.S}` +
`ktest_posix_busybox_sh_pipe3.c`.

## Slice 7.6d.N.15 — Cross-process FILE-fd-through-fork via subshell + slot-position-preserving FILE inheritance

Workload: `busybox sh -c "(cat) < /banner"`.
Output: `hello from initramfs`.

Closes the gap deferred since slice 7.6a: `sys_fork` previously
inherited only HANDLE_CHANNEL.  HANDLE_FILE was skipped out of
caution about per-cursor state, but slice 7.6d.N.8's vfs `retain`
op (added for `dup3`/`fcntl(F_DUPFD)`) gives us exactly the right
primitive: bump the per-open's `refs`; both parent and child
point at the same `struct ramfs_open` (same cursor + flags),
matching POSIX's "fork shares the open file description" semantic.

Production: extend the (already slot-position-preserving — see
N.14) `sys_fork` inheritance loop to also handle `NX_HANDLE_FILE`,
calling `vops->retain` instead of `nx_channel_endpoint_retain`.
Skips the FILE branch if the vfs has no `retain` op (the slot
isn't bound yet during early bootstrap).

**Workload selection note.**  The HANDOFF originally proposed
`exec 3< /banner; head <&3` — `exec` opens fd 3 in ash itself
(no fork), then `head <&3` forks head with fd 3 dup'd onto stdin.
That idiom would have exercised the FILE-inheritance path
beautifully... except it hits a separate, deeper kernel gap: our
encoded-handle scheme `(gen << 8) | (idx + 1)` makes encoded
value 3 == slot 2, which is the pre-installed CONSOLE STDIN slot.
ash's `dup2(open_result, 3)` interprets `3` as encoded value 3
and lands the FILE at slot 2, aliasing it onto fd 0.  Then the
forked head's `dup2(3, 0); close(3)` resolves both the source AND
the destination AND the closed-fd to the SAME slot — close(3)
closes the file head was about to read.  head exits with
`standard input: I/O error`.

That's a slice 7.6d.N.16 issue (POSIX fd ↔ slot alignment).
Switching the workload to `(cat) < /banner` exercises FILE
inheritance via the subshell-fork path (subshell ash applies the
redirect to its own stdin, then forks cat) without using literal
fd 3+ — and that path runs end-to-end on slice N.15's
inheritance work alone.  See "N.16 deferral" below.

Side-effect cap bumps:
- `NX_PROCESS_TABLE_CAPACITY` 64 → 128 in `framework/process.c`.
- `MMU_MAX_ADDRESS_SPACES` 64 → 128 in `core/mmu/mmu.c`.

Both bumps were forced because the cumulative kernel test sweep
now creates more processes than the previous caps allowed.  The
test harness's `sched_rr_purge_user_tasks` only unlinks tasks
from the runqueue — `nx_process_destroy` doesn't run, so every
test's processes (including ash forks for each pipeline stage)
stay allocated until the QEMU run ends.  Real fix is reap-on-wait
(deferred since slice 7.4); the cap bumps are the v1 hack we keep
using.

Test: `posix_busybox_sh_xfile_prog.{c,_blob.S}` +
`ktest_posix_busybox_sh_xfile.c`.

## N.16 deferral

The HANDOFF originally framed N.16 as "drop in-handle generation
encoding (paydown for N.11 fcntl band-aid; skippable if no other
workload hits it)" — replacing `(generation << 8) | (idx + 1)`
with `idx + 1`.  N.15's investigation showed that's incomplete:

- Linux POSIX fds are 0-based integers (fd 0 = STDIN).
- Our encoded-value 0 is reserved for `NX_HANDLE_INVALID`.
- POSIX fd N maps to encoded value `(N+1)` for N ≥ 1, with fd 0
  special-cased to slot 2 in `sys_read` / `sys_handle_close` /
  `sys_dup3`.
- Result: encoded value 3 == slot 2 == CONSOLE STDIN.  POSIX fd 3
  is not addressable as a *new* file descriptor — it aliases onto
  the STDIN slot.

To make `exec 3< /banner; head <&3` work, we'd need POSIX fd N to
map to slot N (no `+1` shift).  That requires choosing a non-zero
sentinel for `NX_HANDLE_INVALID` (e.g. `0xFFFFFFFF`) so encoded
value 0 can name fd 0.  Substantial change, ripples through every
syscall body that compares against `NX_HANDLE_INVALID` and through
the slice-5.3 stale-handle invariants.

Skippable for slice 7.6d.N.final: an interactive `busybox sh`
prompt that runs basic commands doesn't issue literal `dup2(_, 3)`.
Pipes use whatever encoded values `sys_pipe` returns; redirects
target STDIN/STDOUT/STDERR (fds 0/1/2 — already special-cased).
Workloads that explicitly use fd 3+ (e.g. shell scripts using
`exec N< file`) will need N.16 — re-scope that slice when one
such workload becomes a goal.

## Test results

```
$ make test
PASS: 51/51 python tests
PASS: 275/275 host tests, 0 leaks, 0 errors
ktest: 99/99 PASSED, 0 failures
```

Net delta: +4 ktests (one per slice — N.12/N.13/N.14/N.15),
total kernel tests 95 → 99.

## Followups

- **slice 7.6d.N.16 (re-scoped).**  POSIX-fd-to-slot alignment.
  Choose a non-zero `NX_HANDLE_INVALID` sentinel; drop the `+1`
  shift in encode/decode; restate slice-5.3 stale-handle
  invariants against the new encoding.  Lands when a workload
  explicitly demands fd 3+ to be addressable as a normal POSIX
  file descriptor.
- **slice 7.6d.N.final.**  Interactive `busybox sh` over UART RX.
  Production work: PL011 RX IRQ handler → per-process line-
  buffered ring; `nx_console_read` blocks on the ring instead of
  returning 0 = EOF; `tcgetattr`/`tcsetattr`/`ioctl(TIOCGWINSZ)`
  stubs; `isatty(0)` → 1 via the stub; Ctrl-D = 0-byte EOF;
  Ctrl-C = SIGINT delivery via the slice N.12 handler dispatch;
  `/init` switches from libnxlibc test programs to busybox itself.
  Requires user direction on the `/init` swap (production path
  vs test path) and a manual-verification protocol.
- **reap-on-wait.**  `sys_wait` should `nx_process_destroy(child)`
  after collecting the exit status, freeing the process struct +
  handle table + MMU address space + 8 MiB user backing.  Would
  let us drop `NX_PROCESS_TABLE_CAPACITY` + `MMU_MAX_ADDRESS_SPACES`
  back from 128 to 32 and stop bumping them every few slices.
