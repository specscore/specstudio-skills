# Sidekick Capture Lint Rule — Cross-Repo Companion Plan Stub

**Status:** Stub. This plan exists in *this* repo to record the dependency. The actual implementation work happens in [`specscore/specscore-cli`](https://github.com/specscore/specscore-cli).

**Source contract:** REQ `seed-lint-rule` in [`spec/features/sidekick-capture/README.md`](../features/sidekick-capture/README.md).

## What needs to ship in specscore-cli

A new lint rule registered against the SpecScore CLI that:

1. Targets files matching `spec/ideas/seeds/*.md`.
2. Recognizes them as the `sidekick-seed` document type.
3. Rejects:
   - (a) unknown frontmatter keys
   - (b) missing required keys (8 keys from REQ `seed-frontmatter-schema`)
   - (c) `type` values other than `sidekick-seed`
   - (d) `trigger` values outside `{heuristic, explicit}`
   - (e) bodies whose first non-blank line is not an H1 (`# <text>`)
   - (f) bodies (after frontmatter, inclusive of H1) exceeding 2000 characters

## Why it's not implemented here

This repo (`specstudio-skills`) authors the *contract*. The CLI authors the *enforcement*. Keeping the rule in the CLI repo means every SpecScore project gets the rule by upgrading the CLI, without per-project plumbing.

## How to verify the rule is live

After the rule ships in specscore-cli and a SpecScore project upgrades:

```bash
# Write a fixture seed with an unknown key
cat > spec/ideas/seeds/_test.md <<EOF
---
type: sidekick-seed
slug: test
captured_at: 2026-05-18T00:00:00Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
unknown_key: oops
---

# Test seed
EOF

specscore spec lint
# Expected: violation flagging unknown_key under spec/ideas/seeds/_test.md
rm spec/ideas/seeds/_test.md
```

## Tracking

- **Upstream issue:** [`specscore/specscore-cli#6`](https://github.com/specscore/specscore-cli/issues/6) — "Add `sidekick-seed` lint rule"
- Until the rule ships, the contract is enforceable only by visual review; Phase 0 still functions because the sidekick skill enforces frontmatter and body shape at write time (defense-in-depth per the directive).
