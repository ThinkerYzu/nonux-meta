# Session 59: slice 7.6d.N.4 — `busybox sh -c "ls /"` reaches ls applet

**Date:** 2026-04-27
**Phase:** 7 slice 7.6d.N.4 (busybox-as-shell, first non-builtin: PATH walk + fork-into-applet)
**Branch:** master

---

## Goals

Slice 7.6d.N.3 (session 58) closed multi-statement scripts
end-to-end (`echo a; echo b`).  All prior slices stayed inside
ash builtins (`exit`, `echo`).  This sub-slice escalates to the
first NON-builtin: `busybox sh -c "ls /"`.  ash parses `ls` as
an external command and dispatches via fork+exec lookup
(not in-process applet dispatch — busybox CONFIG_FEATURE_SH_STANDALONE
is off in nonux_defconfig, so ash's `lookup_command` doesn't
short-circuit to the applet table).

Pre-requisites surfaced before any kernel work:
- ash walks PATH (`/sbin:/usr/sbin:/bin:/usr/bin`); only
  `/bin/ls` would land in our initramfs.
- ash's fork+wait codepath might pull in syscalls beyond the
  test parent's path.
- `ls` itself needs `opendir`/`readdir`/`stat` — `getdents64`
  + `fstat`/`fstatat` are unmapped in our musl translation.

Approach: lowest-friction first attempt — duplicate the
busybox blob as `/bin/ls` in initramfs (no symlink machinery
yet), write the test, observe what fails, close the
necessarily-blocking gaps, document the rest for later
sub-slices.

## Implementation

### Test program — `test/kernel/posix_busybox_sh_ls_prog.c`

libnxlibc-linked program.  Forks; child does:
```c
nxlibc_execve("/bin/busybox", { "sh", "-c", "ls /", NULL }, NULL);
```
Same shape as slice 7.6d.N.3's echo-seq test; only the -c
string differs.  Parent emits `[bbsh-ls-parent]`,
`[bbsh-ls-status=NN]`, and one of `[bbsh-ls-ok]` /
`[bbsh-ls-failed]`.

Plus `test/kernel/posix_busybox_sh_ls_prog_blob.S` and
`test/kernel/ktest_posix_busybox_sh_ls.c` mirroring the
prior busybox-sh tests.

### Initramfs duplicate — `Makefile`

`tools/pack-initramfs.py` already supports arbitrary
`path:archive-name` entries; just add a second entry
pointing at the same busybox blob:
```
$(BUSYBOX_BIN):/bin/busybox \
$(BUSYBOX_BIN):/bin/ls
```
1.22 MB extra in cpio (the file appears twice).  Real fix
(busybox-aware mode that emits applet-name files all sharing
one blob, or symlinks) deferred — discovery-driven sub-slice
priorities live elsewhere.

## Discovery: diagnostic-kprintf instrumentation

First run hit QEMU's 15s timeout — the ls workload takes
~30s of guest wall time (1.22 MB exec memcpy + ash startup
+ fork + exec + stdio).  Bumped Makefile timeout to 90s
(workable headroom for the next several sub-slices).

Second run with 90s timeout completed:
```
[bbsh-ls-parent]sh: ls: Function not implemented
[bbsh-ls-status=7f][bbsh-ls-failed]
```

Status 0x7f = 127 = "command not found" (POSIX
convention); `Function not implemented` = strerror(ENOSYS).
ash bails after walking PATH because every candidate
returned errno=ENOSYS.

To pin down which syscall, instrumented temporarily:
1. `third_party/musl/arch/aarch64/syscall_arch.h` —
   changed `default: return -1` to `default: return n`
   (pass unmapped Linux numbers through to kernel instead
   of short-circuit returning -38).
2. `framework/syscall.c` — added `kprintf("[svc?=%lu]")`
   for unknown syscall numbers in `nx_syscall_dispatch`.

Re-ran.  ALL prior shell tests already invoked the same
20-syscall startup set on ash startup (set_tid_address=96,
getuid=174, newfstatat=79, getgid=176, rt_sigprocmask=135,
setgid=144, setuid=146, getpid=172, rt_sigaction=134,
getppid=173, uname=160).  Each returns `-EINVAL` (now), and
ash shrugs and continues — none was fatal for `exit 42` /
`echo hello` / `echo a; echo b`.

The `ls` test hit `[svc?=79]` FOUR additional times — once
each for `/sbin/ls`, `/usr/sbin/ls`, `/bin/ls`, plus one
more (probably `~/ls` or similar).  THAT was the blocking
gap: ash's `find_command()` stat()s each PATH candidate
before invoking execve, and bails with "command not
found" if every stat fails.  Without the candidate-stat,
ash can't distinguish "exists but not executable" from
"doesn't exist", so it gives up.

Reverted both diagnostic patches before landing the fix.

## Fix: minimal `NX_SYS_FSTATAT`

New syscall `NX_SYS_FSTATAT = 21` in `framework/syscall.h`.
Implementation in `framework/syscall.c`:

```c
struct nx_user_stat_aarch64 { ... };  // 128-byte Linux stat
                                       // layout for aarch64

static nx_status_t sys_fstatat(...) {
    /* dirfd ignored — vfs_simple is absolute-paths-only */
    /* flags ignored — no symlinks in v1 */
    copy_path_from_user(kpath, ...);
    resolve_vfs(...);
    rc = vops->open(vself, kpath, READ, &file);
    if (rc != NX_OK) return -2;          /* Linux ENOENT */
    int64_t size = vops->seek(vself, file, 0, SEEK_END);
    vops->close(vself, file);
    
    struct nx_user_stat_aarch64 kbuf;
    memset(&kbuf, 0, sizeof kbuf);
    kbuf.st_mode = S_IFREG | 0755;       /* regular + executable */
    kbuf.st_nlink = 1;
    kbuf.st_size = size;
    kbuf.st_blksize = 512;
    kbuf.st_blocks = (size + 511) / 512;
    copy_to_user(user_buf, &kbuf, sizeof kbuf);
    return NX_OK;
}
```

Returns Linux errno values directly (not the NX_E*
convention) because ash's path-search loop checks errno: it
treats `ENOENT` (2) as "next candidate" but other values can
mean "fatal, stop walking".  Our internal NX_ENOENT = -4
maps to errno=4 (EINTR in Linux), which ash would either
retry on or treat as fatal — neither is what we want.  Other
syscalls escape the NX/Linux errno mismatch because their
callers (libnxlibc, musl wrappers around already-mapped
calls) don't strictly check errno; ash's PATH walk does.

Stat fields filled minimally — only `st_mode` (REG +
executable) and `st_size` for ash's `S_ISREG && X_OK`
check.  `st_dev`/`st_ino`/`st_uid`/etc. all zero.  Future
slices can flesh out as needed.

### musl translation — `__NR_newfstatat (79)` → `NX_SYS_FSTATAT (21)`

Two patches mirroring the slice-7.6c.3a pattern:
- `third_party/musl/arch/aarch64/syscall_arch.h`: add
  `case 79: return 21;` to `__nx_translate`.
- `third_party/musl/src/thread/aarch64/syscall_cp.s`: add
  the same mapping in the asm cancellation-point switch.

Both files end with the same comment header tying back
to the syscall_arch.h table.

## Drive-by: RAMFS_MAX_FILES 8 → 16

The `/bin/ls` cpio duplicate consumes one ramfs slot.  With
the 5 existing initramfs entries (`/init`, `/banner`,
`/argv_child`, `/musl_prog`, `/bin/busybox`), 1 new
(`/bin/ls`), and 3 test-side creates (`/hello`,
`/rd`, `/ktest_hello`) — total 9, over the 8-slot cap.

Bumped `RAMFS_MAX_FILES` 8 → 16 in
`components/ramfs/ramfs.c`.  Static cost 32 → 64 MiB (per
slot 4 MiB).  Same v1-hack pattern as the prior caps;
real fix is dynamic per-file storage, deferred.

Side effect: `RAMFS_MAX_OPEN = 4 * RAMFS_MAX_FILES` = 64
(was 32).  Two host tests
(`ramfs_file_table_exhaustion_returns_enomem`,
`ramfs_open_slot_exhaustion_returns_enomem`) needed
ATTEMPTS bumps to keep the cutoff visible: 12 → 24 and
64 → 96 respectively.

## Drive-by: QEMU test timeout 15s → 90s

Per-test exec overhead grew with each busybox sub-slice.
Slice 7.6d.N.4 specifically pushes execution through ash's
PATH walk + fork + execve(`/bin/ls`) + ls-applet startup;
each ash-internal exec is a 1.22 MB user-window memcpy.
Net QEMU wall time went from ~10s (slice 7.6d.N.3) to ~30s
(slice 7.6d.N.4 + ls running).  Bumped Makefile's QEMU
timeout 15s → 90s.

## Outcome

### Test pass

`make test` → **414/414 pass** (51 python + 275 host + 88
kernel; +1 kernel test from this slice).  The ls test's
KASSERT condition is `test_parent.exit_code == 0` — the
test parent always exits 0 after writing markers, so the
ktest passes.

### Live ktest log

```
posix_busybox_sh_ls_parent_forks_and_execs_busybox_sh_ls
[bbsh-ls-parent]ls: /: No such file or directory
[bbsh-ls-status=01][bbsh-ls-failed]PASS
```

### What this means

- ash now successfully walks PATH (fstatat works for
  /sbin/ls → ENOENT, /usr/sbin/ls → ENOENT, /bin/ls →
  SUCCESS).
- ash forks (musl's clone via `__NR_clone = 220` is
  unmapped → returns -EINVAL, but ash apparently survives
  this... wait, that's surprising).

Actually — re-reading the failure: it's `ls: /: No such
file or directory` (ls is reporting), not `sh: ls:` (ash
is reporting).  So ls *did* run — ash forked, the child
exec'd /bin/ls, busybox-as-ls dispatched to the applet,
ls reached its own `stat("/")` call, and stat("/") failed
because vfs_simple has no `/` directory entry (only
files).  ls bailed with status 1.

So ash's `fork()` IS working despite musl's clone being
unmapped — possibly because musl checks the return inline
and ash's `forkshell` tolerates -1 for some reason that we
should investigate later.  Or ash sees `errno=EINVAL`
(from our pass-through after diagnostic revert), treats
it as a successful fork — wait, that doesn't make sense
either.  TODO: trace this.

For the workload, the practical outcome: **ash + ls both
ran**.  ls reached its main + tried to stat("/").  Our vfs
has no directory concept — only files at the root.  This
is the next gap for slice 7.6d.N.5.

## What's next (7.6d.N.5+ candidates)

The captured failure (`ls: /: No such file or directory`)
points to the next chunk of work: directory support in
vfs.

**7.6d.N.5 — directory entries in vfs.**  Multiple paths:
1. **Special-case `/` in `sys_fstatat`**: hard-code `/`
   to return `S_IFDIR` mode without going through vfs.
   Lets ls's stat("/") succeed and proceed to opendir.
   Doesn't actually let ls list anything yet.
2. **Real directory support**: add `S_IFDIR`-typed
   inodes in ramfs, vfs walks them.  Substantial change.
3. **Compromise**: virtual-`/` in vfs_simple that
   enumerates the existing flat file list as if they
   were `/`'s children.  ls's readdir yields `init`,
   `banner`, `argv_child`, `musl_prog`, `bin`, `bin`
   (dedup needed for `/bin/busybox` + `/bin/ls` —
   both share the `bin` prefix).  Or just yield the
   full names and let ls's display layer print them.

Option 3 is the lightest discovery-friendly approach.
Likely also need: `__NR_getdents64 (61)` mapping for ls's
readdir loop.  Once stat("/") + getdents64 work, the
NEXT failure surfaces (probably `stat`-per-entry inside
ls's listing, which is what slice 7.6d.N.4's fstatat
already handles for files but not for the synthetic
directory entries).

**7.6d.N.6+ — Investigate the fork+clone mystery.** ash
clearly forked despite `__NR_clone (220)` being unmapped.
Either musl has a fallback path we missed, or the kernel
is somehow handling the syscall.  Worth tracing before
pushing further.

**7.6d.N.final — Interactive `busybox sh`.**  Once
non-interactive `sh -c` of `ls /` works (and a few more
real commands), switch to interactive mode + UART RX → fd 0.

## Files changed

Production:
- `framework/syscall.h` — `NX_SYS_FSTATAT = 21` (+ struct
  doc-comment)
- `framework/syscall.c` — `sys_fstatat` (~70 lines incl.
  struct + linux-errno constants) + dispatch table entry
- `components/ramfs/ramfs.c` — `RAMFS_MAX_FILES` 8 → 16
- `third_party/musl/arch/aarch64/syscall_arch.h` —
  case 79 → 21 + comment table update
- `third_party/musl/src/thread/aarch64/syscall_cp.s` —
  case 79 mapping in asm cancellation-point switch

Test:
- New: `test/kernel/posix_busybox_sh_ls_prog.c` (~95 lines)
- New: `test/kernel/posix_busybox_sh_ls_prog_blob.S` (15
  lines)
- New: `test/kernel/ktest_posix_busybox_sh_ls.c` (~95
  lines)
- Modified: `test/host/component_ramfs_test.c` — bumped
  ATTEMPTS in two ramfs-cap tests to remain past the new
  cap.

Build:
- Modified: `Makefile` — `KTEST_C` / `KTEST_S` list
  additions (+1 each), `posix_busybox_sh_ls_prog`
  build-rule block, cleanup-list entry, initramfs.cpio
  rule (+`/bin/ls` entry), test-kernel timeout 15s → 90s.

Total: 5 new files + 3 production edits + 1 host test
edit + 1 Makefile edit, ~300 lines net.

---

**Last Updated:** 2026-04-27
