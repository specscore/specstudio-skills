---
name: specify
description: |
  Turns an approved SpecScore Idea (or a clear buildable intent) into a
  SpecScore Feature with requirements and Given/When/Then acceptance
  criteria at spec/features/<slug>/. Gates implementation — no code,
  plans, or scaffolding until the Feature is lint-clean and
  user-approved. Optionally scaffolds Rehearse test stubs.
  Trigger: "specify this", "/specify", "spec this out", or the
  `idea.approved` event.
aliases: [specify]
---

# Specify

Turn approved intent into a lintable, testable SpecScore Feature.

## Hard Gate

<HARD-GATE>
Do NOT invoke `writing-plans`, `frontend-design`, `mcp-builder`, or ANY implementation skill until ALL of the following are true:
  1. The Feature artifact exists at `spec/features/<slug>/README.md` and contains at least one `#### REQ: <slug>` requirement inside the `## Behavior` section.
  2. Each requirement has ≥1 acceptance criterion in `Given / When / Then` format.
  3. `specscore spec lint` passes.
  4. The reviewer gate has released — every entry in `gates.specify.reviewers` (including the `type: human` entry that captures the user's approval) returned `Approved` per the [reviewer-gates](../../spec/features/reviewer-gates/README.md) Feature's AND-composition.

This applies to **every** Feature, regardless of perceived simplicity. The only skill invoked after `specstudio:specify` is `writing-plans`.
</HARD-GATE>

## When to Use

- A SpecScore Idea is `Approved` and ready to become a Feature.
- User has a clear, high-conviction buildable intent (may skip `specstudio:ideate`).
- Behavior of an existing Feature needs to change — **revise in place** (see [path-conventions.md](../shared/path-conventions.md)).

## Anti-Pattern: "This Is Too Simple To Need A Spec"

Every Feature goes through this. A toggle, a one-line config, a single utility — all of them. Simple Features are where unexamined assumptions cost the most. The spec can be short (a few sentences for truly simple Features), but it **must** be written and approved.

## Pre-Flight

1. **Inputs check.** If triggered from an approved Idea, load it and list the Idea's assumptions that this Feature must validate. If no Idea exists, ask: "Is this ready to specify, or should we ideate first?" — don't force `specstudio:ideate` on high-conviction users.
2. **Scope decomposition.** If the intent spans multiple independent subsystems, stop and help the user decompose into multiple Features before continuing.
3. **Revision vs new.** If a Feature with this slug already exists, decide: revise in place (default) or create a successor and set `**Supersedes:** <old-slug>` in the new Feature's body metadata (only when scope change invalidates existing ACs). See [path-conventions.md](../shared/path-conventions.md).

## Checklist

Create a task for each and complete in order:

1. **Explore project context** — existing Features, recent commits, related specs.
2. **Scope decomposition check.**
3. **Offer visual companion** (if visual questions are ahead) — own message, no other content. See [visual-companion.md](references/visual-companion.md) (still TBD — see [the analysis doc §11.1](../../spec/research/ideate-vs-brainstorming-skills-analysis.md)).
4. **Ask clarifying questions** — one at a time, multiple-choice preferred. See [question-cadence.md](../shared/question-cadence.md).
5. **Propose 2–3 approaches** with trade-offs; lead with your recommendation. Use `specstudio:ideate` lenses (inversion, constraint removal, simplification) where useful.
6. **Present spec sections** one at a time, get approval after each.
7. **Author the Feature artifact** — single `README.md` with topic-grouped `### <Topic>` headings inside `## Behavior`, each containing one or more `#### REQ: <slug>` requirements.
8. **Rehearse stub decision** — per-AC heuristic. See [rehearse-heuristic.md](../shared/rehearse-heuristic.md).
9. **Lint** — `specscore spec lint`.
10. **Inline self-review** — placeholders, consistency, scope, ambiguity.
11. **Run the reviewer gate** — load and dispatch `gates.specify.reviewers` from `specscore.yaml` per the [Reviewer Gates](../../spec/features/reviewer-gates/README.md) Feature. See the `## Reviewer Gate` section below.
12. **Emit events** — `feature.specified` after lint passes and before reviewer-gate dispatch; `feature.approved` after the reviewer gate releases (every entry returned `Approved`, including the `type: human` entry that captured the user's approval). See [events.md](../shared/events.md).
13. **Transition to `writing-plans`.**
14. **Throughout** — watch for sidekick ideas per [sidekick-capture.md](../shared/sidekick-capture.md). When an out-of-scope improvement surfaces, invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step immediately. Do not derail to discuss the sideline idea.

## Spec Sections (scale to complexity)

- **Purpose & user job** — from the Idea's HMW if applicable, or restated here.
- **Requirements** — numbered, each with ≥1 acceptance criterion.
- **Architecture & components** — isolation, interfaces, dependencies. Each unit: what does it do, how is it used, what does it depend on?
- **Data flow.**
- **Error handling & failure modes.**
- **Testing strategy** — references Rehearse stubs (or explains why none).
- **Not Doing / Out of Scope** — inherited from Idea + spec-level cuts.
- **Assumption carryover** — which Idea assumptions survive; which are now invalidated or answered.

## Artifact Layout

The schema follows canonical SpecScore: title prefix `# Feature: …` is the dispatch key, body metadata is bold-prefixed lines immediately after the title, requirements are inline `#### REQ: <slug>` sub-headings under topic `### <Topic>` headings inside `## Behavior` — **no YAML front-matter, no separate requirement files**.

```
spec/features/<slug>/
├── README.md                # The Feature artifact (single file)
├── _tests/                  # Optional — Rehearse scenarios (see shared/rehearse-heuristic.md)
│   ├── <scenario-1>.md
│   └── …
└── assets/                  # Optional — diagrams, mockups
```

**`README.md` schema (authoritative):**

```markdown
# Feature: <Title>

**Status:** Draft
**Date:** YYYY-MM-DD
**Owner:** <author identifier>
**Source Ideas:** —              <!-- or: <idea-slug>, <idea-slug> when this Feature originates from one or more Ideas -->
**Supersedes:** —                <!-- or: <old-feature-slug> on wholesale replacement -->

## Summary

1–3 sentences. What this Feature is and who it serves.

## Problem

Why this Feature exists. What gap or pain it addresses.

## Behavior

How the Feature works. Topics use `###` headings; individual rules use `#### REQ: <slug>` under their topic.

### <Topic-1>

Narrative context for this group of rules.

#### REQ: <req-slug-1>

The system MUST/MAY/SHOULD … (one enforceable rule, prose).

#### REQ: <req-slug-2>

…

### <Topic-2>

…

## Acceptance Criteria

Each REQ has ≥1 AC in `Given / When / Then` form. ACs may be inline under each REQ or grouped here with explicit REQ back-references — both forms are valid; pick one and be consistent.

### AC: <ac-slug-1> (verifies REQ:<req-slug-1>)

**Given** <precondition>
**When** <action>
**Then** <observable outcome>

### AC: <ac-slug-2> (verifies REQ:<req-slug-2>)

…

## Outstanding Questions

- <Open question that doesn't block approval but should be tracked>

(Or: "None at this time." — the section is never omitted.)

---
*This document follows the https://specscore.md/feature-specification*
```

Notes:
- The canonical id is the directory slug; there is no separate `id` field.
- `**Source Ideas:**` and `**Supersedes:**` MUST be present with value `—` when empty.
- Topics inside `## Behavior` MUST have a `###` heading; requirements (`#### REQ:`) MUST be scoped under a topic — never directly under `## Behavior`.

## Acceptance Criterion Format

Every AC uses `Given / When / Then`. This is **enforced by lint rule F-004**.

```
Scenario: <short name>
Given <precondition that sets state>
When <action the system or user takes>
Then <observable outcome that can be checked>
```

If you can't phrase an outcome as `Then <observable>`, the AC is too abstract — sharpen it.

## Rehearse Stub Decision

After drafting ACs, for each one apply the heuristic in [rehearse-heuristic.md](../shared/rehearse-heuristic.md):

- **Testable** (has CLI/HTTP/pure-fn/data/UI-selector/fs/event surface): scaffold `spec/features/<slug>/_tests/<req-slug>-<ac-slug>.md` with `**Status:** pending` body metadata.
- **Not testable** (subjective, abstract, undefined observer, doc-only Feature): skip; record reason in the Feature's `README.md` under `## Rehearse Integration`.

The user can always override the heuristic.

## Inline Self-Review

Check:

1. **Placeholder scan** — `TBD`, `TODO`, incomplete sections, vague requirements.
2. **Internal consistency** — architecture matches feature descriptions; requirements align with ACs.
3. **Scope check** — focused enough for one implementation plan? If not, decompose.
4. **Ambiguity check** — could any requirement be interpreted two ways? Pick one; make it explicit.

Fix inline. Don't re-review; move on.

## Reviewer Gate

The reviewer gate — including the user's own approval — is consumed from the typed-per-stage [Reviewer Gates](../../spec/features/reviewer-gates/README.md) contract. The skill carries NO hardcoded baseline reviewer and runs NO separate downstream user-approval step. The reviewer list comes exclusively from `gates.specify.reviewers` in `specscore.yaml`; the user's approval is collected through a `type: human` entry in that list like any other reviewer.

### Step 1 — Update status to `Under Review`

Before dispatching the gate, update the Feature's `**Status:**` body-metadata line from `Draft` to `Under Review`. Re-run lint to confirm the transition is clean.

### Step 2 — Load and validate the gate config

Follow the protocol in [`shared/reviewer-gates/loader.md`](../shared/reviewer-gates/loader.md) with `<skill> = specify`. The loader reads `gates.specify.reviewers` from `specscore.yaml`, validates every entry's shape (per the Reviewer Gates Feature's `reviewer-entry-required-fields`, `mvp-type-set`, `ai-entry-shape`, `human-entry-shape`, `no-untyped-entry` REQs), and returns an ordered list of validated entries.

If the loader refuses (missing `gates:` key, missing `gates.specify`, empty `reviewers: []`, any per-entry validation failure), surface its error verbatim and halt. Do NOT dispatch any reviewer, do NOT fall back to a built-in baseline, do NOT run a separate user-approval step, and do NOT emit any approval event.

### Step 3 — Run the gate

Follow the protocol in [`shared/reviewer-gates/runner.md`](../shared/reviewer-gates/runner.md), passing the validated reviewer list returned by the loader. The runner:

- Dispatches every entry serially in declared list order. For `type: ai`, the Agent tool is invoked with the entry's `prompt` file as the system prompt and the Feature `README.md` as the user-facing message. For `type: human`, the user is presented with the Feature and the approval-phrase recognizer described in `## Approval-Phrase Recognition` below collects the verdict.
- AND-composes the verdicts: the gate releases only when every entry returns `Approved`. On the first `Issues Found` verdict in a pass, the runner halts — subsequent entries in that pass are NOT dispatched.
- On rerun (after the user addresses findings), re-dispatches per the runner's `rerun-policy`: every previously-`Issues Found` reviewer always, every previously-`Approved` reviewer when the fix touched a structural section (`## Behavior`, `## Architecture`, or `## Acceptance Criteria`).

The skill MUST dispatch exactly the entries in `gates.specify.reviewers` — no more, no less. It MUST NOT additionally dispatch any reviewer hardcoded inside this skill's own logic, and MUST NOT silently inject a built-in baseline. The runner's verdict map is the sole gating signal.

### Step 4 — On `Approved`

When the runner returns `Approved` (every entry returned `Approved`):

- Update `**Status:** Under Review → Approved` in the body metadata.
- Re-run lint to confirm the transition is clean.
- Emit `feature.approved` (see Checklist step 12).
- Proceed to `## Transition to Implementation`.

### Step 5 — On `Issues Found`

When the runner returns `Issues Found`, surface the failing reviewer's name and every `Blocker`-severity finding verbatim. Advisory findings SHOULD also be surfaced (clearly labeled). Address every `Blocker` inline (edits to the Feature `README.md`), re-run lint, then re-invoke the runner per its `rerun-policy`. Do NOT emit `feature.approved` while any reviewer's last verdict is `Issues Found`.

## Approval-Phrase Recognition

When the runner dispatches a `type: human` entry, it presents the Feature to the user with a prompt such as:

> "Feature written and lint-clean at `spec/features/<slug>/`. The reviewer gate is waiting on your approval. Please review and let me know if you approve or want changes before we move to `writing-plans`."

The recognizer maps the user's response per the same explicit-approval phrase set used by `specstudio:ideate`:

- **Explicit approval phrase** — English `approve`, `approved`, `accept`, `accepted`, `lgtm`, plus direct semantic equivalents in any language the user is communicating in (e.g., `aprobar`, `承認`, `одобрено`, `批准`). The criterion is semantic: the phrase must function as a verb form meaning "I give explicit approval" in the source language. On detection of any qualifying phrase as a standalone or dominant response → verdict `Approved`.
- **Explicit change request** — the user names a concrete change they want before approving → verdict `Issues Found` with the user's change-request text captured as a single `Blocker` finding under the human entry's `name:`.
- **Vague positive signal** (e.g., `looks good`, `yeah`, `nice`, `ship it`, `+1`, `🚀`, `yes`, `ok`, `sí`, `oui`, `да`, `はい`) — not yet a verdict. Ask one explicit confirmation question (e.g., "Treat that as approval?") and wait. Silent transition on a vague signal is a contract violation.

## Transition to Implementation

Invoke `writing-plans`. Do **not** invoke any other skill. `writing-plans` is the next step.

## Visual Companion (Optional)

**Status:** decision pending — see `spec/research/ideate-vs-brainstorming-skills-analysis.md` §11.1.

Until the visual-companion strategy is decided, prefer these lightweight visual aids in text:

- **Mermaid diagrams** embedded in `README.md` (most IDEs and GitHub render these natively).
- **ASCII diagrams** for simple flows.
- **Static SVGs** committed to `spec/features/<slug>/assets/` for complex visuals.

If the user has `obra/superpowers` installed, we may reuse its browser-based visual companion — pending a formal integration decision.

## Verification

- [ ] `spec/features/<slug>/README.md` exists
- [ ] `## Behavior` contains at least one `#### REQ: <slug>` requirement (scoped under a `###` topic heading)
- [ ] Every requirement has ≥1 acceptance criterion
- [ ] Every AC is `Given / When / Then`
- [ ] `specscore spec lint` passes
- [ ] Reviewer gate released — every entry in `gates.specify.reviewers` (including the `type: human` entry) returned `Approved`
- [ ] `**Status:** Approved` in body metadata
- [ ] Rehearse decision recorded (stubs scaffolded OR skip-reason noted)
- [ ] Source Idea (if any) linked via the `**Source Ideas:**` body-metadata line — lifecycle tooling handles the reverse link
- [ ] `feature.specified` + `feature.approved` events emitted

## Red Flags

- Proceeding to `writing-plans` before the reviewer gate releases
- Dispatching a hardcoded baseline reviewer not present in `gates.specify.reviewers`
- Running a separate downstream user-approval step outside the `type: human` reviewer entry
- Silently downgrading a `Blocker` finding to `Advisory`, or skipping a registered reviewer
- Requirements without ACs
- ACs not in `Given / When / Then`
- "Too simple to spec" rationalization
- Scope spanning multiple subsystems
- Assumptions from the source Idea silently dropped
- Writing to `docs/superpowers/specs/` instead of `spec/features/<slug>/`
- Invoking any skill other than `writing-plans` on transition

## References

- [Reviewer Gates Feature](../../spec/features/reviewer-gates/README.md) — canonical typed-per-stage `gates:` schema, reviewer entry shape, AND-composition, and rerun policy.
- [shared/reviewer-gates/loader.md](../shared/reviewer-gates/loader.md) — load-and-validate protocol for `gates.specify.reviewers`.
- [shared/reviewer-gates/runner.md](../shared/reviewer-gates/runner.md) — dispatch and verdict-aggregation protocol.
- [references/reviewer-prompt.md](references/reviewer-prompt.md) — baseline reviewer prompt; opt-in via a `type: ai` entry in `gates.specify.reviewers`.
- [visual-companion.md](references/visual-companion.md) — visual companion strategy (decision pending).
- [philosophy.md](../shared/philosophy.md) — shared tenets.
- [path-conventions.md](../shared/path-conventions.md) — `spec/` vs `docs/` rules.
- [specscore-lint-rules.md](../shared/specscore-lint-rules.md) — lint contract this skill assumes.
- [events.md](../shared/events.md) — event payloads emitted by this skill.
- [question-cadence.md](../shared/question-cadence.md) — when to batch vs single-question.
- [rehearse-heuristic.md](../shared/rehearse-heuristic.md) — per-AC testability decision.
