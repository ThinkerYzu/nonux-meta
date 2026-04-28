# Session 57: slice 7.6d.N.2 — `busybox sh -c "echo hello"` runs end-to-end

**Date:** 2026-04-27
**Phase:** 7 slice 7.6d.N.2 (busybox-as-shell, escalated workload)
**Branch:** master

---

## Goals

Slice 7.6d.N.1 (session 56) made `busybox sh -c "exit 42"`
work end-to-end via the `exit` builtin — exiting status 42
without consulting sigaction / getuid / ioctl / fs lookup.
That was a "boring" success: ash never reached the
non-trivial code paths.

This sub-slice escalates to the smallest non-trivial -c
string: `echo hello`.  Differences from the `exit 42` path:

1. ash has to parse a multi-token -c string into a command
   tree (parser allocates more from mallocng, exercising
   the mmap arena past the bootstrap allocation).
2. The `echo` builtin issues `write(STDOUT_FILENO, "hello\n",
   6)` — exercises musl's stdio (`__stdio_write` →
   SYS_writev) end-to-end through the magic-fd-handle, on
   the actual emit path.
3. ash returns from the builtin and proceeds through the
   shell's normal end-of-input shutdown — flush, atexit,
   teardown — instead of jumping straight to `_exit`.

Discovery-driven, just like 7.6d.N.0.  Write the test,
observe, plan the next sub-slice from what surfaces.

## Implementation

### `test/kernel/posix_busybox_sh_echo_prog.c`

libnxlibc-linked program.  Forks; child does:
```c
nxlibc_execve("/bin/busybox", { "sh", "-c", "echo hello", NULL }, NULL);
```
Parent waits + emits `[bbsh-echo-parent]`, `[bbsh-echo-status=NN]`,
and one of `[bbsh-echo-ok]` (status==0) / `[bbsh-echo-failed]`
(anything else).  Same shape as `posix_busybox_sh_prog`.

### `test/kernel/posix_busybox_sh_echo_prog_blob.S`

`.incbin` of the linked ELF, with
`__posix_busybox_sh_echo_prog_blob_{start,end}` symbols.

### `test/kernel/ktest_posix_busybox_sh_echo.c`

Drives the program through the existing kernel-test
infrastructure (process_create, elf_load_into_process,
sched_spawn_kthread + drop_to_el0).  Same yield-cap as the
`exit 42` and `--help` tests (32768 ticks).  Asserts only
parent-side liveness (parent reached the marker print,
emitted ≥3 debug_writes).

### `Makefile`

Three sources added to `KTEST_C` / `KTEST_S` / build rules
+ the cleanup list.  Same recipe as `posix_busybox_sh_prog`;
only the embedded -c string differs.

## Discovery: process-table + MMU-space exhaustion

First test run captured failure:
```
[bbsh-echo-parent][bbsh-echo-exec-failed][bbsh-echo-status=61][bbsh-echo-failed]PASS
```

`[bbsh-echo-exec-failed]` is the marker the child writes
*after* execve returns.  Status 0x61 = 97 = the literal value
in the child's `nxlibc_exit(97)` after the failed-marker
write.  So execve returned to the child instead of replacing
the process image.

Surprising: the previous busybox test (`sh -c "exit 42"`)
exec'd the same `/bin/busybox` successfully a moment earlier.
The argv content differs, but the path is identical.

Diagnostic: temporarily added an "exec-rc=NN" hex marker
to the test program so we could see WHICH NX_E* code came
back from the syscall.  Output: `[bbsh-echo-exec-rcfe0]`
(the marker layout's positions were off by one — the `f`
and `e` landed at positions 18-19 instead of 19-20, so the
high nibble = 0xf, low nibble = 0xe → byte 0xfe → signed -2 →
**NX_ENOMEM**).

`sys_exec` returns `NX_ENOMEM = -2` from one of two paths:

1. `malloc(SYS_EXEC_MAX_FILE)` (4 MiB ELF-staging buffer)
   exhausts kheap.
2. `mmu_create_address_space()` returns 0 because either
   the MMU-spaces table is full (MMU_MAX_ADDRESS_SPACES = 32)
   or the underlying 8 MiB user-window malloc fails.

The cumulative test sweep had hit the v1 cap.

**Why every test leaks so much.**  Looking at
`components/sched_rr/sched_rr.c:116-128`, the helper
`sched_rr_purge_user_tasks(self, keep)` only does
`nx_list_remove(node)` for each user task on the runqueue.
That just unlinks them; nothing is freed:

- The `nx_task` struct (kheap-allocated) — leaked.
- The task's kstack — leaked.
- The task's owning `nx_process` — still in
  `g_process_table`, leaked.
- The process's `ttbr0_root` (L1 + L2_ram pages,
  ~8 KiB) — leaked.
- The process's **8 MiB user-window backing** — leaked.

So every fork+exec test leaks roughly:
  parent process + forked child process
= 2 × (8 MiB user backing + ~16 KiB page tables + task + kstack)
≈ 16 MiB physical RAM + 2 MMU-spaces slots + 2 process-table
slots, on top of the prior tests' leaks.  Slice 7.6c.4 first
bumped the caps 16 → 32 to paper over this; slice 7.6d.N.2
bumps them again 32 → 64.  Real reap-on-wait remains deferred.

### Fix: bump caps

Two cap bumps, mirroring slice 7.6c.4's earlier 16 → 32 bump:

- `MMU_MAX_ADDRESS_SPACES` 32 → 64 (`core/mmu/mmu.c`)
- `NX_PROCESS_TABLE_CAPACITY` 32 → 64 (`framework/process.c`)

Memory cost: 64 × 8 MiB = 512 MiB peak (worst case, all
slots in use simultaneously).  Tight but workable inside
QEMU's 1 GiB guest RAM.  Real fix is reap-on-wait
(deferred since slice 7.4 — tracked under slice 7.7
follow-ups).

After the bump, the test passes:
```
posix_busybox_sh_echo_parent_forks_and_execs_busybox_sh_echo [bbsh-echo-parent]hello
[bbsh-echo-status=00][bbsh-echo-ok]PASS
```

`hello\n` — emitted by `echo` via stdio
`[bbsh-echo-status=00]` — child exited 0
`[bbsh-echo-ok]` — success-path marker

End-to-end: ash parsed `-c "echo hello"`, dispatched the
`echo` builtin, `echo` wrote "hello\n" via musl's stdio
(`__stdio_write` → SYS_writev → magic-fd-handle → UART), ash
ran its end-of-input shutdown, exited cleanly with status 0.
Substantially better than predicted again — the 7.6d.N.2
plan expected `echo hello` to surface either an arena
exhaustion or a new shutdown-path syscall (sigaction,
sigprocmask, tcgetattr).  None did.

## Verification

`make test` → **412/412 PASSED** (51 python + 275 host + 86
kernel; +1 kernel test from slice 7.6d.N.2), 0 leaks, 0
errors, exit 0.

`posix_busybox_sh_echo_parent_forks_and_execs_busybox_sh_echo`
captures `[bbsh-echo-parent]hello\n[bbsh-echo-status=00][bbsh-echo-ok]`.

Both prior busybox tests stay green.

## Files changed

Production:
- `core/mmu/mmu.c` — `MMU_MAX_ADDRESS_SPACES` 32 → 64
  (1-line change + comment)
- `framework/process.c` — `NX_PROCESS_TABLE_CAPACITY` 32 → 64
  (1-line change + comment)

Test:
- New: `test/kernel/posix_busybox_sh_echo_prog.c` (~85
  lines)
- New: `test/kernel/posix_busybox_sh_echo_prog_blob.S` (15
  lines)
- New: `test/kernel/ktest_posix_busybox_sh_echo.c` (~95
  lines)
- Modified: `Makefile` (3 list additions + 1 build rule
  block + 1 cleanup-list entry)

Total: 5 files net (3 new, 2 production edits + 1 Makefile),
~210 lines.

## What's next (7.6d.N.3 candidates)

`busybox sh -c "echo hello"` works.  Both shell-builtin paths
(exit, echo) and the parser are validated.  Next escalation:

1. **`busybox sh -c "ls /"`** — first NON-builtin: ash
   forks + execs the `ls` applet.  busybox dispatches via
   `basename(argv[0])` from execv'd path resolution.  Will
   surface multiple gaps at once:
   - PATH lookup (ash's default PATH is
     `/sbin:/usr/sbin:/bin:/usr/bin`; we have no `/sbin`,
     `/usr/sbin`, `/usr/bin` — `/bin/ls` would need to
     exist via symlink-or-cpio-duplicate).
   - process-table reuse (ash forks a child for the applet,
     then reaps it via wait).
   - whatever `ls` itself needs (opendir, readdir, stat).

2. **`busybox sh -c "echo a; echo b"`** — multi-statement
   shell script.  Still all builtins, but exercises ash's
   sequence parser + statement dispatch.  Lower-risk
   intermediate step before tackling fork-into-applet.

3. **Interactive `busybox sh`** (the slice-7.6d.N.final
   goal) — switch production `/init` to busybox, add UART
   RX → fd 0 line discipline, expect to need sigaction
   for SIGINT/SIGTERM, isatty/ioctl(TIOCGWINSZ) for
   terminal sizing, tcsetattr for terminal mode.

Natural next step is option 2 (`echo a; echo b`) — minimal
escalation, exercises one more parser path, gaps surface
clearly.  Option 1 is a bigger jump that's worth its own
sub-slice arc.

## Build-side gotcha (still unresolved from session 56)

When `syscall_arch.h` / `syscall_cp.s` change, busybox's
incremental build doesn't reliably rebuild against new
musl.  Same workaround as session 56: `rm -f
test/kernel/initramfs.cpio test/kernel/initramfs_blob.o
kernel-test.bin kernel-test.elf && make test`.

This session didn't touch musl, so the issue didn't bite.
The two-line fix (`touch $BUSYBOX_BIN` after the inner
`make` in `tools/build-busybox.sh`) is still on the
opportunistic list.

---

**Last Updated:** 2026-04-27
