---
captured_by: user
status: queued
---
# CLI and sidekick skill need a seed change-status (close) verb so implemented seeds can be retired without manual frontmatter edits

## Problem

Sidekick seeds have no CLI-driven terminal status for "the idea shipped." `specscore idea change-status` targets only full Ideas (`spec/ideas/<slug>.md`) with the enum {Draft, Approved, Specifying, Specified, Implementing, Implemented, Archived}; `feature change-status` is Features-only; `specscore sidekick` exposes only `new`. None can set a seed under `spec/ideas/seeds/` to a closed state.

The only existing `status: completed` seed (`specscore-cli-companion-implement-plan-feature-lint-rules`) was closed by a manual frontmatter hand-edit (commit 1d273b9), which contradicts the SpecScore tenet that all status transitions go through a CLI verb (see companion seeds `audit-all-specscore-skills-for-manual-status-edit`, `update-ideate-and-specify-skills-to-transition-artifact`).

## Suggested direction

Add a CLI verb (e.g. `specscore sidekick change-status <slug> --to=<status>`) and define the seed terminal-status vocabulary. Canonical seed terminals today are `promoted` (via `idea promote`) and `deprecated` (consilium reject); a third `completed`/`implemented` terminal is needed for seeds whose idea shipped directly without promotion (common for dogfood-finding seeds). `--to=archived` should also move the seed to an archived path, mirroring `idea change-status --to=archived`. Lint should recognize the chosen vocabulary.

CLI-side tracked at specscore/specscore-cli#72; the skill-side close flow is tracked by companion seed `should-there-be-a-close-lifecycle-skill-that-retires-an`.

## Provenance

Surfaced 2026-06-16 while triaging the seed queue: 5 dogfood-finding seeds were verified as implemented in skills/implement, skills/verify, and skills/sidekick, but there was no CLI path to mark them closed.
