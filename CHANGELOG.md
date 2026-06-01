# Changelog

## 0.0.7

- **publication policy protocol** — producer skills now resolve shared publication policy at lifecycle events and implementation milestones instead of hard-coding stage-only handoffs.
- **manifest-safe publication** — added shared guidance for first-run publication preferences, path manifests, unrelated-index checks, branch-safe pushes, and `publication_result` event metadata.
- **cross-repo policy specs** — added companion Feature specs for durable publication config in `specscore` and CLI mutation/resolution helpers in `specscore-cli`.

## 0.0.6

- **sidekick multi-repo destination resolution** — `specstudio:sidekick` now resolves which SpecScore-managed repo a captured seed belongs to when multiple repos are open in the workspace, with an `UNCERTAIN` escape clause when identity signals conflict.
- **relocate-idea skill shipped** — `specstudio:relocate-idea` is a thin shell over `specscore idea relocate` that moves an Idea or sidekick seed to another SpecScore repo and appends one JSON line to `.specscore/destination-resolution-log.jsonl` for future tuning.
- **manifest description** — lifecycle arrows in the plugin description use `→` instead of `⇒`.

## 0.0.5

- **PRINCIPLES.md added** — top-level repo principles doc; first principle is "Respect the user's time and attention" with three operational sub-principles.
- **plan Feature revised** — additive revision adding `**Mode:** <full|stub>`, `**Status:**`, and `**Depends-On:**` task body fields; lint rules `P-003` (Depends-On cycle / dangling / self-ref) and `P-004` (placeholder body on done-status task in stub Plan); canonical placeholder body token `<!-- implement: pending -->`.
- **implement skill shipped** — `specstudio:implement` dispatches parallel subagents per task in batches computed from the Plan's dependency graph; stages all changes with a mandatory `Verifies: <feature-slug>#ac:<ac-slug>, ...` commit-message trailer; per-batch user-approval gate.
