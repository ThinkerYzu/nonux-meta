# Session 119: Remove generated HTML files from source repo

**Date:** 2026-05-08
**Phase:** Phases 1–9b complete; post-9b fixes through Session 118; Phase 9 not yet started
**Branch:** master

---

## Goals

- Remove the HTML files that `scripts/convert_kb.sh` produced from
  Markdown sources from the `sources/nonux/` repo — they are build
  artifacts and should not be tracked.
- Add a `.gitignore` rule so future conversions don't re-stage them.

## What Was Done

### HTML files removed

15 `.html` files removed via `git rm` from `sources/nonux/`:

- `README.html`
- `components/README.html`
- `core/README.html`
- `docs/README.html`
- `docs/framework-bootstrap.html`
- `docs/framework-components.html`
- `docs/framework-hooks.html`
- `docs/framework-ipc.html`
- `docs/framework-registry.html`
- `framework/README.html`
- `interfaces/README.html`
- `lib/README.html`
- `test/README.html`
- `third_party/README.html`
- `tools/README.html`

Each had a matching `.md` source — confirmed before deletion.

### .gitignore rule added

Added under "Build artifacts" in `sources/nonux/.gitignore`:

```
# HTML files generated from Markdown by ../../scripts/convert_kb.sh.
# Vendored upstream HTML (e.g. third_party/busybox/docs/cgi/*.html) is
# already tracked and unaffected by this rule.
*.html
```

Vendored busybox HTML (`third_party/busybox/docs/cgi/*.html`,
`third_party/busybox/docs/draft-coar-cgi-v11-03-clean.html`) was left
alone because it is upstream content, not a build artifact. Gitignore
only filters untracked files, so the existing tracked busybox HTMLs
are unaffected by the new `*.html` rule.

## Key Findings

- gitignore semantics: rules don't affect already-tracked files. So a
  blanket `*.html` rule is safe even though some HTML files (busybox
  upstream) remain tracked.
- The previous Session 118 session log mentioned HTML conversion as a
  step ("All new/modified markdown files converted to HTML by
  scripts/convert_kb.sh") and the resulting HTML files were committed
  to the source repo. That was the regression this session corrects.

## Decisions Made

- **Keep busybox vendored HTML tracked** — it is upstream content, not
  output of our build, so it belongs in the repo.
- **Single `*.html` ignore rule** rather than per-file or per-directory
  rules — simpler, and gitignore's "no effect on tracked files" rule
  protects the busybox HTMLs.

## Status at End of Session

- 15 HTML deletions + `.gitignore` change staged in `sources/nonux/`.
- No code changes; tests not re-run. Test counts unchanged from
  Session 118: 102/102 tools · 485/485 host · 152/152 kernel.

## Next Steps

- Phase 9 — per-process MM rework (L3 4 KiB pages, VMAs, demand
  paging, COW fork). See
  [IMPLEMENTATION-GUIDE.md §Phase 9](../IMPLEMENTATION-GUIDE.md#phase-9-per-process-memory-management-rework).

---

**Files Changed:**
- `sources/nonux/.gitignore` — added `*.html` rule under "Build artifacts"
- `sources/nonux/README.html` — deleted
- `sources/nonux/components/README.html` — deleted
- `sources/nonux/core/README.html` — deleted
- `sources/nonux/docs/README.html` — deleted
- `sources/nonux/docs/framework-bootstrap.html` — deleted
- `sources/nonux/docs/framework-components.html` — deleted
- `sources/nonux/docs/framework-hooks.html` — deleted
- `sources/nonux/docs/framework-ipc.html` — deleted
- `sources/nonux/docs/framework-registry.html` — deleted
- `sources/nonux/framework/README.html` — deleted
- `sources/nonux/interfaces/README.html` — deleted
- `sources/nonux/lib/README.html` — deleted
- `sources/nonux/test/README.html` — deleted
- `sources/nonux/third_party/README.html` — deleted
- `sources/nonux/tools/README.html` — deleted
