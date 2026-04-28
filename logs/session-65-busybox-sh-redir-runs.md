# Session 65: slice 7.6d.N.8 — `busybox sh -c "echo a > /tmp/foo"` runs end-to-end

**Date:** 2026-04-28
**Phase:** 7 slice 7.6d.N.8 (busybox-as-shell, first stdout-to-file redirection)
**Branch:** master

---

## Goals

Slice 7.6d.N.7 (session 64) drove cat against a real ramfs file —
`HANDLE_FILE` on the read side, `HANDLE_CONSOLE` on the write side.
Clean first try, no kernel gaps.

This sub-slice flips the FILE handle to the **write** side and onto
**stdout**, which is exactly the path slice 7.6a deferred: a FILE
handle inherited / dup'd onto fd 1 and used by a builtin without
forking.  Workload: `busybox sh -c "echo a > /tmp/foo"`.

Concretely this exercises four things at once:

1. `sys_openat` with `O_CREAT | O_WRONLY | O_TRUNC` against a path
   that doesn't exist (ash's `openredirect()` of "/tmp/foo").
2. A redirection bookkeeping path that goes through `fcntl` —
   `save_fd_on_redirect()` calls `fcntl(1, F_DUPFD_CLOEXEC, 10)` to
   stash the original stdout before installing the redirection.
   `__NR_fcntl (25)` is unmapped today; the previous slice's
   ENOSYS hygiene means ash sees a real -ENOSYS rather than
   "Operation not permitted", which `xfunc_die`s ash on entry to
   redirect.  So we have to map fcntl at minimum for F_DUPFD_CLOEXEC.
3. `sys_dup3` on a `HANDLE_FILE` source — slice 7.6d.N.6a wrote the
   close-then-replace logic but only `HANDLE_CHANNEL` was retained.
   With FILE in the mix, the per-open struct now lives in two slots
   and must survive the first close — needs a vfs `retain` op.
4. Redirected stdout itself — the echo builtin writes "a\n" through
   `sys_write` to a `HANDLE_FILE` (not the CONSOLE fallback).  Slice
   6.3's FILE arm of `sys_write` already covers this, but exercising
   it through ash's redirection setup is new.

Discovery-driven again: write the slice, observe what actually
surfaces, fix the gaps that block.

## Implementation

### Test deliverable

- `test/kernel/posix_busybox_sh_redir_prog.c` — libnxlibc-linked
  program; forks + child execs `busybox sh -c "echo a > /tmp/foo"`;
  parent emits `[bbsh-redir-parent]`, waits, emits
  `[bbsh-redir-status=NN]` + one of `[bbsh-redir-{ok,failed}]`.
- `test/kernel/posix_busybox_sh_redir_prog_blob.S` — `.incbin` of
  the linked ELF, weak start/end symbols.
- `test/kernel/ktest_posix_busybox_sh_redir.c` — KASSERTs parent
  exits 0, then re-opens `/tmp/foo` through `vfs_simple` and asserts
  the contents are exactly `"a\n"`.
- `Makefile` — three new build rules (object, ELF, blob), `KTEST_C`
  and `KTEST_S` lists extended, `clean` target updated.

### Production changes

#### `NX_SYS_FCNTL = 26` (`framework/syscall.{h,c}`, ~95 lines)

Minimal cmd set covering exactly what ash uses during builtin
redirection setup:

- **F_DUPFD (0)** / **F_DUPFD_CLOEXEC (1030)** — find the lowest
  free slot whose encoded handle is ≥ `arg`, install a copy of
  fd's `(type, rights, object)`, retain the underlying object
  (CHANNEL endpoint refcount, FILE per-open refcount), and return
  the new encoded handle.  CLOEXEC is moot in v1 since exec
  rebuilds the handle table from scratch in `sys_exec`, so both
  commands collapse to the same body.

- **F_GETFD (1) / F_SETFD (2) / F_GETFL (3) / F_SETFL (4)** —
  return 0.  We don't track per-handle FD or open-file flags.
  ash treats this as "the bit you asked about was clear / was set
  successfully" and continues — exactly the behaviour we need
  without having to actually plumb FD_CLOEXEC.

- anything else — return `-ENOSYS = -38`.  Programs that need
  real fcntl surface the gap explicitly.

Encoded-handle reminder: `(generation << 8) | (idx + 1)`.  For a
fresh slot we use generation 0, so the encoded value is `idx + 1`.
F_DUPFD's `arg` is the minimum POSIX fd; with gen 0 that maps to
`idx ≥ arg - 1` (clamped to 0 for `arg ≤ 0`).  Special case skipped:
`arg <= 0` would make the loop start at `min_idx = 0`, which is fine
(slot 0 is reserved at `nx_process_create` for STDIN, so the search
always passes through it without finding a free slot until slot 3+).

Linux errno values returned directly (`-EBADF = -9`, `-EINVAL =
-22`, `-EMFILE = -24`) — fcntl's failure modes are programs-visible
through musl's errno translation, so they need the standard Linux
numbering, not our internal `NX_E*`.  Same pattern as session 59's
`sys_fstatat`.

#### `sys_dup3` FILE retain branch (`framework/syscall.c`, ~5 lines)

Slice 7.6d.N.6a's `sys_dup3` retained CHANNEL endpoints.  This slice
adds the FILE arm: when the source is `HANDLE_FILE`, call
`vops->retain` on the per-open object so it survives the first
close.  Mirrors the existing CHANNEL branch one-to-one.

#### vfs `retain` op (`interfaces/{fs,vfs}.h` + components/{ramfs,vfs_simple}/*.c)

New `nx_fs_ops.retain(self, file)` and matching `nx_vfs_ops.retain`.
Drivers that don't refcount may leave the op NULL; the syscall
layer falls back to "no retain" — semantically a leak in v1, but
no driver currently needs that fallback.

ramfs: `struct ramfs_open` gains a `refs` counter.  `open()` returns
`refs = 1`, `retain()` bumps, `close()` decrements and frees the
slot only when `refs` reaches 0.  POSIX semantic: dup'd fds share
the cursor + flags.

vfs_simple: `vfs_simple_retain` forwards to the active mount's
driver.

#### musl translation

`__NR_fcntl (25) → NX_SYS_FCNTL (26)` in both
`arch/aarch64/syscall_arch.h` (C-level `__syscallN`) and
`src/thread/aarch64/syscall_cp.s` (asm-level `__syscall_cp_asm`).

## Discovery + bring-up

Started by writing the test program + ktest and running `make test`.
Observed parent exits cleanly, `[bbsh-redir-status=01]` (ash exited
1), `/tmp/foo` did NOT exist or was empty.

To pin where ash gave up I added a heavy diagnostic stack —
*temporary*, all reverted before the commit:

- Kernel-side `kprintf("[>%u(%x,%x,%x)]", ...)` and
  `kprintf("[<%u->%d]", ...)` brackets in `nx_syscall_dispatch`
  to log every entry/exit (skipping svc 1 to keep debug_write
  spam tractable).
- Kernel-side `kprintf("[svc?=%u]")` for the diagnostic syscall
  range (see below).
- Kernel-side `kprintf("[openat-path:\"%s\"]")` in `sys_openat`.
- Musl pass-through: `__nx_translate(n)`'s `default` returns
  `n + 1000` (instead of `-1`); kernel handles `1000 ≤ num <
  1500` by logging `[svc?=N]` and returning `-ENOSYS`.  This
  passes the unmapped Linux syscall numbers through to the kernel
  so they show up in the trace.  Same trick for the asm switch
  in `syscall_cp.s` (`add x8, x1, #1000; b .Lnx_run`).
- ash-side: direct svc 0 instructions in `openredirect()` and
  `redirect()`, plus inlined `write(2, ...)` calls before/after
  each `open()` and at the entry of `save_fd_on_redirect()`.

First instrumented run: trace showed openat for "/tmp/foo" → fd 4
followed *immediately* by `exit(1)` with NONE of the ash-side
markers firing.  Half-an-hour theorising "where did
`openredirect()` go" and grepping the busybox binary for the
markers (`readelf -p .rodata` confirmed they were in the rodata
section).

Root cause: **stale initramfs.cpio**.  busybox had been rebuilt
with my ash debug code, but `test/kernel/initramfs.cpio` was older
than busybox and `make test` didn't trigger a rebuild.  The cpio
rule does declare `$(BUSYBOX_BIN)` as a prerequisite, but on a
fresh-from-distclean build the rule chain didn't kick in — the
binary was still 1224448 bytes from a prior session.  Forcing
`rm -f test/kernel/initramfs.cpio && make test` rebuilt the cpio
and re-embedded the new busybox.

Second instrumented run: trace finally showed the full ash flow:

```
[REDIR-call] [oR-direct] [oR-NCLOBBER]
[>23(...)][openat-path:"/tmp/foo"][<23->4]   openat → fd 4
[oR-f+004]                                    open returned 4
[sFR-enter]                                   save_fd_on_redirect
[>26(1,406,a)][<26->10]                       fcntl(1, F_DUPFD_CLOEXEC, 10) → 10
[>26(a,2,1)][<26->0]                          fcntl(10, F_SETFD, 1) → 0
[>24(4,1,0)][<24->1]                          dup3(4, 1, 0) → 1
[>2(4,0,0)][<2->0]                            close(4)
[>8(1,...,2)][<8->2]                          write(1, "a\n", 2) → 2
[>24(a,1,0)][<24->1]                          dup3(10, 1, 0) → 1 (restore)
[>2(a,0,0)][<2->0]                            close(10)
[>11(0,0,0)]                                  exit(0)
[bbsh-redir-status=00][bbsh-redir-ok]
[/tmp/foo open=ok read=2:"a\n"]
PASS
```

Everything works end-to-end.  fcntl handled F_DUPFD_CLOEXEC and
F_SETFD correctly.  dup3 retained the FILE handle so the
subsequent close didn't tear down the per-open before write could
use it.  ramfs accumulated the writes, vfs read-back saw "a\n".

After confirming the path was clean, all diagnostic instrumentation
was reverted (`framework/syscall.c` dispatch + sys_openat,
`syscall_arch.h` + `syscall_cp.s` musl pass-through, ash.c write
markers).  The kprintf-peek in the ktest was upgraded to a real
`KASSERT_EQ_U(buf[0], 'a')` + `KASSERT_EQ_U(buf[1], '\n')` against
the file content read back through vfs_simple.

## Outcome

`make test` → **417/417 pass** (51 python + 275 host + 91 kernel),
0 leaks, 0 errors, exit 0.

Live ktest log:
```
posix_busybox_sh_redir_parent_forks_and_execs_busybox_sh_redir
[bbsh-redir-parent][bbsh-redir-status=00][bbsh-redir-ok]PASS
```

The slice closes the FILE-fd inheritance gap from slice 7.6a in the
specific form ash actually exercises: `dup3` on a FILE handle into
a builtin's stdout slot, with the FILE per-open surviving close +
re-close around it.  Cross-process FILE-fd inheritance through
fork (the original 7.6a follow-up) is still deferred — a forked
child currently gets an empty handle table, which would leave the
inherited FILE handle dangling.  That's the next gap to surface,
likely with `cat /banner > /tmp/copy` (which forces the redirected
FILE handle to be inherited across the fork-exec to /bin/cat).

## Next milestone

7.6d.N.9+ — escalate further to surface the next real gap.  Likely
candidates:

- `cat /banner > /tmp/copy` — cross-process FILE-fd inheritance
  through fork (the 7.6a residual), forces fork to dup-and-retain
  a FILE handle into the child's table.
- `cat < /banner` — input redirection; slim variant of N.7.
- `ls / | wc -l` — adds wc applet, exercises pipe with multi-line
  input AND `getline` over a CHANNEL.
- `echo $(cat /banner)` — command substitution; probably needs
  pipe + double-fork.

Final target: 7.6d.N.final (interactive `busybox sh` over UART RX —
RX IRQ + per-process line buffer + sigaction for SIGINT +
ioctl(TIOCGWINSZ) stub + tcsetattr stub).

## Lessons / Notes

- **Build-system gotcha for the next slice:** when changing
  `third_party/busybox/shell/*.c`, the busybox binary rebuilds
  (the build script's internal make catches the source change),
  but the initramfs cpio's dependency on busybox isn't always
  enough to trigger re-pack on the *first* rebuild after a clean.
  Defensive habit: `rm -f test/kernel/initramfs.cpio` whenever
  ash is touched, or just `touch third_party/busybox/busybox`
  before `make test`.

- **fcntl as the cheapest "unblock ash" syscall.**  ash uses
  fcntl exclusively as a fancy dup — F_DUPFD_CLOEXEC for stash,
  F_SETFD to mark CLOEXEC, F_GETFD/F_GETFL/F_SETFL as no-ops
  during recipe setup.  None of those need real per-handle flag
  storage in v1.  When the time comes for *real* CLOEXEC
  semantics (fork-exec into a child that needs to see only some
  inherited handles), bump F_SETFD to actually set a per-slot
  bit and have `sys_exec` walk the table dropping the
  cloexec-flagged ones.  Keep that behind the same fcntl entry
  point so the API surface doesn't grow.

- **Refcounted vfs per-open as a single-use fix.**  The
  refcount lives on `struct ramfs_open` and is bumped only by
  `dup3` / `fcntl(F_DUPFD)` of a HANDLE_FILE.  Single-process v1
  doesn't need atomics — RR scheduling means no two tasks
  concurrently mutate one process's handle table without the
  preemption boundary serialising them.  When SMP lands (or
  even when a future per-process kthread races with the
  syscall path), the bump and decrement need C11 atomics on
  `refs` (per the user's preemption-needs-atomics feedback).
  Comment in `ramfs.c` flagging this would be useful but not
  load-bearing yet.
