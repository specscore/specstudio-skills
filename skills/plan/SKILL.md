---
name: plan
description: |
  Turns an approved SpecScore Feature into a lint-clean Plan artifact
  at spec/plans/<slug>.md — an ordered list of tasks where each task
  references one or more AC IDs from its source Feature. Hard-gates on
  full AC coverage (every AC covered or explicitly deferred), lint,
  reviewer-subagent approval, and user approval. Optionally scaffolds
  follow-on artifacts via downstream skills.
  Trigger: "plan", "/plan", "plan this feature", "specstudio:plan", or
  Synchestra event `feature.approved`.
aliases: [plan]
---

# Plan

Turn an approved SpecScore Feature into a lint-clean, AC-mapped, ordered task list.

## Hard Gate

<HARD-GATE>
Do NOT invoke `specstudio:implement`, `writing-plans`, `frontend-design`, `mcp-builder`, or ANY implementation skill until ALL of the following are true:
  1. The Plan artifact exists at `spec/plans/<slug>.md` and contains at least one `### Task N: <name>` task inside the `## Tasks` section.
  2. Every acceptance criterion in the source Feature is either covered by ≥1 task (via the task's `**Verifies:**` line) OR explicitly listed under `## Deferred AC Coverage` with a concrete one-sentence reason.
  3. `specscore spec lint` passes.
  4. The plan-document reviewer subagent returned `Approved`.
  5. The user has explicitly approved the written Plan.

The only skill invoked after `specstudio:plan` is `specstudio:implement` (or — while `implement` is unshipped — a hand-back to the user with that recommendation).
</HARD-GATE>

## When to Use

- A SpecScore Feature is `**Status:** Approved` and the user is ready to decompose it into work.
- The Synchestra event `feature.approved` has fired and the user has confirmed they want to plan.
- The user wants to revise an existing approved Plan (revise-in-place by default).

**Refuse and redirect when:**

- The source Feature is `Draft` or `Under Review` → tell the user to run `specstudio:specify` to approve the Feature first. Do **not** plan against unapproved specs.
- The request spans two or more Features → say so and offer to write two separate Plans.
- The user asks for a DAG / parallel-branch plan → linear-only is the MVP; offer a linearized best-effort order or defer.

## Pre-Flight

1. **Source Feature check.** Resolve the input to a path `spec/features/<feature-slug>/README.md`. Confirm `**Status:**` is one of `{Approved, Implementing, Stable}`. If not, stop and recommend `specstudio:specify`.
2. **Parse AC list.** Read the source Feature's `## Acceptance Criteria` section. Build the full list of AC slugs — each `### AC: <slug>` heading or each inline `Scenario:` AC block with an explicit `verifies REQ:<slug>` reference. This list is the AC-coverage contract; every entry must be accounted for in the final Plan.
3. **Slug derivation.** Default Plan slug = source Feature slug. If the user explicitly asks for an alternative breakdown ("a spike plan", "a stable plan"), use `<feature-slug>-<variant>` (single segment, lowercase, hyphen-separated, URL-safe). Refuse chained-variant suffixes.
4. **Existing-plan check.** If `spec/plans/<slug>.md` already exists, ask the user: revise in place (default), supersede with a new plan that invalidates the prior task list, or pick a different variant suffix.
5. **Single-Feature scope.** If the user's intent references work in more than one Feature, stop and offer to write two Plans.

## Checklist

Create a task for each and complete in order:

1. **Pre-flight** (steps above).
2. **Convergent task-breakdown dialogue.** Walk the AC list with the user. For each AC, decide: dedicated task / bundled into a multi-AC task / deferred to `## Deferred AC Coverage` with a reason. Keep the conversation tight — batched questions, multiple-choice where possible.
3. **Order tasks linearly 1..N.** Respect inferable dependencies (an AC whose precondition is established by another AC's task). No gaps, no DAG, no parallel branches in the MVP.
4. **Author the Plan artifact** — single `spec/plans/<slug>.md` per the schema below.
5. **Auto-create `spec/plans/`** + index `README.md` if absent. Tell the user the directory is being bootstrapped.
6. **Auto-stage** all created files via `git add`. Report the staged paths.
7. **Lint** — `specscore spec lint`. On failure, run `specscore spec lint --fix` exactly once and re-lint; on persistent failure, surface violations with rule IDs and stop.
8. **Inline self-review** — placeholders, empty `**Verifies:**` lines, AC IDs that look misspelled vs the source Feature, task-name vs `**Verifies:**` contradictions.
9. **Status: Draft → Under Review.**
10. **Dispatch the baseline reviewer subagent** — see [reviewer-prompt.md](references/reviewer-prompt.md). Must return `Approved` before user review.
11. **Dispatch additional registered reviewers** (if any) from `specscore.yaml`'s `reviewers:` extension key. AND composition — every reviewer must return `Approved`.
12. **Emit `plan.drafted`** event after all reviewers Approved + lint passes.
13. **User review gate** — present the Plan with the summary of deferred ACs (if any) and explicitly ask for approval.
14. **On user approval** — status Under Review → Approved, re-run lint, emit `plan.approved`.
15. **Transition to `specstudio:implement`** — or, while `implement` is unshipped, hand back with the recommendation that the user implement task-by-task using a general-purpose skill.
16. **Throughout** — watch for sidekick ideas. When an out-of-scope improvement surfaces (e.g., a Feature change), invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step immediately. Do not derail.

## Artifact Layout

Plans are **single files**, not directories. Unlike Features, a Plan has no `_tests/` or `assets/` siblings in the MVP.

```
spec/plans/
├── README.md                # Plans index
└── <slug>.md                # The Plan artifact (single file per plan)
```

**Plan `.md` schema (authoritative):**

```markdown
# Plan: <Title>

**Status:** Draft
**Source Feature:** <feature-slug>
**Date:** YYYY-MM-DD
**Owner:** <author identifier>
**Supersedes:** —              <!-- or: <old-plan-slug> on wholesale replacement -->

## Summary

1–3 sentences. What this Plan covers and how it decomposes the source Feature.

## Approach

≤1 paragraph on the decomposition strategy. What grouped what, what got deferred and why.

## Tasks

### Task 1: <task-name>

**Verifies:** <feature-slug>#ac:<ac-slug>, <feature-slug>#ac:<ac-slug>

1–3 sentence description of what this task does.

### Task 2: <task-name>

**Verifies:** <feature-slug>#ac:<ac-slug>

…

## Deferred AC Coverage

<!-- Omit this section entirely when no ACs are deferred. -->

- <feature-slug>#ac:<ac-slug> — <one-sentence reason that specifies why and when>

## Outstanding Questions

- <Open question that doesn't block approval but should be tracked>

(Or: "None at this time." — the section is never omitted.)

---
*This document follows the https://specscore.md/plan-specification*
```

**Schema notes:**

- The canonical id is the file slug; there is no separate `id` field.
- `**Source Feature:**` is **required** and MUST resolve to an existing Feature at `spec/features/<feature-slug>/README.md` with `**Status:**` in `{Approved, Implementing, Stable}`.
- `**Supersedes:**` MUST be present with value `—` when empty.
- Tasks are numbered linearly starting from 1 with no gaps.
- Every task MUST have a `**Verifies:**` line listing ≥1 AC ID from the source Feature.
- AC IDs use the form `<feature-slug>#ac:<ac-slug>` so they are unambiguous when planning across Features in the future.

## AC Coverage Contract

The defining discipline of `plan`: **every AC in the source Feature is accounted for, exactly once.** Either:

1. Listed in at least one task's `**Verifies:**` line, OR
2. Listed under `## Deferred AC Coverage` with a one-sentence reason that names *why* and *when* (e.g., "Deferred to v2; waits on the third-party SDK release"; "Out of MVP scope; covered by the follow-on `<slug>` plan").

**Lint enforces this as `P-001`** (hard rule, not advisory). The reviewer subagent enforces it semantically (catches stale AC references, vague defer-reasons, AC-wrapper tasks).

Silent omission of an AC is a contract violation.

## Task Format

Each task MUST have:

1. A `### Task N: <task-name>` heading (numbered linearly).
2. A `**Verifies:** <feature-slug>#ac:<ac-slug>, …` line listing one or more AC IDs from the source Feature.
3. A 1–3 sentence prose description.

Tasks MAY include optional fields (e.g., `**Notes:**`). The three above are required.

### Task granularity

**Lower bound (lint-enforced):** every task references ≥1 AC ID.

**Upper bound (reviewer-advisory):** each task SHOULD be sized to be implementable in one focused work session and SHOULD reference no more than ~3 AC IDs. The reviewer flags:

- **AC wrappers** — tasks that restate a single AC's `Then` clause with no implementation work beyond it.
- **Bundled-too-fat tasks** — tasks that reference many ACs and look like they should be split.

These are advisory findings only; the user can override.

## Order Discipline

In the MVP, task order is **strictly linear** — numbered 1..N with no gaps, no DAG, no parallel branches. If the user requests DAG-style ordering, decline and offer one of:

- Linearize the DAG into a best-effort order (the reviewer will check the order respects inferable dependencies).
- Defer until a future plan-schema revision supports `**Depends-On:**` per task.

If the reviewer flags an ordering that violates an inferable dependency, the finding **must cite** the specific REQ slug or prose passage that establishes the dependency. Uncited dependency claims are not actionable.

## Lint and Self-Review

Run `specscore spec lint` after every write. On failure:

1. Run `specscore spec lint --fix` exactly **once**.
2. Re-run `specscore spec lint` to verify.
3. If passing, continue and tell the user what was auto-fixed.
4. If still failing, surface remaining violations with rule IDs and stop.

The skill MUST NOT loop `--fix`, and MUST NOT carry its own knowledge of which rules are auto-fixable.

**Inline self-review** (before dispatching the reviewer subagent) checks:

1. **Placeholders** — `TBD`, `TODO`, `???`, `FIXME`.
2. **Empty `**Verifies:**` lines** — every task has at least one AC ID.
3. **Misspelled AC IDs** — every `<feature-slug>#ac:<ac-slug>` reference resolves in the source Feature.
4. **Name/Verifies contradictions** — a task whose name describes work outside its claimed AC scope.

Fix inline. Don't re-review; move on.

## Reviewer Subagent

Dispatch the **baseline reviewer** using [reviewer-prompt.md](references/reviewer-prompt.md). It enforces the six baseline blocker categories:

1. **AC coverage gap.** An AC in the source Feature appears neither in a task's `**Verifies:**` nor in `## Deferred AC Coverage`.
2. **Stale AC reference.** A task references an AC ID that does not exist in the current source Feature.
3. **Order violates dependency.** A task is ordered before another task whose `**Verifies:**` ACs it depends on. Finding MUST cite the specific REQ slug or prose passage that establishes the dependency.
4. **Task is an AC wrapper.** A task restates a single AC's `Then` clause with no implementation work beyond it.
5. **Hidden multi-Feature scope.** A task description references work in a Feature other than the declared `**Source Feature:**`.
6. **Defer-reason vague.** A `## Deferred AC Coverage` entry has a reason that does not specify why or when (e.g., "later", "TBD", "out of scope" with no follow-up reference).

Findings outside these six categories MAY be returned as `Advisory` severity.

**Additional registered reviewers (third-party).** Read `specscore.yaml` at the repo root. If it contains a top-level `reviewers:` extension key (per the [`third-party-integration`](../../spec/features/third-party-integration/README.md) Feature), dispatch each registered reviewer in addition to the baseline.

For each entry: read the `prompt` file (which MUST be a path inside the repo working tree), confirm it contains an explicit blocker/advisory taxonomy section, and dispatch the reviewer using the same Agent-tool pattern as the baseline. A reviewer prompt missing a documented taxonomy fails the reviewer registration AC — surface as a registry error; do not silently skip.

**Composition is AND.** The User Review Gate releases only when **every** registered reviewer plus the baseline returns `Approved`. Any single `Issues Found` from any reviewer blocks the gate. On `Issues Found`:

1. Address every blocker-severity finding from every failing reviewer.
2. Re-dispatch every reviewer that previously returned `Issues Found`.
3. Re-dispatch reviewers that previously returned `Approved` when the fix changes the Plan's structural sections (Tasks, Deferred AC Coverage) — MUST for structural changes, SHOULD for task-list edits covered by previously-Approved reviewers.

Advisory findings MAY be ignored.

The skill MUST NOT silently downgrade a blocker to advisory, MUST NOT skip a registered reviewer, and MUST NOT proceed past the User Review Gate while any reviewer's last verdict is `Issues Found`.

When `specscore.yaml` does not contain a `reviewers:` key, only the baseline runs — the absence of the key is not an error.

## User Review Gate

After all reviewers return `Approved`:

> "Plan written and lint-clean at `spec/plans/<slug>.md`. Reviewer subagent approved. [N tasks covering M ACs; K ACs deferred — list them here if any.] Please review and let me know if you want changes before we move to `specstudio:implement`."

Wait. If the user requests changes, fix, re-lint, re-review, re-gate. Only proceed once the user approves.

**Approval phrases** (English + semantic equivalents in any language the user is communicating in): `approve`, `approved`, `accept`, `accepted`, `lgtm`. On detection of any qualifying phrase, transition status directly.

**Vague positive signals** (`looks good`, `yeah`, `ship it`, `+1`, `🚀`, `yes`, `ok`): treat as soft signal and ask one explicit confirmation question. Do not silently transition status.

On confirmed approval:

- Update `**Status:** Under Review → Approved`.
- Re-run lint.
- Emit `plan.approved` event.

## Event Emission

The skill participates in the Synchestra event vocabulary with **three events**:

| Event | When |
|---|---|
| `plan.drafted` | After reviewer subagent returns `Approved` and lint passes. First emission carries `previous_revision: null`, `changed_sections: null`, `change_summary: null`. Subsequent emissions during reviewer iteration carry the previous revision. |
| `plan.approved` | Exactly once, after user approval and the Draft → Approved transition completes. |
| `plan.updated` | On every successful lint pass after a subsequent edit while `status ∈ {Approved, Implementing}`. |

All three carry `changed_sections`, `previous_revision`, and a factual `change_summary` (≤2 sentences, no speculation/editorializing) — null on the very first `plan.drafted`, non-null thereafter.

The skill MUST NOT emit `plan.drafted` for an already-approved Plan, and MUST NOT re-emit `plan.approved` for further iteration.

## Auto-Stage in Git

Every file the skill creates — bootstrapped `spec/plans/`, `spec/plans/README.md`, the Plan file — MUST be `git add`-ed to the index. The skill MUST NOT commit on the user's behalf. Report staged paths in the same response as the artifact write. If staging fails (no git repository, lock contention), surface the failure and continue without aborting the artifact write.

## Transition to Build

After `plan.approved`, transition only to `specstudio:implement`. Do **not** invoke any other downstream skill — no `frontend-design`, no `mcp-builder`, no implementation skill of any kind.

While `specstudio:implement` is unshipped, hand back to the user with:

> "Plan approved. The `specstudio:implement` skill is not yet shipped — implement task-by-task using a general-purpose implementation skill. Each commit should reference the AC IDs from the satisfied task's `**Verifies:**` line."

## Revise vs Supersede

When the user wants to change an existing Plan:

- **Default: revise in place.** Git history is the record of evolution.
- **Supersede only when** the change invalidates the existing task list wholesale (e.g., the source Feature's ACs changed enough that the old tasks are stale). Set `**Supersedes:** <predecessor-slug>` in the new Plan's body metadata.

The skill MUST NOT silently delete or rewrite an existing Plan without the user's explicit choice between revise-in-place and supersede.

## Tone

The skill MUST NOT yes-machine weak Plans. When a task is vague, an AC is uncovered without justification, the task order violates a clear dependency, or the user asks to span multiple Features, say so with specificity and propose the alternative. Honest disagreement, not performative agreement.

## Verification

- [ ] `spec/plans/<slug>.md` exists
- [ ] `**Source Feature:**` resolves to an Approved/Implementing/Stable Feature
- [ ] Every task has a `**Verifies:**` line with ≥1 valid AC ID
- [ ] Every source-Feature AC is covered (in a task) or deferred (with a concrete reason)
- [ ] Tasks numbered 1..N with no gaps
- [ ] `specscore spec lint` passes
- [ ] Baseline reviewer subagent returned `Approved`
- [ ] All registered third-party reviewers returned `Approved`
- [ ] User explicitly approved the Plan
- [ ] `**Status:** Approved` in body metadata
- [ ] All created files staged via `git add` (never committed by the skill)
- [ ] `plan.drafted` + `plan.approved` events emitted

## Red Flags

- Proceeding to `specstudio:implement` without user approval
- Tasks without `**Verifies:**` lines
- ACs neither covered nor deferred
- Defer-reasons like "later" or "TBD"
- DAG-shaped task graphs (linear-only in MVP)
- Multi-Feature scope in a single Plan
- Planning against a Draft or Under Review Feature
- Writing to `docs/plans/` instead of `spec/plans/`
- Skipping the reviewer subagent or a registered third-party reviewer
- Invoking any skill other than `specstudio:implement` on transition

## References

- [reviewer-prompt.md](references/reviewer-prompt.md) — plan-document reviewer subagent template.
- [philosophy.md](../shared/philosophy.md) — shared tenets.
- [path-conventions.md](../shared/path-conventions.md) — `spec/` vs `docs/` rules.
- [specscore-lint-rules.md](../shared/specscore-lint-rules.md) — lint contract this skill assumes.
- [events.md](../shared/events.md) — event payloads emitted by this skill.
- [question-cadence.md](../shared/question-cadence.md) — when to batch vs single-question.
- [sidekick-capture.md](../shared/sidekick-capture.md) — sidekick-idea handling during the skill's flow.
- [Plan Skill Feature](../../spec/features/skills/plan/README.md) — the SpecScore Feature this skill implements.
