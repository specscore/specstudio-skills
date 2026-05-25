# Idea: Triage Issue/Ticket

**Status:** Draft
**Date:** 2026-05-25
**Owner:** alexandertrakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** extends:sidekick-issue-tracker-destinations

## Problem Statement

How might we give SpecStudio a skill that takes a GitHub issue URL, classifies it into the SpecScore artifact taxonomy (idea, proposal, issue, or bug), attributes it to an existing Feature when detectable, and scaffolds the right artifact — so a maintainer's triage workflow produces spec-graph-integrated artifacts instead of mental bookkeeping?

## Context

Today maintainers triage GitHub issues mentally: they read the issue, decide whether it is a feature request, a change to existing behavior, a problem report, or a diagnosed bug, and then manually create the corresponding artifact (if they create one at all). The issue-to-spec.md marketing initiative envisions converting GitHub issues into draft SpecScore features (Flow 3: /specstudio:from-issue), but that covers only the idea→feature path. The sidekick-issue-tracker-destinations Idea (Implementing) handles the outbound direction — capturing issues from within a host skill to GitHub. This Idea is the inbound complement: GitHub issue in, SpecScore artifact out. The four output types map to existing or in-flight SpecScore primitives: Idea (exists), Proposal/change-request (forward-referenced from feature/plan specs, canonical Feature spec in Draft via proposals-and-feature-phases Idea), Issue (shipping via issue-artifact-type Feature, Approved), and Bug (deferred to bug-artifact-and-rca sibling Idea — this skill creates a forward-compatible stub). The GitHub App auto-triage surface is explicitly a separate Idea.

## Recommended Direction

Build a specstudio:triage skill that takes a GitHub issue URL, fetches content via the gh CLI, classifies it into {idea, proposal, issue, bug} using the LLM's reading of the issue title, body, labels, and selected comments, attributes it to an existing Feature by matching against spec/features/*/README.md summaries, and scaffolds the right SpecScore artifact with a confirm-before-write gate. For ideas: specscore idea new. For proposals: writes to spec/features/<feature>/proposals/<slug>.md (requires Feature attribution to succeed). For issues: spec/issues/<slug>.md (or Feature-scoped at spec/features/<feature>/issues/<slug>.md when attributed). For bugs: a forward-compatible stub at spec/bugs/<slug>.md with a minimal schema (type: bug, slug, status: reported, reported_at, reported_by, reproduction_steps, expected_behavior, actual_behavior, optional affected_component) that the future bug-artifact-and-rca Idea can adopt or refine — lint accepts the stub as opaque until the bug Feature ships. The skill always confirms the classification with the user before writing. A --batch flag ingests multiple issues (by label, milestone, or 'all open') and produces a triage table for bulk review — but batch is a v2 follow-on, not MVP.

## Alternatives Considered

**Classify-only (no artifact scaffolding).** The skill outputs classification + confidence + suggested command, but doesn't write anything. The user runs `specscore idea new` or `specscore issue new` manually. *Lost because:* the classification step is the easy part; the artifact scaffolding — extracting requirements from the issue body, writing acceptance criteria, populating source links — is where the real value lives. A classify-only skill saves 10 seconds of decision-making but leaves 10 minutes of artifact writing on the table.

**Extend the existing sidekick skill with an inbound mode.** Instead of a new skill, add `--from-url` to `specstudio:sidekick` so it can capture from a GitHub issue URL in addition to capturing from the user's flow. *Lost because:* the use cases are architecturally opposite. Sidekick is "I'm mid-flow, park this observation and keep going" — speed over accuracy, minimal interaction. Triage is "I'm looking at an external issue and deciding what it means for my spec graph" — accuracy over speed, confirm-before-write. Merging them into one skill creates a mode flag that changes the skill's entire interaction model.

**Defer to the GitHub App.** Skip the CLI/SpecStudio skill entirely and build the triage logic directly into the SpecScore GitHub App (the auto-triage surface described in issue-to-spec.md). *Lost because:* the App is a deployment surface that should consume triage logic, not define it. Building triage as a skill first means the logic can be tested locally, dogfooded on first-party repos, and validated before committing to an always-on automation surface. The App becomes a separate Idea that wraps the proven skill logic.

## MVP Scope

One vertical slice: a maintainer runs /triage https://github.com/org/repo/issues/123 in Claude Code. The skill fetches the issue, classifies it (showing the reasoning: 'This looks like a change request to the sidekick-capture feature because it describes a modification to existing behavior'), asks the user to confirm or override the classification and Feature attribution, then scaffolds the right artifact with the GitHub issue URL in the source-link metadata. Success: 10 real issues from specscore-cli or specstudio-skills triaged end-to-end, with ≥80% classification accuracy before user correction, and every output artifact passing specscore spec lint.

## Not Doing (and Why)

- GitHub App auto-triage — separate Idea that consumes the triage logic as a library
- Batch triage of multiple issues in one invocation — clean follow-on once single-issue triage is validated
- Non-GitHub trackers (Linear, Jira, Plane) — GitHub first per issue-to-spec.md initiative; the skill's input interface is generic enough to extend
- Bidirectional sync between the local artifact and the GitHub issue — write-only at triage time; sync is the GitHub App's job
- Auto-classification without user confirmation — the confirm-before-write gate is non-negotiable for MVP
- Full bug-artifact-and-rca pipeline — the skill writes a forward-compatible stub; the lifecycle and RCA workflow belong to the sibling Idea
- Semantic search or embedding-based Feature attribution — keyword/summary matching for MVP; sophistication follows demand

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | LLM classification into the 4-type taxonomy (idea, proposal, issue, bug) is ≥80% accurate on real GitHub issues without fine-tuning or few-shot examples beyond the skill prompt. | Manually classify 20 real issues from specscore-cli and specstudio-skills. Run the skill. Compare. If accuracy is <80%, add few-shot examples to the skill prompt and re-test before adding more complexity. |
| Must-be-true | Feature attribution via keyword/summary matching against spec/features/*/README.md is accurate enough that proposals route to the right Feature ≥70% of the time. | Run against 10 issues in a repo with 5+ Features. Count correct Feature attributions. If <70%, the confirm-before-write gate catches errors, but the skill's value proposition weakens. |
| Must-be-true | The `gh` CLI can fetch issue content (title, body, labels, comments) in a single command or two, with latency ≤2s for a typical issue. | Benchmark `gh issue view <url> --json title,body,labels,comments` on 10 issues of varying size. If latency is consistently >2s, consider fetching only title+body and making comments opt-in. |
| Should-be-true | The confirm-before-write UX feels like a useful checkpoint rather than friction. Users override the classification or attribution in ≤20% of cases; when they do, the override is quick (one selection, not a freeform reclassification). | Dogfood for one week. Track override rate and time-to-override. If override rate exceeds 40%, the classification logic needs improvement. If override takes >30s, the UX needs simplification. |
| Should-be-true | The forward-compatible bug stub (spec/bugs/<slug>.md) doesn't cause confusion or lint noise before the bug-artifact-and-rca Idea ships. Users understand it's a placeholder, not a fully specified artifact. | After 5 bug-type triages, check whether users try to transition the bug through a lifecycle that doesn't exist yet. If they do, add a clear "This is a stub — full bug lifecycle coming in bug-artifact-and-rca" note to the generated artifact. |
| Should-be-true | The proposal output path works even though the canonical proposal Feature spec doesn't exist yet — the forward-referenced schema from feature/README.md and plan/README.md is sufficient to scaffold a valid proposal artifact. | Scaffold 3 proposals via the skill, then lint. If lint rejects them because the proposal spec doesn't exist, the skill must fall back to writing an Idea with a "proposed change to Feature X" framing. |
| Might-be-true | Batch triage (triaging multiple issues in one invocation) is worth building in v2. | Wait for the demand signal: if users request it after using single-issue triage for 2+ weeks, build it. |


## SpecScore Integration

- **New Features this would create:**
  - `triage-skill` — the SpecStudio skill itself: input parsing, GitHub issue fetching, 4-way classification logic, Feature attribution, confirm-before-write UX, and artifact scaffolding dispatch.
  - `bug-artifact-stub` — the minimal bug schema at `spec/bugs/<slug>.md` that the triage skill writes. Forward-compatible surface for the deferred `bug-artifact-and-rca` Idea. Lint accepts it as opaque until the full bug Feature ships.
- **Existing Features affected:**
  - `issue-artifact-type` — the triage skill becomes a producer of issue artifacts. No schema changes; the skill uses the existing issue scaffold path.
  - `sidekick-capture` — conceptually adjacent but not modified. The triage skill is a sibling entry point, not an extension of sidekick.
  - `specscore spec lint` — gains awareness of `spec/bugs/<slug>.md` as an opaque artifact (no validation beyond file presence and frontmatter shape).
- **Dependencies:**
  - `issue-artifact-type` Feature (Approved) — the skill routes issues to `spec/issues/`. If the Feature hasn't shipped to the CLI yet, the skill falls back to writing the issue file directly (matching the schema).
  - `proposals-and-feature-phases` Idea (Draft) — the skill routes proposals to `spec/features/<feature>/proposals/`. If the proposal spec isn't yet canonical, the skill writes the file matching the forward-referenced schema from feature/plan specs.
  - `gh` CLI authenticated on the developer's machine — required for fetching issue content. Clean error if not available.
  - `bug-artifact-and-rca` Idea (not yet drafted) — the bug stub schema must be forward-compatible with whatever this sibling Idea defines. Risk: the sibling Idea may choose a different schema, requiring a migration of early stubs.

## Open Questions

- **Bug stub schema stability.** The triage skill writes `spec/bugs/<slug>.md` with a minimal schema before the `bug-artifact-and-rca` Idea defines the canonical bug artifact. What happens if the sibling Idea chooses a significantly different schema? Lean: keep the stub minimal enough (type, slug, status, repro steps, expected/actual) that any future schema is a superset, not a break.
- **Proposal scaffolding without a canonical spec.** The `proposals-and-feature-phases` Idea is Draft. Can the triage skill confidently write proposals using the forward-referenced schema from feature/plan specs, or should it fall back to writing an Idea with "proposed change to Feature X" framing until the proposal spec ships? Lean: write the proposal directly — the forward-referenced schema is stable enough and the skill includes source links for traceability.
- **Feature attribution failure path.** When the skill can't attribute an issue to any Feature (e.g., cross-cutting concern, new area), what happens to proposals and Feature-scoped issues? Proposals require a Feature target. Lean: if attribution fails and the type is proposal, ask the user to specify the Feature manually; if they can't, reclassify as an Idea (because it's proposing something that doesn't exist yet).
- **Comment selection.** GitHub issues can have dozens of comments. Should the skill (a) fetch all comments, (b) fetch only the first N, or (c) fetch all but let the LLM decide which are relevant? Lean: fetch all (via `gh issue view --json comments`), pass them to the LLM, let it extract relevant context. Cap at 50 comments for token budget.
- **Skill trigger name.** `/triage`, `/specstudio:triage`, or `/specstudio:from-issue` (matching the issue-to-spec.md initiative's naming)? Lean: `/triage` as the primary trigger with `specstudio:triage` as the qualified name. `/from-issue` is a valid alias since that's how the initiative names it.
- **Multiple issues referencing the same problem.** If a user triages issue #45 and it's a duplicate of issue #12 which was already triaged, should the skill detect this? Lean: not in MVP. The user will notice when the scaffolded artifact has a similar slug to an existing one.

---
*This document follows the https://specscore.md/idea-specification*
