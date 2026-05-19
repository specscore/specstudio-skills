# Feature: Plan Skill

> [View in SpecStudio](https://specstudio.synchestra.io/project/features?id=specstudio-skills@synchestra-io@github.com&path=spec%2Ffeatures%2Fskills%2Fplan) — graph, discussions, approvals

**Status:** Approved
**Source Ideas:** specstudio-plan-skill
**Supersedes:** —

## Summary

The `specstudio:plan` skill turns an approved SpecScore Feature into a lint-clean Plan artifact at `spec/plans/<slug>.md` — an ordered list of tasks, each mapped to one or more AC IDs from its source Feature. Its output is the gating input for `specstudio:build`. Like `ideate` and `specify`, the skill hard-gates on lint-clean output, a reviewer-subagent verdict, and explicit user approval. Implementation will live at `skills/plan/` once approved.

**Provenance.** This skill mirrors the shape of `specstudio:specify`: same `<HARD-GATE>` pattern, same body-metadata schema, same lint → reviewer → user gate sequence, same CLI-preferred / fallback-direct-write contract, same auto-stage-on-create discipline. What is *new* in `plan` is the AC-coverage contract — every acceptance criterion in the source Feature must be covered by at least one task, or explicitly deferred with a reason.

## Problem

`specstudio:specify` produces lint-clean Features with `Given / When / Then` acceptance criteria. The next step — decomposing those ACs into ordered, executable tasks — currently has no SpecStudio skill. Users fall back to generic planning skills (`superpowers:writing-plans`, `agent-skills:planning-and-task-breakdown`), which are SpecScore-blind: they do not consume Feature body metadata, do not map tasks to AC IDs, do not lint, and do not emit Synchestra events.

The result is that spec↔code coherence — the central SpecStudio principle — breaks at exactly the handoff where it matters most: the moment ACs become work. A `plan` skill that consumes a Feature, produces tasks bound to its AC IDs, and gates on full coverage closes that gap.

## Behavior

### Invocation and skip conditions

The skill is invoked when a Feature is approved and the user is ready to decompose it into work.

#### REQ: invocation-triggers

The skill MUST respond to the triggers `plan`, `/plan`, `specstudio:plan`, "plan this feature", and the Synchestra event `feature.approved`. It MAY respond to additional natural-language phrasings of the same intent.

#### REQ: requires-approved-feature

The skill MUST refuse to proceed unless its input is a Feature at `spec/features/<slug>/` with `**Status:** Approved`. When given a `Draft` or `Under Review` Feature, the skill MUST tell the user that the Feature must be approved first (via `specstudio:specify`) and stop. The skill MUST NOT plan against unapproved specs.

### Hard gate

The skill enforces a hard gate downstream — once invoked, it must produce a complete, lint-clean, reviewer-approved, user-approved Plan before any build skill can run.

#### REQ: hard-gate

The skill MUST NOT invoke `specstudio:build`, `writing-plans`, or ANY implementation skill until ALL FIVE conditions hold:

1. The Plan artifact exists at `spec/plans/<slug>.md` and contains at least one `### Task N: <name>` task inside the `## Tasks` section.
2. Every acceptance criterion in the source Feature is either covered by at least one task (via the task's `**Verifies:**` line) or explicitly deferred under `## Deferred AC Coverage` with a reason.
3. `specscore lint spec/plans/<slug>.md` exits zero.
4. The plan-document reviewer subagent returned `Approved`.
5. The user has explicitly approved the written Plan.

A skill invocation that bypasses any condition is a contract violation.

### Artifact location and structure

Plan artifacts have one canonical location and a fixed single-file shape — mirroring Ideas, not Features.

#### REQ: artifact-path

The skill MUST write Plan artifacts to `spec/plans/<slug>.md` relative to the project root. Plans are single files, not directories — unlike Features, a Plan has no `_tests/` or `assets/` siblings in the MVP.

#### REQ: plan-slug-derivation

The Plan `<slug>` MUST default to the source Feature's slug 1:1. When the user explicitly asks for an alternative breakdown of the same Feature (e.g., a spike plan and a stable plan), the skill MUST use `<feature-slug>-<variant>` where `<variant>` is a single lowercase, hyphen-separated, URL-safe segment. Chained-variant suffixes (`<feature-slug>-<variant>-<variant>`) are not specified by the MVP — the skill MUST refuse them and ask the user to either pick a single variant or supersede the prior plan. Both forms MUST resolve to the same `**Source Feature:**` body-metadata value. The skill MUST refuse to write a plan whose slug collides with an existing plan unless the user explicitly chooses to overwrite or supersede.

#### REQ: no-docs-path

The skill MUST NOT write Plan artifacts to `docs/plans/`, `docs/superpowers/plans/`, `notes/`, or any path other than `spec/plans/`. `docs/` is reserved for prose documentation; SpecScore artifacts live under `spec/`.

#### REQ: auto-create-plans-dir

When invoked in a project that does not yet have a `spec/plans/` directory, the skill MUST create the directory and a lint-clean `spec/plans/README.md` index file before writing the first Plan. The auto-created index MUST be lint-clean per the canonical SpecScore Plans Index spec (title `# Plans`, `**Status:** Stable`, empty Contents table, "None at this time." Outstanding Questions, adherence footer). Auto-creation MUST NOT happen silently — the skill MUST tell the user it is bootstrapping the directory.

### Schema and body metadata

The Plan artifact conforms to a canonical schema: title-prefix dispatch key, bold-prefixed body metadata, fixed section schema, no YAML front-matter.

#### REQ: plan-schema-conformance

The Plan `.md` MUST follow the SpecScore Plan template: `# Plan: <Title>` heading, body metadata (`**Status:**`, `**Source Feature:**`, `**Date:**`, `**Owner:**`, `**Supersedes:**`) immediately after, `## Summary` (1–3 sentences), `## Approach` (≤1 paragraph on decomposition strategy), `## Tasks` (with `### Task N: <name>` blocks), optional `## Deferred AC Coverage`, `## Outstanding Questions`, and the adherence footer.

#### REQ: plan-status-domain

The `**Status:**` body-metadata value MUST be one of: `Draft`, `Under Review`, `Approved`, `Implementing`, `Completed`. The skill MUST set `**Status:** Draft` on initial write and manage transitions through `Under Review` and `Approved`. The skill does NOT manage transitions to `Implementing` or `Completed` — those are owned by `specstudio:build` (or user-driven).

#### REQ: source-feature-field

The Plan `.md` MUST declare its source via a `**Source Feature:** <feature-slug>` body-metadata line. The referenced slug MUST resolve to an existing Feature at `spec/features/<feature-slug>/README.md` with `**Status:**` in `{Approved, Implementing, Stable}`. The skill MUST NOT permit a Plan to reference a Draft or Under-Review Feature.

#### REQ: task-format

Each task MUST be a `### Task N: <task-name>` sub-heading inside `## Tasks`, numbered linearly starting from 1. Each task block MUST contain:

1. A `**Verifies:** <feature-slug>#ac:<ac-slug>, <feature-slug>#ac:<ac-slug>, …` line listing one or more AC IDs from the source Feature.
2. A 1–3 sentence prose description of what the task does.

Tasks MAY include additional optional fields (e.g., `**Notes:**`) but the two fields above are required.

#### REQ: linear-order-only

In the MVP, tasks MUST be in strict linear order — numbered 1..N with no gaps, no parallel branches, no DAG-style dependencies. Partial-order or DAG-style planning is a future Idea and explicitly out of scope for the MVP. If the user requests DAG-style ordering, the skill MUST say so and offer either (a) linearize the DAG into a best-effort order, or (b) defer until a future plan-schema revision.

### Source Feature linkage and AC coverage

The plan-to-feature linkage is declared by the Plan, and the AC-coverage contract is the Plan's central guarantee.

#### REQ: ac-coverage-required

Every acceptance criterion in the source Feature MUST be covered by at least one task — that is, every `### AC: <ac-slug>` (or inline `Scenario:` AC) in the source Feature's `README.md` MUST appear in at least one task's `**Verifies:**` line. This is a hard lint rule (`P-001`), not advisory.

#### REQ: deferred-ac-mechanism

When the user explicitly wants to defer an AC (e.g., post-MVP scope), the AC MUST be listed under a `## Deferred AC Coverage` section with a one-sentence reason per AC:

```
## Deferred AC Coverage

- <feature-slug>#ac:<ac-slug> — <one-sentence reason>
```

Deferred ACs satisfy the `ac-coverage-required` rule. The skill MUST NOT silently omit an AC — every AC is either in a task's `**Verifies:**` line or in `## Deferred AC Coverage`.

#### REQ: ac-id-validity

Every AC ID referenced by a task's `**Verifies:**` line or by `## Deferred AC Coverage` MUST resolve to an actual AC in the source Feature. Stale or invented AC references are rejected by lint (`P-002`). When the source Feature changes its AC list after a Plan is written, the skill MUST flag the drift to the user on the next invocation and offer to update the Plan.

#### REQ: task-granularity

Each task SHOULD be sized to be implementable in one focused work session and SHOULD reference no more than ~3 AC IDs. The skill MUST NOT enforce these as hard lint rules — they are reviewer-advisory only. The reviewer subagent MUST flag (a) tasks that look like thin wrappers around a single AC's `Then` clause (suggesting the AC and the task are duplicates), and (b) tasks that bundle many ACs (suggesting the task should be split). The user MAY override either finding.

### Single-Feature scope

The MVP plans against one Feature at a time.

#### REQ: single-feature-per-plan

A Plan MUST reference exactly one source Feature via `**Source Feature:**`. Multi-Feature or roadmap-level planning is explicitly out of scope for the MVP and is a separate Idea. If the user asks to plan across two Features, the skill MUST say so and offer to write two Plans instead.

### CLI vs fallback path

Plan crystallization prefers a stable CLI contract when available.

#### REQ: cli-preferred

When a `specscore new plan` (or equivalent) CLI command is on PATH, the skill MUST use it to scaffold the Plan file, passing every field it already has via documented flags. The skill MUST NOT invent flags.

#### REQ: fallback-direct-write

When no scaffolding CLI is available, the skill MUST fall back to direct file writes using the authoritative SpecScore Plan schema. CLI and fallback paths MUST produce schema-equivalent artifacts. They MAY differ in cosmetic ways (whitespace, blank lines).

### Lint and self-review

Every Plan passes through machine validation and a deliberate self-review.

#### REQ: lint-pass

After writing or editing the Plan, the skill MUST run `specscore lint spec/plans/<slug>.md` and confirm a zero exit code before proceeding to the reviewer subagent.

#### REQ: lint-failure-recovery

On `specscore lint` failure, the skill MUST:

1. Run `specscore lint --fix spec/plans/<slug>.md` exactly once.
2. Re-run `specscore lint spec/plans/<slug>.md` to verify.
3. If passing, continue and tell the user what was auto-fixed.
4. If still failing, surface remaining violations to the user with rule IDs.

The skill MUST NOT loop `--fix` more than once. The skill MUST NOT carry its own knowledge of which lint rules are auto-fixable — that policy belongs to the `specscore` CLI.

#### REQ: inline-self-review

Before dispatching the reviewer subagent, the skill MUST scan the Plan for: (a) unresolved placeholders (`TBD`, `TODO`, `???`, `FIXME`), (b) tasks with empty `**Verifies:**` lines, (c) AC IDs that look misspelled relative to the source Feature, and (d) tasks whose names contradict their `**Verifies:**` ACs. Findings MUST be fixed inline.

### Reviewer subagent gate

A subagent provides a structured second opinion on the Plan before the user sees it.

#### REQ: reviewer-subagent-required

The skill MUST dispatch at least the built-in plan-document reviewer subagent (per the prompt at `skills/plan/references/reviewer-prompt.md`) after lint passes and before presenting the Plan to the user. Every dispatched reviewer's verdict MUST be `Approved` before the User Review Gate runs. On `Issues Found` from any reviewer, the skill MUST address every blocker-severity finding and re-dispatch every reviewer that previously returned `Issues Found`. Advisory recommendations MAY be ignored.

#### REQ: reviewer-baseline-blockers

The built-in reviewer MUST treat the following finding categories as **blocker-severity**. These are semantic checks that complement (do not duplicate) `specscore lint`'s syntactic checks:

1. **AC coverage gap.** An AC in the source Feature appears neither in a task's `**Verifies:**` nor in `## Deferred AC Coverage`.
2. **Stale AC reference.** A task references an AC ID that does not exist in the current source Feature.
3. **Order violates dependency.** A task is ordered before another task whose `**Verifies:**` ACs it depends on (the dependency is inferable from the source Feature's REQs or from the prose description). A finding in this category MUST cite the specific REQ slug or prose passage that establishes the dependency, so the user can verify the inference; uncited dependency claims are not actionable and MUST NOT be raised as blockers.
4. **Task is an AC wrapper.** A task's description and `**Verifies:**` line restate a single AC's `Then` clause with no implementation work beyond it.
5. **Hidden multi-Feature scope.** A task description references work in a Feature other than the declared `**Source Feature:**`.
6. **Defer-reason vague.** A `## Deferred AC Coverage` entry has a reason that does not specify why or when the AC will be addressed (e.g., "later", "TBD", "out of scope" with no follow-up reference).

Findings outside these six categories MAY be returned as `Advisory` severity. Future expansions happen via Proposal once the Feature is `Stable`.

#### REQ: reviewer-extension-hook

The skill MUST support running additional reviewer subagents beyond the built-in one, registered via the same `reviewers:` extension key in `specscore.yaml` that the [`third-party-integration`](../../third-party-integration/README.md) Feature defines. Composition is **AND**: every reviewer MUST return `Approved` for the User Review Gate to release. Any single `Issues Found` from any reviewer blocks the gate. The skill MUST NOT silently downgrade a blocker from any reviewer to advisory, and MUST NOT skip a registered reviewer.

### User review and approval

The user — not the skill, not the subagent — owns the final approval decision.

#### REQ: user-approval-required

After the reviewer subagent returns `Approved`, the skill MUST present the Plan to the user with an explicit request for approval. The skill MUST NOT proceed to status transition or `plan.approved` event emission without that approval.

#### REQ: approval-explicit-phrase

The skill MUST recognize the same explicit-approval phrase set as `specstudio:ideate` and `specstudio:specify`: English `approve`, `approved`, `accept`, `accepted`, `lgtm`, plus their direct semantic equivalents in any language the user is communicating in. On detection of any qualifying phrase as a standalone or dominant response, the skill MUST proceed directly to the status transition.

#### REQ: approval-vague-confirmation

When the user's response signals positive sentiment but does not contain a recognized explicit phrase (e.g., `looks good`, `yeah`, `ship it`, `+1`, `🚀`, `yes`, `ok`), the skill MUST treat this as a soft signal and ask one explicit confirmation question before proceeding. The skill MUST NOT silently transition status on a vague signal.

#### REQ: status-transition-under-review

When the skill dispatches the reviewer subagent (and/or first presents the Plan for human review), the skill MUST update the Plan's `**Status:**` body-metadata line from `Draft` to `Under Review`. Subsequent edits during reviewer/user iteration keep `**Status:** Under Review` until either reviewer-and-user-approved (→ `Approved`) or the user explicitly drops back to `Draft` for substantial rework.

#### REQ: status-transition-on-approval

On confirmed user approval (after reviewer subagent returned `Approved` AND the user explicitly approved), the skill MUST update the Plan's `**Status:**` body-metadata line from `Under Review` to `Approved`, re-run lint, and emit `plan.approved`. The transition Approved → Implementing is owned by `specstudio:build`, not by `plan`.

### Auto-stage in git

Files the skill creates are staged for the user but never committed.

#### REQ: auto-stage-on-create

When the skill creates files (the bootstrapped `spec/plans/`, `spec/plans/README.md`, the Plan file), it MUST `git add` those paths to the index and report the staged paths to the user. The skill MUST NOT commit on the user's behalf. If staging fails (no git repository, lock contention), the skill MUST surface the failure and continue without aborting the artifact write.

### Event emission

The skill participates in the Synchestra event vocabulary.

#### REQ: event-drafted

The skill MUST emit `plan.drafted` after the reviewer subagent returns `Approved` and lint passes — that is, when the Plan is structurally and qualitatively ready for user review. The first emission for a Plan carries `previous_revision: null`; subsequent emissions during reviewer iteration carry the previous revision.

#### REQ: event-approved

The skill MUST emit `plan.approved` exactly once, after the user approves the Plan and the status transition Draft → Approved completes successfully.

#### REQ: event-updated

After `plan.approved` has fired, on every successful lint pass after a subsequent edit while `status ∈ {Approved, Implementing}`, the skill MUST emit `plan.updated`. The skill MUST NOT emit `plan.drafted` for an already-approved Plan, and MUST NOT re-emit `plan.approved` for further iteration.

#### REQ: event-payload-change-context

`plan.drafted`, `plan.approved`, and `plan.updated` event payloads MUST carry the same change-context fields as Idea and Feature events: `changed_sections` (H2 section names whose content differs from `previous_revision`, with H3 changes rolled up), `previous_revision` (git SHA), and `change_summary` (≤2 sentences, factual, no speculation/editorializing). On the very first `plan.drafted` emission for a Plan, all three are `null`.

### Promotion boundary

The next skill is `specstudio:build`, and only `specstudio:build`.

#### REQ: transition-to-build

After `plan.approved`, the skill MUST transition only to `specstudio:build` (or, while `build` is not yet shipped, hand back to the user with the recommendation to use a general-purpose implementation skill manually). The skill MUST NOT invoke `specify`, `ideate`, `frontend-design`, `mcp-builder`, or any other skill.

#### REQ: revise-vs-supersede

When the user wants to change an existing Plan, the default is to revise the Plan in place (git history is the record of evolution). The skill MUST create a successor with `**Supersedes:** <predecessor-slug>` only when the change invalidates the existing task list wholesale (e.g., the source Feature's ACs changed). The skill MUST NOT silently delete or rewrite an existing Plan without the user's explicit choice between revise-in-place and supersede.

### Tone

#### REQ: honest-pushback

The skill MUST NOT yes-machine weak Plans. When a task is too vague, an AC is uncovered without justification, the task order violates a clear dependency, or the user asks to span multiple Features, the skill MUST say so with specificity and propose the alternative. The acceptance bar is honest disagreement, not performative agreement.

## Interaction with Other Features

| Feature | Interaction |
|---|---|
| [Specify Skill](../specify/README.md) | `plan` is the downstream gate of `specify`. `plan` consumes the approved Feature via `**Source Feature:**` linkage and reads the Feature's AC list. The `feature.approved` event triggers `plan` (with user confirmation). |
| [Build Skill](../build/README.md) | `plan` is the upstream gate of `build`. `build` consumes the approved Plan one task at a time; `plan` never invokes `build` itself — `transition-to-build` is the explicit handoff. |
| [Third-Party Integration](../../third-party-integration/README.md) | Plan reviewers register via the same `reviewers:` extension key in `specscore.yaml` that `specify` uses. The contract (entry shape, prompt-location constraints, AND-composition) is shared. |
| SpecScore Plan schema | The schema, lint rules, and lifecycle of the produced Plan artifact are owned by SpecScore. `plan` is a producer, not a definer of that schema. The Plan-specific lint rules (`P-001`, `P-002`, …) are reserved by this Feature and defined in the SpecScore lint contract. |
| Synchestra Events | Emits `plan.drafted`, `plan.approved`, and `plan.updated`, all with change-context payloads. Consumers (including Hub and a future `build` skill) observe these to advance their own state. |
| `specscore` CLI | Preferred crystallization path when `specscore new plan` is available. The skill probes for the CLI once per invocation and falls back to direct write only when absent. |

## Acceptance Criteria

### AC: hard-gate-enforced (verifies REQ:hard-gate, REQ:lint-pass, REQ:reviewer-subagent-required, REQ:user-approval-required, REQ:requires-approved-feature)

**Given** an approved Feature at `spec/features/<slug>/` with two ACs (one covered by a task, one not covered and not deferred),
**When** the skill is invoked and asked to hand off to `build`,
**Then** the skill refuses the handoff, names the uncovered AC, and does not emit `plan.approved`.

### AC: refuse-unapproved-feature (verifies REQ:requires-approved-feature)

**Given** a Feature at `spec/features/<slug>/` with `**Status:** Draft`,
**When** the user invokes `specstudio:plan` against it,
**Then** the skill refuses to proceed, explains that the Feature must be approved via `specstudio:specify` first, and exits without writing any Plan file.

### AC: artifact-conformance (verifies REQ:artifact-path, REQ:plan-slug-derivation, REQ:no-docs-path, REQ:auto-create-plans-dir, REQ:auto-stage-on-create, REQ:plan-schema-conformance, REQ:task-format, REQ:linear-order-only)

**Given** an approved Feature with three ACs and no existing `spec/plans/` directory,
**When** the user invokes `specstudio:plan` and the skill completes a writing pass,
**Then** the skill creates `spec/plans/` with a lint-clean `README.md` index, writes `spec/plans/<feature-slug>.md` conforming to the Plan schema (title prefix, required body metadata, `## Tasks` with `### Task N:` blocks numbered 1..N with no gaps, each task with a `**Verifies:**` line), stages all created files via `git add`, and reports the staged paths to the user.

### AC: alternative-plan-slug (verifies REQ:plan-slug-derivation)

**Given** an existing approved Plan at `spec/plans/auth.md` for Feature `auth`,
**When** the user explicitly asks for an alternative breakdown ("a spike plan for auth"),
**Then** the skill writes the new plan at `spec/plans/auth-spike.md` with `**Source Feature:** auth`, leaves `spec/plans/auth.md` untouched, and surfaces both Plans on subsequent invocations.

### AC: source-feature-validity (verifies REQ:source-feature-field, REQ:ac-id-validity)

**Given** a Plan referencing `**Source Feature:** nonexistent-feature` or `**Verifies:** auth#ac:typo-slug` where the AC does not exist,
**When** the skill runs lint,
**Then** lint fails with rule `P-002` (or the source-feature-validity rule), and the skill surfaces the violation with the offending reference and the closest valid alternative.

### AC: ac-coverage-hard (verifies REQ:ac-coverage-required, REQ:deferred-ac-mechanism)

**Given** a source Feature with five ACs, and a draft Plan that covers four via task `**Verifies:**` lines and omits the fifth entirely,
**When** the skill runs lint or the reviewer subagent,
**Then** the gate fails with `P-001` (AC coverage gap), names the uncovered AC, and offers the user three resolutions: (a) add a task that covers it, (b) move it to `## Deferred AC Coverage` with a reason, or (c) revise the source Feature to remove the AC.

### AC: defer-with-reason (verifies REQ:deferred-ac-mechanism)

**Given** a source Feature with an AC the user explicitly wants to defer,
**When** the user marks it under `## Deferred AC Coverage` with a one-sentence reason,
**Then** lint passes, the reviewer subagent treats the AC as covered, and the deferred AC is reported in the Plan summary at user-review time so the user re-consents to the deferral.

### AC: defer-reason-blocker (verifies REQ:reviewer-baseline-blockers)

**Given** a Plan whose `## Deferred AC Coverage` entries list ACs with reasons like "later" or "TBD",
**When** the reviewer subagent runs,
**Then** the reviewer returns `Issues Found` with a blocker-severity finding for each vague defer-reason, and the User Review Gate does not release.

### AC: linear-order-only (verifies REQ:linear-order-only)

**Given** a user asking for a DAG-style plan with parallel branches,
**When** the skill processes the request,
**Then** the skill declines to write a DAG-shaped Plan, explains that linear-only is the MVP contract, and offers either a linearized best-effort ordering or to defer until a future plan-schema revision — and only proceeds with the user's choice.

### AC: single-feature-per-plan (verifies REQ:single-feature-per-plan)

**Given** a user request to plan two Features in one Plan,
**When** the skill processes the request,
**Then** the skill refuses, explains the single-Feature constraint, and offers to write two separate Plans instead.

### AC: cli-vs-fallback (verifies REQ:cli-preferred, REQ:fallback-direct-write)

**Given** two project environments — one with `specscore new plan` on PATH and one without,
**When** the skill writes a Plan in each,
**Then** both produced Plans are schema-equivalent (identical title prefix, identical body-metadata fields, identical sections, identical required content); cosmetic differences (whitespace, blank lines) are allowed.

### AC: lint-and-recovery (verifies REQ:lint-pass, REQ:lint-failure-recovery)

**Given** a freshly written Plan that fails lint on an auto-fixable rule (e.g., heading-spacing),
**When** the skill enters its lint-recovery path,
**Then** the skill runs `specscore lint --fix` exactly once, re-lints, and on success continues to the reviewer subagent while telling the user what was auto-fixed; on persistent failure the skill surfaces remaining violations with rule IDs and stops.

### AC: reviewer-then-user (verifies REQ:reviewer-subagent-required, REQ:reviewer-baseline-blockers, REQ:reviewer-extension-hook, REQ:user-approval-required)

**Given** a lint-clean Plan with an AC-wrapper task ("Task 3: verify the user feels confident" with `**Verifies:** ux#ac:confidence`),
**When** the reviewer subagent runs,
**Then** the reviewer returns `Issues Found` with blocker `Task is an AC wrapper`, the User Review Gate does not release, the skill addresses the finding (splits or rewrites the task), and only after the re-dispatched reviewer returns `Approved` does the user see the Plan.

### AC: approval-detection (verifies REQ:approval-explicit-phrase, REQ:approval-vague-confirmation)

**Given** a reviewer-approved Plan presented to the user,
**When** the user replies with an explicit phrase (`approve`, `lgtm`, `承認`, etc.),
**Then** the skill transitions status to `Approved`, re-runs lint, and emits `plan.approved`.
**And When** the user replies with a vague positive (`looks good`, `ship it`, `🚀`),
**Then** the skill asks one explicit confirmation question and does not transition status until a recognized explicit phrase is received.

### AC: lifecycle-events (verifies REQ:event-drafted, REQ:event-approved, REQ:event-updated, REQ:event-payload-change-context, REQ:status-transition-on-approval)

**Given** a Plan moving through the full lifecycle (Draft → Under Review → Approved → subsequent edits while Approved),
**When** each gate completes,
**Then** the skill emits `plan.drafted` after reviewer-subagent approval (with all three of `previous_revision`, `changed_sections`, and `change_summary` set to `null` on the very first emission, non-null thereafter), `plan.approved` exactly once on user approval, and `plan.updated` on each later successful lint pass — each payload carrying `changed_sections`, `previous_revision`, and a factual `change_summary`.

### AC: promotion-boundary-held (verifies REQ:transition-to-build, REQ:revise-vs-supersede)

**Given** an approved Plan,
**When** the skill completes its work,
**Then** the only transition the skill offers is to `specstudio:build` (or, while `build` is unshipped, a hand-back to the user with that recommendation); the skill MUST NOT invoke `specify`, `ideate`, `frontend-design`, or any other skill. When the user later wants to change the Plan, revise-in-place is the default; `**Supersedes:**` is reserved for changes that invalidate the existing task list wholesale.

### AC: honest-pushback (verifies REQ:honest-pushback)

**Given** a user asking the skill to write a Plan that papers over a clear problem (e.g., omitting an AC without deferral, listing tasks out of dependency order, spanning two Features),
**When** the skill processes the request,
**Then** the skill names the specific problem, proposes the alternative, and does not write the requested Plan until the user resolves the issue.

## Outstanding Questions

- **Plan-specific lint rule numbering.** This Feature reserves `P-001` (AC coverage gap) and `P-002` (stale AC reference). The full rule list, including auto-fixable flags and severity, should live in the SpecScore lint rule registry — to be drafted as a separate Feature on the SpecScore side before the `plan` skill ships.
- **Rehearse integration.** The MVP explicitly omits Rehearse stub scaffolding (deferred per the source Idea). Once Rehearse's markdown format stabilizes, a follow-on Feature should specify how a `**Verifies:**` AC ID links to its Rehearse scenario file under the source Feature's `_tests/` directory.
- **DAG / partial-order plans.** Linear-only is the MVP. The follow-on Idea for DAG planning should specify the schema change (`**Depends-On:**` per task), the lint implications (cycle detection, topological reachability), and the migration path for existing linear Plans.

---
*This document follows the https://specscore.md/feature-specification*
