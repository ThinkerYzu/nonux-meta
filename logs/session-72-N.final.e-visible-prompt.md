# Session 72: slice 7.6d.N.final.e — visible interactive busybox prompt

**Date:** 2026-04-28
**Phase:** Phase 7 (in progress); slice 7.6d.N.final
**Branch:** master

---

## Goals

- Make `make run-busybox` show the busybox prompt **before** the user
  types anything.  Session 71's deliverable booted an interactive
  shell, but the prompt was invisible — typing landed in `read(0)`,
  ash dispatched commands, output was visible, but no `# ` ever
  rendered before input.  Subjectively the shell felt broken even
  though it worked.
- Pin down the *actual* root cause (the slice 71 log proposed "musl
  line-buffers stdout" as the hypothesis without verifying the chain).
- Land the smallest possible fix; preserve `make test` 432/432; keep
  `make test-interactive` passing.

## What Was Done

### 1. Root-cause investigation

Two-pass exploration confirmed and *corrected* the slice 71 hypothesis:

- **Slice 71 said**: musl line-buffers stdout when `isatty(1) == 1`,
  ash's prompt has no trailing newline, so the prompt sits in musl's
  FILE buffer until the next newline-terminated write retroactively
  flushes it.
- **Slice 71 also said**: the fix would be "real `tcsetattr` so ash
  unbuffers itself" — that turned out to be **wrong**.  ash never
  uses termios state to choose stdio buffering; the buffering choice
  is entirely musl-internal, made on the first write to a tty.

Concrete trace:

- `third_party/busybox/.config` → `CONFIG_FEATURE_EDITING=y`.  This
  takes ash's prompt path through `libbb/lineedit.c`.
- `lineedit.c:2704–2712` — `putprompt()` with FEATURE_EDITING just
  *stores* the prompt string in `cmdedit_prompt`; no write.
- `lineedit.c:503–509` — `put_prompt_custom()` calls
  `fputs_stdout(prompt)`.  **No `fflush` follows.**
- `libbb/xfuncs_printf.c:317–320` — `fputs_stdout(s)` is bare
  `fputs(s, stdout)`.
- `third_party/musl/src/stdio/stdout.c:5–18` — musl's `__stdout_FILE`
  is initialised with `.lbf = '\n'` (line-buffered).
- `third_party/musl/src/stdio/__stdout_write.c:4–10` — on first
  write, musl issues a `TIOCGWINSZ` ioctl probe; on **failure**
  (non-tty) it sets `.lbf = -1` (unbuffered).  Our slice 70
  ioctl stub returns 0 (success) for `TIOCGWINSZ` against a
  CONSOLE handle, so musl correctly leaves stdout line-buffered for
  our tty case.

Net: ash writes `# ` (no `\n`) into musl's stdio, musl holds it.  Ash
then enters its read loop (which uses `read(STDIN_FILENO)` directly
via lineedit, not via stdio — so nothing flushes the buffer).  The
user types blind until the next newline-terminated write happens.

Critically: the **non-FEATURE_EDITING** path at `lineedit.c:3025–3032`
already calls `fflush_all()` after the same `fputs_stdout(prompt)`.
The bug is asymmetry between the two paths.  Verified by reading
both branches.

### 2. Fix (vendored busybox patch)

Single-line addition to `libbb/lineedit.c:put_prompt_custom`:

```c
fputs_stdout((is_full ? cmdedit_prompt : prompt_last_line));
+/* nonux patch (slice 7.6d.N.final.e): force-flush ... */
+fflush(stdout);
cursor = 0;
```

**Why `fflush(stdout)` and not `fflush_all()`?**  The
non-FEATURE_EDITING path uses `fflush_all()` (= `fflush(NULL)`).
That works there because the next call is `fgets(stdin, ...)` —
stdin's stdio buffer is filled by fgets immediately afterward.  In
our case, lineedit reads stdin via the `read()` syscall directly
(bypassing stdio), so there are no read-ahead bytes in stdin's FILE
buffer.  But `fflush(NULL)` *also* discards unread bytes in any
stdin FILE buffer — and the `make test-interactive` harness
trickle-feeds bytes that musl's stdio might pre-buffer in some
configurations.  Using the targeted `fflush(stdout)` removes any
risk of racing the harness.  9-line nonux comment block matches the
musl-patch convention used in
`third_party/musl/arch/aarch64/syscall_arch.h`.

### 3. Side fix: `test/interactive/run.sh` line-burst → per-byte trickle

After landing the patch, `make test-interactive` started failing
the `echo_pipe` script even though `make run-busybox` (manual eyeball
test) worked perfectly.  Investigation pinned this down to a separate
preexisting bug exposed by the visible prompt:

QEMU's `-serial stdio` chardev does **not** backpressure when
PL011's 16-byte hardware RX FIFO is full — surplus bytes are silently
dropped at the host/guest boundary.  Our software RX ring (256 bytes,
slice 7.6d.N.final.a) drains the hardware FIFO via IRQ, but if the
host writes 29 bytes faster than the IRQ-handler can drain a 16-byte
FIFO, the surplus is lost on the host side before the kernel ever
sees it.

**Why slice .d's `printf '%s\n' "$line"; sleep 1` worked anyway
*before* slice .e**: baseline (invisible-prompt) busybox didn't echo
typed input back through CONSOLE → UART (the prompt buffer never
flushed, so neither did the input echo characters).  Per-line UART
traffic was just the final command output (~10 bytes for
`PIPE_INPUT\n`).  With visible prompts in slice .e, ash echoes every
typed byte back, doubling the UART traffic per line and tipping the
host→guest path past the FIFO drain rate.

Fix: switch the trickle from line-burst (`printf '%s\n' "$line"`) to
per-byte (`printf '%s' "$c"; sleep 0.05`).  Per-byte feeding gives
the IRQ handler time to drain the FIFO between each byte; the
per-byte cost (50 ms × ~30 bytes/line = 1.5 s/line) is invisible
against the 60 s end-to-end test budget.  Verified by 3× consecutive
runs of `make test-interactive`: 9/9 pass (3 scripts × 3 runs).

### 4. New regression script

`test/interactive/visible_prompt.{script,expected}`:

- `.script`: `exit\n`
- `.expected`: ` #` (the busybox prompt prefix; the final `#` plus
  trailing space `# ` is what ash actually emits, but the substring
  match in `run.sh` only needs the prefix)

This pins down the slice .e fix as a regression test — without the
patch, the captured log would only contain `[init] entering busybox
sh at EL0` plus boot output.

### 5. Doc updates

- `IMPLEMENTATION-GUIDE.md` — added slice 7.6d.N.final.e block + bumped
  Last-Updated.
- `HANDOFF.md` — replaced Session 71 headline with Session 72 (visible
  prompt) headline; added slice .e to the Next-Milestone list under
  7.6d.N.final; added the new "Slice 7.7 (NEXT)" item at the top of
  Next Actions; pushed Session 67 from the live list into
  HANDOFF-ARCHIVE.md (5-session window enforced); refreshed Last
  Updated + Previous chain.
- `TESTING-GUIDE.md` — replaced the "No prompt is currently printed"
  caveat with the new expected eyeball recipe (`# echo HELLO`); added
  the per-byte trickle explanation; added the QEMU-FIFO-no-
  backpressure note to the limitations list.
- `README.md` — bumped both Last-Updated lines; refreshed Previous
  chain.
- `logs/session-72-N.final.e-visible-prompt.md` (this file).

## Key Findings

- **The slice 71 hypothesis pointed at the right symptom but wrong
  fix.**  Real termios state would not have helped — ash is buffering-
  agnostic; musl makes the line-buffered choice solely from a
  TIOCGWINSZ probe result.  A fix on the kernel side (force
  TIOCGWINSZ to fail, making musl unbuffer) would have removed the
  prompt visibility issue, but at the cost of POSIX-violating
  per-byte writes for *all* programs.

- **Why busybox-on-Linux works without any fflush patch (post-mortem
  investigation).**  Empirically verified: glibc on Linux behaves
  identically to musl on nonux for `printf("# "); _exit(0)` — both
  produce empty output when stdout is a tty (line-buffered, no `\n`,
  `_exit` bypasses the atexit flush).  The actual flush trigger on
  Linux is in `libbb/lineedit.c:2553-2556` — the call sequence is
  `get_terminal_width()` → `parse_and_put_prompt()` (the un-flushed
  prompt write) → `ask_terminal()`.  `ask_terminal()` (lines 1908-
  1916) writes `\e[6n` (cursor-position-report query) and calls
  `fflush_all()`; the fflush is meant for the ESC sequence but
  incidentally flushes the buffered prompt too.  The whole branch
  is gated by `safe_poll(&stdin, 1, 0) == 0` (poll-with-zero-timeout
  for "is input pending").  On Linux that returns 0 (no input) →
  branch fires → prompt visible.  On nonux, `__NR_ppoll = 73` is
  not in `third_party/musl/arch/aarch64/syscall_arch.h:68-107`'s
  translation table; ppoll returns -ENOSYS, `safe_poll` returns -1,
  the `== 0` test fails, `ask_terminal()` skips, nothing flushes.
  So the divergence isn't really a libc buffering difference — it's
  an unimplemented kernel syscall breaking a busybox happens-to-
  work-on-Linux path.  Implementing a minimal `NX_SYS_PPOLL` that
  returns 0 (no events) for `timeout == 0` would restore the
  intended ask_terminal flush automatically and help any
  poll/select-using program — but ppoll proper-semantics is non-
  trivial (multi-fd readiness, timeouts, signal-mask juggling) and
  scope-creeps beyond v1.  Recorded as a future-slice candidate.
- **busybox's two prompt paths are inconsistent.**  The
  non-FEATURE_EDITING path at `lineedit.c:3027` already does
  `fflush_all()` after `fputs_stdout(prompt)`.  Only the
  FEATURE_EDITING path was missing it.  This is plausibly an upstream
  busybox bug worth reporting (or just preserving as a nonux-local
  patch since most distros run with FEATURE_EDITING and rarely hit it
  because their terminals are real ttys with their own per-byte
  echoing paths that flush implicitly).
- **QEMU `-serial stdio` does not backpressure on PL011 FIFO
  overflow.**  This is the underlying reason `make test-interactive`
  was always borderline-flaky; the line-burst trickle worked in slice
  .d only by accident.  Per-byte feeding is the right answer for
  scripted tests.
- **`fflush(NULL)` and `fflush(stdout)` are not interchangeable for
  trickle-fed test harnesses.**  Discarding stdin FILE-buffer
  read-ahead is a real semantic difference; the targeted form
  `fflush(stdout)` avoids the race even if our specific code path
  doesn't currently exercise it.

## Decisions Made

- **One-line vendored busybox patch over disabling FEATURE_EDITING**
  — Disabling would lose arrow keys + history + tab completion for
  the eyeball-test recipe; one-line patch keeps the UX intact.
- **`fflush(stdout)` over `fflush_all()`** — see Key Findings above.
- **Per-byte trickle over bumping line-burst sleep** — verified
  empirically: even 5 s boot + 4 s line gap was still flaky on the
  echo_pipe burst.  Per-byte is principled (matches the actual FIFO
  drain dynamic) rather than just "more sleep".
- **Visible_prompt regression script** — captures the slice .e fix
  as a stable test; cheap to maintain (2-line script, 1-line
  expected).
- **Push Session 67 to HANDOFF-ARCHIVE** — 5-session window enforced
  (memory: "Keep last 5 session logs in HANDOFF.md; move older ones
  to HANDOFF-ARCHIVE.md").

## Status at End of Session

- `make run-busybox` boots an interactive busybox sh **with a visible
  `# ` prompt** before the user types anything.  Tested manually:
  prompt visible → typed `echo HELLO` → output `HELLO` → next prompt
  visible → typed `exit` → shell exited cleanly (init kthread
  remains in wfe park, expected v1 behaviour).
- `make test` → **432/432 pass** (51 python + 277 host + 104 kernel),
  0 leaks, 0 errors, exit 0.  Unchanged from session 71 — no kernel
  changes this session.
- `make test-interactive` → **3/3 pass** (visible_prompt, echo_hello,
  echo_pipe).  Stable across 3× consecutive runs.
- Slice 7.6d.N.final closed at v1 (sub-slices a/b/c-minimal/d/e
  done).  Deferred: c-full (signal-handler trampoline), N.16
  (POSIX-fd-to-slot alignment).  Slice 7.7 (interactive smoke tests)
  is the next forward step.

## Next Steps

- **Slice 7.7 — Interactive smoke tests.**  Promote the eyeball
  recipe in TESTING-GUIDE.md into regression-tested canned scripts:
  `ls /` (works today — just script + expected files),
  `echo hello | cat` (works today via slice 7.6d.N.6b machinery),
  `mkdir /foo && touch /foo/bar && ls /foo` (needs new vfs/ramfs
  `mkdir` op + `touch` builtin — judgement call whether mkdir is
  scope or deferred), `ps` (needs `/proc/<pid>` ramfs view *or*
  `NX_SYS_GETPROCS` — likely a separate slice 7.7.x).  See HANDOFF.md
  Next Actions item 0 for the full breakdown.

- **Slice 7.8 (after 7.7) — Wait queues + `poll`/`ppoll`
  (foundation slice).**  Three sub-slices:
  - 7.8a: `core/sched/waitq.{h,c}` primitive + `NX_TASK_BLOCKED`
    state + scheduler integration (~150 lines).
  - 7.8b: `NX_SYS_PPOLL` syscall built on 7.8a + per-handle-type
    `add_to_pollset` ops (~150 lines).
  - 7.8c: migrate the three existing yield-loop call sites
    (`sys_read` CHANNEL arm, `sys_wait`, `nx_console_read`) onto
    waitqs **AND revert this session's `third_party/busybox/
    libbb/lineedit.c` `fflush(stdout)` patch** — once `ppoll`
    works, busybox's intended `ask_terminal()` flush path runs
    again on its own.  Keep
    `test/interactive/visible_prompt.{script,expected}` as a
    regression guard against re-introduction.

  This session's slice .e patch is therefore explicit cleanup
  debt: it lives in vendored busybox source until slice 7.8c
  lands and doesn't have to.

## Cleanup Debt Recorded

- **`third_party/busybox/libbb/lineedit.c:put_prompt_custom`**
  — the `fflush(stdout);` we added (with 9-line nonux comment
  block) is scheduled for removal in slice 7.8c once
  `NX_SYS_PPOLL` is wired.  Without ppoll, busybox's
  `safe_poll(stdin, 0) == 0` test fails on nonux, the
  `ask_terminal()` `\e[6n` + `fflush_all()` branch never runs,
  and our stand-in flush is the only thing keeping the prompt
  visible.  After 7.8b lands ppoll, the upstream lineedit
  works as designed; our patch becomes redundant and should be
  removed to keep the vendored busybox tree as close to upstream
  as possible.

---

**Files Changed:**
- `third_party/busybox/libbb/lineedit.c` — slice 7.6d.N.final.e
  patch: `fflush(stdout);` after `fputs_stdout(...)` in
  `put_prompt_custom`, with 9-line nonux comment block.
- `test/interactive/run.sh` — switch trickle from line-burst
  (`printf '%s\n' "$line"; sleep 1`) to per-byte
  (`printf '%s' "$c"; sleep 0.05`) + bump boot delay 3 s → 5 s + bump
  per-test timeout 30 s → 60 s + updated comment block explaining
  the FIFO-no-backpressure issue.
- `test/interactive/visible_prompt.script` (new) — `exit\n`.
- `test/interactive/visible_prompt.expected` (new) — ` #`.
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — added slice
  7.6d.N.final.e description block + bumped Last-Updated.
- `proj_docs/nonux/HANDOFF.md` — Headline/Tests/Latest-session-log
  refreshed; slice .e added to 7.6d.N.final list under Next
  Milestone; phase checklist updated; Session 72 added at top of
  Session Logs (item 1); Session 67 moved to HANDOFF-ARCHIVE.md;
  Last-Updated/Previous chain refreshed; Slice 7.7 promoted to top
  of Next Actions.
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 67 entry inserted
  at top of archive list.
- `proj_docs/nonux/TESTING-GUIDE.md` — Eyeball-test recipe updated to
  show visible prompt; "No prompt is currently printed" caveat
  removed; per-byte trickle explanation added; QEMU-FIFO-no-
  backpressure note added to limitations.
- `proj_docs/nonux/README.md` — both Last-Updated lines refreshed;
  Previous chain extended.
- `proj_docs/nonux/logs/session-72-N.final.e-visible-prompt.md` (this
  file).
