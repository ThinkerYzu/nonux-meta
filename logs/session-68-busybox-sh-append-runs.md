# Session 68: slice 7.6d.N.11 — `busybox sh -c "echo a >> /tmp/ap; echo b >> /tmp/ap"` runs end-to-end

**Date:** 2026-04-28
**Phase:** 7 slice 7.6d.N.11 (busybox-as-shell, append-redirect + first F_DUPFD-above-cap fix)
**Branch:** master

---

## Goals

Slice 7.6d.N.10 (session 67) closed two bundled FILE-on-stdin
workloads with no production code changes — both composed from
already-tested mechanisms.  HANDOFF.md's slice N.11 plan
identified the next escalation as the first 7.6d.N step since
N.6b expected to need a real kernel/vfs change: `echo a >> /tmp/ap`
(append-redirect).  ash translates `>>` into
`O_WRONLY|O_CREAT|O_APPEND`; Linux `O_APPEND` (= `0o2000` =
`0x400`) was unmapped before this slice — sys_openat would have
silently dropped the bit, leaving each `>>` write to land at
cursor 0 (effectively truncating).

Two `>>` clauses back-to-back to force the append semantic to
actually take effect: a single `>>` against an empty file looks
indistinguishable from `>` (both produce `"a\n"`); but two
opens with non-empty intervening content distinguish a
correctly-implemented `O_APPEND` (file ends as `"a\nb\n"`,
4 bytes) from a dropped-flag implementation (file ends as
`"b\n"`, 2 bytes — the second open's fresh per-open with
cursor 0 stomps the first).

## Implementation

### `interfaces/fs.h` + `interfaces/vfs.h`

Add `NX_FS_OPEN_APPEND = (1U << 3)` and matching
`NX_VFS_OPEN_APPEND` (the VFS-layer mirror of the fs.h flag, kept
bit-for-bit compatible per the existing interface contract).
Brief paragraph in the docstring explaining "every write seeks to
end-of-file before writing" and pointing to slice 7.6d.N.11.

### `framework/syscall.c — sys_openat`

`NX_LINUX_O_APPEND = 02000u` constant.  In the existing
Linux→nonux flag-translation block (just below the access-mode
+ O_CREAT branches), one new line:

```c
if (lin_flags & NX_LINUX_O_APPEND) nx_flags |= NX_VFS_OPEN_APPEND;
```

Doc comment above updated: `O_APPEND=0o2000` added to the Linux
side; `APPEND=8` added to the NX side.

### `components/ramfs/ramfs.c — ramfs_op_open / ramfs_op_write`

`known` flag mask in `ramfs_op_open` extended to include
`NX_FS_OPEN_APPEND`.  In `ramfs_op_write`, before computing
`room`, one new branch:

```c
if (op->flags & NX_FS_OPEN_APPEND) op->cursor = op->file->size;
```

Comment explains the seek-before-write semantic and why it's
load-bearing for our refcounted-per-open layout (slice 7.6d.N.8):
two handle slots can reference the same per-open struct after
`dup3` + the FILE retain branch, so a non-APPEND write between
two APPEND writes would otherwise stomp the appended data when
the cursor was left mid-file.  v1's single-CPU model makes the
POSIX-mandated atomicity (seek+write as one) moot; the
positioning is still load-bearing.

### `framework/syscall.c — sys_fcntl high-arg fallback (the discovery)`

The append flag plumbing alone wasn't enough.  First test run
captured: parent reports status=01, file `/tmp/foo` ends up at
4 bytes `"a\na\n"` (the previous slice 7.6d.N.8 redir test left
`"a\n"` on the same path; first echo a appended; second echo b
never ran).  Live trace via temporary kprintf in sys_openat /
sys_dup3 / sys_fcntl / sys_write:

```
[bbsh-append-parent][openat lin=20441 nx=e][openat=4][fcntl fd=1
  cmd=1030 arg=10][fcntl fd=a cmd=2 arg=1][dup3 old=4 new=1]
  [wr fd=1 len=2 rc=2][dup3 old=a new=1][openat lin=20441
  nx=e][openat=260][fcntl fd=1 cmd=1030 arg=261][fcntl fd=1
  cmd=1030 arg=0][bbsh-append-status=01][bbsh-append-failed]
```

Decoded:
- First iteration's open returns encoded fd `4` (slot 3, gen 0).
- ash's `xdup_CLOEXEC_above(1, /*avoid_fd=*/0)` → `fcntl(1,
  F_DUPFD_CLOEXEC, 10)` succeeds at fd 10 (saves stdout).
  Redirect runs.  Restore via `dup3(10, 1)`.  close(4) bumps
  slot 3's generation to 1.
- Second iteration's open returns encoded fd `260` (slot 3,
  gen 1: `(1 << 8) | (3 + 1) = 0x104 = 260`).
- ash's `xdup_CLOEXEC_above(1, /*avoid_fd=*/260)` →
  `fcntl(1, F_DUPFD_CLOEXEC, 261)` — but our table only has
  64 slots, so the prior code returned `-EINVAL`.  ash bails.

Root cause: our handle encoding `(generation << 8) | (idx + 1)`
intentionally encodes generation in the high bits to detect
stale handles (slice 5.3 invariant), but ash treats fds as
small integers and uses `avoid_fd + 1` as the F_DUPFD floor.
With one close-and-reopen of a low-idx slot, the encoded value
crosses 256 and ash's avoid-pattern overflows our table cap.

Fix in `sys_fcntl`'s F_DUPFD body — when `min_idx >=
NX_HANDLE_TABLE_CAPACITY`, fall back to `min_idx = 0` (find the
lowest free slot) instead of returning `EINVAL`:

```c
size_t min_idx = (arg <= 0) ? 0 : (size_t)(arg - 1);
if (min_idx >= NX_HANDLE_TABLE_CAPACITY) min_idx = 0;
```

This is a *strict* POSIX violation — F_DUPFD's contract
guarantees the result is `>= arg` — but it preserves ash's
*intent* ("don't collide with avoid_fd"): the loop below skips
non-INVALID slots, so `avoid_fd`'s slot stays occupied by the
open that produced it and is never returned.  Userspace
software treating handles as opaque (which our handle layer is
designed for) gets a free slot regardless of where avoid_fd
lives.  Inline comment cross-references the symptom.

Long-term cleaner fix (deferred): drop the generation from
the encoded handle entirely; rely on per-slot in-table
generation purely for stale-detection and return `idx + 1` to
userspace.  That'd match POSIX `lowest fd` semantics at the
cost of slice 5.3's slice 5.3 stale-handle-protection test
expectations.  Not worth the test-bed churn for this slice.

### Test deliverable: `test/kernel/posix_busybox_sh_append_prog.{c,_blob.S}` + `ktest_posix_busybox_sh_append.c`

libnxlibc-linked program; forks; child does:

```c
nxlibc_execve("/bin/busybox",
              { "sh", "-c",
                "echo a >> /tmp/ap; echo b >> /tmp/ap", NULL },
              NULL);
```

Parent emits `[bbsh-append-parent]`, `[bbsh-append-status=NN]`,
and one of `[bbsh-append-ok]` / `[bbsh-append-failed]`.

Path is `/tmp/ap` — distinct from slice 7.6d.N.8/N.9's
`/tmp/foo` and `/tmp/copy` so the cumulative-test-order doesn't
leak content into this slice's assertions.  (Initial draft used
`/tmp/foo` and triggered a 6-bytes-not-4 mismatch from slice
N.8's stale `"a\n"` content; the path rename is the cleanest
isolation.)

ktest re-opens `/tmp/ap` through `vfs_simple` after the parent
exits and `KASSERT`s the contents are exactly `"a\nb\n"` —
4-byte assertion is load-bearing: a regression that drops the
APPEND flag would leave `"b\n"` (2 bytes) instead, since the
second open's fresh per-open struct starts at cursor 0 and the
write at cursor 0 against existing content `"a\n"` overwrites
it byte-by-byte, then `size=max(size, cursor)` clamps the
visible length to 2.  The strict 4-byte length + `buf[2]='b'`
+ `buf[3]='\n'` checks together catch every plausible
mis-implementation.

### Makefile

Added `ktest_posix_busybox_sh_append.c` to `KTEST_C`,
`posix_busybox_sh_append_prog_blob.S` to `KTEST_S`, and a
per-blob compile/link recipe block mirroring the existing
busybox-sh variant blocks.  `clean` target list updated to
include the new `.elf`.

## Outcome

Live ktest log:

```
posix_busybox_sh_append_parent_forks_and_execs_busybox_sh_append
  [bbsh-append-parent][bbsh-append-status=00][bbsh-append-ok]PASS
```

`make test` → **421/421 pass** (51 python + 275 host + 95 kernel),
0 leaks, 0 errors, exit 0.

Kernel test count: 94 → 95 (+1).
Host: 275 (unchanged).
Python: 51 (unchanged).

## Non-results / things deferred

- **Cross-process FILE-fd-through-fork inheritance still
  deferred** — N.11 doesn't surface it either (ash opens the
  redirect file in the child after fork, not before).  Needs a
  workload where the parent holds a FILE handle open *before*
  forking.
- **The handle-encoding generation issue is patched, not
  fixed.**  ash's `avoid_fd + 1` pattern hits the same gap any
  time a freshly-opened fd encodes above 64 — i.e., any time
  the chosen slot has `gen > 0`.  The fcntl high-arg fallback
  is a focused workaround.  Long-term, drop the in-handle
  generation (or shrink it to a few low bits) and lean on
  per-slot in-table generation for stale-detection only.
- **Other tolerable-syscall stubs (HANDOFF Next Action #7)
  still unmapped.**  This slice deliberately scopes to the
  append-redirect minimum — the wider stubs sweep waits for
  a workload that demands them.

## Lessons

1. **Cross-test ramfs state matters.**  ramfs is a flat
   per-instance file table; ktests don't reset it between
   runs.  Re-using path names from previous ktests (especially
   `/tmp/foo`) produces order-dependent assertions.  Pick a
   distinct path per ktest, or `vfs_simple_open ... TRUNCATE`
   in the test setup once a TRUNCATE flag exists.

2. **POSIX-fd-as-small-int collides with our generation
   encoding.**  Anywhere userspace uses arithmetic on fd
   values (`avoid_fd + 1`, `dup_above(fd, max(fds))`), our
   encoded handles can produce values larger than the table
   capacity once any slot has been closed-and-reopened.  Each
   collision needs a defensive fallback OR the long-term
   encoding fix.  Fcntl's F_DUPFD is the first; keep an eye
   out in select/poll/dup2-from-explicit-N as more workloads
   land.

3. **Trace early, decide late.**  The minute the append slice
   wasn't a clean first run, kprintf-around-syscalls showed
   the actual timeline in seconds — much faster than reading
   ash's redirect.c trying to predict the gap.  Both the
   tracing (~12 lines added/removed) and the wrong-path-
   contamination (`/tmp/foo` vs `/tmp/ap`) were 5-minute
   detours, not multi-iteration debug sessions, because the
   trace pinpointed the failure point precisely.

## Updated docs

- `proj_docs/nonux/HANDOFF.md` — current status, phase
  checklist, next actions (close 7.6d.N.11; surface
  generation-encoding follow-up; promote N.final + 7.7).
- `proj_docs/nonux/README.md` — recent updates.
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — slice 7.6d.N.11
  closure note.
- `proj_docs/nonux/logs/session-68-busybox-sh-append-runs.md`
  (this file).
