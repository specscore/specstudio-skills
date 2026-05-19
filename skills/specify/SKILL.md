---
name: specify
description: |
  Turns an approved SpecScore Idea (or a clear buildable intent) into a
  SpecScore Feature with requirements and Given/When/Then acceptance
  criteria at spec/features/<slug>/. Gates implementation — no code,
  plans, or scaffolding until the Feature is lint-clean and
  user-approved. Optionally scaffolds Rehearse test stubs.
  Trigger: "specify this", "/specify", "spec this out", or Synchestra
  event `idea.approved`.
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
  4. The spec-document reviewer subagent returned `Approved`.
  5. The user has explicitly approved the written Feature.

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
11. **Dispatch spec-document-reviewer subagent** — see [reviewer-prompt.md](references/reviewer-prompt.md). Must return `Approved` before user review.
12. **User review gate** — user reviews the written Feature; wait for approval.
13. **Emit events** — `feature.specified` on reviewer approval; `feature.approved` on user approval. See [synchestra-events.md](../shared/synchestra-events.md).
14. **Transition to `writing-plans`.**
15. **Throughout** — watch for sidekick ideas per [sidekick-capture.md](../shared/sidekick-capture.md). When an out-of-scope improvement surfaces, invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step immediately. Do not derail to discuss the sideline idea.

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

## Reviewer Subagent

Dispatch the **built-in baseline reviewer** using [reviewer-prompt.md](references/reviewer-prompt.md). It enforces the six baseline blocker categories (scope spans subsystems / unobservable Then / AC coverage gap / architecture↔requirements contradiction / vague REQ / missing source-Idea reasoning). Status must be `Approved` before the user review gate.

**Additional registered reviewers (third-party).** Read `specscore.yaml` at the repo root. If it contains a top-level `reviewers:` extension key (per the [`third-party-integration`](../../spec/features/third-party-integration/README.md) Feature's `reviewer-registration-mechanism` REQ), dispatch each registered reviewer **in addition to** the baseline:

```yaml
# Example shape — see third-party-integration Feature for the canonical contract
reviewers:
  - name: security-baseline
    prompt: spec/reviewers/security-baseline/prompt.md
    description: Enforces auth/data-handling/secret-management blockers
  - name: ux-accessibility
    prompt: spec/reviewers/ux-accessibility/prompt.md
```

For each entry: read the `prompt` file (which MUST be a path inside the repo working tree per `reviewer-prompt-location`), confirm it contains an explicit blocker/advisory taxonomy section (per `reviewer-contract` and `AC: reviewer-registration-and-composition`), and dispatch the reviewer using the same Agent-tool dispatch pattern as the baseline. A reviewer prompt missing a documented taxonomy fails the reviewer registration AC — surface this to the user as a registry error, do not silently skip the reviewer.

**Composition is AND.** The User Review Gate releases only when **every** registered reviewer plus the baseline returns `Approved`. Any single `Issues Found` from any reviewer blocks the gate. On `Issues Found`:

1. Address every blocker-severity finding from every failing reviewer.
2. Re-dispatch every reviewer that previously returned `Issues Found`.
3. Re-dispatch reviewers that previously returned `Approved` when the fix changes the Feature's structural sections (Behavior, Architecture, Acceptance Criteria) — `reviewer-composition` makes this MUST for structural changes, SHOULD for REQ/AC changes covered by previously-Approved reviewers.

Advisory findings MAY be ignored.

The skill MUST NOT silently downgrade a blocker to advisory severity, MUST NOT skip a registered reviewer, and MUST NOT proceed past the User Review Gate while any reviewer's last verdict is `Issues Found`.

When `specscore.yaml` does not contain a `reviewers:` key, only the built-in baseline reviewer runs — the absence of the key is not an error.

## User Review Gate

After reviewer subagent returns `Approved`:

> "Feature written and lint-clean at `spec/features/<slug>/`. Reviewer subagent approved. Please review and let me know if you want changes before we move to `writing-plans`."

Wait. If the user requests changes, fix, re-lint, re-review, re-gate. Only proceed once the user approves.

On approval:
- Update `**Status:** Under Review → Approved` in the body metadata.
- Re-run lint.
- Emit `feature.approved` event.

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
- [ ] Reviewer subagent returned `Approved`
- [ ] User explicitly approved the written Feature
- [ ] `**Status:** Approved` in body metadata
- [ ] Rehearse decision recorded (stubs scaffolded OR skip-reason noted)
- [ ] Source Idea (if any) linked via the `**Source Ideas:**` body-metadata line — Synchestra handles the reverse link
- [ ] `feature.specified` + `feature.approved` events emitted

## Red Flags

- Proceeding to `writing-plans` without user approval
- Requirements without ACs
- ACs not in `Given / When / Then`
- "Too simple to spec" rationalization
- Scope spanning multiple subsystems
- Assumptions from the source Idea silently dropped
- Writing to `docs/superpowers/specs/` instead of `spec/features/<slug>/`
- Skipping the reviewer subagent
- Invoking any skill other than `writing-plans` on transition

## References

- [reviewer-prompt.md](references/reviewer-prompt.md) — spec-document reviewer subagent template.
- [visual-companion.md](references/visual-companion.md) — visual companion strategy (decision pending).
- [philosophy.md](../shared/philosophy.md) — shared tenets.
- [path-conventions.md](../shared/path-conventions.md) — `spec/` vs `docs/` rules.
- [specscore-lint-rules.md](../shared/specscore-lint-rules.md) — lint contract this skill assumes.
- [synchestra-events.md](../shared/synchestra-events.md) — event payloads emitted by this skill.
- [question-cadence.md](../shared/question-cadence.md) — when to batch vs single-question.
- [rehearse-heuristic.md](../shared/rehearse-heuristic.md) — per-AC testability decision.
