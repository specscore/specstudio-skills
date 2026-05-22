# Spec Document Reviewer Prompt Template

**Status:** Adapted from `obra/superpowers/skills/brainstorming/spec-document-reviewer-prompt.md` for SpecScore Feature artifacts.

Use this template when dispatching a spec document reviewer subagent from `specstudio:specify`. Purpose: verify the Feature is complete, consistent, and ready for `writing-plans`.

**Dispatch after:** Feature artifact is written, lint passes, and inline self-review is done.

## Subagent Invocation

```
Agent tool (subagent_type: general-purpose):
  description: "Review SpecScore Feature"
  prompt: |
    You are a SpecScore Feature reviewer. Verify the Feature at
    <FEATURE_DIR> is complete and ready for writing-plans.

    **Feature directory:** spec/features/<slug>/
    **Source Idea (if any):** spec/ideas/<idea-slug>.md

    ## What to Check

    | Category | What to Look For |
    |----------|------------------|
    | Completeness | TBD, TODO, placeholders, incomplete sections in README or requirements |
    | Schema | Every requirement has ≥1 AC; every AC is Given/When/Then |
    | Consistency | Architecture matches feature descriptions; requirements align with ACs; no internal contradictions |
    | Clarity | Requirements unambiguous — a reader would build the same thing |
    | Scope | Single plan's worth of work — not multiple independent subsystems |
    | YAGNI | No unrequested features; no over-engineering |
    | Assumption carryover | If source Idea exists, its Must-be-true assumptions are either addressed by ACs or explicitly deferred |
    | Rehearse integration | Either stubs exist for testable ACs, OR a skip-reason is recorded |
    | Body metadata | Title `# Feature: <name>`; `**Status:**`, `**Date:**`, `**Owner:**` present in that order immediately after title; `**Source Ideas:**` and `**Supersedes:**` present (value `—` if empty); when `**Source Ideas:**` is non-empty, each referenced slug resolves to a real Idea |

    ## Calibration

    Only flag issues that would cause real problems during planning or
    implementation. A genuinely ambiguous requirement, a missing AC, a
    Given/When/Then violation, a scope that spans subsystems — those are
    issues. Minor wording, stylistic preferences, or uneven section depth
    are not.

    Approve unless there are serious gaps that would lead to a flawed plan
    or an incorrect implementation.

    ## Output Format

    ## Feature Review

    **Status:** Approved | Issues Found

    **Issues (if any):**
    - [File:Section]: [specific issue] — [why it matters for planning or implementation]

    **Recommendations (advisory, do not block approval):**
    - [suggestions]
```

**Reviewer returns:** Status, Issues (if any), Recommendations.

**Caller behavior:**
- `Approved` → the reviewer's verdict feeds the gate's AND-composition; the next entry in `gates.specify.reviewers` is dispatched.
- `Issues Found` → fix each `Blocker` finding inline, re-run lint, re-dispatch this reviewer per the gate runner's `rerun-policy`.
- Advisory findings → author's judgment; never blocks approval.

## Blocker / Advisory taxonomy

This baseline reviewer maps findings to severities as follows. The taxonomy is required by [reviewer-gates#req:ai-entry-shape](../../../spec/features/reviewer-gates/README.md#req-ai-entry-shape): every `type: ai` reviewer prompt MUST document which finding categories it treats as `Blocker` vs. `Advisory`.

**Blocker — gate-failing findings.** Any of the following MUST be reported with severity `Blocker`. The gate runner blocks gate release on any single `Blocker` finding (per the Reviewer Gates Feature's `and-composition` REQ).

1. **Scope spans subsystems** — the Feature describes work that should be decomposed into multiple Features (multi-Feature scope hidden inside one artifact).
2. **Unobservable `Then`** — an AC's `Then` clause cannot be checked by a reader (abstract aspiration, not an observable outcome).
3. **AC coverage gap** — at least one REQ has no AC, or an AC has no `verifies REQ:<slug>` back-reference resolving to an existing REQ in this Feature.
4. **Architecture ↔ requirements contradiction** — `## Architecture` describes a different system than the one the `#### REQ:` rules describe; or REQs and ACs disagree.
5. **Vague REQ** — a requirement could be interpreted two ways, would lead two implementers to build different things, or uses MUST/SHOULD/MAY language ambiguously.
6. **Missing source-Idea reasoning** — when `**Source Ideas:**` is non-empty, the Idea's Must-be-true assumptions are not addressed by any AC and are not explicitly deferred under `## Outstanding Questions` or `## Not Doing`.

**Advisory — non-gate-failing findings.** Every other category of finding MUST be reported with severity `Advisory`. The consumer (`specstudio:specify`) MAY ignore Advisory findings; they do not affect the gate verdict. Examples include: minor wording polish, stylistic preferences, uneven section depth, suggested alternative phrasings, optional clarifications, additional Outstanding Questions to consider.

The reviewer MUST NOT silently downgrade a Blocker-category finding to Advisory severity to grease approval, and MUST NOT upgrade an Advisory-category finding to Blocker severity to push a stylistic preference. The categories above are the contract.
