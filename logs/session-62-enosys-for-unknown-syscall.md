# Session 62: hygiene fix — return `NX_ENOSYS = -38` for unknown syscall numbers

**Date:** 2026-04-27
**Phase:** 7 (opportunistic cleanup, not a sub-slice — closes the side discovery captured in slice 7.6d.N.6a / Session 61)
**Branch:** master

---

## Goals

Close the misleading-error-message hygiene gap captured in
session 61.  `framework/syscall.c`'s `nx_syscall_dispatch`
returned `NX_EINVAL = -1` for unknown syscall numbers (out-of-
range or NULL dispatch slot), but `NX_EINVAL`'s numeric value
collides with Linux's `EPERM = -1` — so musl's `__syscall_ret`
translates the value byte-for-byte and userspace surfaces
unmapped syscalls as `errno = EPERM` ("Operation not
permitted") instead of the conventionally correct
`errno = ENOSYS` ("Function not implemented").

This bites every diagnostic message in busybox + every other
musl-linked program that hits an unmapped syscall.  Session 61
saw `sh: can't create pipe: Operation not permitted` instead
of `Function not implemented` for the unmapped `pipe2` (now
mapped — but the failure mode applies generally).

This session is a narrow, standalone fix: introduce a
Linux-compat `NX_ENOSYS = -38` constant and use it from the
single "unknown syscall" return path.  All other NX_E* uses
stay as-is — kernel-internal handler errors don't go through
musl's errno translation, so the in-handler `NX_EINVAL`
return values remain semantically correct.

## What landed

Five files, +29/-15 lines.  No new syscalls, no new tests.

### `framework/registry.h` — new constant

Added immediately after the existing `NX_EAGAIN = -11` block:

```c
/*
 * Linux-ABI errno value, intentionally outside the NX_E* sequence above.
 * Used by `nx_syscall_dispatch` for unknown syscall numbers so musl's
 * `__syscall_ret` translates the value to userspace `errno = ENOSYS`
 * ("Function not implemented") instead of `EPERM` ("Operation not
 * permitted") — which is what the rest of the NX_E* codes would resolve
 * to since `NX_EINVAL = -1` collides with Linux `EPERM = -1`.
 */
#define NX_ENOSYS   -38
```

The deliberate gap (-12 through -37 unused) signals that
NX_ENOSYS isn't part of the dense NX_E* sequence — it's a
Linux-ABI value chosen to round-trip through musl's errno
translation.  A future cleanup might renumber the entire
NX_E* family to match Linux errno values, but that's a much
bigger change and not required to make unmapped-syscall
diagnostics readable.

### `framework/syscall.c` — single-site change

`nx_syscall_dispatch`'s unknown-syscall fallback:

```c
if (num >= NX_SYSCALL_COUNT || g_syscall_table[num] == NULL) {
    rc = NX_ENOSYS;          // was NX_EINVAL
} else {
    ...
}
```

Plus two comment updates: the file header (referring to the
"crisp NX_EINVAL" for stray `svc #0` with x8 unset → "crisp
NX_ENOSYS"), and the inline comment on `NX_SYS_RESERVED_0` in
the dispatch table.

### `framework/syscall.h` — ABI comment

Updated the ABI comment to distinguish the two error paths:

```
Unknown syscall numbers return NX_ENOSYS (Linux-compat -38) so musl
surfaces them as `errno = ENOSYS` instead of EPERM; invalid argument
pointers (and other in-handler errors) return NX_EINVAL or other
NX_E* codes.
```

This is the contract that future syscall implementations
should follow — handlers continue to return `NX_EINVAL` for
bad arguments (NULL pointers, out-of-window addresses, etc.),
since those errors *are* semantically "invalid argument" and
the dispatcher-level "no such syscall" path is the only one
that needs the ENOSYS distinction.

### Tests — rename + value update

Two tests pinned the unknown-syscall return value, both
needed updating:

- `test/host/syscall_test.c`: renamed
  `syscall_unknown_number_returns_einval_on_host` →
  `..._returns_enosys_on_host`; updated three assertions
  (slot 0 / x8=9999 / x8=NX_SYSCALL_COUNT) from `NX_EINVAL`
  to `NX_ENOSYS`.
- `test/kernel/ktest_syscall.c`: same rename
  (`syscall_unknown_number_returns_einval` →
  `..._returns_enosys`), same three-assertion update,
  plus expanded the comment to spell out the Linux-compat
  motivation.

Test names are auto-registered via the `TEST` / `KTEST`
macros — grepped to confirm no other file references the
old names.

## What didn't land (and why)

The session-61 notes flagged that NX_EINVAL = -1 colliding
with Linux EPERM is a *general* concern: every NX_E* code
collides with *some* Linux errno (NX_ENOMEM=-2 vs ENOENT,
NX_EEXIST=-3 vs ESRCH, etc.).  This session intentionally
does *not* renumber the NX_E* family — that's a much bigger
change with broader test impact, and the immediate
diagnostic-message problem only requires the unknown-syscall
case.  Other NX_E* return paths from individual syscall
handlers either:

- Don't reach userspace at all (they're returned to a
  kernel-internal caller that interprets them via the NX_E*
  convention), or
- Do reach userspace but the specific collision isn't
  causing observable confusion yet (e.g. NX_ENOMEM for `mmap`
  becomes EPERM at musl's `__syscall_ret`, but mallocng's
  callers treat any negative return as failure regardless of
  the specific errno).

If userspace diagnostics start surfacing other misleading
messages, that's the trigger for a bigger renumbering pass.
For now, the targeted ENOSYS fix is sufficient.

## What this unblocks

7.6d.N.6b's diagnostic trace will now show "Function not
implemented" for any syscall that slips past the musl-side
translation table (e.g. `__NR_sendfile = 71` — observed in
session 61's cat-hang failure mode).  This makes the
post-7.6d.N.6b investigation pinch-point easier: we can read
the error message at face value instead of having to
remember the EPERM↔ENOSYS swap.

It also slightly reduces the "what to add to the syscall
translation table" search space — when an unmapped syscall
*does* land in the kernel (rather than being short-circuited
by musl's translation `default: return -ENOSYS`), the
diagnostic message matches the standard Linux convention
that everyone reading the code already understands.

## Test results

`make test` → 414/414 pass (51 python + 275 host + 88 kernel),
0 leaks, 0 errors.  Same numbers as session 61, since the
test count isn't changing — only two existing tests'
expected values rotated.

## Next

Resume slice 7.6d.N.6b: introduce `NX_HANDLE_CONSOLE` +
pre-install at slots 0/1/2 in `nx_process_create`, delete
the magic-fd fallbacks, make slots 0/1/2 survive exec,
extend fork inheritance to console handles, re-enable the
pipe ktest.
