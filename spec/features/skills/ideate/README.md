# Feature: Ideate Skill

> [View in SpecScore Studio](https://specscore.studio/project/features?id=specstudio-skills@specscore@github.com&path=spec%2Ffeatures%2Fskills%2Fideate) — graph, discussions, approvals

**Status:** Approved

## Summary

The `specstudio:ideate` skill refines raw, vague concepts into lint-clean SpecScore Idea artifacts through a three-phase dialogue (Understand & Expand, Evaluate & Converge, Crystallize). Its output (`spec/ideas/<slug>.md`) is the gating input for `specstudio:specify` and the only sanctioned way to produce a SpecScore Idea inside SpecStudio. Implementation lives at [`skills/ideate/`](../../../../skills/ideate/).

**Provenance.** This skill is adapted from `addyosmani/agent-skills`'s `idea-refine` skill — its three-phase divergent/convergent structure, lens library (inversion, constraint removal, audience shift, combination, simplification, 10x, expert lens), refinement criteria (painkiller vs. vitamin, differentiation tiers, Must/Should/Might assumption audit), and Jobs-flavored tone are inherited and rebuilt against canonical SpecScore conventions (single-file `spec/ideas/<slug>.md`, body-metadata header fields, lint-enforced Not-Doing list and assumption tiers). Scope-decomposition discipline and the mandatory-artifact policy ("unsaved ideation is waste") are grafted from `obra/superpowers`'s `brainstorming`. The full mapping of inherited vs. grafted vs. dropped patterns lives in [`spec/research/ideate-vs-brainstorming-skills-analysis.md`](../../../research/ideate-vs-brainstorming-skills-analysis.md).

## Problem

Vague intent silently corrupts every downstream artifact. When a developer tells an AI agent "build me X" without first articulating the problem, alternatives, and explicit non-goals, the agent fills in the gaps stochastically — producing features that solve the wrong problem, miss the obvious "Not Doing" boundary, or inherit unstated assumptions as if they were specifications.

`ideate` exists to pay the ideation cost up front and capture it in a typed artifact, instead of paying it implicitly across every later phase of the lifecycle. The Feature defines exactly what that capture must look like, so the artifact is machine-verifiable and downstream skills can rely on it.

## Behavior

### Invocation and skip conditions

The skill is invoked when a concept is vague, multiple directions are open, or the user wants to stress-test an idea before specifying it.

#### REQ: invocation-triggers

The skill MUST respond to the triggers `ideate`, `/ideate`, "refine this idea", and "stress-test this". It MAY respond to additional natural-language phrasings of the same intent.

#### REQ: skip-when-clear-intent

The skill MUST NOT proceed when the user already has a clear, high-conviction feature to specify. In that case it MUST hand off to `specstudio:specify` and explain why. The decision criterion is whether the user can articulate the problem, the recommended direction, and an explicit Not-Doing boundary in one short statement.

### Hard gate

The skill is gated downstream — once invoked, it must produce a complete, lint-clean, user-approved Idea before any other SpecStudio skill can run.

#### REQ: hard-gate

The skill MUST NOT invoke `specstudio:specify`, `writing-plans`, or any implementation skill until ALL THREE conditions hold:

1. An Idea artifact exists at `spec/ideas/<slug>.md`.
2. `specscore spec lint` exits zero.
3. The user has explicitly approved the Recommended Direction.

A skill invocation that bypasses any condition is a contract violation.

### Artifact location

Idea artifacts have one canonical location.

#### REQ: artifact-path

The skill MUST write Idea artifacts to `spec/ideas/<slug>.md` relative to the project root. The `<slug>` MUST be lowercase, hyphen-separated, URL-safe, and stable for the artifact's lifetime.

#### REQ: no-docs-path

The skill MUST NOT write Idea artifacts to `docs/ideas/`, `notes/`, or any path other than `spec/ideas/`. `docs/` is reserved for prose documentation; SpecScore artifacts live under `spec/`.

#### REQ: auto-create-ideas-dir

When invoked in a project that does not yet have a `spec/ideas/` directory, the skill MUST create the directory and an empty `spec/ideas/README.md` index file before writing the first artifact. The auto-created index MUST be lint-clean per the canonical SpecScore Ideas Index spec (title `# Ideas Index`, `**Status:** Stable`, empty Contents table, "None at this time." Outstanding Questions, adherence footer). Auto-creation MUST NOT happen silently — the skill MUST tell the user it is bootstrapping the directory.

#### REQ: auto-stage-on-create

When the skill creates files (the bootstrapped `spec/ideas/`, `spec/ideas/README.md`, or any new `spec/ideas/<slug>.md`), it MUST `git add` those paths to the index and MUST report the staged paths to the user in the same response that confirms the write. The skill MUST NOT commit on the user's behalf — staging only. If staging fails (no git repository, detached worktree, lock contention), the skill MUST surface the failure to the user and continue without aborting the artifact write.

### Three-phase dialogue

The skill runs a fixed three-phase structure: divergent exploration, convergent evaluation, and crystallization into the artifact.

#### REQ: phase-1-divergent

In Phase 1 the skill MUST: (a) restate the user's input as a single "How Might We…" sentence, (b) ask 3–5 sharpening questions in batched form covering at minimum *who is this for* and *what does success look like*, and (c) generate 5–8 directional variations using lenses such as inversion, constraint removal, audience shift, combination, simplification, 10x, or expert lens.

#### REQ: phase-2-convergent

In Phase 2 the skill MUST: (a) cluster the variations into 2–3 meaningfully different directions, (b) stress-test each direction on user value, feasibility, and differentiation, and (c) surface hidden assumptions tagged Must-be-true / Should-be-true / Might-be-true.

#### REQ: phase-3-crystallize

In Phase 3 the skill MUST produce the Idea artifact at the canonical path. The artifact's content MUST conform to the Idea schema documented in the skill manifest, including all required sections.

### CLI vs fallback path

Crystallization prefers a stable CLI contract over ad-hoc file writes.

#### REQ: cli-preferred

When the `specscore` CLI is on PATH, the skill MUST use `specscore new idea <slug>` to scaffold the artifact, passing every field it already has via the documented flags (`--title`, `--owner`, `--hmw`, `--context`, `--recommended-direction`, `--mvp`, `--not-doing`).

#### REQ: cli-flag-discipline

The skill MUST NOT invent CLI flags. If a needed input has no flag, the skill MUST fill that section via `Edit` after scaffolding.

#### REQ: fallback-direct-write

When the `specscore` CLI is NOT on PATH, the skill MUST fall back to a direct file write using the same authoritative schema documented in the skill manifest.

#### REQ: schema-equivalence-cli-fallback

CLI and fallback paths MUST produce **schema-equivalent** artifacts: identical title prefix, identical body-metadata fields, identical section headings, identical required content. They MAY differ in cosmetic ways (whitespace, blank-line counts, comment style, default ordering of optional fields). The contract is that downstream consumers (`specscore spec lint`, `specstudio:specify`, human readers) cannot tell which path produced the artifact based on its functional content.

### Required artifact content

Some sections are non-negotiable.

#### REQ: not-doing-required

The Idea artifact's `Not Doing (and Why)` section MUST be non-empty. This corresponds to lint rule `I-002`. An Idea without explicit exclusions is rejected.

#### REQ: assumption-tiers

The Idea artifact MUST include at least one Must-be-true assumption in the `Key Assumptions to Validate` table. This corresponds to lint rule `I-003`.

### Lint and self-review

Every artifact passes through both machine validation and a deliberate human-style review.

#### REQ: lint-pass

After writing or editing the artifact, the skill MUST run `specscore spec lint` and confirm a zero exit code before proceeding to user review.

#### REQ: lint-failure-recovery

On `specscore spec lint` failure after a write or edit, the skill MUST:

1. Run `specscore spec lint --fix` once to attempt automated repair.
2. Re-run `specscore spec lint` to verify the result.
3. If lint now passes, continue and tell the user what was auto-fixed.
4. If lint still fails, surface the remaining violations to the user with rule IDs and affected sections.

The skill MUST NOT loop `--fix` more than once. **The skill MUST NOT carry its own knowledge of which lint rules are auto-fixable** — that policy belongs to the `specscore` CLI. If `--fix` silently repairs a violation that should require human input, that is a CLI bug to file against `specscore`, not a workaround to encode in this skill.

#### REQ: inline-self-review

Before requesting user review, the skill MUST scan the artifact for: (a) unresolved placeholders (`TBD`, `TODO`, `???`, `FIXME`), (b) internal contradictions (Recommended Direction vs. MVP Scope, assumptions vs. Not-Doing), (c) hidden multi-Idea scope, and (d) requirements interpretable two ways. Findings MUST be fixed inline.

### User review and approval

The user — not the skill — owns the approval decision.

#### REQ: user-approval-required

The skill MUST present the lint-clean artifact to the user with an explicit request to approve the Recommended Direction. The skill MUST NOT proceed to status transition or event emission without that approval.

#### REQ: approval-explicit-phrase

The skill MUST recognize the following English phrases (case-insensitive, optional surrounding punctuation) as unambiguous approval: `approve`, `approved`, `accept`, `accepted`, `lgtm`. The skill MUST also recognize their **direct semantic equivalents in any language the user is communicating in** — for example, `aprobar` / `aprobado` (Spanish), `approuver` / `approuvé` (French), `承認` / `承認する` (Japanese), `одобрить` / `одобрено` (Russian), `批准` / `同意` (Chinese), `genehmigen` / `genehmigt` (German). The criterion is semantic, not lexical: the phrase MUST mean "I give explicit approval" as a verb form in the source language, not a generic affirmative.

On detection of any qualifying phrase as a standalone user response (or as the dominant content of a short response), the skill MUST proceed directly to the status transition without asking for confirmation.

The boundary is intentionally tight in every language — generic affirmatives like English `yes` / `ok`, Spanish `sí`, French `oui`, Russian `да`, Japanese `はい`, Chinese `好`, as well as ambiguous tokens like `+1`, `ship it`, `🚀`, `confirm` are NOT in the explicit set. They fall into the vague-signal tier per `approval-vague-confirmation`. When in doubt, the skill MUST treat the signal as vague.

#### REQ: approval-vague-confirmation

When the user's response signals positive sentiment but does not contain a recognized explicit approval phrase (e.g., "looks good", "yeah", "nice", "ship it", "+1", `🚀`), the skill MUST treat this as a soft signal and ask one explicit confirmation question (e.g., "Treat that as approval?") before proceeding. The skill MUST NOT silently transition status on a vague signal.

#### REQ: status-transition-under-review

When the skill first presents the lint-clean Idea to the user for review, the skill MUST update the artifact's `**Status:**` body-metadata line from `Draft` to `Under Review`. Subsequent edits during user iteration keep `**Status:** Under Review` until either the user approves (→ `Approved`) or explicitly drops back to `Draft` for substantial rework.

#### REQ: status-transition-on-approval

On confirmed user approval (per `approval-explicit-phrase` or `approval-vague-confirmation`), the skill MUST update the artifact's `**Status:**` body-metadata line from `Under Review` to `Approved` and re-run lint to confirm the transition is still valid.

### Event emission

The skill participates in the Synchestra event vocabulary defined in [`shared/synchestra-events.md`](../../../../skills/shared/synchestra-events.md).

#### REQ: event-drafted

While the artifact's `**Status:**` is `Draft`, the skill MUST emit `idea.drafted` after every successful `specscore spec lint` pass that follows a write or edit. The first emission carries the same event name as subsequent ones — Synchestra dedupes by event uuid.

#### REQ: event-approved

The skill MUST emit `idea.approved` exactly once, after the user approves the Recommended Direction and the status transition Draft → Approved completes successfully.

#### REQ: event-updated

After `idea.approved` has fired, the artifact's `status` is `Approved`. While in that state, the skill MUST emit `idea.updated` after every successful `specscore spec lint` pass that follows a subsequent write or edit. The skill MUST NOT emit `idea.drafted` for an Approved artifact, and MUST NOT re-emit `idea.approved` for further iteration.

#### REQ: event-payload-change-context

Both `idea.drafted` and `idea.updated` event payloads MUST carry three change-context fields beyond the base payload (slug, hmw, target_user, approved):

- `changed_sections`: a list of H2 section names whose content differs from `previous_revision`. H3 changes within a section roll up to that section.
- `previous_revision`: the git SHA the diff is computed against.
- `change_summary`: a string of at most two sentences summarizing the change (see `change-summary-discipline`).

On the **first** `idea.drafted` emission for an Idea (initial write, no prior revision), all three fields MUST be `null`. On every subsequent emission of either event, all three MUST be present and non-null.

#### REQ: change-summary-discipline

The `change_summary` field MUST describe the change in **factual, observable terms** — what content was added, removed, or modified in which section. The summary MUST NOT speculate about the user's intent, motivation, or downstream implications, and MUST NOT editorialize about the quality or wisdom of the change. If the only change is whitespace or formatting, the summary MUST say so explicitly. Maximum length: two sentences.

### Promotion boundary

Promotion to a Feature is the responsibility of `specstudio:specify` and Synchestra tooling, not of `ideate`.

#### REQ: no-manual-promotes-to

The skill MUST NOT manually edit the `**Promotes To:**` body-metadata line. That line is managed by Synchestra in response to a Feature declaring this Idea in its `**Source Ideas:**` line.

#### REQ: promotion-out-of-scope

The skill MUST NOT scaffold, write, or modify SpecScore Features. When a user requests promotion immediately after approval, the skill MUST hand off to `specstudio:specify`.

### Tone

#### REQ: honest-pushback

The skill MUST NOT yes-machine weak ideas. When a direction has clear problems, the skill MUST say so with specificity and propose the alternative. The acceptance bar is honest disagreement, not performative agreement.

## Interaction with Other Features

| Feature | Interaction |
|---|---|
| [Specify Skill](../specify/README.md) | `ideate` is the upstream gate of `specify`. `specify` consumes the approved Idea via `Source Ideas` linkage; `ideate` never invokes `specify` itself — the user does. |
| [SpecScore Idea](https://github.com/synchestra-io/specscore/blob/main/spec/features/idea/README.md) | The schema, lint rules, and lifecycle of the produced artifact are owned by SpecScore's Idea feature. `ideate` is a producer, not a definer of that schema. |
| Synchestra Events | Emits `idea.drafted` and `idea.approved`. Consumers — including `specify` and Hub — observe these to advance their own state. |
| `specscore new idea` CLI | Preferred crystallization path. The skill probes for the CLI once per invocation and falls back to direct write only when absent. |

## Acceptance Criteria

### AC: hard-gate-enforced

**Requirements:** ideate#req:hard-gate, ideate#req:lint-pass, ideate#req:user-approval-required

The skill cannot invoke `specify`, `writing-plans`, or any implementation skill until the artifact exists at `spec/ideas/<slug>.md`, `specscore spec lint` passes, and the user has approved the Recommended Direction. Any invocation that bypasses any of these conditions is rejected.

### AC: artifact-conformance

**Requirements:** ideate#req:artifact-path, ideate#req:no-docs-path, ideate#req:auto-create-ideas-dir, ideate#req:auto-stage-on-create, ideate#req:phase-3-crystallize, ideate#req:not-doing-required, ideate#req:assumption-tiers

Every produced artifact lives at the canonical path `spec/ideas/<slug>.md`, conforms to the Idea schema, has a non-empty `Not Doing (and Why)` section, and lists at least one Must-be-true assumption. When `spec/ideas/` does not exist, the skill bootstraps it (with a lint-clean index README) and tells the user. All files the skill creates are staged in git and the staged paths are reported to the user; the skill never commits on the user's behalf. Artifacts written elsewhere or missing required sections are rejected by `specscore spec lint`.

### AC: phase-discipline

**Requirements:** ideate#req:phase-1-divergent, ideate#req:phase-2-convergent

The skill executes the three-phase dialogue in order. Phase 1 produces a HMW restatement, sharpening questions covering at minimum *who* and *success*, and 5–8 variations. Phase 2 produces 2–3 stress-tested directions with assumptions tiered into Must / Should / Might.

### AC: cli-vs-fallback

**Requirements:** ideate#req:cli-preferred, ideate#req:cli-flag-discipline, ideate#req:fallback-direct-write, ideate#req:schema-equivalence-cli-fallback

The skill probes for the `specscore` CLI once per invocation. When present, scaffolding goes through `specscore new idea <slug>` with only documented flags. When absent, the skill writes the artifact directly using the authoritative schema. Both paths produce schema-equivalent artifacts — identical title prefix, identical body-metadata fields, identical sections, identical required content — though they MAY differ in cosmetic ways (whitespace, blank lines, comment style).

### AC: lifecycle-events

**Requirements:** ideate#req:event-drafted, ideate#req:event-approved, ideate#req:event-updated, ideate#req:event-payload-change-context, ideate#req:change-summary-discipline, ideate#req:status-transition-on-approval

While `**Status:** Draft`, every successful lint pass after a write/edit emits `idea.drafted`. On confirmed user approval, `**Status:**` transitions Draft → Approved, lint is re-run, and `idea.approved` is emitted exactly once. While `**Status:** Approved`, every successful lint pass after a subsequent write/edit emits `idea.updated` (never `idea.drafted` and never a second `idea.approved`). Both `idea.drafted` and `idea.updated` payloads carry `changed_sections`, `previous_revision`, and `change_summary` (null on the first `idea.drafted`, non-null thereafter); the `change_summary` is factual and bounded to two sentences. Skipping, reordering, misclassifying, or generating speculative/editorializing summaries is a contract violation.

### AC: approval-detection

**Requirements:** ideate#req:approval-explicit-phrase, ideate#req:approval-vague-confirmation, ideate#req:user-approval-required

The skill detects approval in two tiers: explicit phrases (`approve`, `approved`) trigger immediate transition without further confirmation; vague positive signals trigger a single explicit confirmation prompt. Silent transition on vague signals is a contract violation.

### AC: lint-failure-recovery

**Requirements:** ideate#req:lint-pass, ideate#req:lint-failure-recovery

On lint failure after a write/edit, the skill runs `specscore spec lint --fix` exactly once, re-runs lint, and either continues (logging what was fixed) or surfaces the remaining violations to the user. The skill never loops `--fix`, and never encodes its own list of which rules are auto-fixable — the `specscore` CLI owns that policy.

### AC: skip-condition-respected

**Requirements:** ideate#req:skip-when-clear-intent

When the user can articulate the problem, recommended direction, and Not-Doing boundary up front, the skill hands off to `specstudio:specify` without running the three-phase dialogue. The hand-off is explicit, not silent.

### AC: promotion-boundary-held

**Requirements:** ideate#req:no-manual-promotes-to, ideate#req:promotion-out-of-scope

The skill never edits `**Promotes To:**`, never scaffolds a Feature, and never modifies an existing Feature. Promotion requests are routed to `specstudio:specify`.

## Outstanding Questions

None at this time.

---
*This document follows the https://specscore.md/feature-specification*
