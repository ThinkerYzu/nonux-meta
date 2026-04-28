# Session 73: slice 7.7a — trivial interactive smoke tests

**Date:** 2026-04-28
**Phase:** Phase 7 (in progress); slice 7.7 (sub-sliced; 7.7a closes here)
**Branch:** master

---

## Goals

- Open slice 7.7 (interactive smoke tests).  Slice 7.6d closed at v1
  in Session 72; slice 7.7's IMPLEMENTATION-GUIDE.md exit criteria
  asks for four scripted commands: `ls /`, `echo hello | cat`, `ps`,
  `mkdir /tmp` + `ls /tmp`.  HANDOFF.md flagged the split between
  trivial (existing kernel surface) and non-trivial (needs new
  kernel mechanism) work as a sub-slice judgement call.
- Land the trivial half this session — `ls /` and `echo hello | cat`
  — with no kernel changes.  Both workloads already work end-to-end
  at the ktest level (slices 7.6d.N.5 and 7.6d.N.6b respectively);
  the only question is whether they survive the interactive UART
  harness from Session 71/72.
- Sub-slice the non-trivial half (`mkdir`, `ps`) explicitly into
  slice 7.7b as a follow-up session, since both require new kernel
  surface (vfs/ramfs `mkdir` + `NX_SYS_MKDIRAT`; procfs view OR
  `NX_SYS_GETPROCS` + busybox patch).
- Preserve `make test` 432/432; promote `make test-interactive`
  3/3 → 5/5.

## What Was Done

### 1. Two new interactive scripts under `test/interactive/`

Drop-in additions to the slice-7.6d.N.final.d harness — no code
changes.  Each script is a `*.script` file fed line-by-line through
`tools/qemu-stdin-feed.sh` plus a `*.expected` file containing
substring patterns that must appear in the captured kernel log.

- **`ls_root.{script,expected}`** — drives `ls /` then `exit`.
  Expected contains `argv_child`, `musl_prog`, `banner` — three
  ramfs entry names that:
  - are visible in busybox ls's coloured columnar output (the
    `\x1b[1;32m` ANSI codes around each name don't break substring
    match because grep -F looks for `argv_child` literally inside
    `\x1b[1;32margv_child\x1b[m`);
  - won't false-match anywhere in the boot log (`init`, `busybox`,
    `cat` would all collide with kernel boot tags or status
    markers);
  - prove `getdents64` actually walked the root directory cursor
    (Session 60's slice 7.6d.N.5 deliverable) and returned multiple
    entries through the FILE arm of musl-stdio writev.

- **`echo_cat.{script,expected}`** — drives `echo HELLO_FROM_ECHO_CAT
  | cat` then `exit`.  Expected: the literal `HELLO_FROM_ECHO_CAT`
  string.  This is the headline two-process pipeline from Session
  63 (slice 7.6d.N.6b); it composes:
  - ash's `pipe(2)` → CHANNEL handles installed at slot 3+;
  - ash's first fork+`dup3` for the echo stage;
  - ash's second fork+`dup3` for the cat stage;
  - cat's CHANNEL-arm `read(0)` looping until peer-closed-EOF
    (`nx_channel_recv` returns 0 when queue empty AND peer
    endpoint closed, set in slice 7.6d.N.6b);
  - cat's CONSOLE-arm write(1) to UART;
  - ash's waitpid(-1) loop reaping both children.

  Distinctive marker (`HELLO_FROM_ECHO_CAT`) chosen for the same
  uniqueness reason as the slice 71's `HELLO_FROM_INTERACTIVE` and
  slice 71's `PIPE_INPUT` markers: substring matching against the
  ~150-line boot log needs to be deterministic.

### 2. First-run `ls_root` failure + correction

First-iteration `ls_root.expected` was `/init\n/bin/busybox\n` —
based on the assumption that `ls /` would print stored ramfs paths
verbatim.  That's wrong.  Tracing through the captured log:

- `sys_getdents64` (slice 7.6d.N.5; `framework/syscall.c:1873–1964`)
  strips one leading `/` from each entry's stored name before
  packing into `struct linux_dirent64.d_name`.  So `/bin/busybox`
  goes out the syscall as `bin/busybox`.
- busybox ls then treats the embedded `/` as a path separator and
  prints just the basename — `busybox`.  Confirmed by inspecting
  the actual log:

  ```
   # ls /
  [1;32margv_child[m  [1;32mcat[m         [1;32mid[m          [1;32mmusl_prog[m   [1;32mwc[m
  [1;32mbanner[m      [1;32mecho[m        [1;32minit[m        [1;32mtr[m
  [1;32mbusybox[m     [1;32mhead[m        [1;32mls[m          [1;32muname[m
   # exit
  ```

  13 visible names.  Updated `ls_root.expected` to the three unique
  ones (`argv_child`, `musl_prog`, `banner`) — `init`, `busybox`,
  `ls`, `cat`, `echo`, `head`, `id`, `tr`, `uname`, `wc` were
  rejected as expected-substrings because they collide with
  `[init] entering busybox sh at EL0` (init), the same line's
  "busybox" substring, or busybox's banner printout in other
  ktests' captured output that bleeds into the log.

  Re-ran `make test-interactive` → 5/5 pass.

### 3. Sub-slicing decision: 7.7 → 7.7a (this session) + 7.7b (next)

IMPLEMENTATION-GUIDE.md slice 7.7 originally listed four exit
criteria.  This session lands two; the other two need new kernel
surface that's bigger than a "drop in a script" change:

- **`mkdir /tmp` + `ls /tmp`** — needs:
  1. New `nx_vfs_ops.mkdir` interface op + ramfs implementation.
  2. New `NX_SYS_MKDIRAT` syscall (musl uses `__NR_mkdirat = 34`).
  3. Either real hierarchical paths in ramfs (substantial rework —
     ramfs is currently flat-namespace per `components/ramfs/ramfs.c`
     line 405–409 comment) OR a v1 hack that adds a directory marker
     to the file table without cracking open the namespace assumption.
  4. `sys_open` + `sys_getdents64` + `sys_fstatat` need to recognize
     directory entries at non-`/` paths (current code special-cases
     just `/` per slice 7.6d.N.5).  This isn't trivial.
- **`ps`** — needs a way to enumerate processes from userspace.  Two
  realistic shapes:
  1. New `procfs` component plumbed into vfs as `/proc/<pid>/stat`
     etc., so busybox's stock ps applet just works.  Requires the
     same hierarchical-path rework as mkdir, plus a procfs component
     mirroring `g_process_table` through ramfs-shaped reads.
  2. New `NX_SYS_GETPROCS` syscall returning a packed array of
     `(pid, ppid, name, state)` records, plus a vendored busybox
     patch teaching its ps applet to use it instead of /proc walks.
     Smaller kernel surface but adds a vendored-busybox patch.

Either path is realistically a session of its own (the mkdir
hierarchical-paths work has knock-on effects across `sys_open`,
`sys_fstatat`, `sys_getdents64`, ramfs `ramfs_op_open`, and the
vfs_simple front).  Pushing `7.7b` to a follow-up session (with
the path rework being the load-bearing piece) keeps slice 7.7a
small and low-risk.  The interactive-smoke-tests harness is now
live and proven against three workload classes (single-process
no-pipe, single-process getdents64, two-process pipe), so the
pattern for adding `7.7b` workloads later is clear: add
`mkdir_tmp.{script,expected}` and `ps_smoke.{script,expected}`
next to the others.

### 4. Doc updates

- `IMPLEMENTATION-GUIDE.md` — replaced the slice 7.7 block with a
  7.7a (this session, done) / 7.7b (next, mkdir + ps) split;
  preserved the four exit-criteria workloads as the union of 7.7a
  + 7.7b.
- `HANDOFF.md` — updated Current Status (slice 7.7a closes), Next
  Milestone (7.7b is now the next forward step), Phase checklist
  (added 7.7a checked + 7.7b pending), Next Actions (slice 7.7
  reframed as 7.7a/7.7b), Session Logs (added Session 73, moved
  Session 68 to archive to keep the 5-session window), Last
  Updated + Previous chain.
- `HANDOFF-ARCHIVE.md` — Session 68 inserted at the top of the
  archive list.
- `README.md` — bumped both Last Updated lines + refreshed Previous
  chain.
- `logs/session-73-7.7a-trivial-interactive-scripts.md` (this file).

## Key Findings

- **`getdents64`'s leading-`/` strip is a v1 hack with a visible
  side effect.**  The strip in `sys_getdents64` (`framework/syscall.c:1925-1926`)
  was added in slice 7.6d.N.5 to make ramfs's stored-as-absolute
  names look like POSIX basenames to userspace.  That works for
  flat top-level files (`/init` → `init`) but produces oddly-named
  entries for files under fake subdirectories (`/bin/busybox` →
  `bin/busybox`) — busybox's ls then strips the embedded `/` as a
  path separator, displaying `busybox`.  Cosmetically consistent
  but architecturally lossy: `ls /` returns 13 entries, but
  `bin/busybox`, `bin/cat`, `bin/ls`, etc. all share an apparent
  parent that doesn't exist in the file table.  Slice 7.7b's
  hierarchical-path rework is the right time to clean this up
  (real ramfs directories with real readdir scoping).  Recorded
  as a follow-up note; no change this session.
- **The interactive harness scales linearly with new workloads.**
  Adding two scripts pushed `make test-interactive` from 3/3 to
  5/5 with no harness changes.  Confirms slice 7.6d.N.final.d's
  design (per-byte trickle + substring grep against captured log)
  is the right primitive for slice 7.7+ smoke tests; we don't
  need to stand up a full pexpect-style state machine.  A future
  slice could promote substring grep to ordered-line-match if a
  workload needs ordering invariants (none does today).
- **Substring-uniqueness is the load-bearing test discipline.**
  The first-iteration `ls_root.expected` failed not because the
  kernel surface was broken but because the expected lines weren't
  unique to ls's output (`/bin/busybox` appears literally inside
  the boot log's `[fw] composition` line that mentions the busybox
  init runner indirectly).  Future smoke-test scripts should
  prefer markers that:
  - aren't English words used by the boot/dispatch/exit logging
    layer;
  - are visible in the workload's output without prefix-stripping;
  - are unique even after busybox's coloured-column formatting.
  Chosen format for 7.7+: ALL_CAPS_UNDERSCORE_MARKERS (matches
  slice 71's existing convention).

## Tests

- `make test-interactive` → **5/5 pass** (was 3/3 before this
  session): `echo_cat`, `echo_hello`, `echo_pipe`, `ls_root`,
  `visible_prompt`.  Stable across the iteration.
- `make test` → **432/432 pass** (51 python + 277 host + 104
  kernel), 0 leaks, 0 errors, exit 0 — unchanged from Session 72
  (no kernel changes this session).
- `make run-busybox` — manual eyeball test still works; `ls /`
  + `echo HELLO | cat` + `exit` all work interactively in the
  live shell.

## Files Changed

- `test/interactive/ls_root.script` (NEW, 2 lines)
- `test/interactive/ls_root.expected` (NEW, 3 lines)
- `test/interactive/echo_cat.script` (NEW, 2 lines)
- `test/interactive/echo_cat.expected` (NEW, 1 line)
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — slice 7.7 sub-sliced
  into 7.7a + 7.7b
- `proj_docs/nonux/HANDOFF.md` — Current Status, Next Milestone,
  Phase checklist, Next Actions, Session Logs, Last Updated
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 68 added
- `proj_docs/nonux/README.md` — Last Updated + Previous chain
- `proj_docs/nonux/logs/session-73-7.7a-trivial-interactive-scripts.md`
  (NEW, this file)

No kernel source changed.  No new ktests, no new host tests, no
new python tests, no new musl translations.

## Next Steps

- **Slice 7.7b** — `mkdir /tmp` + `ls /tmp` + `ps` interactive
  scripts.  Headline kernel work:
  - new `nx_vfs_ops.mkdir` op + ramfs implementation;
  - hierarchical-path support in ramfs (cleans up the `getdents64`
    leading-`/` strip hack discovered above);
  - new `NX_SYS_MKDIRAT = N` syscall + musl translation
    `__NR_mkdirat (34)`;
  - new procfs OR `NX_SYS_GETPROCS` for ps.
  Estimated 1–2 sessions depending on how the path-rework lands.
- **Slice 7.8** stays queued behind 7.7b (foundation slice for
  wait queues + ppoll; the slice 7.6d.N.final.e busybox patch
  reverts in 7.8c once `ppoll` is wired).
