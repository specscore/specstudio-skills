# Feature: Change Publication Policy

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/change-publication-policy?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/change-publication-policy?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/change-publication-policy?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/change-publication-policy?op=request-change) |
**Status:** Draft
**Date:** 2026-06-01
**Owner:** alex
**Source Ideas:** configurable-change-publication-policy
**Supersedes:** —

## Summary

SpecStudio skills consume a shared publication policy to decide, at artifact lifecycle events and command milestones, whether to leave edits unstaged, stage them, commit them, and push them. The durable config schema is owned by `specscore`'s `publication-policy-config` Feature, and mutation/resolution commands are owned by `specscore-cli`'s `cli/publication-policy` Feature; this Feature owns how SpecStudio skills resolve, present, and enforce the policy while preserving user intent around git state.

## Problem

SpecStudio skills currently hard-code their handoff behavior. Most skills stage files and require the user to commit; `implement` is being revised toward an explicit commit-on-behalf override. That is too rigid for users who want different publication granularity at different lifecycle checkpoints, such as staging an `idea.drafted` artifact for review but committing and pushing once `idea.approved` fires.

The policy also cannot be a simple "auto" switch. Commit and push have different risk profiles, branch rules matter, and git state may contain unrelated staged or unstaged work. SpecStudio needs a shared policy model that is easy to choose in conversation, durable when users want defaults, and strict enough that skills do not accidentally publish work the user did not approve.

## Behavior

### Cross-Repo Ownership

#### REQ: companion-spec-boundaries

This Feature MUST treat the publication policy as a three-repo contract:

- `specscore` owns the durable config schema for project, user, event, command, milestone, and branch-safety policy in its `publication-policy-config` Feature.
- `specscore-cli` owns deterministic config mutation and resolution helpers, including branch-rule validation and path-manifest-oriented git helpers, in its `cli/publication-policy` Feature.
- `specstudio-skills` owns conversational collection of user preference, milestone selection, manifest tracking, user approval gates, and invocation of the CLI helpers.

SpecStudio skills MUST NOT define a competing config schema in this repo.

#### REQ: cli-mutates-durable-config

When a user chooses to save a publication preference to durable user or project config, a SpecStudio skill MUST ask the `specscore` CLI to perform the mutation when the CLI provides the needed command. The skill MUST NOT hand-edit `specscore.yaml` or user-level config for durable publication settings when an equivalent CLI mutation exists.

If the needed CLI mutation does not exist yet, the skill MAY keep the preference at run or session scope, but durable persistence MUST be blocked with a clear message that the companion CLI config command is required.

### Policy Model

#### REQ: action-list-model

The canonical resolved policy MUST be an ordered action list, not a combined enum. The MVP action set is exactly:

- `stage`
- `commit`
- `push`

An empty list means "just edit." `commit` implies `stage` for validation purposes, and `push` implies both `stage` and `commit`. The resolver or consuming skill MUST reject invalid sequences such as `[push]` or `[stage, push]` before any git operation runs.

#### REQ: first-run-choice-mapping

When no effective policy is configured at any scope, the first SpecStudio producer skill that reaches a publication checkpoint MUST ask for the user's preferred workflow with exactly these user-facing options:

| User-facing option | Resolved actions |
|---|---|
| `just edit` | `[]` |
| `stage` | `[stage]` |
| `commit` | `[stage, commit]` |
| `commit & push` | `[stage, commit, push]` |

`stage` MUST be the default highlighted option because it preserves current SpecStudio behavior.

#### REQ: save-scope-options

The first-run prompt MUST ask where to apply the selected preference: this run only, this session, user default, or project default. Run and session preferences are held by the skill runtime. User and project defaults are durable and MUST be persisted via `specscore-cli` per `cli-mutates-durable-config`.

### Resolution Axes

#### REQ: event-first-resolution

Policy SHOULD be configured primarily against artifact lifecycle events, not commands. A policy targeting `idea.approved` applies whenever that event is emitted, regardless of which command or future UI surface caused it.

The resolver MUST support event-level policy for at least the existing events in `skills/shared/events.md`, including `idea.drafted`, `idea.approved`, `idea.updated`, `feature.specified`, `feature.approved`, and `feature.updated`.

#### REQ: command-and-milestone-overrides

The resolver MUST support command-level overrides and command-scoped event or milestone overrides. Command overrides are secondary to event policy and exist for cases where one producer command needs different behavior for the same event.

When a useful checkpoint has no canonical event, this Feature MAY define a milestone name such as `implement.batch-approved`; the milestone MUST be documented in a canonical catalog before users can configure against it.

#### REQ: precedence

The effective policy MUST be resolved by specificity and scope in this order:

1. Task override.
2. Session override.
3. Command-scoped event or milestone override.
4. Event or milestone override.
5. Command default.
6. Project default.
7. User default.
8. Built-in default.

Branch deny rules are safety constraints and MUST NOT be loosened by a more-specific lower-safety setting. If any applicable project or user branch rule denies push, `push` is removed or blocked even when a more-specific action list includes it.

### Git Safety

#### REQ: branch-guard-before-push

Before executing a `push` action, the skill or CLI helper MUST resolve the current branch and apply configured allow and deny patterns. Push MUST be refused on detached HEAD, missing upstream, denied branch names, or branches that match no allow pattern when an allow list is configured.

The default deny list SHOULD include `main`, `master`, and `release/*` unless a project explicitly configures otherwise in the companion `specscore` schema.

#### REQ: manifest-based-staging

Each SpecStudio skill MUST track a manifest of paths it created or edited for the current checkpoint. Staging MUST apply only to manifest paths unless the user explicitly approves including broader work.

When the `specscore` CLI edits files on behalf of a skill, such as status transitions or config mutations, the CLI helper MUST return or print the touched paths so the skill can add them to the manifest.

#### REQ: no-unrelated-auto-commit

Before an automatic or ask-confirmed commit, the skill MUST compare the approved manifest to the staged index. If unrelated staged paths are present, the skill MUST stop and ask whether to include, unstage, or abort. The skill MUST NOT silently commit unrelated staged changes.

Unstaged unrelated changes MAY remain in the working tree; they are not included unless the user explicitly adds them to the manifest.

### Skill Integration

#### REQ: checkpoint-disclosure

At every publication checkpoint, the skill MUST disclose the resolved policy before acting. The disclosure MUST name the source of the policy when known, the event or milestone being handled, the resolved action list, and any branch-safety effect such as "push removed because branch is `main`."

#### REQ: approval-gates-preserved

Publication policy MUST NOT bypass existing human or reviewer gates. For example, `idea.approved: [stage, commit, push]` runs only after the user explicitly approves the Idea's Recommended Direction and the Idea status transition has completed.

#### REQ: event-payload-publication-result

When a skill emits an event after applying publication policy, the event payload or envelope extension MUST record the publication outcome: resolved actions, executed actions, skipped actions with reasons, commit SHA when a commit was created, and push target when a push succeeded.

The exact payload field names are owned by the companion event/schema work, but SpecStudio consumers MUST expose this information to downstream automation.

## Acceptance Criteria

### AC: durable-config-is-cli-owned (verifies REQ: companion-spec-boundaries, REQ: cli-mutates-durable-config)

**Given** a user chooses `commit` and asks to save it as a project default,
**When** the SpecStudio skill persists that preference,
**Then** it invokes the companion `specscore` CLI config mutation command rather than hand-editing `specscore.yaml`.

### AC: first-run-prompt-defaults-to-stage (verifies REQ: first-run-choice-mapping, REQ: save-scope-options)

**Given** no user, project, session, command, event, milestone, or task publication policy applies,
**When** a producer skill reaches its first publication checkpoint,
**Then** it prompts with `just edit`, `stage`, `commit`, and `commit & push`, highlights `stage` as the default, and asks whether the preference applies to this run, this session, user default, or project default.

### AC: idea-draft-stage-approved-push (verifies REQ: event-first-resolution, REQ: approval-gates-preserved)

**Given** project policy maps `idea.drafted` to `[stage]` and `idea.approved` to `[stage, commit, push]`,
**When** `specstudio:ideate` writes a lint-clean draft and later the user explicitly approves the Recommended Direction,
**Then** the draft checkpoint stages the manifest without committing, and the approval checkpoint stages, commits, and attempts a guarded push only after the approval transition completes.

### AC: command-scoped-override-wins (verifies REQ: command-and-milestone-overrides, REQ: precedence)

**Given** event policy maps `feature.approved` to `[stage, commit, push]` and command-scoped policy maps `specify.events.feature.approved` to `[stage, commit]`,
**When** `specstudio:specify` emits `feature.approved`,
**Then** the effective action list is `[stage, commit]` for that command while other producers of `feature.approved` still use the event-level policy.

### AC: branch-deny-blocks-push (verifies REQ: branch-guard-before-push, REQ: precedence)

**Given** the resolved action list includes `push` and branch policy denies `main`,
**When** the current branch is `main`,
**Then** the skill refuses to push, reports the branch-rule reason, and does not bypass the denial because a task, command, session, or user override requested push.

### AC: manifest-prevents-unrelated-commit (verifies REQ: manifest-based-staging, REQ: no-unrelated-auto-commit)

**Given** a skill manifest contains `spec/ideas/foo.md` and an unrelated file is already staged,
**When** the resolved policy includes `commit`,
**Then** the skill stops before committing and asks whether to include, unstage, or abort rather than silently committing the unrelated staged file.

### AC: cli-touched-paths-join-manifest (verifies REQ: manifest-based-staging)

**Given** a skill invokes a CLI helper that transitions an artifact status,
**When** the CLI edits one or more files,
**Then** the CLI returns the touched paths and the skill adds those paths to the checkpoint manifest before staging or committing.

### AC: checkpoint-disclosure-before-action (verifies REQ: checkpoint-disclosure)

**Given** any resolved publication policy,
**When** a skill reaches a configured event or milestone,
**Then** it displays the event or milestone name, the resolved action list, and the policy source before running stage, commit, or push actions.

### AC: event-records-publication-result (verifies REQ: event-payload-publication-result)

**Given** a skill emits an event after applying publication policy,
**When** downstream automation reads the event,
**Then** it can determine which actions were resolved, which actions executed, which were skipped and why, and the commit SHA or push target when those actions succeeded.

## Open Questions

- Should this Feature define exact config key names, or should that wait for the companion `specscore` Feature? Lean: companion `specscore` owns exact schema; this Feature should show examples only.
- Should the CLI helper that commits and pushes live under a general `specscore git` namespace or under publication-specific commands? Lean: publication-specific, because the helper needs policy context and manifest semantics.
- Should `push` require an upstream branch to already exist, or may a policy opt into `--set-upstream` for new feature branches? Lean: require upstream in MVP; adding upstream creation is a separate opt-in.
- What is the minimal milestone catalog for `implement`? Batch-level approval is needed, but the exact stable names should be specified alongside the `implement` Feature revision.

---
*This document follows the https://specscore.md/feature-specification*
