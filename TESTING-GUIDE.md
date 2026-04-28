# Testing Guide: nonux

**Scope:** How to run tests, what each test validates, expected output, and troubleshooting.

**Current test count (Phase 3 slice 3.2):** 62 host-side + 6 in-kernel = **68 tests**.

---

## Navigation

**Project Docs:** [README](README.md) | [SPEC](SPEC.md) | [DESIGN](DESIGN.md) | [IMPLEMENTATION-GUIDE](IMPLEMENTATION-GUIDE.md) | [HANDOFF](HANDOFF.md) | [TESTING-GUIDE](TESTING-GUIDE.md) *(you are here)*

**This Document:**
- [Quick Start](#quick-start)
- [Two Harnesses, Two Scopes](#two-harnesses-two-scopes)
- [Test Inventory](#test-inventory)
- [Running Tests](#running-tests)
- [Expected Output](#expected-output)
- [Writing a New Test](#writing-a-new-test)
- [Troubleshooting](#troubleshooting)

---

## Quick Start

```bash
cd sources/nonux
make test
```

Expected: host tests (62 pass) then kernel tests (6 pass) then QEMU exits cleanly with status 0. Final lines:

```
PASS: 62/62 tests passed, 0 leaks, 0 errors
...
ktest: 6/6 PASSED, 0 failures
```

If both sections report all-pass and `make` exits 0, the tree is healthy.

---

## Two Harnesses, Two Scopes

| Harness | Location | Compiler | What it validates |
|---|---|---|---|
| **Host** | `test/host/` | system `cc` + libc | Pure-C logic — PMM algorithm + mem_track + framework-level code (registry). Fast iteration; runs in ~milliseconds. PMM tests use a `posix_memalign`'d buffer to stand in for physical memory; framework tests use caller-owned auto/static structs. |
| **Kernel** | `test/kernel/` | cross `aarch64-linux-gnu-gcc`, freestanding | End-to-end kernel behavior: PMM against real physical RAM, IRQ path (vectors → GIC → dispatch → handler → ERET), timer. Boots under QEMU with `-semihosting`; the guest exits the VM with a status code that `make` propagates. |

Both harnesses use a self-registering `TEST()` / `KTEST()` macro: each test emits an entry into a dedicated linker section and the runner walks it. Adding a test is one `TEST(name) { … }` block; no manual registration.

The host harness's `mt_track` layer catches leaks and red-zone overflows in test code, but framework code that allocates internally (e.g. `framework/registry.c`) uses plain `malloc`/`free` so its lifetime is decoupled from the per-test `mt_reset()`. Framework tests verify "no internal leaks" by asserting on counts (e.g. `nx_graph_*_count() == 0`) after a register/unregister cycle.

---

## Test Inventory

### Host — `test/host/pmm_test.c` (11 tests — PMM + mem_track)

| # | Test Name | What It Validates |
|---|-----------|-------------------|
| 1 | `pmm_init_counts_pages_and_reserves_bitmap` | After `pmm_init`, `pmm_total_count` / `pmm_free_count` are correct and the bitmap's own pages are reserved. |
| 2 | `pmm_alloc_page_returns_nonnull_and_aligned` | First alloc is non-NULL, page-aligned, and the free count drops by 1. |
| 3 | `pmm_alloc_distinct_pages` | Two allocs return different pages. |
| 4 | `pmm_alloc_until_exhausted_then_null` | After `total` allocs, the next returns NULL and free count is 0. |
| 5 | `pmm_free_then_realloc` | Freeing a page and allocating again returns the same page (hint reuse). |
| 6 | `pmm_full_cycle_no_leak` | Alloc all, free all → free count equals initial total. |
| 7 | `pmm_alloc_pages_contiguous_two` | 2-page contiguous alloc is page-aligned; next single alloc is distinct from both. |
| 8 | `pmm_alloc_pages_too_big_returns_null` | Requests larger than the pool return NULL rather than succeeding. |
| 9 | `pmm_alloc_pages_finds_contiguous_range_after_fragmentation` | Freeing the middle 3 of 5 allocated pages creates a gap that `pmm_alloc_pages(3)` finds. |
| 10 | `mem_track_tracks_live_and_released` | `mt_live_count` goes up on tracked_alloc and down on tracked_free. |
| 11 | `mem_track_zero_initializes_payload` | Tracked allocations hand back zero-filled memory. |

### Host — `test/host/registry_test.c` (51 tests — Component Graph Registry)

Exercises `framework/registry.c`. Every test starts with `nx_graph_reset()` so it begins on a clean global state regardless of what the previous test left behind. Caller-owned `struct nx_slot` / `struct nx_component` are auto-storage in the test body.

#### Slice 3.1 — registry skeleton (25 tests)

| # | Test Name | What It Validates |
|---|-----------|-------------------|
| 1 | `registry_starts_empty` | After `nx_graph_reset()`, all counts and the generation counter are zero. |
| 2 | `slot_register_and_lookup` | A registered slot is findable by name; count goes to 1; generation bumps. |
| 3 | `slot_register_duplicate_name_fails` | Second register with the same name returns `NX_EEXIST` and leaves count at 1. |
| 4 | `slot_register_null_inputs_fail` | NULL slot or NULL `name` field returns `NX_EINVAL`. |
| 5 | `slot_lookup_misses_return_null` | Lookup of an unregistered name returns NULL. |
| 6 | `slot_unregister_removes_entry` | Unregister drops the slot from lookup and decrements count. |
| 7 | `slot_unregister_with_active_connection_returns_ebusy` | `nx_slot_unregister` refuses to drop a slot still referenced by a connection edge — caller must tear wiring down first. |
| 8 | `component_register_and_lookup` | Register + `(manifest_id, instance_id)` lookup roundtrip. |
| 9 | `component_register_duplicate_pair_fails` | Same `(manifest, instance)` pair returns `NX_EEXIST`. |
| 10 | `component_register_distinct_instances_of_same_manifest` | `uart_pl011/uart0` and `uart_pl011/uart1` coexist. |
| 11 | `component_state_set_updates_state_and_bumps_gen` | State transition writes through to the caller's struct and bumps generation. |
| 12 | `component_unregister_blocked_while_active_in_a_slot` | `EBUSY` while a slot still names this component as `active`; succeeds after `slot_swap(s, NULL)`. |
| 13 | `slot_swap_sets_active_and_bumps_gen` | Swap binds the new component into `slot->active` and bumps generation. |
| 14 | `slot_swap_to_unknown_component_fails` | Swap to a component that was never registered returns `NX_ENOENT` and leaves `active` untouched. |
| 15 | `connection_register_basic` | New connection appears in global count; `from_slot` and `to_slot` survive round-trip; `installed_gen` is set. |
| 16 | `connection_register_with_null_from_is_a_boot_edge` | `from_slot == NULL` is legal — used by the boot sequence's initial wiring. |
| 17 | `connection_register_null_to_fails` | `to_slot == NULL` returns `NX_EINVAL`. |
| 18 | `connection_register_unknown_slot_fails` | An unregistered endpoint returns `NX_ENOENT`. |
| 19 | `connection_retune_changes_mode_and_stateful` | `nx_connection_retune` updates mode and stateful flag and bumps generation. |
| 20 | `connection_unregister_removes_from_lists` | Unregister cleans both the global list and the per-slot incoming/outgoing lists; afterwards the slots are unrefe­renced and can themselves be unregistered. |
| 21 | `graph_foreach_visits_every_slot` | Global slot enumeration visits every registered slot exactly once. |
| 22 | `graph_foreach_visits_every_component_and_connection` | Same, for components and connections. |
| 23 | `slot_neighbourhood_walks_in_and_out_edges` | `slot_foreach_dependent` walks incoming edges; `slot_foreach_dependency` walks outgoing. |
| 24 | `generation_monotonically_increases` | Generation only ever goes up across register / retune / unregister. |
| 25 | `register_unregister_cycle_leaves_no_residue` | 100 cycles of register-all → unregister-all leaves all counts at zero (catches internal leaks). |

#### Slice 3.2 — events, change log, snapshot, JSON (26 tests)

| # | Test Name | What It Validates |
|---|-----------|-------------------|
| 26 | `subscribe_then_register_slot_fires_slot_created_event` | A subscribed callback receives `NX_EV_SLOT_CREATED` when a slot is registered; payload `slot.name` round-trips. |
| 27 | `subscribe_observes_full_event_taxonomy` | One scripted scenario emits one of every event type (12 events: 2× created, 2× swapped, 1× component_registered, 1× component_state, 1× connection_added, 1× retuned, 1× removed, 1× component_unregistered, 2× destroyed) and verifies each union payload. |
| 28 | `unsubscribe_stops_delivery` | After `nx_graph_unsubscribe`, no further events arrive. |
| 29 | `subscribe_duplicate_returns_eexist` | Re-subscribing the same `(cb, ctx)` pair returns `NX_EEXIST`. |
| 30 | `subscribe_distinct_ctx_is_not_a_duplicate` | Same callback with different `ctx` is a separate subscriber; both receive each event. |
| 31 | `subscribe_null_callback_is_einval` | `nx_graph_subscribe(NULL, …)` returns `NX_EINVAL`. |
| 32 | `unsubscribe_unknown_pair_is_a_noop` | Unsubscribing a never-subscribed `(cb, ctx)` doesn't crash. |
| 33 | `change_log_starts_empty_after_reset` | After `nx_graph_reset()`, `total = size = 0`. |
| 34 | `change_log_records_each_mutation` | Two mutations produce two log entries with monotonically-increasing generations. |
| 35 | `change_log_read_filters_by_since_gen` | `nx_change_log_read(since_gen, …)` returns only events with `generation > since_gen`. |
| 36 | `change_log_overflow_keeps_most_recent` | 600 events into a 256-slot ring → `total=600`, `size=256`, oldest retained generation = 345. |
| 37 | `change_log_read_max_caps_output` | `max` parameter caps the returned count. |
| 38 | `change_log_read_with_null_or_zero_returns_zero` | NULL output or zero max returns zero, doesn't crash. |
| 39 | `graph_reset_clears_subscribers_and_log` | After reset, log is empty and no subscribers fire on subsequent mutations. |
| 40 | `snapshot_of_empty_registry_is_empty` | `nx_graph_snapshot_take()` of a clean registry returns a snapshot with all-zero counts. |
| 41 | `snapshot_captures_current_state` | Snapshot of a wired registry surfaces the right slot/component/connection counts and `active_manifest`/`active_instance`. |
| 42 | `snapshot_is_stable_against_subsequent_mutation` | Slot count and generation in a held snapshot don't change when the live registry mutates. |
| 43 | `snapshot_refcount_keeps_alive_until_last_put` | `_retain` + two `_put`s keep the snapshot alive across the first `_put`. |
| 44 | `snapshot_put_null_is_a_noop` | `nx_graph_snapshot_put(NULL)` doesn't crash. |
| 45 | `snapshot_index_out_of_range_returns_null` | Out-of-range indexed accessors return NULL. |
| 46 | `snapshot_to_json_empty_shape` | JSON of empty snapshot contains `"slots":[]`, `"components":[]`, `"connections":[]`, `"generation":0`. |
| 47 | `snapshot_to_json_full_shape` | JSON of a wired snapshot contains the expected enum strings (`"hot"`, `"shared"`, `"uninit"`, etc.) and identifiers. |
| 48 | `snapshot_to_json_truncated_returns_enomem` | Tiny buffer returns `NX_ENOMEM` and remains NUL-terminated. |
| 49 | `snapshot_to_json_escapes_special_chars` | Names containing `"` and `\` are emitted as `\"` and `\\`. |
| 50 | `change_log_to_json_emits_each_event` | Change log JSON contains `"total"`, `"held"`, and one event-object per logged event with the correct `"type"` strings. |
| 51 | `change_log_to_json_handles_null_optional_strings` | A connection with `from_slot=NULL` (boot edge) emits `"from":null` rather than crashing. |

### Kernel — `test/kernel/ktest_pmm.c` (4 tests)

| # | Test Name | What It Validates |
|---|-----------|-------------------|
| 1 | `pmm_alloc_page_is_aligned_and_above_kernel` | Real-PMM alloc is page-aligned and lives above `__free_mem_start` (i.e., not overlapping the kernel image or stack). |
| 2 | `pmm_alloc_page_is_writable_and_readable` | Allocated page actually maps to usable RAM — writes to offsets 0, 8, and `PAGE_SIZE-8` read back the same values. |
| 3 | `pmm_full_cycle_restores_free_count` | 128 real-PMM allocs followed by 128 frees restore the free count exactly. |
| 4 | `pmm_contiguous_alloc_is_contiguous` | `pmm_alloc_pages(4)` gives a single contiguous 16 KiB block; first and last bytes are writable. |

### Kernel — `test/kernel/ktest_irq.c` (2 tests)

| # | Test Name | What It Validates |
|---|-----------|-------------------|
| 5 | `irq_timer_ticks_advance_under_busy_wait` | Spinning on `CNTPCT_EL0` for 250 ms sees `timer_ticks()` advance, proving the EL1 physical timer IRQ (PPI 30) fires and the handler runs. |
| 6 | `irq_wfi_wakes_on_timer` | A `wfi` instruction wakes on the next timer IRQ — proves the full IRQ path delivers to a halted CPU. |

---

## Running Tests

### Full suite

```bash
make test
```

Runs `make test-host` then `make test-kernel`. Fails (non-zero exit) if either side fails.

### Host-only

```bash
make test-host
```

Builds and runs `test/host/test_host` on the dev machine. Fast — useful while iterating on algorithm changes.

### Kernel-only

```bash
make test-kernel
```

Builds `kernel-test.bin` (same objects as `kernel.bin` except `boot.c` is recompiled with `-DNX_KTEST` so `boot_main` dispatches into `ktest_main` after setup), boots it under QEMU with `-semihosting`, and propagates the guest's exit code. An outer `timeout 15` catches hangs.

### Single kernel test

There isn't a per-test filter yet — all kernel tests run together. To iterate on one test, comment out the others' `KTEST(...)` blocks or move them out of the registry temporarily. A filter is a future polish item.

### Production kernel (no tests)

```bash
make                         # builds kernel.bin
tools/run-qemu.sh -t 4       # boots it; prints [tick] once/sec
```

`kernel.bin` does not include ktest code.

### Interactive busybox shell (slice 7.6d.N.final.b)

```bash
make run-busybox             # builds kernel-busybox.bin and runs it in QEMU
```

This builds a kernel binary that links the same initramfs as
`kernel-test.bin` *except* `pack-initramfs.py --busybox-init` rewrites
the `/init` entry to be the busybox blob.  `boot.c` (compiled with
`-DNX_INIT_BUSYBOX`) creates an `init` process at boot, drops it to EL0
with a small stub that issues `execve("/init", { "sh", NULL }, NULL)` —
busybox dispatches to the ash applet via `basename(argv[0])`.  PL011 RX
(slice 7.6d.N.final.a) wires bytes typed at the QEMU UART into the
`read(0, ...)` path in EL0, so anything you type lands in ash's line
editor.

**Eyeball-test smoke recipe.**  Drive `make run-busybox` from a host
terminal so QEMU's `-serial stdio` connects your TTY to the kernel UART.
After the framework composition dump, you'll see the busybox `# ` prompt
(slice 7.6d.N.final.e wired the lineedit-side `fflush`).  Type a line
and press Enter:

```
[init] entering busybox sh at EL0
sh: can't access tty; job control turned off
 # echo HELLO
HELLO
 # exit
```

Quit QEMU with **Ctrl-A x** (the QEMU monitor escape) — ash's `exit`
builtin runs but does not yet propagate cleanly to a kernel halt; the
init kthread is left in a wfe park until a future slice wires up real
exit-and-shutdown semantics.  Slice 7.6d.N.final.d adds
`tools/qemu-stdin-feed.sh` + `make test-interactive` for canned-script
regression coverage.

Known v1 limitations (eyeball-test caveats):
- Ctrl-C arms the SIGTERM-to-all-user-processes path (slice
  7.6d.N.final.c minimal); the shell session ends rather than just
  the foreground command, because v1 has no process groups and no
  signal-handler dispatch.  Real ash-style "interrupt the current
  command, return to prompt" lands once the trampoline is wired
  (deferred slice `7.6d.N.final.c-full`).
- Ctrl-D returns 0 (EOF) on the next `read(0, ...)`; ash exits
  cleanly because end-of-input on its main loop terminates the
  shell.
- The QEMU `-serial stdio` chardev does not backpressure when PL011's
  16-byte hardware RX FIFO is full — surplus bytes are silently
  dropped at the host/guest boundary.  Real-time typing is fine
  (humans don't push >16 bytes within a single FIFO drain interval);
  pasted long lines may drop characters.  The `make test-interactive`
  harness works around this by feeding bytes per-character with a
  50 ms gap (see `test/interactive/run.sh`).
- ash prints `sh: can't access tty; job control turned off` at start
  because `tcgetpgrp(2)` fails on our `NX_HANDLE_CONSOLE` (we have no
  process groups).  The shell still runs interactively; only job-
  control features (`fg`, `bg`, `&` backgrounding with terminal
  signal delivery) are disabled.

### Scripted busybox tests (slice 7.6d.N.final.d + .e)

```bash
make test-interactive        # runs each test/interactive/*.script
                             # against kernel-busybox.bin and verifies
                             # output via test/interactive/*.expected
```

Each script lands inside QEMU's UART one byte at a time (with a 50 ms
gap between bytes so PL011's 16-byte hardware FIFO doesn't overflow —
slice 7.6d.N.final.e changed the trickle from line-burst to per-byte
once visible prompts doubled per-line UART traffic and exposed the
silent byte-drop that line-bursts had been getting away with); the
captured kernel log + busybox stdout is checked for substring matches
against the corresponding `.expected` file.  Logs are written to
`test/kernel-output-busybox-<name>.log`.

Current scripts (3): `visible_prompt` (asserts `# ` appears in the
captured log — the slice .e regression), `echo_hello`
(`echo HELLO_FROM_INTERACTIVE`), `echo_pipe`
(`echo PIPE_INPUT | tr a-z A-Z` — first multi-process pipeline).
3/3 pass, stable across consecutive runs.

The substring match (rather than strict `diff`) is intentional —
busybox prompts and the kernel's framework-bootstrap dump share the
same UART stream and would tank a strict comparison.  A `.expected`
should list only the lines the test cares about.

`tools/qemu-stdin-feed.sh SCRIPT [TIMEOUT]` is the underlying driver
if you want to play a script manually:

```bash
tools/qemu-stdin-feed.sh test/interactive/echo_hello.script
```

---

## Expected Output

### `make test-host`

```
Running 36 test(s):
  pmm_init_counts_pages_and_reserves_bitmap PASS
  pmm_alloc_page_returns_nonnull_and_aligned PASS
  ...
  mem_track_zero_initializes_payload       PASS
  registry_starts_empty                    PASS
  slot_register_and_lookup                 PASS
  ...
  register_unregister_cycle_leaves_no_residue PASS

PASS: 36/36 tests passed, 0 leaks, 0 errors
```

### `make test-kernel`

```
========================================
  nonux — composable microkernel
  ARM64 / QEMU virt
========================================

[boot] kernel loaded at 0x40080000 (QEMU's -kernel offset)
[boot] BSS:  0x40084000 — 0x40084844
[boot] kernel end:    0x40085000
[boot] free memory:   0x400c5000 — 0x80000000
[pmm]  total=261939 free=261939 pages (1047756 KiB)
[cpu]  exception vectors installed at 0x40080800
[gic]  distributor + CPU interface enabled
[timer] cntfrq=62500000 Hz, interval=6250000 ticks, rate=10 Hz

=== ktest: running 6 test(s) ===
  pmm_contiguous_alloc_is_contiguous              PASS
  pmm_full_cycle_restores_free_count              PASS
  pmm_alloc_page_is_writable_and_readable         PASS
  pmm_alloc_page_is_aligned_and_above_kernel      PASS
  irq_wfi_wakes_on_timer                          PASS
  irq_timer_ticks_advance_under_busy_wait         PASS

ktest: 6/6 PASSED, 0 failures
```

### Known acceptable variations

- **Test order** in the kernel output may vary from the source order: it reflects the order `KTEST()` entries land in the `kernel_test_registry` linker section, which depends on link order, not source order.
- **PMM page counts** change if you adjust `QEMU_MEM` or if the kernel image size changes; `261939` is for 1 GiB of RAM with the current image.

---

## Writing a New Test

### Host test (pure algorithm)

Add to `test/host/pmm_test.c` (or a new file under `test/host/`):

```c
TEST(your_test_name)
{
    /* setup — for PMM tests, call setup_region() */
    /* exercise */
    /* ASSERT / ASSERT_EQ_U / ASSERT_NOT_NULL / ASSERT_NULL */
    /* teardown */
}
```

If you added a new `.c` file, list it in `test/host/Makefile` under `SRCS`.

### Framework test (registry / lifecycle / IPC / hooks)

Add to `test/host/registry_test.c` (or a new file alongside it):

```c
#include "test_runner.h"
#include "framework/registry.h"

TEST(your_test_name)
{
    nx_graph_reset();   /* mandatory — start on a clean global state */

    struct nx_slot s = { .name = "slot", .iface = "iface" };
    nx_slot_register(&s);
    /* exercise + ASSERT_* */
}
```

Things to know:

- **Always call `nx_graph_reset()` first.** The previous test may have left the registry holding pointers to its own auto-storage structs — those are now dangling. `nx_graph_reset()` drops all internal nodes without dereferencing the cached pointers, so calling it unconditionally is safe.
- **Slot/component storage is caller-owned.** The registry only bookkeeps pointers. Auto-storage is fine within a single test; never leak a caller-owned struct's address into the registry past the struct's lifetime.
- **Internal leaks are detected by counts**, not by `mt_track`. Pattern: do work, then `ASSERT_EQ_U(nx_graph_*_count(), 0)` after teardown. The 100-cycle test (`register_unregister_cycle_leaves_no_residue`) is the canonical example.
- **No threads, no SMP.** Slice 3.1 is single-threaded; concurrency primitives land with the recomposer.

If you added a new `.c` file, list it in `test/host/Makefile` under `SRCS` and (if it pulls in additional framework code) under the `vpath` line.

### Kernel test (real hardware path)

Add to `test/kernel/ktest_pmm.c` or `test/kernel/ktest_irq.c` (or a new file):

```c
#include "ktest.h"
#include "core/some/thing.h"

KTEST(your_test_name)
{
    /* exercise real kernel APIs */
    /* KASSERT / KASSERT_EQ_U / KASSERT_NOT_NULL / KASSERT_NULL */
}
```

If you added a new `.c` file, list it in the top-level `Makefile` under `KTEST_C`.

Kernel tests run with IRQs unmasked and the timer armed at 10 Hz. Treat the environment as "real" — no setjmp/longjmp, `KASSERT*` on failure prints + sets a flag + `return`s. Don't clobber global state needed by later tests (the runner does not reset PMM state between tests; alloc what you need, free before returning).

---

## Troubleshooting

### `make test-kernel` hangs until the 15-second timeout

**Symptom:** No output or output stops mid-run; the `timeout` wrapper kills QEMU.

**Likely cause:** A kernel test caused an unhandled exception (sync fault, double fault) and the vector loops infinitely. Or an interrupt path is broken and a test is waiting on a tick that never fires.

**Fix:** Run with QEMU's interrupt log:

```bash
timeout 5 qemu-system-aarch64 -M virt,gic-version=2 -cpu cortex-a53 -nographic \
    -kernel kernel-test.bin -m 1G -semihosting \
    -d int -D /tmp/ktest-int.log
head -30 /tmp/ktest-int.log
```

The first exception record tells you which vector was taken and what `ELR_EL1` pointed at. If `ELR` is inside one of your tests, you have an illegal memory access or alignment fault in the test body.

### Kernel test output shows `%-40s` instead of test names

**Symptom:** Literal `%-40s` in the runner output.

**Cause:** `kprintf` only supports the formats listed in `core/lib/printf.c` (`%s %d %u %x %p %c %lu %lx %ld`). Width modifiers (`%-40s`, `%5d`) are not parsed.

**Fix:** Pad manually with `uart_putc(' ')` in a loop (see `ktest_main` for the pattern), or extend `kprintf` if you need this widely.

### `undefined reference to __start_kernel_test_registry`

**Symptom:** Link failure when building `kernel-test.elf`.

**Cause:** GNU ld only auto-generates `__start_/__stop_` markers for *orphan* sections (not mentioned in the linker script). Once `KEEP(*(kernel_test_registry))` is placed inside an output section, the markers must be defined explicitly.

**Fix:** `core/boot/linker.ld` already defines them inside `.rodata`. If you see this error, something removed the markers — restore them.

### Host tests link but fail with `Relocations in generic ELF (EM: 183)`

**Symptom:** `cc` errors out linking `test_host`, complaining about aarch64 objects.

**Cause:** The host Makefile's VPATH picked up a stale `.o` from a prior cross-compile at the top.

**Fix:** `make -C test/host clean && make -C test/host`. The host Makefile uses `vpath %.c ...` (not plain `VPATH`) so only `.c` files are pulled from the kernel tree, which prevents this, but clean is always a reset.

### `make test-kernel` says `Error 1` but the tests all printed PASS

**Symptom:** Tests pass, Make still reports failure.

**Cause:** Semihosting isn't enabled — the guest's `hlt #0xf000` fell through to the vector handler, which halted but never called QEMU's exit path. QEMU eventually gets SIGTERM from the outer `timeout`, which returns 128+15=143.

**Fix:** Confirm the `-semihosting` flag is on the QEMU command line in `make test-kernel` (it should be — check the Makefile).

---

**Last Updated:** 2026-04-20 (Session 7 — registry tests added, 42 tests total)
