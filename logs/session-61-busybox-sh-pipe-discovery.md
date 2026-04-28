# Session 61: slice 7.6d.N.6a — first pipe attempt: `busybox sh -c "echo hello | cat"`

**Date:** 2026-04-27
**Phase:** 7 slice 7.6d.N.6 (busybox-as-shell, first pipe — split into 7.6d.N.6a discovery + 7.6d.N.6b proper fix)
**Branch:** master

---

## Goals

Slice 7.6d.N.5 closed `ls /` end-to-end.  Sessions 7.6d.N.0..5
covered builtins-only (`exit`, `echo`, `echo;echo`) plus a
single non-builtin fork+exec (`ls`).  This sub-slice escalates
to the smallest pipeline: `echo hello | cat`.  Two stages,
two forks, an actual pipe wired between them, and cat's read
loop on its stdin.

Discovery-driven: write the test, run it, capture failures,
iteratively map missing syscalls and implement small kernel
pieces until either it runs or the failure surface mandates
a bigger slice.

## What landed

Four musl translations + two new kernel syscalls + an EOF
fix to `sys_read` + two new entries in the initramfs.

### `__NR_pipe2 (59)` → `NX_SYS_PIPE (15)` mapping

ash's first pipeline syscall.  musl 1.2.5's `pipe(int[2])` on
aarch64 calls `pipe2(fds, 0)` (no plain `pipe` syscall on
aarch64 — same shape as the openat / getdents64 situation).
NX_SYS_PIPE already existed (slice 7.5) with signature `(int
*fds)`; the kernel ignores extra register args, so dropping
the flags arg works as a one-line translation.

**Why discovery here matters:** the captured failure was
`sh: can't create pipe: Operation not permitted` — *not*
"Function not implemented".  Root cause: `NX_EINVAL = -1`
(see `framework/registry.h:33`) collides with Linux's
`EPERM = -1`.  Our kernel returns NX_EINVAL for unknown
syscall numbers; musl interprets that as Linux EPERM,
giving the misleading "Operation not permitted" message
for any unmapped syscall the kernel actually receives.
This collision now affects *every* unmapped syscall that
slips past the musl-side translation table.  Worth a
follow-up cleanup: change the kernel's "unknown syscall"
return from `NX_EINVAL` to a Linux-shape `-ENOSYS = -38`
so the error messages stop lying.

### `__NR_clone (220)` → `NX_SYS_FORK (12)` mapping

Session 59's ls test puzzle is now fully resolved: ash's
`fork()` on aarch64 calls `__NR_clone (220)` with
`SIGCHLD, NULL, NULL, NULL, NULL`.  Our previous translation
dropped clone to -ENOSYS via the C-level wrapper's default,
and ash silently fell into a single-command-shortcut path
where no fork was actually needed (because `ls /` could be
exec'd directly — when ash is the only thing in the
pipeline, it execve's in place rather than forking).  For
`echo hello | cat` ash *must* fork twice (one process per
pipeline stage), so the unmapped clone bites.

The translation drops all five clone arguments — `flags`,
`child_stack`, `parent_tid`, `child_tid`, and `tls`.  musl's
`fork()` always passes `child_stack=NULL` (which selects
Linux's "duplicate parent's stack" mode = our fork
semantics) and the only flag bit is `SIGCHLD` (which we
don't implement parent-notify-on-exit anyway).  pthread
clone (`CLONE_VM | CLONE_FILES | ...`) would need real
thread support — not on the v1 roadmap.

### `__NR_dup3 (24)` → `NX_SYS_DUP3 (24)` — new syscall

ash uses `dup3(oldfd, newfd, 0)` to redirect stdin/stdout
to pipe ends before exec'ing each pipeline stage.  musl's
`dup2(o, n)` calls `dup3(o, n, 0)` on aarch64 (no plain
`dup2`/`dup` syscall).

`sys_dup3` (~95 lines in `framework/syscall.c`):

1. POSIX semantic: `oldfd == newfd` returns `newfd` unchanged
   (matches musl's `dup2` shape; real Linux dup3 returns
   `-EINVAL`, but musl unconditionally issues `SYS_dup3` for
   both shapes and the dup2 semantic is friendlier for ash).
2. Look up oldfd via standard `nx_handle_lookup`.  Stale or
   invalid → return immediately.
3. Decode newfd's slot index + generation from the encoded
   handle layout (low 8 bits = idx + 1, high 24 = gen).
   Special case: `newfd == 0` (POSIX `STDIN_FILENO`) has no
   encoded form in our scheme — encoded value 0 is reserved
   for `NX_HANDLE_INVALID`.  Treat newfd=0 as "install at
   slot 0 with gen=0" so the matching `h==0` special case
   in `sys_read` resolves it back via direct slot lookup.
4. Close-on-replace: if newfd's slot has an entry, decref/
   close per type (channel endpoint close, vfs close, dir
   cursor free).  *Don't* bump generation — we're about to
   overwrite with a specific gen.
5. Retain the source's underlying object (channel endpoint
   ref+1 for HANDLE_CHANNEL).
6. Install at the destination slot, forcing generation to
   match newfd's encoded gen so the encoded handle for the
   slot equals newfd exactly (callers like ash assume the
   literal value they passed in keeps working).

The forced-generation step trades the "stale handle
detection bumps gen" invariant for that specific slot —
acceptable because dup3 is an explicit replace operation.

### `__NR_readv (65)` → `NX_SYS_READV (25)` — new syscall

cat's main loop reads stdin via `bb_copyfd_eof()` →
`safe_read()` → musl's `__stdio_read` → `readv()`.  We had
writev (slice 7.6c.3c) but not readv.  Implementation
mirrors `sys_writev` exactly: copy_from_user the iovec
array (cap 16 entries), dispatch each entry through
`sys_read` so all the magic-fd / handle-table / CHANNEL /
FILE branches apply, stop on first short read per Linux's
readv semantic, return running total.

### `sys_read` h=0 magic-fd extension

The existing `sys_read` magic-fd fallback was
`if (rc == NX_ENOENT && h == 0) return 0;`.  But `h == 0`
(= `NX_HANDLE_INVALID`) makes `nx_handle_lookup` return
`NX_EINVAL`, not `NX_ENOENT` — so the fallback never
fired for the actual STDIN fd value.  cat's read on fd 0
saw `-NX_EINVAL = -1` = Linux EPERM, printed
`cat: read error: Operation not permitted`, and busy-
looped.

New shape: when `h == 0`, look up slot 0 directly (bypass
the encoded-value generation check).  If slot 0 has a
real handle (e.g. a pipe read end installed by `dup3`),
dispatch it normally.  If empty, return 0 = EOF.  This
makes cat exit cleanly on empty stdin instead of busy-
looping on EPERM.

### Initramfs additions

`Makefile`'s `pack-initramfs.py` invocation now adds two
cpio duplicates of `/bin/busybox`:

- `/bin/cat` — needed as soon as ash forks the pipeline
  stage 2 and execve's `cat`.
- `/bin/echo` — preemptive; future builtins-vs-applet
  escalations may want the externally-exec'd echo.

Each is +1.22 MB in the initramfs.  RAMFS_MAX_FILES = 16
still has slack (now 11/16 used: init + banner +
argv_child + musl_prog + bin/busybox + bin/ls + bin/cat +
bin/echo + 3 test-side creates).

## Captured failure mode

With everything above wired, the test reaches an actual
hang inside cat after running ash startup, forking twice,
exec'ing `/bin/cat`, and reaching cat's read loop.  The
kernel test eventually times out under QEMU's 90s
test-kernel limit.  Held out of `KTEST_C` (with a
`#`-comment block in the Makefile cross-referencing this
session log) so `make test` stays green at 414/414 while
the proper fix lands as a separate sub-slice.

Diagnostic instrumentation (temporary musl pass-through
of unmapped Linux numbers + kernel `kprintf("[svc?=N]")`,
both reverted before commit) captured the syscall trace:

- ash startup: 96 (set_tid_address), 174/176/144/146/172/
  173 (uid/gid/pid/ppid family), 134/135 (rt_sigaction /
  rt_sigprocmask × many), 160 (uname).  All -EPERM via
  `NX_EINVAL`; ash shrugs through them.
- ash sigprocmask + clone — fork the first stage.
- Stage 1 child startup: 96, 134, 135 — same family,
  shorter (a forked child reuses some of the parent's
  init state).  Then **24** (dup3 — *now mapped*).
- Stage 2 child: similar, then 24 again.
- *Inside cat*: 71 (sendfile) + more 135 (sigprocmask)
  before the trace dries up.

The captured `[svc?=71]` is the smoking gun for the
hang: busybox's `bb_copyfd_size_eof` tries `sendfile(out,
in, 0, count)` first as a fast path for regular files.
With our `sys_fstatat` reporting *every* path as
`S_IFREG | 0755` (slice 7.6d.N.4 chose this for ash's
PATH walk + slice 7.6d.N.5 added the `/` → `S_IFDIR`
special case but kept everything else as REG), cat's
fstat on stdin returns a regular file shape, so it
takes the sendfile path.  sendfile returns -EPERM; cat
falls back to a copy_loop, which then issues another
read — but that read is what loops indefinitely.  Best
guess: cat's loop also hits something else unmapped or
gets stuck in a sigprocmask-pair around an interrupted
syscall that never gets retried correctly.

Pinning the exact infinite-loop point is 7.6d.N.6b's
job, alongside the bigger structural fix below.

## The deeper structural problem (slice 7.6d.N.6b)

Even with cat's hang fixed, the pipe wouldn't actually
carry data.  Walk-through:

1. `pipe()` allocates handles in slots 0 and 1, encoded
   as `1` and `2`.  ash treats these as Linux fds.
2. Linux convention: pipe returns fds ≥ 3 because 0/1/2
   are STDIN/STDOUT/STDERR.  Our pipe returns fds 1 and
   2 — collision with STDOUT_FILENO and STDERR_FILENO.
3. Stage 1 calls `dup2(p[1]=2, STDOUT_FILENO=1)` —
   *overwriting slot 0 (which held the pipe READ end)*
   with the pipe write end.
4. Stage 1's subsequent `close(p[0]=1)` and
   `close(p[1]=2)` close both pipe-write refs in stage
   1, *without ever wiring the pipe read end into stage
   2's stdin*.  echo's stdout flushes to slot 0 → magic-
   fd → debug_write → UART.  The pipe is fully bypassed.

The fix is to **reserve slots 0/1/2 in `nx_handle_alloc`**
so pipe (and every other handle allocator) starts at
slot 3.  Then:

- `pipe()` returns slots 3, 4 → encoded handles 4, 5 →
  ash sees Linux-shape fds.
- `dup3(_, 1, 0)` installs at slot 0 (the reserved STDOUT
  slot) without colliding with the pipe handles.
- `read(0, ...)` looks up slot 0 — either finds the
  dup3'd stdin or falls through to magic-fd EOF.
- `write(1, ...)` finds the dup3'd stdout (or magic-fd
  → UART).

Affected by the change:

- `framework/handle.c`'s `nx_handle_alloc` — bump the
  scan start from 0 to a `NX_HANDLE_FIRST_USER_SLOT = 3`
  constant.
- Host handle tests (`test/host/handle_test.c`) — the
  capacity-fill case asserts `count ==
  NX_HANDLE_TABLE_CAPACITY` after filling all slots; with
  3 reserved that becomes `capacity - 3`.  Other tests
  that assert specific handle values (e.g. "first slot
  is 1") update to the new first slot 4.
- Kernel handle tests (`test/kernel/ktest_handle.c`) —
  same shape.

7.6d.N.6b is also where the `[svc?=71]` cat-hang
investigation lands, and probably mapping
`__NR_rt_sigaction (134)` + `__NR_rt_sigprocmask (135)`
as no-op-success stubs (already on the opportunistic
list under "Tolerable-syscall stubs sweep" in HANDOFF.md).
With those three together — slot reservation + sendfile
fast-path skipped (or stub'd) + signal stubs — the pipe
test should run end-to-end.

## What works now

- `pipe()` succeeds, returns valid handles.
- `fork()` works (clone via musl is mapped).
- `dup3(_, 1, 0)` installs at slot 0; the encoded-handle
  generation alignment is correct (return value = newfd).
- `dup3(_, 0, 0)` special-cases newfd=0 and installs at
  slot 0 with gen=0; subsequent `read(0, ...)` resolves
  via the new slot-0 direct-lookup path.
- `readv` works (mirrors writev one-to-one).
- ash startup, fork, exec(`/bin/cat`) all run cleanly.

## What doesn't work

- The pipe-data path (pipe writes from stage 1 don't
  reach stage 2's stdin) — slot collision.
- cat's actual read loop hangs after sendfile fallback
  — exact failure not yet pinned.
- The pipe ktest is held out of `KTEST_C` until both of
  the above land.

## File changes

**Source repo (`sources/nonux`):**

- `Makefile` — added `/bin/cat` and `/bin/echo` cpio
  duplicates; added the new test-side `.c` / `_blob.S` to
  `KTEST_S` (built but unused) but held the corresponding
  `ktest_…_pipe.c` out of `KTEST_C` with an explanatory
  comment block.
- `framework/syscall.h` — added `NX_SYS_DUP3 = 24` and
  `NX_SYS_READV = 25` enum entries with rationale comments.
- `framework/syscall.c` — added `sys_dup3` (~95 lines) +
  `sys_readv` (~35 lines) + dispatch table entries; fixed
  `sys_read` magic-fd path so `h == 0` resolves via
  direct slot-0 lookup before falling through to EOF.
- `third_party/musl/arch/aarch64/syscall_arch.h` — added
  C-level translation table entries for 24, 59, 65, 220.
- `third_party/musl/src/thread/aarch64/syscall_cp.s` —
  matching asm-level translation entries.
- `test/kernel/posix_busybox_sh_pipe_prog.c` — new EL0
  test program: forks + execve's `/bin/busybox sh -c
  "echo hello | cat"`.
- `test/kernel/posix_busybox_sh_pipe_prog_blob.S` — blob
  wrapper.
- `test/kernel/ktest_posix_busybox_sh_pipe.c` — kernel-
  side ktest harness (held out of KTEST_C).

**Doc repo (`proj_docs/nonux`):**

- `HANDOFF.md` — current-status / next-actions /
  session-log additions.
- `IMPLEMENTATION-GUIDE.md` — slice 7.6d.N.6 sub-
  division + 7.6d.N.6b plan.
- `README.md` — Last-Updated banner.
- This session log.

## Test status

`make test` → **414/414 pass** (51 python + 275 host +
88 kernel).  Same count as session 60 — the new pipe
ktest is built but held out of the test list.

## Next

**7.6d.N.6b** — slot-0/1/2 reservation in
`nx_handle_alloc` + cat-hang investigation +
sigaction/sigprocmask no-op stubs.  Re-enable the
pipe ktest at the end and assert `[bbsh-pipe-ok]`
with status 0.
