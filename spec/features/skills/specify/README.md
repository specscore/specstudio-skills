---
format: https://specscore.md/feature-specification
status: Approved
---

# Feature: Specify Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/specify?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/specify?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/specify?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/specify?op=request-change) |

**Status:** Approved
**Source Ideas:** —

## Summary

The `specstudio:specify` skill turns an approved SpecScore Idea — or a clear buildable intent — into a lint-clean SpecScore Feature with requirements and `Given / When / Then` acceptance criteria. Its output (`spec/features/<slug>/`) is the gating input for `specstudio:plan` and any implementation skill. Unlike `ideate` (single-file artifact), `specify` produces a multi-file directory with a README, one requirement per file, optional Rehearse stubs, and optional assets. Implementation lives at [`skills/specify/`](../../../../skills/specify/).

**Provenance.** This skill is adapted from `obra/superpowers`'s `brainstorming` skill — its checklist flow, `<HARD-GATE>` pattern, "too simple to spec" anti-pattern, one-question-at-a-time cadence, spec-document reviewer (now consumed via the [reviewer-gates](../../reviewer-gates/README.md) Feature as an opt-in `type: ai` entry), and `writing-plans` terminal handoff are inherited and rebuilt against canonical SpecScore conventions (multi-file `spec/features/<slug>/`, body-metadata header fields, lint-enforced `Given / When / Then` acceptance criteria). Lens-based divergence (inversion, constraint removal, simplification, 10x, …) and the painkiller-vs-vitamin evaluation tier are grafted from `addyosmani/agent-skills`'s `idea-refine`. The full mapping of inherited vs. grafted vs. dropped patterns lives in [`spec/research/ideate-vs-brainstorming-skills-analysis.md`](../../../research/ideate-vs-brainstorming-skills-analysis.md).

## Problem

Even when an Idea is well-formed, the jump from "approved direction" to "buildable contract" is non-trivial. Without a structured skill, that jump happens ad-hoc: requirements get scattered into prose, acceptance criteria get omitted or written as vibes, the Source Idea linkage gets lost, Rehearse stubs are forgotten, and the Feature ends up as a description rather than a contract that downstream skills (`plan`, `build`, `verify`) can consume.

`specify` exists to enforce the structural discipline of a SpecScore Feature — canonical title prefix, body-metadata header fields, inline `#### REQ:` requirement decomposition, G/W/T ACs, reviewer gate (consumed from [reviewer-gates](../../reviewer-gates/README.md)), Rehearse coverage decision — without forcing the user to memorize the schema. The Feature defines exactly what that discipline must look like, so the artifact is machine-verifiable and downstream skills can rely on it.

## Behavior

### Invocation and skip conditions

The skill is invoked when an Idea has been approved, or when the user has clear buildable intent that bypasses the ideation phase.

#### REQ: invocation-triggers

The skill MUST respond to the triggers `specify`, `/specify`, "spec this out", and the event `idea.approved`. It MAY respond to additional natural-language phrasings of the same intent.

#### REQ: accepts-idea-or-intent

The skill MUST accept two kinds of input: (a) an approved Idea (path to `spec/ideas/<slug>.md` with `**Status:** Approved`), or (b) a clear buildable intent stated directly by the user. When (a) is the input, the skill MUST load the Idea and surface its assumptions, alternatives, and Not-Doing list as initial Feature material. When (b) is the input, the skill MUST still elicit the same structural elements before proceeding.

#### REQ: clear-intent-criterion

For the no-Idea path, "clear buildable intent" is satisfied when the user can articulate the problem, the recommended direction, and an explicit Not-Doing boundary in one short statement. If the intent fails this criterion, the skill MUST recommend running `specstudio:ideate` first instead of producing a low-quality Feature.

### Hard gate

The skill enforces a hard gate downstream — once invoked, it must produce a complete, lint-clean, reviewer-approved, user-approved Feature before any other SpecStudio skill can run.

#### REQ: hard-gate

The skill MUST NOT invoke `writing-plans`, `frontend-design`, `mcp-builder`, or ANY implementation skill until ALL FOUR conditions hold:

1. The Feature artifact exists at `spec/features/<slug>/README.md` and contains at least one `#### REQ: <slug>` requirement inside the `## Behavior` section.
2. Each requirement has ≥1 acceptance criterion in `Given / When / Then` format.
3. `specscore spec lint` exits zero.
4. The reviewer gate has released — every entry in `gates.specify.reviewers` (including the `type: human` entry that captures the user's approval) returned `Approved` per the [reviewer-gates](../../reviewer-gates/README.md) Feature's AND-composition.

A skill invocation that bypasses any condition is a contract violation.

### Artifact location and structure

Feature artifacts have one canonical location and a fixed multi-file shape.

#### REQ: artifact-path

The skill MUST write Feature artifacts under `spec/features/<slug>/` relative to the project root. The `<slug>` MUST be lowercase, hyphen-separated, URL-safe, and stable for the artifact's lifetime.

#### REQ: artifact-structure

The Feature directory MUST contain a single `README.md` — the Feature artifact per the canonical SpecScore Feature spec, with topic-grouped `### <Topic>` headings and `#### REQ: <slug>` requirements inline inside `## Behavior`. Requirements are a heading convention, not a separate file artifact (per `https://specscore.md/requirement-specification`). Optional sub-directories: `_tests/` (Rehearse scenarios), `assets/` (diagrams, mockups), `proposals/` (post-Stable change requests).

#### REQ: no-docs-path

The skill MUST NOT write Feature artifacts to `docs/features/`, `docs/superpowers/specs/`, `notes/`, or any path other than `spec/features/<slug>/`. `docs/` is reserved for prose documentation; SpecScore artifacts live under `spec/`.

#### REQ: auto-create-features-dir

When invoked in a project that does not yet have a `spec/features/` directory, the skill MUST create the directory and a lint-clean `spec/features/README.md` index file before writing the first Feature. The auto-created index MUST be lint-clean per the canonical SpecScore Features Index spec (title `# Features`, `**Status:** Stable`, empty Contents table, "None at this time." Outstanding Questions, adherence footer). Auto-creation MUST NOT happen silently — the skill MUST tell the user it is bootstrapping the directory.

### Schema and body metadata

The Feature artifact conforms to the canonical SpecScore Feature spec — title-prefix dispatch key, bold-prefixed body metadata, fixed section schema, no YAML front-matter.

#### REQ: feature-schema-conformance

The Feature `README.md` MUST follow the SpecScore Feature template: `# Feature: <Title>` heading, `**Status:**` field immediately after, `## Summary` (1–3 sentences), `## Problem`, `## Behavior` (with `#### REQ: <slug>` rules under topic headings), `## Acceptance Criteria` (with `### AC: <slug>` blocks bundling REQs), `## Outstanding Questions`, and the adherence footer.

#### REQ: feature-status-domain

The `**Status:**` body-metadata value MUST be one of: `Draft`, `Under Review`, `Approved`, `Implementing`, `Stable`, `Deprecated`. The skill MUST set `**Status:** Draft` on initial write and manage transitions through `Under Review` and `Approved` per the rules below. The skill does NOT manage transitions to `Implementing`, `Stable`, or `Deprecated` — those are owned by downstream skills (`writing-plans` for `Implementing`) or user-driven (for `Stable` and `Deprecated`).

#### REQ: requirement-format

Requirements MUST be `#### REQ: <slug>` sub-headings scoped under a `### <Topic>` heading inside `## Behavior` — never separate files, never directly under `## Behavior` without a topic heading. Each requirement is followed by a one-paragraph prose description that states the enforceable rule (typically using MUST/SHOULD/MAY language). Acceptance criteria for the requirement live either inline beneath the REQ block or grouped in `## Acceptance Criteria` with explicit `verifies REQ:<slug>` back-references — both forms are valid; choose one and be consistent within a Feature.

#### REQ: ac-format

Every acceptance criterion MUST use `Given / When / Then` form. This corresponds to lint rule `F-004`. ACs that cannot be phrased as `Then <observable outcome>` MUST be sharpened or removed; abstract aspirations are not acceptance criteria.

### Source Idea linkage

When a Feature originates from an Idea, the linkage is declared by the Feature.

#### REQ: source-idea-field

When the Feature originates from one or more approved Ideas, the Feature `README.md` MUST declare them via a `**Source Ideas:** <slug>, <slug>, ...` body-metadata line immediately after the `**Status:**` field. Every listed slug MUST resolve to an existing Idea at `spec/ideas/<slug>.md` with `Status ∈ {Approved, Specified}`.

#### REQ: no-manual-promotes-to

The skill MUST NOT manually edit any referenced Idea's `**Promotes To:**` or `**Status:**` body-metadata lines. Synchestra reconciles those in response to the Feature declaring its `**Source Ideas:**`. The Feature is the only authoritative input; the Idea side is managed.

#### REQ: related-idea-surfacing

When the user invokes `specify` on the **clear-intent path** (no Idea explicitly named), the skill MUST scan `spec/ideas/` for Approved Ideas semantically related to the user's intent and surface any candidates to the user with their slugs and HMWs before writing the Feature. The user MAY (a) link any subset via `**Source Ideas:**`, (b) waive — proceed without linking — or (c) revise the intent. The skill MUST acknowledge the user's choice in its response and proceed accordingly. Silent orphaning of related Approved Ideas is a contract violation; the user's waiver is not blocked but MUST be a conscious choice the skill explicitly acknowledged.

### CLI vs fallback path

Crystallization prefers a stable CLI contract over ad-hoc file writes when one becomes available.

#### REQ: cli-preferred

When a `specscore feature new` (or equivalent) CLI command is on PATH, the skill MUST use it to scaffold the Feature directory, passing every field it already has via documented flags. The skill MUST NOT invent flags.

#### REQ: fallback-direct-write

When no scaffolding CLI is available, the skill MUST fall back to direct file writes using the authoritative SpecScore Feature schema. CLI and fallback paths MUST produce schema-equivalent artifacts (identical title prefix, identical body-metadata fields, identical sections, identical required content). They MAY differ in cosmetic ways (whitespace, blank lines, comment style).

### Lint and self-review

Every artifact passes through machine validation, automated repair, and a deliberate human-style review.

#### REQ: lint-pass

After writing or editing the Feature, the skill MUST run `specscore spec lint` and confirm a zero exit code before proceeding to the reviewer gate.

#### REQ: lint-failure-recovery

On `specscore spec lint` failure, the skill MUST:

1. Run `specscore spec lint --fix` exactly once.
2. Re-run `specscore spec lint` to verify.
3. If passing, continue and tell the user what was auto-fixed.
4. If still failing, surface remaining violations to the user with rule IDs and affected files.

The skill MUST NOT loop `--fix` more than once. **The skill MUST NOT carry its own knowledge of which lint rules are auto-fixable** — that policy belongs to the `specscore` CLI. If `--fix` silently repairs a violation that should require human input, that is a CLI bug to file against `specscore`, not a workaround to encode in this skill.

#### REQ: inline-self-review

Before dispatching the reviewer gate, the skill MUST scan the Feature for: (a) unresolved placeholders (`TBD`, `TODO`, `???`, `FIXME`), (b) internal contradictions (architecture vs. requirements, requirements vs. ACs), (c) hidden multi-Feature scope, and (d) requirements interpretable two ways. Findings MUST be fixed inline.

### Reviewer gate

The skill's reviewer gate — including the user's own approval — is consumed from the typed-gate model defined by the [reviewer-gates](../../reviewer-gates/README.md) Feature, not from any built-in baseline or separate downstream user-approval step.

#### REQ: reviewer-gate-via-reviewer-gates

The skill MUST consume its reviewer gate from `gates.specify.reviewers` in `specscore.yaml` per the [reviewer-gates](../../reviewer-gates/README.md) Feature's `specify-loads-gate` and `missing-gates-block-refuses` REQs. The skill MUST NOT carry a hardcoded baseline reviewer and MUST NOT dispatch any reviewer outside the registry; the prior baseline-reviewer prompt at `skills/specify/references/reviewer-prompt.md` is opt-in as a `type: ai` entry in the registry like any other reviewer.

#### REQ: user-approval-via-human-reviewer-entry

The user's approval MUST be collected through a `type: human` entry in `gates.specify.reviewers` per the [reviewer-gates](../../reviewer-gates/README.md) Feature's `human-entry-shape` REQ — dispatched serially with the other reviewers and contributing its verdict to the gate's AND-composition. The skill MUST NOT run a separate downstream user-approval step outside the gate; the explicit-approval phrase set is still recognized but only inside the `type: human` reviewer dispatch.

### Approval phrase recognition and status transitions

The user's approval is collected inside the gate's `type: human` reviewer entry; the phrase set the skill recognizes inside that dispatch — and the Feature `**Status:**` transitions that follow a successful gate — are defined here.

#### REQ: approval-explicit-phrase

The skill MUST recognize the same explicit-approval phrase set as `specstudio:ideate`: English `approve`, `approved`, `accept`, `accepted`, `lgtm`, plus their direct semantic equivalents in any language the user is communicating in (e.g., `aprobar`, `承認`, `одобрено`, `批准`). The criterion is semantic — the phrase must function as a verb form meaning "I give explicit approval" in the source language. On detection of any qualifying phrase as a standalone or dominant response, the skill MUST proceed directly to the status transition.

#### REQ: approval-vague-confirmation

When the user's response signals positive sentiment but does not contain a recognized explicit phrase (e.g., `looks good`, `yeah`, `nice`, `ship it`, `+1`, `🚀`, `yes`, `ok`, `sí`, `oui`, `да`, `はい`), the skill MUST treat this as a soft signal and ask one explicit confirmation question (e.g., "Treat that as approval?") before proceeding. The skill MUST NOT silently transition status on a vague signal.

#### REQ: status-transition-under-review

When the skill begins dispatching the reviewer gate (per `reviewer-gate-via-reviewer-gates`), the skill MUST update the Feature's `**Status:**` body-metadata line from `Draft` to `Under Review`. The transition signals to consumers that the Feature is in active review; subsequent edits during reviewer iteration keep `**Status:** Under Review` until the gate releases (→ `Approved`) or the user explicitly drops back to `Draft` for substantial rework.

#### REQ: status-transition-on-approval

On the reviewer gate releasing (every reviewer entry in `gates.specify.reviewers` returned `Approved` per AND-composition, including the `type: human` entry that captured the user's explicit-phrase approval), the skill MUST update the Feature's `**Status:**` body-metadata line from `Under Review` to `Approved`, re-run lint, and emit `feature.approved`. The transition Approved → Implementing is owned by `writing-plans` (when implementation work begins), not by `specify`.

### Rehearse stub decision

For each AC, a per-AC decision determines whether to scaffold a Rehearse test stub.

#### REQ: rehearse-per-ac-decision

For every AC in the Feature, the skill MUST apply the heuristic in [`shared/rehearse-heuristic.md`](../../../../skills/shared/rehearse-heuristic.md). When the AC is testable (CLI, HTTP, pure function, data, UI selector, filesystem, or event surface), the skill MUST scaffold `spec/features/<slug>/_tests/<scenario-slug>.md` with `**Status:** pending` body metadata. When the AC is not testable, the skill MUST record the skip-reason in the Feature `README.md` under a `## Rehearse Integration` subsection.

#### REQ: user-can-override-rehearse

The user MAY override the heuristic at any point — both directions (force a stub for a deemed-untestable AC, or skip a stub for a deemed-testable AC). The override MUST be recorded with reasoning.

### Auto-stage in git

Files the skill creates are staged for the user but never committed.

#### REQ: auto-stage-on-create

When the skill creates files (the bootstrapped `spec/features/`, `spec/features/README.md`, the Feature directory and its contents, Rehearse stubs), it MUST `git add` those paths to the index and report the staged paths to the user. The skill MUST NOT commit on the user's behalf. If staging fails (no git repository, lock contention), the skill MUST surface the failure and continue without aborting the artifact write.

### Event emission

The skill participates in the event vocabulary.

#### REQ: event-specified

The skill MUST emit `feature.specified` after lint passes and before dispatching the reviewer gate — that is, when the Feature is structurally ready for the gate to evaluate. The first emission for a Feature carries `previous_revision: null`; subsequent emissions during gate iteration carry the previous revision.

#### REQ: event-approved

The skill MUST emit `feature.approved` exactly once, after the user approves the Feature and the status transition Draft → In Progress completes successfully.

#### REQ: event-updated

After `feature.approved` has fired, the Feature is alive but not frozen (it remains `Approved`, transitions to `Implementing` when `writing-plans` runs, then to `Stable` on user instruction). On every successful lint pass after a subsequent write or edit while `status ∈ {Approved, Implementing, Stable}`, the skill MUST emit `feature.updated`. The skill MUST NOT emit `feature.specified` for an already-approved Feature, and MUST NOT re-emit `feature.approved` for further iteration.

#### REQ: event-payload-change-context

`feature.specified`, `feature.approved`, and `feature.updated` event payloads MUST carry the same change-context fields as the Idea events: `changed_sections` (H2 section names whose content differs from `previous_revision`, with H3 changes rolled up), `previous_revision` (git SHA), and `change_summary` (≤2 sentences, factual, no speculation/editorializing). On the very first `feature.specified` emission for a Feature, all three are `null`.

### Promotion boundary

The next skill is `writing-plans`, and only `writing-plans`.

#### REQ: transition-to-writing-plans

After `feature.approved`, the skill MUST transition only to `writing-plans`. The skill MUST NOT invoke any other downstream skill — no `frontend-design`, no `mcp-builder`, no implementation skill of any kind.

#### REQ: revise-vs-supersede

When the user wants to change an existing Feature, the default is to revise the Feature in place (git history is the record of evolution). The skill MUST create a successor with `supersedes: [<predecessor-slug>]` only when the scope change invalidates existing acceptance criteria. The skill MUST NOT silently delete or rewrite an existing Feature without the user's explicit choice between revise-in-place and supersede.

### Tone

#### REQ: honest-pushback

The skill MUST NOT yes-machine weak Features. When a requirement is vague, an AC is unobservable, or a Feature spans multiple subsystems, the skill MUST say so with specificity and propose the alternative. The acceptance bar is honest disagreement, not performative agreement.

## Interaction with Other Features

| Feature | Interaction |
|---|---|
| [Ideate Skill](../ideate/README.md) | `specify` is the downstream gate of `ideate`. `specify` consumes the approved Idea via `Source Ideas` linkage and surfaces its assumptions, alternatives, and Not-Doing list as initial Feature material. The `idea.approved` event triggers `specify` (with user confirmation). |
| [Reviewer Gates](../../reviewer-gates/README.md) | `specify` consumes its reviewer gate — including the user's own approval — from `gates.specify.reviewers` per `reviewer-gates`'s `specify-loads-gate`, `dispatch-serial`, `and-composition`, `rerun-policy`, and `missing-gates-block-refuses` REQs. No built-in baseline reviewer and no separate downstream user-approval step. |
| [Plan Skill](../plan/README.md) | `specify` is the upstream gate of `plan`. `plan` consumes the approved Feature; `specify` never invokes `plan` itself — `writing-plans` is the explicit transition. |
| [SpecScore Feature](https://github.com/specscore/specscore/blob/main/spec/features/feature/README.md) | The schema, lint rules, and lifecycle of the produced Feature artifact are owned by SpecScore's Feature feature. `specify` is a producer, not a definer of that schema. |
| Synchestra Events | Emits `feature.specified`, `feature.approved`, and `feature.updated`, all with change-context payloads. Consumers — including downstream Features that declare this Feature as a dependency, and Hub — observe these to advance their own state. |
| `specscore` CLI | Preferred crystallization path when a `specscore feature new` (or equivalent) command is available. The skill probes for the CLI once per invocation and falls back to direct write only when absent. |
| Rehearse | The `_tests/` directory under each Feature holds Rehearse-executable scenario files. `specify` decides per-AC whether to scaffold a stub via the rehearse-heuristic. |

## Acceptance Criteria

### AC: hard-gate-enforced

**Requirements:** specify#req:hard-gate, specify#req:lint-pass, specify#req:reviewer-gate-via-reviewer-gates

The skill cannot invoke `writing-plans` or any implementation skill until the Feature exists with at least one requirement and ≥1 G/W/T AC, `specscore spec lint` passes, and the reviewer gate (consumed from `gates.specify.reviewers` per the [reviewer-gates](../../reviewer-gates/README.md) Feature) has released with every entry — including the `type: human` entry that captures the user's approval — returning `Approved`. Bypassing any condition is rejected.

### AC: artifact-conformance

**Requirements:** specify#req:artifact-path, specify#req:no-docs-path, specify#req:auto-create-features-dir, specify#req:auto-stage-on-create, specify#req:artifact-structure, specify#req:feature-schema-conformance, specify#req:requirement-format, specify#req:ac-format

Every produced Feature lives under `spec/features/<slug>/` as a single canonical `README.md` (plus optional `_tests/`, `assets/`), conforms to the SpecScore Feature schema, has body metadata with the required fields, decomposes behavior into inline `#### REQ: <slug>` requirements scoped under `### <Topic>` headings inside `## Behavior`, and uses `Given / When / Then` for every AC. When `spec/features/` does not exist, the skill bootstraps it (with a lint-clean index) and tells the user. All created files are staged in git; the skill never commits.

### AC: source-idea-linkage

**Requirements:** specify#req:source-idea-field, specify#req:no-manual-promotes-to, specify#req:accepts-idea-or-intent, specify#req:related-idea-surfacing

When a Feature originates from one or more approved Ideas, the Feature declares them via `**Source Ideas:**`. Every referenced Idea exists with `Status ∈ {Approved, Specified}`; missing or wrong-status references are rejected by lint. The skill never edits the referenced Idea's `promotes_to` or `Status` directly — Synchestra reconciles those. On the clear-intent path, the skill scans for semantically related Approved Ideas and surfaces them to the user before writing; the user may link, waive, or revise, but waiving must be a conscious acknowledged choice.

### AC: cli-vs-fallback

**Requirements:** specify#req:cli-preferred, specify#req:fallback-direct-write

When a `specscore feature new` (or equivalent) CLI is on PATH, the skill uses it; otherwise it falls back to direct write. Both paths produce schema-equivalent Features (identical title prefix, body-metadata fields, sections, required content) — cosmetic differences are allowed.

### AC: lint-and-recovery

**Requirements:** specify#req:lint-pass, specify#req:lint-failure-recovery

After each write/edit, lint is run; on failure, `specscore spec lint --fix` is attempted exactly once before re-lint. If still failing, remaining violations are surfaced to the user with rule IDs and affected files. The skill never loops `--fix`, and never encodes its own list of which rules are auto-fixable — the `specscore` CLI owns that policy.

### AC: consumes-gates-specify

**Requirements:** specify#req:reviewer-gate-via-reviewer-gates, specify#req:user-approval-via-human-reviewer-entry

**Given** a `specscore.yaml` with a valid `gates.specify.reviewers` block containing one `type: ai` entry (pointing at `skills/specify/references/reviewer-prompt.md`) followed by one `type: human` entry,
**When** `specstudio:specify` runs the reviewer gate after lint passes,
**Then** the skill MUST dispatch exactly the listed entries in list order via the [reviewer-gates](../../reviewer-gates/README.md) Feature's loader/runner per its `specify-loads-gate` REQ, MUST NOT additionally dispatch any built-in baseline reviewer or any reviewer hardcoded in the skill's own logic, MUST NOT run a separate downstream user-approval step outside the gate, and the gate's `Approved` verdict (AND-composition across the two entries) MUST be the sole gating signal for the Draft → Approved status transition.

### AC: approval-detection

**Requirements:** specify#req:approval-explicit-phrase, specify#req:approval-vague-confirmation

The skill detects approval in two tiers inside the gate's `type: human` reviewer dispatch: explicit phrases (English set + semantic equivalents in any language the user is communicating in) trigger an immediate `Approved` verdict for the entry; vague positive signals trigger a single confirmation prompt. Silent transition on vague signals is a contract violation.

### AC: rehearse-coverage

**Requirements:** specify#req:rehearse-per-ac-decision, specify#req:user-can-override-rehearse

Every AC has a recorded Rehearse decision: either a scaffolded stub at `spec/features/<slug>/_tests/<scenario-slug>.md` with `**Status:** pending` body metadata, or a documented skip-reason under `## Rehearse Integration` in the Feature README. The user can override either direction with reasoning.

### AC: lifecycle-events

**Requirements:** specify#req:event-specified, specify#req:event-approved, specify#req:event-updated, specify#req:event-payload-change-context, specify#req:status-transition-on-approval

`feature.specified` fires after lint pass and before reviewer-gate dispatch; `feature.approved` fires exactly once on the reviewer gate releasing (every entry returned `Approved`) and the Under Review → Approved transition; `feature.updated` fires on every successful lint pass after a subsequent edit. All three carry `changed_sections`, `previous_revision`, and a factual `change_summary` (null on the first `feature.specified`, non-null thereafter). Misclassifying or skipping an emission is a contract violation.

### AC: promotion-boundary-held

**Requirements:** specify#req:transition-to-writing-plans, specify#req:revise-vs-supersede

The skill never transitions to any skill other than `writing-plans`. When the user wants to change an existing Feature, revise-in-place is the default; supersede is reserved for scope changes that invalidate existing ACs and is the user's explicit choice.

## Open Questions
 Currently the spec lists three examples (missing Source Ideas, non-G/W/T ACs, empty Outstanding Questions); the canonical list should live in `shared/specscore-lint-rules.md` and be referenced.
- ~~**Reviewer registration mechanism.**~~ **Resolved by [`reviewer-gates`](../../reviewer-gates/README.md)**: reviewers register as ordered entries under `gates.specify.reviewers` in `specscore.yaml`, with a `type:` discriminator (`ai` / `human` in MVP) and type-specific fields. The contract for entry shape, dispatch order, AND-composition, and re-run policy lives in that Feature; the consumer-side wiring lives in [`skills/specify/SKILL.md`](../../../../skills/specify/SKILL.md).
- ~~**When does the reviewer concept earn its own Feature?**~~ **Resolved by [`reviewer-gates`](../../reviewer-gates/README.md)**. The prior baseline reviewer prompt at `skills/specify/references/reviewer-prompt.md` is now an opt-in `type: ai` registry entry rather than a hardcoded baseline.

---
*This document follows the https://specscore.md/feature-specification*
