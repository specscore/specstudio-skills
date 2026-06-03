# Changelog

## 0.0.8

- **reviewer-gates event-keyed revision** — gate keys migrate from command/skill names to events (`gates.feature.approved`); legacy command-keyed gates are rejected with a migration error. Adds `deterministic` (verdict from a `run:` command's exit code) and `noop` (always-approve placeholder) reviewer types, and registers the multi-fire gate-point events `implementation.pre_commit` / `implementation.pre_push`.
- **per-branch gate masks** — a reviewer entry may carry an optional `when: "branch =~ <anchored-regex>"` condition; the entry participates in the gate only when the current branch matches, keeping per-branch autonomy in the gates layer.
- **implement-autonomy layer** — `specstudio:implement` now fires the `implementation.pre_commit` / `implementation.pre_push` gates at its checkpoints, so per-batch approval is gate-config-driven (a `noop`/`deterministic`-only gate commits autonomously; a `type: human` reviewer is the human checkpoint) rather than a hardcoded step.
- **autonomy execution knobs** — adds the top-level `autonomy:` config namespace with `autonomy.implement.commit_cadence` (`task`/`batch`/`plan`, default `batch`, scope-ladder resolved), anomaly-halts (sibling conflict / BLOCKED subagent / unresolved-after-`--fix` lint / source-Feature drift) that fire regardless of gate config, explicit `continue` re-arm scoped to the run, a cumulative review fed to a `type: human` push reviewer, and an explicit push branch-safety floor autonomy cannot bypass.
- **current-branch topology reconciled (Option B)** — `protected-branch-commit-confirmation` no longer requires a commit-time prompt onto a protected branch; the checkpoint relocates to promote/push.

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
