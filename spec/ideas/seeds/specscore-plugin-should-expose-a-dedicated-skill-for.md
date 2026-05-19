---
type: sidekick-seed
slug: specscore-plugin-should-expose-a-dedicated-skill-for
captured_at: 2026-05-19T18:39:12Z
captured_by: user
captured_during: spec/features/skills/implement
trigger: explicit
status: queued
synchestra_task: null
---

# specscore plugin should expose a dedicated skill for changing Feature/Idea lifecycle status, separating it from the broad specscore:feature / specscore:idea navigation skills

**Repo:** [`ai-plugin-specscore`](https://github.com/specscore/ai-plugin-specscore).

## Observed problem

In this session the agent missed `specscore feature change-status` and defaulted to direct file edit + `specscore spec lint --fix` to flip a Feature's `**Status:**`. The CLI commands exist (`specscore feature change-status`, `specscore idea change-status`) and produce the same result. The miss happened because `specscore:feature` bundles seven capabilities (list/tree/info/deps/refs/new/change-status) into one skill description — the lifecycle-transition capability isn't surfaced as a dedicated skill or slash trigger, unlike the focused `/sidekick`, `/ideate`, `/specify`, `/plan` patterns.

## Suggested direction

Add dedicated skills:
- `specscore:feature-status` (or `/feature-status`) — wraps `specscore feature change-status`. Validates target status, runs lint --fix after, reports the diff.
- `specscore:idea-status` (or `/idea-status`) — wraps `specscore idea change-status` with same pattern.

The existing `specscore:feature` and `specscore:idea` skills stay as navigation/inspection; the new status skills are focused on the transition action. Alternative: single `specscore:lifecycle` skill handling both. Less duplication but less discoverable.

## Why this matters

The Status field is managed state — Synchestra reconciles linkage based on it. Direct edits bypass the CLI's validation (allowed-transitions check, audit-event emission). Surfacing the CLI as a dedicated skill makes the correct path the easy path.
