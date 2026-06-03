# Idea: Configurable Change Publication Policy

**Status:** Specifying
**Date:** 2026-05-31
**Owner:** alex
**Promotes To:** change-publication-policy
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we let SpecStudio users choose, per user, project, session, command, and task, whether skills stop at staged changes, commit automatically, push automatically, or ask at each boundary without making protected branches unsafe?

## Context

Triggered by revising specstudio:implement from strict stage-only behavior to an explicit commit-on-behalf override. The broader pattern applies to every SpecStudio producer skill: users may want fully manual control in some repos, automatic commits in low-risk branches, and guarded automatic pushes only where branch policy allows it. Existing repo config already uses a top-level specscore.yaml with per-command keys such as gates.specify, so this should use a comparable configuration model rather than one-off skill prose.

## Recommended Direction

Define a shared change-publication policy consumed by all SpecStudio producer commands. The user-facing setup prompt should offer four plain workflow choices: `just edit`, `stage`, `commit`, and `commit & push`. Internally, durable config stores ordered action lists so the skill can reason safely: edit-only is `[]`, stage is `[stage]`, commit is `[stage, commit]`, and commit-and-push is `[stage, commit, push]`.

When no effective policy is configured at any scope, the first producer skill that reaches a handoff point should ask the user for their preference instead of silently assuming automation. `stage` is the default highlighted choice because it preserves today's behavior. The same prompt should ask where to save the choice: this run only, this session, user default, or project default. Auto-push choices require a branch-safety rule before they can be saved beyond the current run.

Resolve policy by precedence across task, session, project, user, and built-in defaults, with optional per-command and per-milestone overrides at the user and project scopes. Milestones are named workflow checkpoints such as `idea.drafted`, `idea.approved`, `feature.specified`, `feature.approved`, `implement.batch-approved`, and `verify.completed`. Event names should be reused where the event already exists; when a useful handoff point has no event yet, the Feature should either add a real event or define a local milestone name explicitly.

Automatic push must be branch-aware. A policy may allow auto-push only for branch patterns such as `feature/*` while denying protected names such as `main`, `master`, `release/*`, or any project-configured denylist. The effective policy is always the most specific applicable setting, but safety constraints are monotonic: a lower-precedence allow cannot override a higher-precedence branch deny.

Start with configuration and enforcement in the skills, not a new daemon. Each skill resolves the effective policy before each configured milestone, reports the resolved commit/push behavior in its user-facing gate, and records whether the action was manual, ask-confirmed, or automatic in the relevant event payload.

Example project intent:

```yaml
publication:
  default:
    actions: [stage]

  commands:
    ideate:
      milestones:
        idea.drafted:
          actions: [stage]
        idea.approved:
          actions: [stage, commit, push]

  push:
    allow_branches: ["feature/*", "fix/*"]
    deny_branches: ["main", "master", "release/*"]
```

That configuration means an Idea draft is staged for human review, while the approved Idea is committed and pushed once the user explicitly approves the Recommended Direction and branch policy allows the push.

## Alternatives Considered

- **Keep only per-batch commit-on-behalf prompts.** This is the smallest continuation of the current `implement` revision: the skill stages by default and the user may explicitly ask it to commit. Lost because it does not scale to users who want a standing preference, does not cover push, and does not apply consistently to `ideate`, `specify`, `plan`, `verify`, or `recap`.
- **One global automation boolean.** A single setting such as `auto_publish: true` is easy to explain. Lost because commit and push have different risk profiles. Auto-commit with manual push is reasonable for many users; auto-push to `main` is not. A boolean also cannot express branch policies or per-command behavior.
- **Project-only config in `specscore.yaml`.** This would match the existing project config style and avoid user-local state. Lost because commit/push preference is partly personal. One contributor may want manual commit review while another wants auto-commit on feature branches in the same repo. The project should define safety rails and team defaults; the user should still own personal defaults within those rails.
- **Git hooks or CI-driven publication.** Let skills stage changes and rely on hooks, CI, or external automation to commit or push. Lost because the user intent is about the skill's handoff behavior inside the conversational workflow. External hooks are still valid, but they do not tell the skill whether to stop, ask, commit, push, or report the outcome.

## MVP Scope

A shared policy contract plus first consumer wiring across SpecStudio producer skills. The MVP should support a first-run no-config prompt with four options (`just edit`, `stage`, `commit`, `commit & push`), user defaults, project defaults in specscore.yaml, session overrides, per-command overrides, per-milestone overrides, and per-task overrides for plan-backed implementation tasks. It should separate staging, commit, and push internally, enforce branch allow/deny patterns before push, and preserve today's conservative behavior as the default highlighted option: stage/manual commit/manual push unless configured otherwise.

## Architecture Sketch

The durable config format belongs in the main `specscore` repo because it extends the canonical `specscore.yaml` / user-config schema. This repo should own the SpecStudio skill behavior that consumes that schema: when policies are resolved, what user prompts appear, how event/milestone names map to handoff points, and what safety gates are applied before a skill proceeds.

Git intent should remain agent-owned, not CLI-inferred. The agent/skill knows which files it created or edited, which staged files pre-existed, what diff the user approved, and whether the user asked to include unrelated work. The CLI can provide deterministic helpers — parse config, resolve policies, validate branch allow/deny rules, write status transitions, output touched paths, maybe perform a commit/push when given an explicit manifest — but it should not decide that arbitrary staged or unstaged files belong in a SpecStudio commit.

Staging should therefore be manifest-based. Each skill tracks the paths it touched and stages only those paths unless the user explicitly asks to include broader changes. Before auto-commit, the skill compares the approved manifest to `git diff --staged --name-only`; if unrelated staged changes are present, it stops and asks rather than committing them silently. When the CLI itself edits a file, such as a status transition, it should return or print the exact changed path so the agent can add it to the same manifest.

## Not Doing (and Why)

- Replacing git branch protection — remote branch rules remain authoritative; this policy only decides what the skill may attempt
- Background auto-publish daemon — actions happen only inside an explicit skill invocation
- Silent pushes to protected branches — branch deny rules and remote failures are surfaced, never bypassed
- One global boolean for automation — commit and push are separate policies because auto-commit/manual-push is a valid workflow

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | Users can understand two separate knobs (`commit_policy`, `push_policy`) better than a single "automation" mode. | Dogfood three flows: manual everything, auto-commit/manual-push, and auto-commit/auto-push on feature branches. If users keep asking what mode they are in, simplify the vocabulary before specifying. |
| Must-be-true | The four setup choices (`just edit`, `stage`, `commit`, `commit & push`) map cleanly to the lower-level policy fields without hiding safety-critical push behavior. | Prototype the resolver table and show the resolved low-level actions in the prompt. If users expect "commit & push" to bypass branch rules, rename the option to "commit & guarded push." |
| Must-be-true | Branch allow/deny matching can be made deterministic enough for safety-critical push decisions. | At Feature time, define the exact pattern syntax and precedence, then test `main`, `master`, `release/*`, `feature/*`, detached HEAD, and no-upstream branch cases. |
| Must-be-true | Event and milestone names can be made stable enough to configure against. | Audit every producer skill and build a canonical milestone table. If a user-facing handoff point has no stable event, either add an event or define an explicit non-event milestone in the Feature. |
| Must-be-true | Per-task overrides have a stable home in Plan-backed workflows without polluting non-Plan flows. | Prototype a `**Publication Policy:**` or split `**Commit Policy:**` / `**Push Policy:**` task metadata line in a Plan and run `specscore spec lint` design review against it. |
| Should-be-true | The same policy resolver can serve every producer skill even though their handoff points differ. | Map handoff points for `ideate`, `specify`, `plan`, `implement`, `verify`, and `recap`; confirm each can call the resolver after staging and before final event emission. |
| Should-be-true | Project config should be allowed to constrain user config, especially for push. | During specify, decide whether project branch denylists are hard safety rails that user/session/task overrides cannot loosen. Bias: yes for push, no for commit. |
| Might-be-true | Users will want named workflow presets such as `manual`, `local-auto`, and `feature-branch-auto`. | Keep presets out of the core contract initially; add them later only if raw policy config proves repetitive. |


## SpecScore Integration

- **New Features this would create:** a shared `change-publication-policy` Feature, likely under `spec/features/change-publication-policy/`, plus follow-on wiring Features for each consumer skill if the first Feature stays contract-only.
- **Existing Features affected:** [`skills/ideate`](../features/skills/ideate/README.md), [`skills/specify`](../features/skills/specify/README.md), [`skills/plan`](../features/skills/plan/README.md), [`skills/implement`](../features/skills/implement/README.md), [`skills/verify`](../features/skills/verify/README.md), and [`skills/recap`](../features/skills/recap/README.md) all currently describe staging/commit handoff behavior directly. The [`init`](../features/skills/init/README.md) Feature may also be affected if it should scaffold default project policy in `specscore.yaml`.
- **Dependencies:** the canonical SpecScore repo config contract for `specscore.yaml`, the existing event vocabulary in [`skills/shared/events.md`](../../skills/shared/events.md), and Plan/task metadata parsing in the `specscore` CLI if per-task override lines become canonical.
- **Cross-repo ownership:** the config schema and any deterministic CLI helpers belong in the main `specscore` repo; the SpecStudio skill docs in this repo own the conversational policy application and git-intent safeguards.

## Open Questions

- **User config location.** Should user defaults live in a SpecScore-specific file under the user's home directory, in platform agent config, or in git config? Lean: SpecScore user config, with future git-config bridge if users ask for it.
- **Session override syntax.** Should session settings be natural-language only ("for this session, auto-commit but don't push"), a CLI-style flag, or a durable ephemeral file? Lean: natural-language/session memory inside the skill, surfaced in the resolved-policy summary before action.
- **First-run prompt shape.** Should the no-config prompt ask both questions at once (workflow choice plus save scope), or first ask workflow choice and then ask whether to save it? Lean: one batched prompt so users can choose `stage` + `session` quickly without a second interruption.
- **Per-command naming.** Should config keys be bare command names (`implement`) like `gates.specify`, or fully qualified names (`specstudio:implement`) to avoid cross-plugin collisions? Reviewer gates chose bare names for MVP; reuse that unless a collision becomes concrete.
- **Milestone catalog.** Which checkpoints are canonical configuration targets for each command? Ideate already has `idea.drafted`, `idea.approved`, and `idea.updated`; lifecycle tooling also has `idea.implementing` and `idea.specified`. Implement has batch-level checkpoints that may need names more precise than the current event list.
- **Commit message ownership.** When `commit_policy: auto`, does the skill always use its proposed template, or may config define message prefixes/scopes? Lean: template only in MVP; message customization is separate scope.
- **Push target.** Should auto-push only push to the current branch's configured upstream, or may config define a remote and branch pattern? Lean: current upstream only in MVP; no implicit `--set-upstream` without an ask gate.

---
*This document follows the https://specscore.md/idea-specification*
