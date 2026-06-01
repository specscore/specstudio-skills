# `plan` skill

The `specstudio:plan` Claude Code skill — turns an approved SpecScore Feature into a lint-clean Plan artifact at `spec/plans/<slug>.md`, an ordered list of tasks where each task references one or more AC IDs from its source Feature.

- **Skill manifest:** [`SKILL.md`](./SKILL.md) — the canonical, machine-readable skill definition.
- **Feature specification:** [`spec/features/skills/plan/`](../../spec/features/skills/plan/README.md) — the SpecScore Feature this skill implements.
- **Skills index:** [`../README.md`](../README.md) — all SpecStudio skills with status and lifecycle position.

## What it does

Produces a single `spec/plans/<slug>.md` file with tasks numbered 1..N, each bound to one or more AC IDs from the source Feature. Hard-gated on:

1. **AC coverage** — every AC in the source Feature is in a task's `**Verifies:**` line OR explicitly listed under `## Deferred AC Coverage` with a concrete reason. Enforced by lint rule `P-001`.
2. **Lint** — `specscore spec lint` passes.
3. **Reviewer** — the baseline plan-document reviewer subagent returns `Approved`, plus any third-party reviewers registered in `specscore.yaml` (AND composition).
4. **User approval** — explicit phrase, with a single confirmation step for vague positive signals.

The MVP is intentionally narrow: single Feature per Plan, strict linear task order, no Rehearse stub scaffolding, no runner dispatch. DAG ordering and Rehearse integration are follow-on Ideas.

## Status

**Shipped.** Skill exists at `skills/plan/` and is usable today via Claude Code.

## Triggers

`plan`, `/plan`, `specstudio:plan`, "plan this feature", or the event `feature.approved`.

## What it does NOT do

- Plan against unapproved Features (refuses with a redirect to `specstudio:specify`).
- Plan across multiple Features in one artifact (single-Feature scope).
- Write DAG / parallel-branch plans (linear-only in MVP).
- Generate code or bypass publication policy. Staging, committing, and pushing Plan artifacts are handled by the shared publication checkpoint protocol after reviewer and user gates release.
- Invoke any skill other than `specstudio:implement` on transition.
