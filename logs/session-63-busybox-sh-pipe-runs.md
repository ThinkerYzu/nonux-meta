# Session 63: slice 7.6d.N.6b — `busybox sh -c "echo hello | cat"` runs end-to-end

**Date:** 2026-04-27
**Phase:** 7 slice 7.6d.N.6 (busybox-as-shell, first pipe — closes 7.6d.N.6b: console device + slot reservation + slots-survive-exec + waitpid-any + pipe-EOF)
**Branch:** master

---

## Goals

Slice 7.6d.N.6a (Session 61) added the four-syscall scaffolding
(pipe2 / clone / dup3 / readv translation + `sys_dup3` + `sys_readv`)
for the first pipeline but the ktest was held out of `KTEST_C` because
of three structural gaps: (1) pipe()'s natural slot 0/1 allocation
collided with POSIX `STDOUT_FILENO=1` / `STDERR_FILENO=2`, (2) the
magic-fd fallback in `sys_read`/`sys_write` papered over the missing
console primitive, and (3) the kernel had no per-process EOF semantics
for pipes (the writer's `nx_process_exit` left the channel handle
dangling).

This session closes all three.  Plus three others surfaced during
debug:
- (4) `sys_wait` didn't support `pid == -1` (POSIX waitpid-any-child;
      ash uses this for shell pipelines)
- (5) `sys_wait` returned `NX_ENOENT = -4` for "no children" which
      musl translates to userspace `errno = EINTR` instead of
      `ECHILD = 10`, causing ash to spin
- (6) Channel `recv` returned `NX_EAGAIN` even when the peer (writer)
      side was fully closed, so cat's read-after-EOF saw "Resource
      temporarily unavailable" instead of EOF

## What landed

15 files, +347/-115 lines.  Two new files (`framework/console.{h,c}`),
a new `NX_HANDLE_CONSOLE` enum entry, two new `nx_process` fields
(`parent_pid` + `reaped`), and a refactor of `sys_read`/`sys_write`/
`sys_handle_close`/`sys_dup3`/`sys_wait` + `nx_channel_recv` +
`nx_process_exit` + `nx_process_fork`.

### `framework/console.{h,c}` — new files

Singleton sentinel `g_nx_console` (just `int g_nx_console;` — the
*address* is what matters; handle dispatch only checks the type tag)
plus two ops:

- `nx_console_write(buf, len)` — per-byte `uart_putc` loop on kernel,
  no-op-success on host.  Mirrors `sys_debug_write` but goes through
  the handle layer.
- `nx_console_read(buf, cap)` — returns 0 = EOF in v1 (UART RX not
  wired).  Slice 7.6d.N.final replaces this with a real receive path
  (IRQ-driven line buffer + wait queue).

Plus a `nx_console_write_calls` / `nx_console_reset_for_test` pair
mirroring `nx_syscall_debug_write_calls` for tests that need to
verify "EL0 program reached UART through stdio fd 1/2".  Pre-7.6d.N.6b
those tests watched `debug_write_calls`; the rename reflects the
new dispatch path.

`framework/handle.h` gains `NX_HANDLE_CONSOLE` between `NX_HANDLE_DIR`
and `NX_HANDLE_TYPE_COUNT`.

### `nx_process_create` pre-installs CONSOLE at slots 0/1/2

After `nx_handle_table_init(&p->handles)` the create function does
three `nx_handle_alloc` calls back-to-back:

```c
nx_handle_alloc(&p->handles, NX_HANDLE_CONSOLE, NX_RIGHT_WRITE, &g_nx_console, &h0);  // slot 0 → STDOUT_FILENO=1
nx_handle_alloc(&p->handles, NX_HANDLE_CONSOLE, NX_RIGHT_WRITE, &g_nx_console, &h1);  // slot 1 → STDERR_FILENO=2
nx_handle_alloc(&p->handles, NX_HANDLE_CONSOLE, NX_RIGHT_READ,  &g_nx_console, &h2);  // slot 2 → STDIN_FILENO=0 (h==0 routing)
```

The encoded handle values 1/2/3 land at slots 0/1/2 deterministically
because `nx_handle_alloc`'s linear scan starts at slot 0 on a fresh
table.  POSIX `STDOUT_FILENO=1` / `STDERR_FILENO=2` map to encoded
1/2 directly; `STDIN_FILENO=0` has no natural encoding (encoded value
0 is reserved for `NX_HANDLE_INVALID`) so it's routed via a `h == 0`
special case in the syscall layer.

This implicitly reserves slots 0/1/2 — `pipe()` allocations now land
at slot 3+ (encoded 4+), avoiding the POSIX-fd collision noted in
slice 7.6d.N.6a.

### Syscall-layer refactor

**`sys_read`** — the existing `h == 0` fallback (which formerly
looked up slot 0 and returned 0 on empty) now looks up *slot 2*
(STDIN console) and returns `NX_ENOENT` on empty (no magic).  Adds
a `NX_HANDLE_CONSOLE` arm dispatching to `nx_console_read`.  The
`NX_HANDLE_CHANNEL` arm wraps the `nx_channel_recv` call in a
yield-loop:

```c
for (;;) {
    got = nx_channel_recv(obj, staging, cap);
    if (got != NX_EAGAIN) break;
    nx_task_yield();
}
```

This converts non-blocking `EAGAIN` into POSIX-blocking read
semantics for pipelines.  The yield-loop terminates either with
data (positive return) or EOF (0, when peer is closed, see channel
update below).

**`sys_write`** — deletes the magic-fd fallback `if (rc == NX_ENOENT
&& (h == 1 || h == 2)) return sys_debug_write(...)`.  Adds a
`NX_HANDLE_CONSOLE` arm dispatching to `nx_console_write` after
`copy_from_user`-staging the payload.

**`sys_handle_close`** — adds `h == 0` routing to slot 2 (STDIN
console).  Type-aware destructor switch unchanged for CHANNEL/FILE/
DIR; CONSOLE's "destructor" is a no-op (singleton).

**`sys_dup3`** — `newfd == 0` now installs at slot 2 (was slot 0
in 7.6d.N.6a; slot 0 is now permanently reserved for STDOUT
CONSOLE).  All other shapes unchanged.

**`sys_wait`** — `pid == (uint32_t)-1` (POSIX waitpid-any-child)
loop scans `g_process_table` via new `nx_process_find_exited_child`
helper.  Returns the first EXITED non-reaped child; if no EXITED
children but at least one ACTIVE, yields and retries; if no children
at all, returns `-10` (Linux `ECHILD`) directly.  Critically NOT
`NX_ENOENT = -4` because musl's `__syscall_ret` translates the
return to userspace `errno`, and `-4` collides with `EINTR` —
ash sees EINTR, retries, infinite spin.  Marks `target->reaped =
true` after status delivery so a subsequent `waitpid(-1)` doesn't
return the same zombie.

### Channel: pipe-EOF on writer-closed

`nx_channel_recv` previously returned `NX_EAGAIN` when the queue
was empty regardless of peer state.  Slice 7.6d.N.6b adds:

```c
if (e->head == e->tail) {
    if (peer_of_const(e)->closed) return 0;   // EOF
    return NX_EAGAIN;
}
```

So a read on an empty pipe whose writer has been fully closed returns
0 = EOF.  Combined with the `sys_read` yield-loop, this gives
POSIX-shape pipe semantics: read blocks while writers attached,
returns EOF when last writer closes.

### `nx_process_exit` closes channel handles

The exiting process's handle table gets walked at exit time:

```c
for (i = 0..NX_HANDLE_TABLE_CAPACITY) {
    e = &p->handles.entries[i];
    if (e->type == INVALID) continue;
    if (e->type == CHANNEL && e->object) nx_channel_endpoint_close(e->object);
    nx_handle_close(&p->handles, computed_handle);
}
```

CHANNEL endpoints' `handle_refs` decrement; when the last ref drops
the endpoint becomes `closed`, which is what triggers the
peer-side EOF.  CONSOLE entries are singletons (no per-handle
destructor); FILE/DIR are deferred to a real reap-on-wait.

The process struct itself is NOT freed here — `wait()` still needs
the `state` + `exit_code` visible.  Real reap-on-wait (free the
struct) is a follow-up; the new `reaped` boolean is the v1
substitute, hiding zombies from `nx_process_find_exited_child` once
their status has been collected.

### `nx_process_fork` parent tracking

`child->parent_pid = parent->pid;` set right after `nx_process_create`
returns.  Drives the new `nx_process_find_exited_child` helper.

The handle-inheritance loop stays at "CHANNEL only" (slice 7.6a's
shape).  `CONSOLE` doesn't need fork inheritance because
`nx_process_create` already pre-installs CONSOLE at the child's
slots 0/1/2 with the same shape as the parent's.  Edge case: if
the parent dup3'd a CHANNEL onto slot 0/1/2 *before* fork, the
child gets the pre-installed CONSOLE there, NOT the parent's
redirected CHANNEL — the workloads in 7.6d.N.6b dup3 *after* fork
in each child, so this isn't currently exercised.  Slot-position-
preserving fork is a follow-up.

### Test updates

- **`test/host/process_test.c`** — count assertions account for the
  3 pre-installed CONSOLE handles (`count == 3` after fresh create,
  `count == 5` after two VMO allocs).  Test name renamed to
  `process_create_allocates_fresh_pid_and_pre_installs_console_handles`.
- **`test/host/file_syscall_test.c`** — `sys_write_to_unopened_stdout_
  routes_to_debug_write` test deleted (premise gone with magic-fd
  fallback).  Replaced with `sys_write_to_console_handle_routes_
  through_nx_console_write` — manually pre-installs CONSOLE handles
  (the host fixture skips `nx_process_create`) and verifies fd 1/2
  writes route through the type-aware dispatch.  Uses `memset(t, 0,
  sizeof *t)` to also reset slot generations (the kernel process's
  table accumulates generation history across tests; without the
  zero-init the encoded handles are 1025/1026/1027 instead of 1/2/3).
- **`test/kernel/ktest_syscall.c`** — `syscall_handle_close_invalid_
  handle_returns_einval` renamed to `..._zero_handle_with_no_stdin_
  returns_enoent`; expectation flipped from `NX_EINVAL` to
  `NX_ENOENT` (h=0 now routes to slot 2; slot 2 empty in
  g_kernel_process → ENOENT).
- **`test/kernel/ktest_posix_musl.c` + `ktest_posix_musl_printf.c`**
  — switched from `nx_syscall_debug_write_calls() >= N` to
  `nx_console_write_calls() >= N` (musl write fd 1 now goes through
  CONSOLE, not debug_write fallback).
- **`test/kernel/ktest_musl_exec.c`** — split: parent's libnxlibc
  markers still go through `debug_write` (libnxlibc shortcuts fd 1/2
  to NX_SYS_DEBUG_WRITE), child's musl markers go through
  `console_write`.  Asserts `debug_write >= 2 && console_write >= 1`
  for total ≥ 3.

### Re-enabling the pipe ktest

`Makefile`: `KTEST_C` includes `test/kernel/ktest_posix_busybox_sh_pipe.c`
again (the `#`-comment block from slice 7.6d.N.6a is gone).
`KTEST_S` adds `test/kernel/posix_busybox_sh_pipe_prog_blob.S`.
The .c/.S/.elf rules from 7.6d.N.6a stay as-is.

## Test results

Live ktest log (final):

```
posix_busybox_sh_pipe_parent_forks_and_execs_busybox_sh_pipe
[bbsh-pipe-parent]hello
[bbsh-pipe-status=00][bbsh-pipe-ok]PASS
```

The pipeline runs end-to-end: ash creates the pipe, forks twice for
echo + cat, redirects via dup3, execs both stages, waits for both
via `waitpid(-1)`, exits with status 0.  Cat's read-loop blocks on
empty pipe until echo's `nx_process_exit` closes the writer, then
sees EOF (0 bytes), exits cleanly.

`make test` → **415/415 pass** (51 python + 275 host + 89 kernel),
0 leaks, 0 errors.

## Debugging arc

The bring-up took several discovery iterations:

1. **First attempt: cat saw "Resource temporarily unavailable"** —
   pipe was working (data flowed), but cat's read-after-EOF returned
   `NX_EAGAIN` instead of 0.  Fixed by adding the `peer->closed →
   return 0` branch in `nx_channel_recv`.

2. **Second attempt: cat hangs after reading "hello"** — tracing
   showed `[chan-FINAL]` events confirmed write_e WAS closing
   eventually, and `recv-empty peer-closed=1` confirmed my code
   correctly returned EOF when the writer was closed.  But cat
   still blocked.  Root cause: the writer hadn't closed YET at
   the time cat's first empty-read happened — echo had emitted
   "hello" but the close-on-exit hadn't run.

3. **Third attempt: added exit-time channel close in
   `nx_process_exit`** — broke 5 unrelated kernel tests because
   I'd left a stale build artifact.  Full clean fixed those.

4. **Fourth attempt: cat exits, but ash hangs** — tracing showed
   ash calling `wait pid=4294967295 [wait-noent]` repeatedly.
   `wait4(-1)` was unimplemented.  Added pid=-1 support via new
   `nx_process_find_exited_child` helper.

5. **Fifth attempt: ash gets both children's status, then
   spins** — `[wait-any-nochild]` repeating forever.  Root cause:
   `NX_ENOENT = -4` translated to musl `errno = EINTR`, ash
   retried.  Returned `-10` (Linux `ECHILD`) directly.

6. **Sixth attempt: ash got both then would re-find the same
   exited child** — `wait4(-1)` returned echo, then echo again,
   etc.  Root cause: zombies stay in `g_process_table` after
   wait — second `find_exited_child` returns the same one.
   Added `reaped` boolean, set after wait copies status.

Total debug instrumentation added: 4 different kprintf points.
All removed before commit.

## What this unblocks

Slice 7.6d.N.6 closes the first-pipe milestone.  The remaining
items in the slice 7.6d.N.* arc are:

- **7.6d.N.7+**: harder pipelines (3+ stages, redirections to/from
  files, `&&`/`||`, command substitution).  Each will likely
  surface another small kernel gap.
- **7.6d.N.final**: interactive `busybox sh` over UART RX —
  currently `nx_console_read` returns 0 = EOF unconditionally.
  Needs UART RX IRQ + per-process line buffer + `sigaction`
  for SIGINT + `ioctl(0, TIOCGWINSZ)` stub + `tcsetattr` stub.

The structural pieces (CONSOLE, slot reservation, magic-fd-removal,
exit-time channel close, waitpid-any, pipe-EOF) are all now in
place and don't need to be revisited for those follow-ups.

## Next

Resume slice 7.6d.N.7+ (next escalation in the busybox-shell
pipeline) — exact target chosen based on the smallest workload
that surfaces a new kernel gap.  Candidates: `cat /banner` (file →
stdout pipeline, exercises FILE handle inheritance across exec),
`ls / | wc -l` (3-stage pipeline if we count ash, or just a 2-stage
that exercises a non-builtin downstream), or `echo a > /tmp/foo`
(stdout redirection to a file — needs O_CREAT + O_TRUNC + writes
into ramfs).
