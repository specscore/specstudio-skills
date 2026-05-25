# Idea: Synchestra Removal — Follow-up

**Status:** Draft
**Date:** 2026-05-25
**Owner:** alexander.trakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we close out the remaining ~330 lines of synchestra references in specstudio-skills — concentrated in competitor research, large sidekick-consilium Plans, and AC text inside deeply-integrated Features — without forcing a mechanical scrub through content that is either historical audit trail or requires a Feature redesign?

## Context

The 2026-05-24 `synchestra-removal` Idea shipped 11 commits across specscore-cli and specstudio-skills, scrubbing the active runtime surfaces (READMEs, all skills/*/SKILL.md, the events.md contract document, GitHub-org URLs, the winget identifier, the init skill and its source Idea + Feature). The remaining ~330 lines of synchestra references in specstudio-skills clustered into three distinct categories during that work, and each category warrants a different treatment that doesn't fit the original Idea's mechanical-scrub framing: (1) competitor research at `spec/research/competitors/*` (~150 refs across 4 files) and `spec/research/ideate-vs-brainstorming-skills-analysis.md` (~15 refs) — these DESCRIBE Synchestra as a system in head-to-head comparison; scrubbing breaks the analysis and erases the comparison's value, matching the audit-trail logic that preserved `docs/superpowers/2026-04-01-specscore-synchestra-restructuring*.md` in specscore-cli. (2) Two large implementation Plans — `spec/plans/sidekick-consilium.md` (19 refs across 2104 lines) and `spec/plans/sidekick-consilium-task-companion.md` — document executed implementation that integrated synchestra task lifecycle; they are now historical records of work that was done, similar to the preserved 2026-04-01 docs. (3) AC text inside two active Features — `spec/features/skills/init/README.md` (~15 refs in AC bodies) and `spec/features/sidekick-consilium/README.md` (~8 refs) — describes synchestra integration in concrete behavioral detail; the surrounding REQs and the corresponding SKILL.md files have been scrubbed, but the AC text was not because rewriting requires DESIGN decisions about post-synchestra behavior (does sidekick-consilium still create tasks? what task store?) that are out of brand-scrub scope. Plus a handful of smaller Ideas (`sidekick-ideas` 9 refs, `sidekick-consilium` 7, `sidekick-issue-tracker-destinations` 6) that warrant case-by-case decisions.

## Recommended Direction

Split the follow-up into two workstreams, each cleanly scoped: (1) **Preservation declarations** — extend the `synchestra-removal` Idea's preservation list to explicitly cover `spec/research/competitors/*`, `spec/research/ideate-vs-brainstorming-skills-analysis.md`, and `spec/plans/sidekick-consilium*.md`. The Idea text gets an addendum naming these files as audit-trail artifacts (matching the `docs/superpowers/2026-04-01-*` precedent). No content edits to those files. (2) **Feature redesign work** — for the two deeply-integrated Features (`cli/skills/init` and `sidekick-consilium`), draft separate redesign Ideas that answer the genuine design questions: For init, the synchestra orchestration step is dead — should the wizard offer ANY orchestration option, or is init now strictly specscore-only? For sidekick-consilium, the consilium currently drains `consilium-review` tasks from the synchestra task system — does it pivot to a new task store (e.g., a file-backed queue in `.specscore/consilium-queue/`)? does it merge with the seed file as the source of truth? does it defer until a new orchestrator surfaces? Each redesign Idea then promotes to a Feature revision and a Plan, following the standard SDD lifecycle. The smaller Ideas (`sidekick-ideas`, `sidekick-consilium`, `sidekick-issue-tracker-destinations`) get a single batched scrub pass once the redesign decisions clarify what their content should reflect — out of scope for this Idea's MVP.

## Alternatives Considered

- **Mass-scrub the remaining 330 lines with sed and accept the awkward prose.** Loses: the competitor research docs become incoherent (you can't compare against a system that's never named); the implementation Plans lose their historical referent; the AC text in the init and sidekick-consilium Features becomes nonsensical ("when the synchestra task is created" → "when the task is created" — what task? created by what?). The whole point of those passages is the specific system they describe. Scrubbing them mechanically destroys their value without producing a coherent replacement.
- **Do the init and sidekick-consilium Feature redesigns inside this Idea.** Loses: scope explosion. The redesigns are real spec work — what does init do without the orchestration step? does sidekick-consilium still create tasks at all, and if so against what backing store? Each is its own Idea → Feature → Plan cycle. Bundling them into this Idea conflates the "close the brand scrub" framing with substantial design work and leaves both half-done.
- **Declare the remaining files preserved-by-default and never touch them.** Loses: the AC text in the init and sidekick-consilium Features is genuinely stale — the SKILL.md files and surrounding REQs have already been rewritten to drop synchestra; only the AC bodies still describe behavior the skill no longer implements. Leaving them indefinitely creates spec-vs-implementation drift, which is a worse failure mode than acknowledging the redesign work needs to happen.

## MVP Scope

One commit per workstream: (a) a content edit to `spec/ideas/synchestra-removal.md` adding the explicit preservation list extension (`spec/research/*`, `spec/plans/sidekick-consilium*.md`) under the existing "What's deliberately preserved" clause. (b) Two scaffolded follow-up Ideas at `spec/ideas/init-skill-post-synchestra.md` and `spec/ideas/sidekick-consilium-post-synchestra.md` — title + HMW + brief context pointing at the redesign questions, status Draft. No Feature work, no Plan work, no Feature redesigns in THIS Idea's MVP. Timeboxed: ship within the working session that approves this Idea.

## Not Doing (and Why)

- Actually performing the init Feature redesign — captured as a follow-up Idea; the redesign itself is its own SDD lifecycle
- Actually performing the sidekick-consilium Feature redesign — captured as a follow-up Idea; same reasoning
- Scrubbing the competitor research docs and Plans — explicitly preserved as audit-trail per workstream 1
- Scrubbing the smaller Ideas (sidekick-ideas, sidekick-consilium, sidekick-issue-tracker-destinations) — deferred to a single batched pass after the two redesign Ideas settle, since their content references the systems being redesigned
- Any code or runtime behavior change in either specscore-cli or specstudio-skills — this Idea is purely about closing the documentation scrub
- Other workspace repos (specscore, specscore-studio, ai-plugin-specscore, etc.) — still out of scope per the original synchestra-removal Idea

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | Extending the `synchestra-removal` Idea's preservation list to include `spec/research/competitors/*` and `spec/plans/sidekick-consilium*.md` does not contradict any lint rule or downstream consumer expectation. | Audit: confirm `specscore spec lint` still passes after the preservation-list addendum (the addendum is prose, not metadata, so no lint rule should care). Confirm no Synchestra-aware consumer (Hub, etc.) parses the preservation list for routing decisions. |
| Must-be-true | The init and sidekick-consilium Features can have follow-up redesign Ideas drafted without the redesigns themselves being in this Idea's scope. | This Idea ships two Draft Ideas at known paths with HMW + brief context; the actual redesign happens in subsequent SDD lifecycle cycles per each new Idea. Acceptance: this Idea's MVP definition-of-done references only the two scaffolded Ideas, never a Feature or Plan revision. |
| Should-be-true | The smaller Ideas (`sidekick-ideas`, `sidekick-consilium`, `sidekick-issue-tracker-destinations`) genuinely benefit from waiting for the two redesign Ideas to settle before being scrubbed — their content references the same systems that are being redesigned. | At the time the redesign Ideas reach Approved, re-survey the smaller Ideas. If the redesigns have clarified the post-synchestra architecture for the relevant concepts, the scrub becomes a mechanical pass; if not, the smaller Ideas need their own per-file judgment, which costs more than waiting. |
| Should-be-true | The `synchestra-removal` Idea's existing preservation clause can be extended via a content edit (an `idea.updated` emission) without re-running its approval gate. | Per the ideate skill's "Post-Approval Iteration" rules, the Idea is alive but not frozen — content edits emit `idea.updated`, no re-approval needed. |
| Might-be-true | The two redesign Ideas drafted by this Idea's MVP will be promoted to Approved status soon enough that the smaller-Ideas batched scrub doesn't sit indefinitely. | Defer; check in a quarter. If the redesigns are still Draft after a month, escalate the smaller-Ideas scrub to a separate Idea so it isn't blocked. |


## SpecScore Integration

- **New Features this would create:** None directly. This Idea's MVP creates two **scaffolded Draft Ideas** (`spec/ideas/init-skill-post-synchestra.md`, `spec/ideas/sidekick-consilium-post-synchestra.md`); each of those Ideas, once approved, may promote to its own Feature(s). This Idea is a coordination Idea, not a delivery Idea.
- **Existing Features affected:** `spec/features/skills/init/README.md` and `spec/features/sidekick-consilium/README.md` carry stale AC text describing synchestra integration; the redesign Ideas this Idea scaffolds will eventually amend those Features. No edits to those Features happen in THIS Idea's MVP.
- **Dependencies:** The original [`synchestra-removal`](synchestra-removal.md) Idea must be Approved (it currently is). The preservation-list extension is an `idea.updated` to that Idea, not a new Idea.

## Open Questions

- **Should the two scaffolded redesign Ideas declare each other as a `**Related Ideas:**` cross-link?** init's redesign and sidekick-consilium's redesign are independent in scope but share the post-synchestra architecture context. Cross-linking lets a reader landing on one find the other. Default: yes, cross-link bidirectionally — settle when scaffolding.
- **Naming of the redesign Ideas.** `init-skill-post-synchestra` / `sidekick-consilium-post-synchestra` is descriptive but verbose; alternatives like `init-orchestration-cutover` / `sidekick-consilium-task-store-pivot` are tighter but less searchable. Settle at scaffold time; ASCII-slug constraints apply either way.
- **Whether to also extend the preservation list to `docs/superpowers/plans/2026-04-22-specscore-cli-decoupling.md`.** That file was preserved by judgment in specscore-cli (it documents Synchestra Hub URL rewriting as part of cli decoupling work) but isn't explicitly named in the original `synchestra-removal` Idea's Not Doing list. Confirm preservation status as part of the addendum.

---
*This document follows the https://specscore.md/idea-specification*
