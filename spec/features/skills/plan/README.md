# Feature: Plan Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/plan?op=explore) | [Edit](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/plan?op=edit) | [Ask question](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/plan?op=ask) | [Request change](https://specscore.studio/app/p/github.com/synchestra-io/specstudio-skills/spec/features/skills/plan?op=request-change) |

**Status:** Approved
**Source Ideas:** specstudio-plan-skill, specstudio-implement-skill
**Supersedes:** —

## Summary

The `specstudio:plan` skill turns an approved SpecScore Feature into a lint-clean Plan artifact at `spec/plans/<slug>.md` — an ordered list of tasks, each mapped to one or more AC IDs from its source Feature. Its output is the gating input for `specstudio:implement`. Like `ideate` and `specify`, the skill hard-gates on lint-clean output, a reviewer-subagent verdict, and explicit user approval. Implementation will live at `skills/plan/` once approved.

**Provenance.** This skill mirrors the shape of `specstudio:specify`: same `<HARD-GATE>` pattern, same body-metadata schema, same lint → reviewer → user gate sequence, same CLI-preferred / fallback-direct-write contract, same auto-stage-on-create discipline. What is *new* in `plan` is the AC-coverage contract — every acceptance criterion in the source Feature must be covered by at least one task, or explicitly deferred with a reason.

**Revision history.** This Feature was revised in-place to support `specstudio:implement` per the [`specstudio-implement-skill`](../../../ideas/specstudio-implement-skill.md) Idea. The revision is **additive in every direction** — existing approved Plans (without any of the new fields described below) continue to lint and run unchanged. The seven additions: (a) `**Depends-On:**` task field for dependency graphs, (b) `**Status:**` task field for per-task progress, (c) `**Mode:** <full|stub>` Plan body-metadata for plan posture, (d) placeholder-body permission for stub-mode tasks, (e) softened `REQ:linear-presentation-numbering` (linear numbering as presentation only; execution order follows the dependency graph), (f) lint rule `P-003` for cycle detection, (g) lint rule `P-004` preventing placeholder bodies in stub Plans for `done`-status tasks. Reviewer-baseline blocker categories extended to cover the new fields.

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

The skill enforces a hard gate downstream — once invoked, it must produce a complete, lint-clean, reviewer-approved, user-approved Plan before any implement skill can run.

#### REQ: hard-gate

The skill MUST NOT invoke `specstudio:implement`, `writing-plans`, or ANY implementation skill until ALL FIVE conditions hold:

1. The Plan artifact exists at `spec/plans/<slug>.md` and contains at least one `### Task N: <name>` task inside the `## Tasks` section.
2. Every acceptance criterion in the source Feature is either covered by at least one task (via the task's `**Verifies:**` line) or explicitly deferred under `## Deferred AC Coverage` with a reason.
3. `specscore spec lint` exits zero.
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

The Plan `.md` MUST follow the SpecScore Plan template: `# Plan: <Title>` heading, body metadata (`**Status:**`, `**Source Feature:**`, `**Date:**`, `**Owner:**`, `**Supersedes:**`, `**Mode:**`) immediately after, `## Summary` (1–3 sentences), `## Approach` (≤1 paragraph on decomposition strategy), `## Tasks` (with `### Task N: <name>` blocks), optional `## Deferred AC Coverage`, `## Outstanding Questions`, and the adherence footer.

The `**Mode:**` body-metadata line MAY be absent in pre-existing Plans authored before this revision; absence means `frozen` semantics (treated as `**Mode:** full`). New Plans SHOULD declare `**Mode:**` explicitly.

#### REQ: plan-status-domain

The `**Status:**` body-metadata value MUST be one of: `Draft`, `Under Review`, `Approved`, `Implementing`, `Completed`. The skill MUST set `**Status:** Draft` on initial write and manage transitions through `Under Review` and `Approved`. The skill does NOT manage transitions to `Implementing` or `Completed` — those are owned by `specstudio:implement` (or user-driven).

#### REQ: source-feature-field

The Plan `.md` MUST declare its source via a `**Source Feature:** <feature-slug>` body-metadata line. The referenced slug MUST resolve to an existing Feature at `spec/features/<feature-slug>/README.md` with `**Status:**` in `{Approved, Implementing, Stable}`. The skill MUST NOT permit a Plan to reference a Draft or Under-Review Feature.

#### REQ: task-format

Each task MUST be a `### Task N: <task-name>` sub-heading inside `## Tasks`, numbered linearly starting from 1. Each task block MUST contain:

1. A `**Verifies:** <feature-slug>#ac:<ac-slug>, <feature-slug>#ac:<ac-slug>, …` line listing one or more AC IDs from the source Feature.
2. A `**Status:** <pending|in-progress|done|blocked>` line. Default is `pending`. Absent on pre-existing Plans authored before this revision; absence is treated as `pending`. See `REQ:task-status-field` for write semantics.
3. A `**Depends-On:** —` (or `**Depends-On:** <task-number>, <task-number>, …`) line declaring predecessor tasks by their task numbers. Default `—` (no dependencies). Absent on pre-existing Plans; absence is treated as `—`. See `REQ:depends-on-field`.
4. A 1–3 sentence prose description of what the task does, **OR** — when the Plan's `**Mode:** stub` — the canonical placeholder marker `<!-- implement: pending -->` (HTML comment, invisible in rendered markdown, byte-exact parseable). See `REQ:posture-stub-placeholder`.

Tasks MAY include additional optional fields (e.g., `**Notes:**`) but the four above are required.

#### REQ: linear-presentation-numbering

Tasks MUST be numbered linearly 1..N with no gaps. **The numbering is presentation order only** — it determines the order in which tasks appear in the rendered Plan, but **not** the order in which `specstudio:implement` executes them. Execution order is determined by the dependency graph declared via `**Depends-On:**` (see `REQ:depends-on-field` and `REQ:dependency-driven-execution`).

When every task has `**Depends-On:** —` and every task's numbering is interpreted as an implicit dependency on its predecessor (i.e., task N depends on task N-1), the Plan degrades to the original strict-linear semantics — this is the default behavior of pre-existing Plans authored before this revision and is preserved for backward compatibility. New Plans SHOULD declare `**Depends-On:**` explicitly on every task.

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

### Task dependencies and execution order

The dependency graph encoded by `**Depends-On:**` is what enables parallel-batch execution by `specstudio:implement` while keeping the task-numbering stable as the human-readable presentation order.

#### REQ: depends-on-field

Each task in a Plan MUST declare a `**Depends-On:**` body field. The value is either `—` (no predecessors, eligible for execution as soon as the user approves the Plan) or a comma-separated list of predecessor task numbers (e.g., `**Depends-On:** 1, 3`). Predecessor numbers MUST resolve to actual task numbers in the same Plan; references to nonexistent task numbers are a lint violation. Self-references (a task depending on itself) are a lint violation.

#### REQ: dependency-driven-execution

`specstudio:implement` MUST compute the execution order from the dependency graph, not from the task numbering. A task is eligible for execution when all its declared predecessors (per `**Depends-On:**`) are in `**Status:** done`. Tasks with no unsatisfied predecessors may execute in parallel (subject to `specstudio:implement`'s own max-parallel cap; that cap is owned by the implement skill, not by `plan`).

#### REQ: cycle-detection-lint

The Plan dependency graph MUST be acyclic. Cycles (e.g., task 1 depends on task 2, task 2 depends on task 1) are detected by lint rule `P-003`. The lint MUST surface the full cycle path (e.g., "Task 1 → Task 3 → Task 2 → Task 1") so the user can fix it. `P-003` is a hard rule, not advisory.

### Plan posture: full vs stub

Each Plan declares a posture that controls whether task bodies are authored at plan time or journaled at implement time.

#### REQ: plan-mode-field

A Plan MAY declare its posture via a `**Mode:** <full|stub>` body-metadata line in the Plan header. Permitted values: `full`, `stub`. Default when absent: `full` (backward-compatible with pre-existing Plans). The skill MUST set `**Mode:**` explicitly on all new Plans.

#### REQ: posture-full-default

In `full` posture (the default), each task body MUST be a 1–3 sentence prose description of what the task does, authored at plan time and locked before the Plan is approved. Task bodies in `full` Plans are NOT written back by `specstudio:implement`; only `**Status:**` is updated as tasks progress.

#### REQ: posture-stub-placeholder

In `stub` posture, each task body MAY be the canonical placeholder marker `<!-- implement: pending -->` (HTML comment, invisible in rendered markdown, byte-exact parseable by `specscore` and `specstudio:implement`) instead of a prose description. `specstudio:implement` writes back the actual prose description (1–2 sentence post-hoc summary referencing the commit SHA) after the task lands. Stub-mode Plans approve with placeholder bodies; their bodies become canonical only post-implement.

#### REQ: posture-one-way

Once a Plan is declared `**Mode:** stub` or `**Mode:** full`, the posture is fixed for the Plan's lifetime. The skill MUST NOT support mid-flight posture re-classification (no `--switch-mode` flag, no automatic switching). A user who needs the other posture for an in-flight Plan MUST create a successor Plan with `**Supersedes:**` set.

#### REQ: posture-no-auto-classification

The skill MUST NOT auto-classify Plans as `full` or `stub` based on AC count, task count, LLM judgement, or any other heuristic. Posture is **user-declared explicitly** at `plan` time. Auditability beats convenience. (External skills such as consilium MAY *recommend* a posture; the user MUST confirm.)

### Per-task status tracking

Status is the at-a-glance progress signal in the rendered Plan; the authoritative progress source is the git log's `Verifies:` trailer (parsed and cross-checked on every `specstudio:implement` invocation).

#### REQ: task-status-field

Each task MUST have a `**Status:**` body-metadata field with one of: `pending`, `in-progress`, `done`, `blocked`. Default value: `pending`. The field MAY be absent on pre-existing Plans authored before this revision; absence is treated as `pending` and the skill SHOULD add the field on the next revise-in-place opportunity.

#### REQ: task-status-write-authority

`specstudio:implement` is the canonical writer of `**Status:**`. It writes `pending → in-progress` on subagent dispatch, `in-progress → done` on subagent DONE return, and `→ blocked` on subagent BLOCKED return. The user MAY manually edit `**Status:**` (e.g., to reset a task from `blocked` back to `pending` after fixing the blocker); `specstudio:implement` MUST detect manual edits on the next invocation and cross-check them against the git-log `Verifies:` trailer authority. The **divergence-resolution policy** (what `implement` does when the Plan's `**Status:**` and the git-log evidence disagree — surface as warning, refuse to proceed, auto-overwrite, etc.) is owned by the `implement` Feature, not this one; the reviewer-blocker category at item 10 of `REQ:reviewer-baseline-blockers` enforces that any such divergence is at least surfaced.

#### REQ: status-applies-to-both-postures

Per-task `**Status:**` writes apply to **both** `full` and `stub` Plans. The body-writeback exclusion in `posture-full-default` is specifically about *task bodies*, not about status. Status updates are not body edits.

#### REQ: stub-placeholder-done-lint

A task whose `**Status:** done` MUST NOT have a placeholder body — its body MUST be the prose summary written by `specstudio:implement` post-batch. This is enforced by lint rule `P-004`: a placeholder body on a `done`-status task in a `stub` Plan is a lint violation. The lint MUST cite the offending task number and reference both the placeholder rule and the writeback contract.

### CLI vs fallback path

Plan crystallization prefers a stable CLI contract when available.

#### REQ: cli-preferred

When a `specscore new plan` (or equivalent) CLI command is on PATH, the skill MUST use it to scaffold the Plan file, passing every field it already has via documented flags. The skill MUST NOT invent flags.

#### REQ: fallback-direct-write

When no scaffolding CLI is available, the skill MUST fall back to direct file writes using the authoritative SpecScore Plan schema. CLI and fallback paths MUST produce schema-equivalent artifacts. They MAY differ in cosmetic ways (whitespace, blank lines).

### Lint and self-review

Every Plan passes through machine validation and a deliberate self-review.

#### REQ: lint-pass

After writing or editing the Plan, the skill MUST run `specscore spec lint` and confirm a zero exit code before proceeding to the reviewer subagent.

#### REQ: lint-failure-recovery

On `specscore spec lint` failure, the skill MUST:

1. Run `specscore spec lint --fix` exactly once.
2. Re-run `specscore spec lint` to verify.
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

The built-in reviewer MUST treat the following finding categories as **blocker-severity**. These are semantic checks that complement (do not duplicate) `specscore spec lint`'s syntactic checks:

1. **AC coverage gap.** An AC in the source Feature appears neither in a task's `**Verifies:**` nor in `## Deferred AC Coverage`.
2. **Stale AC reference.** A task references an AC ID that does not exist in the current source Feature.
3. **Missing or wrong dependency declaration.** Two sub-cases: (a) a task's prose description implies a dependency on another task (e.g., "uses the output of task N") but its `**Depends-On:**` field doesn't declare it; (b) a task's `**Depends-On:**` declares a predecessor whose `**Verifies:**` ACs are unrelated to the dependent task's work (a spurious dependency that would serialize execution unnecessarily). A finding in this category MUST cite the specific REQ slug or prose passage that establishes (or contradicts) the dependency, so the user can verify the inference; uncited dependency claims are not actionable and MUST NOT be raised as blockers.
4. **Task is an AC wrapper.** A task's description and `**Verifies:**` line restate a single AC's `Then` clause with no implementation work beyond it.
5. **Hidden multi-Feature scope.** A task description references work in a Feature other than the declared `**Source Feature:**`.
6. **Defer-reason vague.** A `## Deferred AC Coverage` entry has a reason that does not specify why or when the AC will be addressed (e.g., "later", "TBD", "out of scope" with no follow-up reference).
7. **Cycle in the dependency graph.** Tasks form a cycle via their `**Depends-On:**` references. Lint rule `P-003` catches syntactic cycles; the reviewer additionally flags *semantic* cycles inferable from task descriptions (e.g., task A's description says "needs the output of task B" but `**Depends-On:**` doesn't declare it, and task B's description says the inverse). Finding MUST cite the inferred cycle path.
8. **Stale Depends-On task number.** A `**Depends-On:**` field references a task number that does not exist in the current Plan (e.g., the user renumbered tasks but left a stale dependency). Lint rule `P-003`'s validity check catches dangling references; the reviewer additionally flags this if a Plan-author note describes a dependency on a renamed/renumbered task.
9. **Posture-inconsistent placeholder body.** A task body is a placeholder marker in a `**Mode:** full` Plan, or a `done`-status task has a placeholder body in a `**Mode:** stub` Plan. Lint rule `P-004` covers the latter; the reviewer flags the former.
10. **Status inconsistent with git-log Verifies cross-check.** A task with `**Status:** done` whose `**Verifies:**` ACs do not appear in any commit's `Verifies:` trailer in the repo's git log — or, conversely, a task with `**Status:** pending` whose ACs DO appear in a `Verifies:` trailer. Finding MUST cite the specific commit (or commit-absence) and AC slug, so the user can reconcile manually or rerun `specstudio:implement` to refresh the Status field.

Findings outside these ten categories MAY be returned as `Advisory` severity. Future expansions happen via Proposal once the Feature is `Stable`.

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

On confirmed user approval (after reviewer subagent returned `Approved` AND the user explicitly approved), the skill MUST update the Plan's `**Status:**` body-metadata line from `Under Review` to `Approved`, re-run lint, and emit `plan.approved`. The transition Approved → Implementing is owned by `specstudio:implement`, not by `plan`.

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

The next skill is `specstudio:implement`, and only `specstudio:implement`.

#### REQ: transition-to-implement

After `plan.approved`, the skill MUST transition only to `specstudio:implement` (or, while `implement` is not yet shipped, hand back to the user with the recommendation to use a general-purpose implementation skill manually). The skill MUST NOT invoke `specify`, `ideate`, `frontend-design`, `mcp-builder`, or any other skill.

#### REQ: revise-vs-supersede

When the user wants to change an existing Plan, the default is to revise the Plan in place (git history is the record of evolution). The skill MUST create a successor with `**Supersedes:** <predecessor-slug>` only when the change invalidates the existing task list wholesale (e.g., the source Feature's ACs changed). The skill MUST NOT silently delete or rewrite an existing Plan without the user's explicit choice between revise-in-place and supersede.

### Tone

#### REQ: honest-pushback

The skill MUST NOT yes-machine weak Plans. When a task is too vague, an AC is uncovered without justification, the task order violates a clear dependency, or the user asks to span multiple Features, the skill MUST say so with specificity and propose the alternative. The acceptance bar is honest disagreement, not performative agreement.

## Interaction with Other Features

| Feature | Interaction |
|---|---|
| [Specify Skill](../specify/README.md) | `plan` is the downstream gate of `specify`. `plan` consumes the approved Feature via `**Source Feature:**` linkage and reads the Feature's AC list. The `feature.approved` event triggers `plan` (with user confirmation). |
| [Implement Skill](../implement/README.md) | `plan` is the upstream gate of `implement`. `implement` consumes the approved Plan one task at a time; `plan` never invokes `implement` itself — `transition-to-implement` is the explicit handoff. |
| [Third-Party Integration](../../third-party-integration/README.md) | Plan reviewers register via the same `reviewers:` extension key in `specscore.yaml` that `specify` uses. The contract (entry shape, prompt-location constraints, AND-composition) is shared. |
| SpecScore Plan schema | The schema, lint rules, and lifecycle of the produced Plan artifact are owned by SpecScore. `plan` is a producer, not a definer of that schema. The Plan-specific lint rules (`P-001`, `P-002`, …) are reserved by this Feature and defined in the SpecScore lint contract. |
| Synchestra Events | Emits `plan.drafted`, `plan.approved`, and `plan.updated`, all with change-context payloads. Consumers (including Hub and a future `implement` skill) observe these to advance their own state. |
| `specscore` CLI | Preferred crystallization path when `specscore new plan` is available. The skill probes for the CLI once per invocation and falls back to direct write only when absent. |

## Acceptance Criteria

### AC: hard-gate-enforced (verifies REQ:hard-gate, REQ:lint-pass, REQ:reviewer-subagent-required, REQ:user-approval-required, REQ:requires-approved-feature)

**Given** an approved Feature at `spec/features/<slug>/` with two ACs (one covered by a task, one not covered and not deferred),
**When** the skill is invoked and asked to hand off to `implement`,
**Then** the skill refuses the handoff, names the uncovered AC, and does not emit `plan.approved`.

### AC: refuse-unapproved-feature (verifies REQ:requires-approved-feature)

**Given** a Feature at `spec/features/<slug>/` with `**Status:** Draft`,
**When** the user invokes `specstudio:plan` against it,
**Then** the skill refuses to proceed, explains that the Feature must be approved via `specstudio:specify` first, and exits without writing any Plan file.

### AC: artifact-conformance (verifies REQ:artifact-path, REQ:plan-slug-derivation, REQ:no-docs-path, REQ:auto-create-plans-dir, REQ:auto-stage-on-create, REQ:plan-schema-conformance, REQ:task-format, REQ:linear-presentation-numbering)

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

### AC: linear-presentation-numbering (verifies REQ:linear-presentation-numbering)

**Given** a user asking for non-linear *task numbering* in the rendered Plan (e.g., gapped numbers like 1, 3, 5; sub-task notation like 1a/1b; side-by-side parallel-column layout),
**When** the skill processes the request,
**Then** the skill declines, explains that task numbering is linear-1..N for presentation only (with parallel *execution* expressed via `**Depends-On:**` annotations on linearly-numbered tasks), and either rewrites the request as a numbered list with explicit `**Depends-On:**` or stops for user clarification.

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
**Then** the skill runs `specscore spec lint --fix` exactly once, re-lints, and on success continues to the reviewer subagent while telling the user what was auto-fixed; on persistent failure the skill surfaces remaining violations with rule IDs and stops.

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

### AC: promotion-boundary-held (verifies REQ:transition-to-implement, REQ:revise-vs-supersede)

**Given** an approved Plan,
**When** the skill completes its work,
**Then** the only transition the skill offers is to `specstudio:implement` (or, while `implement` is unshipped, a hand-back to the user with that recommendation); the skill MUST NOT invoke `specify`, `ideate`, `frontend-design`, or any other skill. When the user later wants to change the Plan, revise-in-place is the default; `**Supersedes:**` is reserved for changes that invalidate the existing task list wholesale.

### AC: honest-pushback (verifies REQ:honest-pushback)

**Given** a user asking the skill to write a Plan that papers over a clear problem (e.g., omitting an AC without deferral, listing tasks out of dependency order, spanning two Features),
**When** the skill processes the request,
**Then** the skill names the specific problem, proposes the alternative, and does not write the requested Plan until the user resolves the issue.

### AC: depends-on-graph (verifies REQ:depends-on-field, REQ:dependency-driven-execution, REQ:linear-presentation-numbering)

**Given** a Plan with five tasks numbered 1..5 where `**Depends-On:**` declares task 3 → task 1 and tasks 4, 5 → task 2 (tasks 1 and 2 have `**Depends-On:** —`),
**When** `specstudio:implement` computes the execution order,
**Then** tasks 1 and 2 are eligible in batch 1 (no predecessors), task 3 is eligible in batch 2 (after task 1 done), tasks 4 and 5 are eligible in batch 2 (after task 2 done). The task numbering (1..5) determines presentation in the rendered Plan only — not execution order.

### AC: cycle-detected (verifies REQ:cycle-detection-lint, REQ:reviewer-baseline-blockers)

**Given** a Plan where task 1 declares `**Depends-On:** 3` and task 3 declares `**Depends-On:** 1`,
**When** the skill runs lint,
**Then** lint rule `P-003` fails with a finding that cites the full cycle path ("Task 1 → Task 3 → Task 1") and the user is shown a non-empty exit with no Plan write/approval permitted.

### AC: stale-depends-on (verifies REQ:depends-on-field, REQ:reviewer-baseline-blockers)

**Given** a Plan with four tasks numbered 1..4 where task 3 declares `**Depends-On:** 7` (task 7 does not exist),
**When** the skill runs lint,
**Then** lint fails citing the stale reference, names task 3 and the dangling number 7, and suggests the closest valid alternative (highest task number in the Plan).

### AC: posture-default-and-explicit (verifies REQ:plan-mode-field, REQ:posture-full-default)

**Given** two new Plans — one declaring `**Mode:** full` explicitly, one omitting `**Mode:**` entirely,
**When** the skill writes both,
**Then** both Plans behave identically (treated as `full` posture). The skill MUST set `**Mode:**` explicitly on new Plans but MUST accept Plans without the field as `full` for backward compatibility with pre-existing approved Plans.

### AC: stub-posture-placeholder-body (verifies REQ:posture-stub-placeholder, REQ:task-format)

**Given** a Plan with `**Mode:** stub` and three tasks whose bodies are the canonical placeholder marker `<!-- implement: pending -->`,
**When** the skill runs lint,
**Then** lint passes — placeholder bodies are explicitly permitted in stub-mode Plans for tasks whose `**Status:**` is not `done`.

### AC: stub-placeholder-blocked-when-done (verifies REQ:stub-placeholder-done-lint, REQ:reviewer-baseline-blockers)

**Given** a Plan with `**Mode:** stub`, three tasks, and task 2 has `**Status:** done` but its body is still a placeholder marker,
**When** the skill runs lint,
**Then** lint rule `P-004` fails, cites task 2 with both the placeholder rule and the writeback contract reference, and the user is told to either re-run `specstudio:implement` to write back the prose summary or revert the Status to `pending`.

### AC: posture-no-mid-flight-switch (verifies REQ:posture-one-way, REQ:revise-vs-supersede)

**Given** an approved Plan with `**Mode:** stub`,
**When** the user asks to switch the Plan to `**Mode:** full` mid-implementation,
**Then** the skill refuses the in-place switch, explains the one-way constraint, and offers two paths: (a) create a successor Plan with `**Supersedes:** <slug>` and `**Mode:** full`, or (b) keep the current Plan and proceed as-is.

### AC: posture-explicit-declaration (verifies REQ:posture-no-auto-classification)

**Given** a Plan with eight ACs (the kind of size that might tempt auto-classification as `full`) or one with one AC (the kind that might be auto-classified as `stub`),
**When** the user invokes `plan` without declaring `**Mode:**`,
**Then** the skill MUST NOT pick a posture silently — it MUST either default to `full` (current backward-compatible behavior) or explicitly ask the user to declare. The skill MUST NOT use AC count, task count, LLM judgement, or any other heuristic to auto-select posture.

### AC: task-status-default-and-writes (verifies REQ:task-status-field, REQ:task-status-write-authority, REQ:status-applies-to-both-postures)

**Given** a new Plan with four tasks, all with default `**Status:** pending`,
**When** `specstudio:implement` dispatches task 1's subagent,
**Then** task 1's `**Status:**` transitions `pending → in-progress`; when the subagent returns DONE, `**Status:**` transitions to `done`. The same transitions apply identically in both `**Mode:** full` and `**Mode:** stub` Plans.

### AC: status-cross-check (verifies REQ:task-status-write-authority, REQ:reviewer-baseline-blockers)

**Given** a Plan with task 2 marked `**Status:** done` whose `**Verifies:**` ACs do not appear in any commit's `Verifies:` trailer in the git log,
**When** the reviewer subagent runs,
**Then** the reviewer returns `Issues Found` with a blocker citing the inconsistency between Plan-stated Status and git-log authority, names the specific AC slug, and tells the user to either rerun `specstudio:implement` to refresh the Status or investigate why the commit is missing.

## Open Questions

- **Plan-specific lint rule registry.** This Feature reserves `P-001` (AC coverage gap), `P-002` (stale AC reference), `P-003` (Depends-On cycle / dangling reference / self-reference / non-linear numbering), and `P-004` (placeholder body on done-status task in stub Plan + invalid `**Mode:**`/`**Status:**` token values). **Cross-repo dependency resolved:** all four rules and the parser extensions (`**Mode:**`, `**Status:**`, `**Depends-On:**`, the canonical placeholder body token) are shipped on [`specscore-cli@main`](https://github.com/specscore/specscore-cli) under Feature `cli/spec/lint/plan-rules` (unreleased at commit 76b6b29 as of 2026-05-19; early adopters install from source via `go install`). Tracked-and-completed in sidekick seed [`specscore-cli-companion-implement-plan-feature-lint-rules`](../../../ideas/seeds/specscore-cli-companion-implement-plan-feature-lint-rules.md).
- **Rehearse integration.** The MVP explicitly omits Rehearse stub scaffolding (deferred per the source Idea). Once Rehearse's markdown format stabilizes, a follow-on Feature should specify how a `**Verifies:**` AC ID links to its Rehearse scenario file under the source Feature's `_tests/` directory.
- ~~**Placeholder body token for stub Plans.**~~ **Resolved 2026-05-19.** The canonical placeholder body token is **`<!-- implement: pending -->`** — HTML comment, invisible in rendered markdown, byte-exact parseable. Chosen over the visible alternatives (`**Implementation:** _pending_`, `_to be journaled by `implement`_`) for zero visual noise in the rendered Plan (stub Plans are primarily seen *through* `implement`, not directly read by humans). Encoded in the `specscore-cli@main` parser and lint rules.
- **CLI flag for posture declaration.** Should `specstudio:plan` accept a `--mode=full|stub` flag in addition to body-metadata declaration? Recommendation: yes (CLI flag writes the metadata line); if both differ on revise-in-place, the file wins. Confirm at implementation time.
- **`plan.updated` payload in stub mode.** When `specstudio:implement` writes back task bodies in stub Plans, the `plan.updated` event MUST list affected task slugs in `changed_sections`. Discipline confirmed at the Idea level; the exact payload shape lands during implement-skill spec work.
- **Status-vs-git-log reconciliation on first invocation of `specstudio:implement` against a pre-existing Plan.** If a Plan was approved before this revision (no Status fields), and the user runs `specstudio:implement` against it, how does the skill initialize Status fields without misreading the git log? Recommendation: scan git log for `Verifies:` trailers referencing any of the Plan's ACs and set Status to `done` for matched tasks; leave the rest `pending`. Spec-time work for the implement Feature.

## Sidekick Seeds Generated

- [specscore-cli-companion-implement-plan-feature-lint-rules](../../../ideas/seeds/specscore-cli-companion-implement-plan-feature-lint-rules.md) — captured 2026-05-19 by specstudio:specify

---
*This document follows the https://specscore.md/feature-specification*
