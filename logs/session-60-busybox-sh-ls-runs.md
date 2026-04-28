# Session 60: slice 7.6d.N.5 — `busybox sh -c "ls /"` runs end-to-end

**Date:** 2026-04-27
**Phase:** 7 slice 7.6d.N.5 (busybox-as-shell, directory listing)
**Branch:** master

---

## Goals

Slice 7.6d.N.4 (session 59) got `ls` to fork+exec successfully
through ash's PATH walk via the new `NX_SYS_FSTATAT`, but ls
itself bailed at `stat("/")` because vfs_simple has no
directory entry for `/`.  This sub-slice closes that gap: make
`busybox sh -c "ls /"` actually list the root directory.

Required pieces (identified during 7.6d.N.4):
1. `stat("/")` must return `S_IFDIR` mode.
2. `open("/")` must succeed with a directory-typed handle.
3. `getdents64(fd, buf, count)` must populate `buf` with Linux-
   shape directory entries.
4. `close(fd)` must release the directory handle.
5. musl-translation entries for `__NR_openat (56)` and
   `__NR_getdents64 (61)`.

Approach: minimal kernel surface — special-case `/` in the
existing path resolution, add one new handle type
(`NX_HANDLE_DIR`), one new syscall (`NX_SYS_GETDENTS64`), and
one Linux-shape openat wrapper (`NX_SYS_OPENAT`) that
converts Linux O_* flag bits to our `NX_VFS_OPEN_*` and
forwards to `sys_open`.

## Implementation

### `framework/handle.h` — `NX_HANDLE_DIR`

New enum entry between `NX_HANDLE_FILE` and the
`NX_HANDLE_TYPE_COUNT` sentinel.  Same shape as the other
type values.

### `framework/syscall.h` — two new syscall numbers

- `NX_SYS_GETDENTS64 = 22` — Linux ABI: `(int fd,
  struct linux_dirent64 *buf, size_t count)`.  Returns bytes
  written, 0 at EOF, small Linux-errno on failure.  fd must
  reference a `NX_HANDLE_DIR`.
- `NX_SYS_OPENAT = 23` — Linux ABI: `(int dirfd, const char
  *path, int flags, mode_t mode)`.  dirfd ignored (vfs_simple
  is absolute-paths-only); flags interpreted as Linux O_*
  bits; mode ignored (no perms in v1).  Forwards to sys_open
  after converting flag layout.

### `framework/syscall.c`

**Special-case `/` in `sys_fstatat`** (slice 7.6d.N.4 already
established the function): if `path == "/"`, skip the vfs
round-trip and fill `st_mode = S_IFDIR | 0755`.  Without
this, the open-then-fail dance returned ENOENT for the root.

**Special-case `/` in `sys_open`** (gated under
`!__STDC_HOSTED__` since it allocates from kheap): if
`kpath == "/"`, allocate a small `struct nx_dir_cursor`
(holds just the readdir cookie), allocate a `HANDLE_DIR`
pointing at it via `nx_handle_alloc`, return the handle.
Falls through to the regular vfs path otherwise (including
on host builds).

**`sys_handle_close` HANDLE_DIR branch**: call `free` on the
cursor object before the slot itself is freed.  Same pattern
as the existing `HANDLE_CHANNEL` and `HANDLE_FILE` branches
that delegate type-specific cleanup.

**`sys_getdents64` (~75 lines)**: looks up the fd as a
`HANDLE_DIR`, walks `vops->readdir` against the per-handle
cursor, packs entries into a 4 KiB kernel staging buffer using
the Linux `struct linux_dirent64` ABI (8-byte aligned
records, `d_type = DT_REG` for v1's flat namespace), copies
out via `copy_to_user`.  Caps the output at `min(count,
4096)`; if a record doesn't fit, rewinds the cursor by 1 so
the next call resumes with the same entry.  Returns total
bytes written; returns 0 at EOF.  Strips leading `/` from
ramfs entry names since cpio convention stores the leading
slash but Linux convention has bare basenames.

**`sys_openat` (~25 lines)**: Linux ABI shape.  Converts
Linux O_* flag bits (`O_RDONLY=0`, `O_WRONLY=1`, `O_RDWR=2`,
`O_CREAT=0o100`, `O_DIRECTORY=0o200000`) to our
`NX_VFS_OPEN_*` flags.  `O_DIRECTORY` is informational —
sys_open's `/` branch already returns `HANDLE_DIR`.  Other
O_* bits (`O_CLOEXEC`, `O_NONBLOCK`, `O_TRUNC`) are dropped
silently.  Forwards path + converted flags to `sys_open`.

### `third_party/musl/arch/aarch64/syscall_arch.h`

Added two cases to `__nx_translate`:
- `case 56: return 23;` — `__NR_openat → NX_SYS_OPENAT`
- `case 61: return 22;` — `__NR_getdents64 → NX_SYS_GETDENTS64`

### `third_party/musl/src/thread/aarch64/syscall_cp.s`

Same two mappings added to the asm-level cancellation-point
dispatcher.

## Discovery debugging notes

Encountered three quirks during the session:

1. **Stale handle.o.**  After adding `NX_HANDLE_DIR` to
   `framework/handle.h`, `nx_handle_alloc` initially returned
   `NX_EINVAL` even though `NX_HANDLE_DIR < NX_HANDLE_TYPE_COUNT`.
   Cause: handle.c hadn't been rebuilt against the new header
   despite the timestamp dependency.  Fixed by `rm -f
   framework/handle.o framework/syscall.o && touch
   framework/handle.h && make`.  Forces a rebuild.  Worth
   adding a Makefile dep `framework/handle.c: framework/handle.h`
   later (opportunistic).

2. **Host build malloc/free.**  `sys_open`'s `/` branch and
   `sys_handle_close`'s `HANDLE_DIR` branch use `malloc`/
   `free` (kheap).  Host build has no kheap, so an
   unconditional call breaks the host test compile.  Gated
   both with `#if !__STDC_HOSTED__` — same pattern as
   `sys_exec`.  Host builds fall through to vfs_simple's
   normal path resolution (which returns NX_ENOENT for "/"
   on host fixtures, matching pre-slice-7.6d.N.5 behaviour).

3. **`grep | head` masking output.**  Spent ~15 minutes
   debugging "ls only prints `argv_child`" — turned out to
   be a `head -3` filter on the `make test` output truncating
   the multi-line ls listing.  The actual log file showed
   all 9 entries.  Lesson: when a UART log doesn't match
   expectations, dump the raw `test/kernel-output.log` first
   before adding kernel-side traces.

## Outcome

`make test` → **414/414 PASSED** (51 python + 275 host + 88
kernel; same as slice 7.6d.N.4 — no new ktests, the existing
`posix_busybox_sh_ls` ktest now exercises the success path).

### Live ktest log

```
posix_busybox_sh_ls_parent_forks_and_execs_busybox_sh_ls [bbsh-ls-parent]argv_child
banner
busybox
hello
init
ktest_hello
ls
musl_prog
rd
[bbsh-ls-status=00][bbsh-ls-ok]PASS
```

ash forks, execs `/bin/ls`, busybox-as-ls dispatches to the
ls applet, ls calls `opendir("/")` → `__NR_openat (56)` →
`NX_SYS_OPENAT (23)` → `sys_open` `/` branch → `HANDLE_DIR`.
Then ls calls `getdents64` → `NX_SYS_GETDENTS64 (22)` →
walks ramfs's flat namespace, packs 9 records into a 272-byte
buffer.  ls sorts alphabetically (argv_child, banner,
busybox, hello, init, ktest_hello, ls, musl_prog, rd) and
prints one-per-line (non-tty stdout defaults to single-column
in busybox ls).  `bin/busybox` and `bin/ls` get rendered as
basenames (`busybox`, `ls`) — busybox ls's display layer
strips path prefixes since it's listing dir contents.

ash's normal-exit path runs (atexit, stdio flush via writev),
exits 0.  Test parent reaps and prints status markers.

## Why the bin/ prefix gets stripped

Some ramfs entries have embedded `/`: `/bin/busybox`,
`/bin/ls`.  Our cpio loader stores them with the leading `/`
intact (e.g. `s->files[i].name = "/bin/busybox"`).  My
getdents64 implementation strips just the FIRST `/` so the
emitted d_name becomes `bin/busybox`.  ls receives that as
the d_name and treats it as a path component.  busybox ls's
display layer takes the basename (`busybox`).  Effectively
fine for `ls /` behaviour; would be a problem if a script
checked for exact directory contents.

A cleaner long-term shape: ramfs grows real directory
inodes, the cpio loader creates a `/bin` directory and puts
`busybox` and `ls` under it, getdents64 returns just `bin`
when listing `/` (with `d_type = DT_DIR`).  `ls /bin` would
then list `busybox` and `ls`.  Multi-level directory support
is its own slice — out of scope for 7.6d.N.

## What's next (7.6d.N.6+ candidates)

Builtins: `exit`, `echo`, multi-statement.  
Non-builtin (single command): `ls /`. ✓

The next escalation step is more demanding:

- **`busybox sh -c "ls /; echo done"`** — non-builtin
  followed by builtin in same script.  Tests ash's wait+
  reap of the ls child + sequence dispatch.
- **`busybox sh -c "echo hello | cat"`** — first pipe.
  ash forks twice (echo + cat), creates a pipe, reaps both.
  Tests `pipe()` (we have NX_SYS_PIPE since slice 7.5)
  + writev redirection through the pipe + cat reading
  via readv.
- **`busybox sh -c "cat /init"`** — non-builtin reading a
  real file from vfs.  Tests musl's `__stdio_read` path
  (uses readv — we don't have `__NR_readv (65)` mapped
  yet).
- **Investigate the fork+clone mystery** (still open from
  7.6d.N.4).  ash now successfully fork+execs `ls` despite
  `__NR_clone (220)` being unmapped.  Possibly musl returns
  -ENOSYS but ash treats pid=-1 as "exec inline" via the
  single-command shortcut.  Worth pinning down.

Lowest-friction next step: `busybox sh -c "echo hello | cat"`
— exercises pipe + readv + cat applet.  Will likely surface
the readv mapping gap.

## Files changed

Production:
- `framework/handle.h` — `NX_HANDLE_DIR` enum entry (1 line)
- `framework/syscall.h` — `NX_SYS_GETDENTS64`, `NX_SYS_OPENAT`
  (~30 lines doc-comments)
- `framework/syscall.c`:
  - `struct nx_dir_cursor` (slice-7.6d.N.5 internal)
  - `sys_open` `/` branch (~15 lines, gated `!__STDC_HOSTED__`)
  - `sys_handle_close` HANDLE_DIR branch (~5 lines, gated)
  - `sys_fstatat` `/` branch (~8 lines)
  - `sys_getdents64` (~75 lines, gated)
  - `sys_openat` (~25 lines)
  - dispatch table entries (2 lines)
  - Linux errno + S_IFDIR + DT_DIR macros (a few lines)
- `third_party/musl/arch/aarch64/syscall_arch.h` —
  `case 56` + `case 61` translations (2 lines + comments)
- `third_party/musl/src/thread/aarch64/syscall_cp.s` —
  same mappings in asm (2 lines + comments)

No test code added (existing ktest_posix_busybox_sh_ls
already covers `ls /`; it now passes the workload check
instead of just structural reach-the-marker).

Total: 1 header + 1 implementation + 2 musl patches.  ~150
lines of net new logic.

---

**Last Updated:** 2026-04-27
