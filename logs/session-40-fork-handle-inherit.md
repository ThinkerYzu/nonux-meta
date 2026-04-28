# Session 40: slice 7.6a — fork handle-table inheritance for HANDLE_CHANNEL

**Date:** 2026-04-25
**Phase:** 7 slice 7.6a (busybox-prereq sub-slice)
**Branch:** master

---

## Goals

Open slice 7.6 by landing the prereq that has been called out in
HANDOFF since slice 7.4a — fork handle-table inheritance.  Without
it, the canonical POSIX `pipe + fork + close-the-other-side +
read/write` pattern can't work because the forked child has an
empty handle table and can't see the parent's pipe fds.  Slice
7.5's pipe demo had to be single-process for that reason.

Sub-scope to this session: HANDLE_CHANNEL only.  HANDLE_FILE
inheritance has thornier semantics (per-open vfs cursor) and lands
later.

## Scope choices

- **Per-endpoint refcount, not per-channel refcount.**  The
  existing channel layer kept a single `_Atomic int refcount` on
  `struct nx_channel` that started at 2 (one per endpoint).  Each
  `endpoint_close` flipped its endpoint's `closed` bit and
  decremented the shared count; on 0 the channel was freed.
  That works when each endpoint has exactly one handle holder.
  Fork inheritance breaks the invariant: after fork, the read
  endpoint has TWO handles (parent + child).  If parent closes
  its handle, the existing logic flips `closed = true` on the
  endpoint — but the child still wants to read from it!  The
  fix is to track refs per endpoint instead of per channel: each
  handle that points at the endpoint owns one ref, and only when
  the last ref drops do we close it.  When both endpoints' refs
  hit zero, the channel allocation is freed.
- **Retain/Release semantics in the channel API surface.**
  Added `nx_channel_endpoint_retain(e)` to bump the count when
  the framework duplicates a channel handle (i.e., during fork
  inheritance).  `nx_channel_endpoint_close` becomes the
  release primitive.  Symmetric, easy to reason about, future-
  proof for slice 7.6c–d busybox flows that may dup channel
  handles via other mechanisms (`dup`, `dup2` in posix_shim).
- **Inherit only HANDLE_CHANNEL in this slice.**  HANDLE_FILE
  inheritance needs vfs_simple to grow a `dup` op (clone the
  per-open with its own seek cursor) or for the framework to
  track cursors outside the driver.  Either is bigger than this
  slice.  Files-across-fork lands when busybox actually needs
  it — likely slice 7.6d when the shell pipes file output.  Note
  that this means a child inheriting from a parent with open
  files will see those file handles missing — POSIX-incompatible
  but acceptable for v1's "no real shell yet" state.
- **Walk the handle table linearly in sys_fork.**  The handle
  table is a fixed 64-entry array; a linear scan after
  `nx_process_fork` is the obvious shape.  Each CHANNEL entry
  triggers a retain + an alloc into the child's table.
  Allocation failure unwinds the just-issued retain and falls
  through to the existing `NX_ENOMEM` rollback path (which
  destroys the partial child process; the child's
  already-allocated entries get torn down by
  `nx_process_destroy`'s handle-table walk).
- **`posix_close` then `posix_read` / `posix_write` close-the-
  other-side discipline in the demo.**  This is the canonical
  POSIX pipe + fork pattern; matches what busybox + a real
  shell will do.  The demo therefore serves double duty —
  proves the inheritance works AND establishes the call shape
  that slice 7.6c–d will replicate at scale.

## What Was Done

### `framework/channel.{h,c}` — refcount surgery

- `struct nx_channel_endpoint` grows `_Atomic int handle_refs`.
- `struct nx_channel`'s `_Atomic int refcount` removed (the
  endpoint refcounts collectively replace it).
- `nx_channel_create` now `atomic_init`s each endpoint's
  `handle_refs` to 1.
- `nx_channel_endpoint_close` decrements the endpoint's
  `handle_refs`.  Only on transition from 1 → 0 does it set
  `closed = true`.  When the closing endpoint becomes the last
  open one (peer's `handle_refs == 0` already), the channel
  allocation is freed.  The peer's count is re-read inside
  the close path rather than cached because two final closers
  on opposite endpoints could race; only one sees both refs
  at zero, and that one frees.
- New `nx_channel_endpoint_retain(e)` — bumps `handle_refs`.
  Caller must already hold a reference, so a relaxed
  `atomic_fetch_add` is correct on a single CPU.

### `framework/syscall.c` — sys_fork inheritance

After `nx_process_fork(caller->process)` succeeds, walk
parent's handle table:

```c
for (size_t i = 0; i < NX_HANDLE_TABLE_CAPACITY; i++) {
    const struct nx_handle_entry *src = &parent_tbl->entries[i];
    if (src->type != NX_HANDLE_CHANNEL) continue;
    if (!src->object) continue;
    nx_channel_endpoint_retain(src->object);
    nx_handle_t dup_h = NX_HANDLE_INVALID;
    int rc = nx_handle_alloc(child_tbl, src->type,
                             src->rights, src->object, &dup_h);
    if (rc != NX_OK) {
        nx_channel_endpoint_close(src->object);  /* drop the retain */
        nx_process_destroy(child);
        return NX_ENOMEM;
    }
}
```

Skips:
- `NX_HANDLE_INVALID` (empty slots).
- Anything except `NX_HANDLE_CHANNEL` — files specifically
  excluded for this slice.

### Tests

- `test/kernel/posix_pipe_xproc_prog.c` — C-compiled EL0
  demo.  Parent `pipe()`s, `fork()`s, parent closes fds[0]
  + writes "hello\n" + closes fds[1] + waits, child closes
  fds[1] + reads from fds[0] + asserts the bytes + emits
  marker + exits 41.  Three live-log markers:
  `[xpipe-parent]`, `[xpipe-child]`, `[xpipe-ok]`.  Six
  distinct exit codes (1..5 for various error points, 41 for
  success) so the ktest can pinpoint regressions.
- `test/kernel/posix_pipe_xproc_prog_blob.S` — `.incbin`
  wrapper.
- `test/kernel/ktest_posix_pipe_xproc.c` — same load + drop +
  yield + assert pattern as the other slice-7.x ktests.
  Verifies all three markers + the child's exit_code == 41.

### Makefile

- New POSIX_PROG_CFLAGS rules for `posix_pipe_xproc_prog.{o,
  elf}` + `posix_pipe_xproc_prog_blob.o`.
- `KTEST_C += test/kernel/ktest_posix_pipe_xproc.c`.
- `KTEST_S += test/kernel/posix_pipe_xproc_prog_blob.S`.
- `clean` drops the new `.elf`.

## Key Findings

- **Per-channel refcount was a footgun for handle aliasing.**
  The old design implicitly assumed each endpoint had exactly
  one handle.  Easy to overlook because at create time it's
  true (sys_pipe / sys_channel_create allocate one handle per
  endpoint and never alias).  Fork inheritance is the first
  caller that violates the assumption; the symptom would have
  been "child's read mysteriously fails because parent closed
  the read fd before child got a chance to read".  Catching
  the structural issue and fixing it via per-endpoint refcounts
  is more robust than special-casing inheritance.
- **The peer-refs reload in `endpoint_close` matters.**  My
  first draft cached the peer's refs before deciding to free
  and got a TOCTOU bug: parent and child can close their last
  references in either order, and a stale snapshot could lead
  to either a double-free or a leak.  Re-reading after my own
  decrement is correct because acq_rel ordering on the
  fetch_sub means I see the peer's final write if it
  happened before mine.  Single-CPU v1 makes this academic
  but the API is now SMP-safe by construction.
- **One-shot test is enough for the whole stack.**  The
  `posix_pipe_xproc` test exercises: NX_SYS_PIPE, sys_fork's
  inheritance walk, channel endpoint retain, fork's
  trap-frame replay, child's address-space replication, both
  sides' `posix_close` / `posix_read` / `posix_write` (which
  hit the type-polymorphic dispatch from slice 7.5),
  posix_waitpid, and the parent's final exit-code observation.
  If anything in the chain regresses, this test catches it
  through one of the six distinct failure exit codes — much
  cleaner than carving the layers into separate tests.

## Decisions Made

- **Skip files.**  Files-across-fork is a v1 gap.  Acceptable
  for slice 7.6a's scope ("unblock busybox-style pipes");
  busybox's own design path uses pipes for IPC and reopens
  files in children, so the lack of file inheritance won't
  bite us until much later.  HANDOFF.md's Next Actions calls
  this out as an opportunistic follow-up before slice 7.6d.
- **Don't ship a `nx_handle_table_clone` helper yet.**  Could
  factor out the inheritance loop as a generic helper on
  `framework/handle.c`.  But the loop is type-aware (needs to
  call CHANNEL-specific retain) and per-type rules will keep
  branching as more types get inheritance plumbing.  Inline
  in sys_fork keeps the type-specific dispatch local; a
  refactor to a helper lands when there are >1 callers.
- **Match `make test` numbering with the slice number.**  After
  this slice the kernel ktest count is 72, total 397.  Same
  cadence as slice 7.5 (71 → 72; +1 ktest per slice).
- **Roll Session 35 to archive.**  HANDOFF.md's session list
  is capped at 5; after adding Session 40 the old Session 35
  (slice 7.4a, fork) moves to the archive.

## Status at End of Session

- `make test` → **397/397 pass (51 python + 274 host + 72
  kernel), 0 leaks, 0 errors, exit 0**.  +1 kernel test.
- `make run` boots cleanly.
- Live EL0 marker catalog (now 21 distinct milestones):
  `[el0] hello` · `[el0-chan-ok]` · `[el0-file-ok]` ·
  `[el0-rdr-ok]` · `[el0-elf-ok]` · `[fork-parent]` ·
  `[fork-child]` · `[wait-parent]` · `[wait-child]` ·
  `[wait-ok]` · `[exec-parent]` · `[exec-ok]` ·
  `[posix-parent]` · `[posix-child]` · `[posix-ok]` ·
  `[pipe-ok]` · `[signal-parent]` · `[signal-child]` ·
  `[signal-ok]` · `[xpipe-parent]` · `[xpipe-child]` ·
  `[xpipe-ok]`.
- The fork+exec+wait+pipe+signal POSIX surface is now feature-
  complete enough that a real shell (busybox `sh`) can do
  `pipe → fork → close-one-side → exec` end-to-end without
  hitting an EXIST-only-in-future-slice gap.

## Next Steps

**Slice 7.6b — initramfs format + ramfs slurp.**

- Pick cpio-newc as the on-disk format (busybox already
  speaks it natively).
- Write `tools/pack-initramfs.py` — takes a list of files +
  symlinks, emits a cpio-newc blob.
- Bake the blob into `kernel-test.bin`'s `.rodata` (or a
  dedicated section we can locate at boot).
- Teach `components/ramfs/ramfs.c`'s `init` to walk the blob
  + populate its inode table.
- Drop the per-test `seed_init_file` boilerplate in
  `ktest_exec` etc.; once `/init` is in the live FS, those
  tests can just exec it.

**7.6c then 7.6d** as outlined in IMPLEMENTATION-GUIDE.md.

**Deferred (unchanged):**

- HANDLE_FILE fork inheritance — needs a vfs_simple `dup` op.
- C-level crt0 for `main(argc, argv)`.
- Proper cross-test task reap.
- Per-test `SCHED_RR_DEFAULT_QUANTUM_TICKS` override.

---

**Files Changed:**
- `sources/nonux/framework/channel.h` — header docs + retain API
- `sources/nonux/framework/channel.c` — per-endpoint handle_refs; retain/release semantics
- `sources/nonux/framework/syscall.c` — sys_fork handle-table inheritance walk
- `sources/nonux/test/kernel/posix_pipe_xproc_prog.c` — new (C-compiled EL0 demo)
- `sources/nonux/test/kernel/posix_pipe_xproc_prog_blob.S` — new (.incbin wrapper)
- `sources/nonux/test/kernel/ktest_posix_pipe_xproc.c` — new (kernel test)
- `sources/nonux/Makefile` — posix_pipe_xproc_prog.{o,elf} + blob rules; KTEST_C/S updates; clean
- `proj_docs/nonux/IMPLEMENTATION-GUIDE.md` — §Slice 7.6 sub-sliced into 7.6a–d; 7.6a filled in
- `proj_docs/nonux/HANDOFF.md` — status / checklist / next-actions / session-log
- `proj_docs/nonux/HANDOFF-ARCHIVE.md` — Session 35 rolled in
- `proj_docs/nonux/README.md` — status + last-updated
