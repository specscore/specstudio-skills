# Seed Template

This is a reference file showing the exact shape of a seed at `spec/ideas/seeds/<slug>.md`. It is documentation, not a real seed — it does not live under `spec/ideas/seeds/`, so the lint rule does not target it.

## Minimal seed (H1 only)

```markdown
---
type: sidekick-seed
slug: persist-debug-logs-across-restarts
captured_at: 2026-05-18T14:32:00Z
captured_by: specstudio:specify
captured_during: spec/features/skills/init
trigger: heuristic
status: queued
synchestra_task: null
---

# Persist debug logs across Claude Code restarts so post-mortems don't lose context
```

## Seed with optional body

```markdown
---
type: sidekick-seed
slug: caching-strategy-for-search-index
captured_at: 2026-05-18T15:01:00Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# Caching strategy for the search index

## Why it surfaced
Three places in `search/indexer.py` and `search/query.py` re-compute the
same shard map within a single request lifecycle.

## Affected files
- `search/indexer.py:42-78`
- `search/query.py:91-104`

## Out of scope for current task
This is a follow-up to the current refactor; capturing so we don't lose it.
```

## Frontmatter contract

The 8 keys in the YAML block are required. Their values follow REQ `seed-frontmatter-schema` in the [`sidekick-capture` Feature](../../../spec/features/sidekick-capture/README.md):

| Key | Type | Notes |
|---|---|---|
| `type` | string | Literal `sidekick-seed`; never any other value |
| `slug` | string | Kebab-case; matches filename without `.md` |
| `captured_at` | ISO-8601 | UTC preferred |
| `captured_by` | string | `<plugin>:<skill>` for skills, literal `"user"` for direct user invocation |
| `captured_during` | string or null | Spec path of the active artifact, or null when no active spec context |
| `trigger` | enum | `heuristic` or `explicit` |
| `status` | string | Literal `queued` at capture time; Phase 1 consilium modifies the value |
| `synchestra_task` | null | Literal `null` at capture time; Phase 1 consilium populates with task ID |

## Body contract

- First non-blank line MUST be an H1 (`# <one-liner>`) containing the verbatim one-liner.
- Optional markdown content may follow (subheadings, lists, code blocks, etc.).
- Total body length (everything after the closing `---`, inclusive of the H1 line) MUST NOT exceed 2000 characters.
- Bodies that need more context have outgrown "seed" status and should become full SpecScore Ideas via `specstudio:ideate`.
