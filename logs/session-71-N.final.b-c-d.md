# Session 71 — slices 7.6d.N.final.b + .c (minimal) + .d

**Date:** 2026-04-28
**Branch:** master
**Goal:** Push through the remaining sub-slices in the 7.6d.N.final
arc — `/init` swap policy (b), Ctrl-C/Ctrl-D semantics (c), and
scriptable QEMU stdin injection (d).  Slice c was scoped down to a
minimal v1 (polled SIGTERM on Ctrl-C, EOF flag on Ctrl-D); the full
signal-handler trampoline is deferred.

---

## Outcome

`make test` → **432/432 pass (51 python + 277 host + 104 kernel),
0 leaks, 0 errors, exit 0**.  +2 ktests vs. session 70's 430/430:
both new tests live in `test/kernel/ktest_console_rx.c`.

`make test-interactive` (new) drives kernel-busybox.bin under QEMU
with two canned scripts and confirms expected output.

`make run-busybox` boots an interactive busybox sh.  Tested
manually: `echo HELLO` runs end-to-end; busybox writes the result
to UART; bytes typed at QEMU's stdin (`-serial stdio`) reach
`read(0, ...)` from EL0 via the slice 7.6d.N.final.a RX ring.

---

## Slice 7.6d.N.final.b — `/init` swap policy

**Goal:** New build target `make run-busybox` produces a kernel binary
that auto-execs busybox sh as init, with the QEMU UART wired to the
host TTY.  Default `/init` (init_prog.elf, used by ktests) stays
unchanged.

**Deliverables:**

- `tools/pack-initramfs.py`: gains `--busybox-init=PATH` flag.  When
  set, any entry whose archive-name is `/init` gets its on-disk path
  rewritten to PATH.  Lets the same `INITRAMFS_ENTRIES` Makefile
  variable feed both the test build (`/init = init_prog.elf`) and the
  busybox-init build (`/init = busybox`) without restructuring.
  ~12 lines including arg parser.

- `test/kernel/init_busybox_prog.S`: hand-rolled EL0 stub (~70
  bytes, layout-identical to `user_prog_exec.S`).  Builds
  `argv = {"sh", NULL}` on the user stack via `adr` (PC-relative —
  the link-time absolute would be wrong since the blob is copied
  verbatim to user-window base 0x48000000).  `svc #0` with x8 = 14
  invokes `NX_SYS_EXEC("/init", argv, NULL)`; busybox sees
  `basename(argv[0]) == "sh"` and dispatches to the ash applet (same
  trick slice 7.6d.N.0 established).  Falls back to `exit(1)` on
  exec failure.

- `core/boot/boot.c`: gains `#ifdef NX_INIT_BUSYBOX` block (~50
  lines).  `nx_init_busybox_main()` creates an "init" process,
  spawns a kthread bound to it, then idles in `wfi`.  The kthread
  copies the EL0 stub into the process's user backing,
  `dsb/ic/dsb/isb` for cache coherency, computes
  `sp_el0 = (top - 16) & ~0xf`, and `drop_to_el0`s.

- `Makefile`: new `INITRAMFS_ENTRIES` variable (lazy `=` because
  `BUSYBOX_BIN` is defined further down); `initramfs-busybox.cpio`
  rule running pack-initramfs with `--busybox-init`; new
  `initramfs_busybox_blob.S` (parallel to `initramfs_blob.S` but
  `.incbin`s the busybox-init cpio); `core/boot/boot-busybox.o`
  rule with `-DNX_INIT_BUSYBOX`; `kernel-busybox.{elf,bin}` link
  rules; `make run-busybox` target invoking QEMU with
  `-nographic -kernel kernel-busybox.bin`.  ~30 lines tooling +
  blob.S.

- `proj_docs/nonux/TESTING-GUIDE.md`: gains an "Interactive busybox
  shell" recipe under Running Tests, plus a known-limitations
  callout listing v1's caveats.

**Bring-up trace.**  First build of kernel-busybox.bin booted past
the framework composition dump but emitted no busybox output.  Added
diagnostic kprintfs in `nx_init_busybox_kthread` (entry print) and
in `sys_exec` (path read, byte count, ELF parse, entry handoff);
confirmed:
- `[init] entering busybox sh at EL0` → kthread reaches drop_to_el0
- `[sys_exec] entered path="/init"` → SVC dispatched correctly
- `[sys_exec] read 1224584 bytes` → ramfs slurp + read worked (full
  busybox binary; no truncation)
- `[sys_exec] elf parsed, entry=0x480043b4` → ELF parse succeeded
- `[sys_exec] handing off to entry=0x480043b4 sp=0x487fff60` →
  trap-frame setup landed

After exec succeeded, busybox was running but silent.  Adding a
ring-byte trace (`[isr] got 0x..`) confirmed bytes were arriving
when piped via `-serial stdio` and busybox was processing them
correctly — the prompt-not-flushing was a stdio-buffering quirk, not
a kernel bug.  All diagnostic kprintfs reverted before commit.

**Caveat captured in TESTING-GUIDE.md.**  Without a real signal-
handler trampoline (deferred to a future slice c-full), `isatty(1) →
1` causes musl to line-buffer ash's prompt, but ash's prompt has no
trailing newline.  The shell IS reading input — typed bytes are
processed even though the prompt is invisible until the first
command output produces a newline.

---

## Slice 7.6d.N.final.c — Ctrl-C / Ctrl-D (MINIMAL)

**Goal:** Recognise Ctrl-C (0x03) and Ctrl-D (0x04) as control bytes
in the RX path; provide tractable v1 semantics for both.

**Spec deviation: minimal version.**  The original spec called for
promoting N.12's sigaction stubs from no-op to actually-installs-
handler, plus a full kernel-side signal trampoline (handler frame on
the EL0 stack, `NX_SYS_RT_SIGRETURN` to restore).  That's real
kernel-mode work — careful trap-frame manipulation, AAPCS-conformant
handler invocation, sigreturn semantics, re-entrancy concerns —
and was higher-risk than slice b/d.  Punted to a future
**`7.6d.N.final.c-full`** sub-slice.  The minimal version landed
here gives users a tractable Ctrl-C / Ctrl-D experience without the
trampoline:

- **Ctrl-C (0x03)** → posts SIGTERM to every ACTIVE non-kernel
  process via the existing polled `sched_check_resched` delivery
  path.  In v1 with no process groups, this brings the whole shell
  session down (busybox dies; the init kthread is left in `wfe`).
  Acceptable as a "force-quit-out-of-the-shell" gesture.

- **Ctrl-D (0x04)** → arms an EOF flag.  `nx_console_read` consumes
  it once the ring is drained: returns 0 (POSIX read EOF) on the
  next call, then the flag clears.  busybox sees end-of-input on
  its main loop and exits cleanly.

**Deliverables:**

- `framework/console.{h,c}`: two new `_Atomic int` flags
  (`g_eof_pending`, `g_intr_pending`); RX ISR + `_test_inject_bytes`
  *no longer* push 0x03/0x04 into the ring (live-RX-ISR path
  intercepts them as control events; the test inject keeps pushing
  raw bytes verbatim so existing 0..0x7F-byte-range ktests don't
  trip control events accidentally — separate
  `_test_inject_intr` / `_test_inject_eof` APIs let tests drive the
  control-byte paths explicitly); `nx_console_read` checks
  `g_eof_pending` via `__atomic_exchange_n` once-shot and returns 0
  if armed; new `nx_console_drain_intr()` posts SIGTERM to every
  ACTIVE non-kernel process when `g_intr_pending` is set, returns 1
  if drained.  ~75 lines net.

- `framework/syscall.c`: `sys_read` calls
  `nx_console_drain_intr()` at entry — any read syscall is the ack
  point for the input-side control byte.  Without this, the CONSOLE
  arm yields, never noticing the pending interrupt.  ~5 lines.

- `test/kernel/ktest_console_rx.c`: +2 ktests
  (`console_eof_returns_zero`, `console_intr_posts_sigterm`); both
  assert the one-shot semantics (rearm + observe a second time).
  ~80 lines.

**Sub-slices remaining as `7.6d.N.final.c-full` (deferred):**
- Per-process `sigaction[NSIG]` table; promote
  `sys_rt_sigaction` from no-op stub to actually-installs-handler.
- Kernel-side signal frame setup on the EL0-return path: when a
  signal is pending and a handler is installed, push the saved
  trap_frame onto the user stack, set PC to handler, set x0 = signo,
  set LR to a small EL0 trampoline.
- `NX_SYS_RT_SIGRETURN` restores trap_frame from the user stack.
- ash's installed SIGINT handler then runs (resets line-edit state,
  re-prompts) instead of v1's kernel-kills-everyone.

---

## Slice 7.6d.N.final.d — Scriptable QEMU stdin injection

**Deliverables:**

- `tools/qemu-stdin-feed.sh`: thin wrapper that pipes a script file
  into `qemu -serial stdio -kernel kernel-busybox.bin`.  Captures
  combined output to `test/kernel-output-busybox.log`.  ~30 lines
  shell + heredoc help.

- `test/interactive/run.sh`: drives every `*.script` in the
  directory through QEMU; verifies `*.expected` lines appear in the
  captured log via `grep -F` (substring match — kernel's boot dump
  + ash's prompts share the UART stream and would tank a strict
  diff).  ~40 lines.

- `test/interactive/{echo_hello,echo_pipe}.{script,expected}`:
  two seed canned scripts.  echo_hello tests the simple `echo X` →
  builtin path; echo_pipe tests the cross-process pipe path
  (`echo PIPE_INPUT | tr a-z A-Z` — first 2-stage interactive
  pipeline since slice 7.6d.N.6b's bbsh-pipe).

- `Makefile`: new `make test-interactive` target.

**Bring-up.**  First run failed: `cat script | qemu` blasted bytes
into PL011 faster than the 16-byte hardware FIFO could drain.  Fix:
trickle the script in line-by-line with `sleep 1` between lines and
a `sleep 3` initial gap so the kernel finishes booting before the
first byte.  `sleep 2` post-script keeps QEMU's chardev pipe open
long enough to flush the last command's output.

**`make test-interactive` is opt-in.**  Each canned-script run takes
~10 s of wall-clock time (mostly the trickle-pace sleeps); it would
roughly double `make test`'s runtime.  Lives as a separate target;
CI can decide whether to invoke it.

---

## Notes / observations

- **PL011 RX flakiness when bytes arrive too early.**  Initial test
  runs occasionally lost the first command's output even when the
  kernel was clearly past `[init] entering busybox sh at EL0`.
  Adding the `sleep 3` initial delay in `test/interactive/run.sh`
  resolved it.  Suspected cause: QEMU's chardev pipe internally
  buffers + delivers in chunks, and bytes that arrive during ELF
  loading get queued in the chardev rather than the PL011 — by the
  time the PL011 IRQ fires, the chardev's queue is half-drained but
  some characters were dropped by the FIFO depth limit during a
  burst handover.  Real-time typing has no problem because each
  keystroke triggers an immediate IRQ.

- **Substring-match `.expected` rather than strict diff.**  The
  alternative — strip the kernel boot dump from the log before
  diffing — would have been cleaner but brittle (every boot-time
  log change would break every interactive test).  Substring match
  is robust to log-format changes and only false-positives if a
  later test's log stream coincidentally contains the searched
  string, which is unlikely with our distinctive uppercase markers.

- **Slice c-full is genuinely deferred, not just out of scope here.**
  The trampoline work is non-trivial: aarch64 calling convention
  preservation across the handler invocation, sigreturn from
  user-stack-corrupted state, re-entrancy of nested signals, etc.
  Best done in a dedicated session.  The minimal slice here gives
  most of the user-visible Ctrl-C/Ctrl-D experience until then.

---

## Test totals delta

|              | Session 70 | Session 71 | Δ   |
|--------------|------------|------------|-----|
| python       | 51         | 51         | 0   |
| host         | 277        | 277        | 0   |
| kernel       | 102        | 104        | +2  |
| **total**    | **430**    | **432**    | +2  |

Plus a new `make test-interactive` target (2 canned scripts;
opt-in, not part of `make test`).

---

## Next actions

1. **`7.6d.N.final.c-full`** — full signal-handler dispatch
   trampoline.  Promote `sys_rt_sigaction` from no-op to handler-
   storing; build EL0 signal frame on return path; add
   `NX_SYS_RT_SIGRETURN`; switch Ctrl-C from "kill everyone" to
   "post SIGINT, deliver via handler".  Estimated ~200-300 lines
   kernel + ktest using mock-RX 0x03 injection that confirms an
   EL0 handler ran.

2. **Slice 7.7** — interactive smoke tests.  Now blocked only on
   c-full; could partially proceed against the minimal slice c.
   Scripted harness drives `ls /`, `echo hello | cat`, `ps`,
   `mkdir /tmp` against the busybox shell over the serial console;
   captured output is compared against expected.  Closes Phase 7.

3. **Reap-on-wait (slice-7.7 follow-up).**  Have `sys_wait` call
   `nx_process_destroy(child)` after collecting exit status.
   Would let us drop `NX_PROCESS_TABLE_CAPACITY` +
   `MMU_MAX_ADDRESS_SPACES` from 128 back to 32.  Not blocking
   N.final-c-full.
