# Session 77 — slice 7.8b: `NX_SYS_PPOLL` built on waitqs

**Date:** 2026-04-29
**Slice:** 7.8b — `NX_SYS_PPOLL` syscall on top of slice 7.8a's wait-queue primitive
**Outcome:** **Closed.**  Real readiness-driven blocking poll lands.  CHANNEL endpoints + the singleton CONSOLE grow per-handle pollset-listener machinery; producers (channel send/close, console RX ISR + test-inject + EOF-arm) wake every registered listener's parent waitq.  `make test` 452 → **453/453** (+1 EL0 ktest); `make test-interactive` 7/7.  No callers in production code path yet — busybox's `ask_terminal()` will pick up `safe_poll(stdin, 0) == 0` automatically because musl's `__NR_ppoll` now translates, but the slice 7.6d.N.final.e `fflush(stdout)` patch in `libbb/lineedit.c` is still in place; slice 7.8c reverts it.

---

## Goal

Slice 7.8a landed the wait-queue primitive (`core/sched/waitq.{h,c}`) with no production callers.  Slice 7.8b promotes that primitive into a userspace-facing readiness-poll syscall.

The slice plan (HANDOFF.md Next Actions §0 + IMPLEMENTATION-GUIDE.md §Slice 7.8b) called for:

1. Per-handle-type "register on this object's wakeups" mechanism for `NX_HANDLE_CHANNEL` + `NX_HANDLE_CONSOLE`.  `NX_HANDLE_FILE` / `NX_HANDLE_DIR` always-ready (no registration).
2. `sys_ppoll(struct pollfd *fds, nfds_t nfds, struct timespec *timeout, sigset_t *sigmask, size_t sigsz)` that walks the user array, registers listeners, computes initial readiness, blocks on a pollset waitq with deadline, re-checks on wake, and copies `revents` back.
3. Producer-side wakes: `nx_channel_send` (peer becomes readable), `nx_channel_endpoint_close` (peer-closed → POLLHUP), console RX ISR (byte arrived → POLLIN), `nx_console_test_inject_*` (test parity).
4. musl translation: `__NR_ppoll = 73 → NX_SYS_PPOLL = 41` in both `syscall_arch.h` and `syscall_cp.s`.
5. EL0 ktest covering: initial-ready (write-then-poll), deadline expiry (50 ms timeout, no producer), peer-close hangup (close-then-poll).

All landed.  Real cost across kernel + posix shim + EL0 test ≈ 450 lines of new code (vs. the plan's ~150 estimate, mostly because the listener mechanism is duplicated between two object types and a thin POSIX-shim wrapper grew alongside the syscall to drive the EL0 program).

---

## What landed

### `framework/pollset.h` — shared listener type (~50 lines)

New header.  Provides `struct nx_pollset_listener { struct nx_list_node node; struct nx_waitq *waitq; }` + an inline `nx_pollset_wake_all(list)` that walks a per-object listener list and calls `nx_waitq_wake_all` on each entry's parent waitq.  Channel and console both `#include` it; sys_ppoll uses it to allocate one listener per pollfd on the kstack.

Storage discipline: listeners are *borrowed* by the per-object lists — sys_ppoll's kstack owns the listener struct, the channel/console module just sees a `struct nx_list_node` linked into its list.  sys_ppoll unregisters every listener before returning so the per-object lists are always empty between calls.

### `framework/syscall.h` — ABI surface

```c
NX_SYS_PPOLL = 41,
struct nx_pollfd { int fd; short events; short revents; };  /* 8 bytes */
#define NX_POLLIN    0x001
#define NX_POLLOUT   0x004
#define NX_POLLERR   0x008
#define NX_POLLHUP   0x010
#define NX_POLLNVAL  0x020
#define NX_PPOLL_MAX_FDS 32
```

`NX_SYSCALL_COUNT` bumps 41 → 42 (sentinel).  POLL flag values match Linux for ABI compat with the musl `poll.h` exposed to busybox.

### `framework/channel.{h,c}` — per-endpoint listener list

`struct nx_channel_endpoint` grows `struct nx_list_head pollset_listeners` (initialised in `nx_channel_create`).  Three new public functions:

- `nx_channel_endpoint_register_pollset(e, listener)` — appends to tail (FIFO).
- `nx_channel_endpoint_unregister_pollset(listener)` — `nx_list_remove`; idempotent.
- `nx_channel_endpoint_readiness(e, want) → short` — POSIX-shape revents.  POLLIN if recv would not return NX_EAGAIN (ring non-empty OR peer closed); POLLOUT if peer's ring has space and neither side is closed; POLLHUP whenever peer is closed (always set independent of `want` — POSIX semantics).

Producer wakes:

- `nx_channel_send(e, ...)` after pushing to peer's ring: `nx_pollset_wake_all(&peer_of(e)->pollset_listeners)`.  Peer's read-readiness rose from "would EAGAIN" to "has data".
- `nx_channel_endpoint_close(e)` after the last close: wake both endpoints' lists.  Peer's listeners want the EOF transition; this side's listeners (if any survived a race) want POLLHUP-on-self.

### `framework/console.{h,c}` — singleton listener list

New static `g_console_pollset_listeners` (statically initialised self-pointing).  Three new public functions mirroring channel: `nx_console_register_pollset`, `nx_console_unregister_pollset`, `nx_console_readiness(want)`.

Console readiness: POLLIN if `rx_count() > 0` OR `g_eof_pending` is armed (Ctrl-D state from slice 7.6d.N.final.c).  POLLOUT always set (UART writes are unconditional).

Producer wakes, all in console.c:

- `nx_console_rx_isr` — after draining the PL011 FIFO into the ring + arming any control flags, calls `nx_pollset_wake_all` if a real byte was pushed OR a Ctrl-D EOF was armed.  (Ctrl-C alone does not wake — it doesn't change read-readiness.)
- `nx_console_test_inject_bytes` — host-test parity: wake after pushing.
- `nx_console_test_inject_eof` — host-test parity: wake after arming EOF.

### `framework/syscall.c` — `sys_ppoll` (~140 lines)

Flow:

1. Validate `nfds <= NX_PPOLL_MAX_FDS` (32); decode optional timeout (`struct timespec` shape: 16 bytes on aarch64; budget = `tv_sec * 1e9 + tv_nsec`).
2. `copy_from_user` the pollfd array into a kstack copy.
3. Walk handles.  STDIN special-case (encoded fd 0 → slot 2) mirrors `sys_read`.  Per-type registration:
   - `CONSOLE` → `nx_console_register_pollset`
   - `CHANNEL` → `nx_channel_endpoint_register_pollset`
   - `FILE` / `DIR` → no registration (always-ready)
   - bad lookup → `revents = NX_POLLNVAL`
4. **After registration**, compute initial readiness via a per-type dispatch (`ppoll_compute_readiness`).  Order matters: a wake during the registration→check window reaches the empty pollset waitq (no-op), but the underlying state is already mutated, so the readiness check catches it.
5. If any fd is ready OR `timeout` is zero (non-blocking), skip the sleep.
6. Otherwise `nx_waitq_wait_with_deadline(&pollset_wq, budget_ns)` — blocks on the kstack pollset's waitq.  Wake (NX_OK) or deadline expiry (NX_EDEADLINE) both fall through to a re-check.
7. Unregister every listener.  Copy revents back via `copy_to_user`.  Return ready count.

Dispatch table gains `[NX_SYS_PPOLL] = sys_ppoll`.

Used `nx_process_current()` rather than `nx_task_current()` for the handle-table lookup so the host build (which only includes `core/sched/task.h` under `#if !__STDC_HOSTED__`) compiles cleanly without pulling kernel-only inline asm.

### `third_party/musl` — translation

`arch/aarch64/syscall_arch.h`: `case 73:  return 41;`.
`src/thread/aarch64/syscall_cp.s`: `cmp x1, #73; b.eq .Lnx_ppoll` + `.Lnx_ppoll: mov x8, #41; b .Lnx_run`.

`make musl-libc` rebuilds; `touch third_party/musl/lib/libc.a` + `make busybox` relinks busybox against the new musl (the lesson from slice 7.7b.1's bring-up).  busybox shrinks/grows by exactly the bytes for the new translation entry.

### `components/posix_shim/posix.h` — EL0-direct shim

New `nx_posix_svc4` helper (4 args) so the EL0 test program can invoke ppoll without going through musl.  Plus:

```c
#define NX_POSIX_SYS_PPOLL  41
struct nx_posix_pollfd { int fd; short events; short revents; };
struct nx_posix_timespec { int64_t tv_sec; int64_t tv_nsec; };
#define NX_POSIX_POLLIN    0x001
... (mirror of NX_POLL* constants from syscall.h)

static inline int
nx_posix_ppoll(struct nx_posix_pollfd *fds, uint64_t nfds,
               const struct nx_posix_timespec *timeout,
               const void *sigmask);
```

The `sigmask` argument is accepted but ignored — same v1 posture as the existing rt_sigaction / rt_sigprocmask stubs.

### `test/kernel/posix_ppoll_prog.c` + `_blob.S` + `ktest_posix_ppoll.c`

EL0 program runs three subtests in sequence, each with a discrete failure exit code (1..11):

1. **Initial readiness.**  Open a pipe, write a byte, ppoll the read end with timeout=0.  Asserts ppoll returns 1 with `revents & POLLIN`.
2. **Deadline expiry.**  Fresh empty pipe, 50 ms timeout, no producer.  Asserts ppoll returns 0 with `revents == 0`.
3. **Peer-close hangup.**  Close the writer side, ppoll the reader side with timeout=0.  Asserts ppoll returns 1 with `revents & (POLLIN | POLLHUP)`.

If all three pass: emit `[ppoll-ok]` + `exit(29)`.  Matching ktest spawns the EL0 program in a fresh process, then waits for the EXITED state via a `nx_task_yield(); asm ("wfi");` loop (yields are insufficient — the deadline-expiry subtest needs wall-clock time to advance, which only timer interrupts do; `wfi` lets the CPU sleep until the next tick).  Asserts `exit_code == 29`.

### `Makefile`

`KTEST_S` gains `posix_ppoll_prog_blob.S`; `KTEST_C` gains `ktest_posix_ppoll.c`; new build rules for `posix_ppoll_prog.{o,elf}` follow the slice-7.5 pattern (same `POSIX_PROG_CFLAGS`, same `init_prog.ld`).  `clean` target gets the new ELF.

---

## Discoveries

1. **`nx_task_current()` is kernel-only.**  First-iteration `sys_ppoll` called `nx_task_current()` to fetch the current process's handle table; host build failed `error: implicit declaration` because `core/sched/task.h` is included under `#if !__STDC_HOSTED__` in `framework/syscall.c`.  Switched to `nx_process_current()` (in `framework/process.h`, included unconditionally) — same result.

2. **`NX_SYSCALL_COUNT` change requires explicit `.o` invalidation.**  First test run reported `syscall_unknown_number_returns_enosys` failing at the kernel ktest level: `svc0(NX_SYSCALL_COUNT)` returned `-22 = EINVAL` instead of `-38 = ENOSYS`.  Cause: `ktest_syscall.o` was compiled when `NX_SYSCALL_COUNT == 41` (so the test's "out-of-range" probe used 41 — which is now `NX_SYS_PPOLL`, a valid handler returning `EINVAL` for the trash arg in `x1`).  Fix: `rm test/kernel/ktest_syscall.o` and rebuild.  The Makefile's header-dep tracking doesn't catch enum-value bumps.  Future-self protection: bumping `NX_SYSCALL_COUNT` should be paired with a kernel-test rebuild.

3. **The `KASSERT_EQ_U(g_ppoll_entry, mmu_user_window_base())` invariant from slice 7.5's `posix_pipe_prog` ktest was too strict.**  My `posix_ppoll_prog.elf` has `nx_posix_exit` placed at offset 0 (compiler ordered the inline-helper before `_start`) so `e_entry == 0x4800000c`, not `0x48000000`.  posix_pipe_prog happened to land `_start` at offset 0 in its ELF; ours doesn't.  Fix: assert entry is *inside* the user window, not exactly at base — the `ENTRY(_start)` linker directive makes `e_entry` track the symbol regardless of layout.

4. **Yield-only wait loops can't observe a deadline-expiry wake.**  My first ktest used `for (i = 0; i < 4096; i++) nx_task_yield();` — the same shape as slice 7.5's pipe ktest.  Failed with `KASSERT(reached)`: 4096 yields finish in microseconds, but the deadline-expiry subtest needs ~50 ms of *wall clock* to elapse before `nx_waitq_tick_deadlines` wakes the EL0 task.  yields are context-switch primitives, not sleeps — they don't advance the timer.  Fix: emit `wfi` between yields (mirrors the idle-task body); CPU sleeps until the next timer tick fires + the wake happens via the IRQ-return path.  Loop bound dropped 4096 → 256 because each iteration is now ≥ 1 tick (100 ms), capping wall-time at ~25 s — still well inside the 90 s QEMU budget.

5. **Initial-readiness check after listener registration is the right ordering.**  Classic doubled-locked-check pattern: register listeners FIRST, then check readiness.  A wake that fires during the gap reaches an empty pollset waitq (no-op), but the readiness predicate has already mutated, so the check catches it.  If we checked readiness first then registered, a wake in the gap would update state but the listener wouldn't yet be plumbed to wake us, and we'd sleep with no chance of being woken.

---

## Test results

```
make test:           453/453 pass (51 python + 283 host + 119 kernel,
                                    0 leaks, 0 errors, exit 0)
make test-interactive: 7/7 pass
                       (echo_cat + echo_hello + echo_pipe + ls_root +
                        mkdir_tmp + ps_smoke + visible_prompt)
```

Up from session 76's baseline of `make test` 452/452 (51 python + 283 host + 118 kernel).  +1 ktest (`posix_ppoll_initial_ready_deadline_and_peer_close`).

The new ktest emits `[ppoll-ok]` after all three subtests pass; failure of any subtest exits with a discrete code (1..11) that the ktest's exit-code KASSERT pinpoints.

---

## Cleanup debt + next steps

The slice-7.6d.N.final.e busybox patch in `third_party/busybox/libbb/lineedit.c:put_prompt_custom` (the `fflush(stdout)` after the prompt write) is still in place.  It was added because busybox's intended path — `ask_terminal()` calling `fflush_all()` after `\e[6n` — is gated on `safe_poll(stdin, 0) == 0`, which fails if `__NR_ppoll = 73` is unmapped.  Now that ppoll is wired, busybox's intended path will run, and the patch can be reverted.

That revert + migrating the three existing yield-loops (`sys_read` CHANNEL arm, `sys_wait`, `nx_console_read`) onto `nx_waitq_wait_with_deadline` is **slice 7.8c**.  Outcome after .c: yield-loop CPU waste under load is gone (Phase 9 benchmarks no longer noisy); the lineedit patch is removed (no vendored-busybox local diff to maintain).
