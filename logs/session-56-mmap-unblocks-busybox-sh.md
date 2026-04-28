# Session 56: slice 7.6d.N.1 — minimal anonymous `mmap` unblocks `busybox sh`

**Date:** 2026-04-27
**Phase:** 7 slice 7.6d.N.1 (busybox-as-shell, mmap addition)
**Branch:** master

---

## Goals

Close the gap surfaced in slice 7.6d.N.0 (session 55): musl
mallocng calls `mmap(MAP_PRIVATE|MAP_ANON)` for every slab
allocation, and our translation table left `__NR_mmap (222)`
unmapped → -ENOSYS → malloc NULL → ash's `bb_msg_memory_exhausted`
exit.

Add the smallest mmap implementation that covers the mallocng
shape, then re-run `posix_busybox_sh` and observe the next
failure mode.

## Scope choices

- **Bump-arena, not real demand-paging.**  The full memory rework
  (per-process VMAs + L3 4 KiB pages + page-fault-driven page
  allocation) is a multi-slice project and explicitly deferred
  to post-Phase-7 (per the long-term plan in HANDOFF.md → "Real
  per-process memory management").  This sub-slice keeps the
  current "8 MiB contiguous user backing per process" model and
  carves out a sub-region as a per-process anonymous-mmap
  arena.  Bump pointer only; no reclaim until process exit.

- **Arena placement: `[base + 2 MiB, base + 5 MiB)`.**  Sits in
  the unused window region between busybox's text+data+bss end
  (~+1.91 MiB) and the kernel-pre-init TLS area (+5 MiB).
  3 MiB is enough for ash's startup mallocng allocations
  (mallocng's first slab is 64 KiB, growing geometrically) and
  doesn't overlap with the brk heap at +6 MiB or the stack near
  the top.

- **No MAP_FIXED, no per-page protection.**  The user_window
  is uniformly EL0-RW; we don't have the L3 page tables to
  enforce per-page prot.  mallocng asks for MAP_FIXED only for
  guard pages — and ignores the return — so rejecting it is
  safe.

- **`munmap` as a no-op success.**  mallocng treats `munmap`
  failure as fatal (it ASSUMES it succeeds and forgets the
  pointer), so we can't return -ENOSYS.  PMM reclaim happens
  at process exit when the whole 8 MiB user_window is freed;
  for ash's startup we leak at most a few hundred KiB until
  the parent reaps the child.  Adequate for v1.

- **Pages are zeroed on the way out.**  MAP_ANONYMOUS's contract
  is zeroed pages.  Our underlying user_window backing is plain
  `malloc()` memory and may carry stale bytes from earlier exec
  cycles or from un-touched memcpy alignment slack.  `memset`
  the returned region to zero before returning the VA to
  userspace.

## Implementation

### `framework/syscall.h`

Two new enum values, `NX_SYS_MMAP = 19` + `NX_SYS_MUNMAP = 20`,
with header comments documenting the supported shape.
`NX_SYSCALL_COUNT` advances from 19 to 21 automatically.

### `framework/process.h`

New per-process bump pointer field:
```c
uint64_t mmap_bump;
```
Plus the layout constants:
```c
#define NX_PROCESS_MMAP_OFFSET  (2u << 20)   /* 2 MiB into window */
#define NX_PROCESS_MMAP_LIMIT   (5u << 20)   /* 5 MiB into window */
```
The window-layout comment is updated to include the new region.

### `framework/process.c`

Initialize `mmap_bump` to `mmu_user_window_base() +
NX_PROCESS_MMAP_OFFSET` in `nx_process_create` (kernel only;
host stays 0 since there's no MMU).  Inherit it across `fork`
the same way `brk_addr` is inherited — the forked backing
already contains the parent's mmap'd pages byte-for-byte.

### `framework/syscall.c`

`sys_mmap` body (~50 lines including comments):

1. Parse the 6 args.  Reject zero-length, non-(-1) fd, non-zero
   offset, missing MAP_ANONYMOUS, MAP_FIXED.  Per-page prot is
   silently ignored — we don't have it.
2. Round length up to 4 KiB.
3. Range-check the bump against the arena.  Lazy re-init if the
   bump drifts out of range (defensive for a hypothetical future
   munmap).
4. Allocate from the bump.  `memset` the returned region to 0.
5. Return the new VA.  Failure returns small negative
   (NX_EINVAL or NX_ENOMEM) which musl's `__syscall_ret` turns
   into MAP_FAILED + errno.

`sys_munmap` body: 1 line (`return NX_OK`) plus a header
comment explaining why arg validation is intentionally loose
(mallocng can pass arbitrary derived pointers).

`sys_exec` updates: the address-space-rebuild logic resets
`brk_addr`; same logic added for `mmap_bump`.  Three lines:
save old, reset to fresh, restore on rollback.

Dispatch table: two new entries (`NX_SYS_MMAP → sys_mmap`,
`NX_SYS_MUNMAP → sys_munmap`).

### `third_party/musl/arch/aarch64/syscall_arch.h`

Two new entries in `__nx_translate`:
```c
case 215: return 20;  /* __NR_munmap -> NX_SYS_MUNMAP */
case 222: return 19;  /* __NR_mmap   -> NX_SYS_MMAP */
```
Header comment block updated.

### `third_party/musl/src/thread/aarch64/syscall_cp.s`

Mirror of the C-level table for the cancellation-point
syscall path (matches the slice 7.6c.3a discipline of keeping
the asm-level table in lockstep with the C one):
```
cmp x1, #215;  b.eq .Lnx_munmap
cmp x1, #222;  b.eq .Lnx_mmap

.Lnx_munmap:   mov x8, #20; b .Lnx_run    // NX_SYS_MUNMAP
.Lnx_mmap:     mov x8, #19; b .Lnx_run    // NX_SYS_MMAP
```

## Result

Live ktest log fragment:

```
posix_busybox_sh_parent_forks_and_execs_busybox_sh [bbsh-parent][bbsh-status=2a][bbsh-ok]PASS
```

`[bbsh-status=2a]` = 0x2a = 42.  `[bbsh-ok]` is the marker
for the success path (`status == 42`).

End-to-end: ash startup ran without OOM, ash parsed `-c "exit 42"`,
ran the `exit` builtin, and exited cleanly with status 42.
Substantially better than expected — the 7.6d.N.0 plan
predicted that mmap would unblock malloc but ash would then
hit a sigaction / getuid / ioctl gap.  For the simple `-c
"exit 42"` path none of those were needed; ash exited via the
builtin directly without consulting any of those subsystems.

## Build issues encountered (and what to fix)

When `syscall_arch.h` and `syscall_cp.s` change, the top-level
`make musl-libc` rebuilds libc.a, but `make test` doesn't
propagate the change to busybox's link line.  Two reasons:

1. The Makefile rule `$(BUSYBOX_BIN): $(MUSL_LIBC)` re-invokes
   `tools/build-busybox.sh`, which delegates to busybox's own
   `make`.  busybox's incremental build only rebuilds .o files
   whose source/header dependencies changed — the libc.a is a
   link-time artifact, and the previous link's outputs are still
   considered up-to-date.  busybox's link step is skipped.

2. Even when busybox IS rebuilt (via `make busybox-clean &&
   make busybox`), the `initramfs.cpio` rule's $(BUSYBOX_BIN)
   prereq doesn't reliably trigger a regeneration on the next
   `make test` — the cpio stays at its old timestamp and so
   does kernel-test.bin.

Workaround used in this session: `rm -f
test/kernel/initramfs.cpio test/kernel/initramfs_blob.o
kernel-test.bin kernel-test.elf && make test` after the musl
patch.  Forces the full test-binary chain to rebuild against
the new busybox.

Real fix (deferred, opportunistic): make `tools/build-busybox.sh`
`touch $BUSYBOX_BIN` after the inner `make` so the timestamp
always advances even when nothing actually rebuilt.  Or add an
`.libc-stamp` file that the busybox link line consumes as a
phony "force re-link if libc changed" indicator.  Either is a
2-line change.  Lands when a future slice burns time on this
again.

## Verification

`make test` → **411/411 PASSED** (51 python + 275 host + 85
kernel), 0 leaks, 0 errors, exit 0.

Same test count as session 55 — no new ktests added.  The
`posix_busybox_sh_parent_forks_and_execs_busybox_sh` test
that captured the OOM in 7.6d.N.0 now captures status 42 +
emits `[bbsh-ok]`.  No code-side assertion change needed; the
test asserts parent liveness only.

## Files changed

Production:
- `framework/syscall.h` — 2 new enum values + comments (~60
  lines)
- `framework/syscall.c` — 2 new syscall bodies + 2 dispatch
  entries + sys_exec mmap_bump reset (~120 lines)
- `framework/process.h` — `mmap_bump` field + layout
  constants + comment update (~20 lines)
- `framework/process.c` — init + fork-inherit (~5 lines)

Vendored musl patches:
- `third_party/musl/arch/aarch64/syscall_arch.h` — 2 new
  translate-table entries
- `third_party/musl/src/thread/aarch64/syscall_cp.s` — 2 new
  cmp+branch + 2 new label blocks

Total: 6 files, ~210 lines of net change (mostly comments).

## What's next (7.6d.N.2 candidates)

`busybox sh -c "exit 42"` is now boring — the test exits
cleanly with status 42.  The next discovery-driven sub-slice
should escalate the workload to surface what's still missing:

1. **`busybox sh -c "echo hello"`** — exercises `write(1, ...)`
   from inside the shell (already works via stdio magic-fd
   handling) plus argv parsing + builtin dispatch from a
   non-trivial -c string.
2. **`busybox sh -c "ls /"`** — first non-builtin: ash forks +
   execs the `ls` applet via `/proc/self/exe` discovery (which
   we don't have) or via PATH lookup (no `/etc/profile` or
   `$PATH` in our env).  Will surface multiple gaps at once
   (process management, fs lookup, etc).  Skip until simpler
   tests stabilize.
3. **`busybox sh` (interactive, with a UART RX → fd 0 line
   discipline)** — the actual slice-7.6d.N.final goal.  Needs
   UART RX plumbing in the kernel (currently TX-only) plus
   ash's interactive setup (sigaction for SIGINT + SIGTERM,
   isatty / ioctl(TIOCGWINSZ), terminal mode set via tcsetattr).

The natural next step is option (1) — minimal escalation,
exercises one new busybox path (`echo` builtin), and any new
gap surfaces clearly.

## Long-term concerns

The bump-arena is fundamentally a v1 hack.  Real mmap should
be backed by demand-paged 4 KiB L3 entries with VMAs, so:
- `length` doesn't have to fit in a hardcoded 3 MiB region;
- pages aren't allocated until first touch (current scheme
  pre-allocates the entire 8 MiB user_window per process);
- `munmap` actually reclaims;
- `mprotect` becomes possible for the guard-page idiom.

The relevant DESIGN.md callout already exists in HANDOFF.md
under "Real per-process memory management (long-term,
post-Phase-7)".  This sub-slice doesn't change the long-term
plan — it just buys us busybox-shell coverage in the simple
v1 model.

---

**Last Updated:** 2026-04-27
