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

Build a specstudio:triage skill that takes a GitHub issue URL, fetches content via the gh CLI, classifies it into {idea, proposal, issue, bug} using the LLM's reading of the issue title, body, labels, and selected comments, attributes it to an existing Feature by matching against spec/features/*/README.md summaries, and scaffolds the right SpecScore artifact with a confirm-before-write gate. For ideas: specscore idea new. For proposals: writes to spec/features/<feature>/proposals/<slug>.md (requires Feature attribution to succeed). For issues: spec/issues/<slug>.md (or Feature-scoped at spec/features/<feature>/issues/<slug>.md when attributed). For bugs: a forward-compatible stub at spec/bugs/<slug>.md with a minimal schema (type: bug, slug, status: reported, reported_at, reported_by, reproduction_steps, expected_behavior, actual_behavior, optional affected_component) that the future bug-artifact-and-rca Idea can adopt or refine — lint accepts the stub as opaque until the bug Feature ships. The skill always confirms the classification with the user before writing.

**Clarification loop.** When the issue lacks enough information to classify confidently or to scaffold a useful artifact — missing actor, unclear expected behavior, no reproduction steps for a bug, ambiguous scope — the skill does NOT guess. Instead it: (1) posts a clarifying comment on the GitHub issue asking specific, grounded questions (at most 3 per comment, derived from what's actually missing in the issue text, never generic); (2) adds a label (e.g., `specscore:awaiting-clarification`) so the maintainer and author can see the issue is pending; (3) records the pending state locally (a lightweight tracking file at `.specscore/triage/<slug>.json` with the issue URL, questions asked, timestamp, and current classification hypothesis). When the maintainer re-runs `/triage <url>` later — or when a future scheduled check detects a new comment — the skill fetches the updated issue, evaluates whether the reply resolves the open questions, and either scaffolds the artifact or posts follow-up questions. The loop terminates in one of three ways: (a) enough information → scaffold artifact, remove the label; (b) author doesn't respond after one follow-up → the skill stops (no nagging — matches the issue-to-spec.md posture of "do not repeat if the maintainer ignores the app"); (c) maintainer explicitly closes the triage (`/triage --close <url>`) → remove label, delete tracking file.

**Participant role awareness.** The skill distinguishes three roles in the issue thread using the GitHub API's `author_association` field: the **issue submitter** (the person who opened it — their replies are clarification input), **external users** (commenters with association NONE or CONTRIBUTOR — supplementary context, lower weight), and **maintainers** (OWNER, MEMBER, COLLABORATOR — their comments are authoritative triage context). A maintainer comment like "this is a duplicate of #12" or "this affects sidekick-capture" should directly influence classification and attribution. A submitter reply to a clarifying question provides the missing detail. External user comments are read but not treated as authoritative.

Comment posture follows the rules from the issue-to-spec.md initiative: ask at most 3 questions per comment, ask only questions grounded in the issue text, do not repeat if ignored, do not use marketing language. Good comment shape: "SpecScore can draft this into a [type] spec, but two details are missing: 1. Who is the actor? 2. What should happen when [specific edge case from issue]?"

A --batch flag ingests multiple issues (by label, milestone, or 'all open') and produces a triage table for bulk review — but batch is a v2 follow-on, not MVP.

## Alternatives Considered

**Classify-only (no artifact scaffolding).** The skill outputs classification + confidence + suggested command, but doesn't write anything. The user runs `specscore idea new` or `specscore issue new` manually. *Lost because:* the classification step is the easy part; the artifact scaffolding — extracting requirements from the issue body, writing acceptance criteria, populating source links — is where the real value lives. A classify-only skill saves 10 seconds of decision-making but leaves 10 minutes of artifact writing on the table.

**Extend the existing sidekick skill with an inbound mode.** Instead of a new skill, add `--from-url` to `specstudio:sidekick` so it can capture from a GitHub issue URL in addition to capturing from the user's flow. *Lost because:* the use cases are architecturally opposite. Sidekick is "I'm mid-flow, park this observation and keep going" — speed over accuracy, minimal interaction. Triage is "I'm looking at an external issue and deciding what it means for my spec graph" — accuracy over speed, confirm-before-write. Merging them into one skill creates a mode flag that changes the skill's entire interaction model.

**Defer to the GitHub App.** Skip the CLI/SpecStudio skill entirely and build the triage logic directly into the SpecScore GitHub App (the auto-triage surface described in issue-to-spec.md). *Lost because:* the App is a deployment surface that should consume triage logic, not define it. Building triage as a skill first means the logic can be tested locally, dogfooded on first-party repos, and validated before committing to an always-on automation surface. The App becomes a separate Idea that wraps the proven skill logic.

## MVP Scope

One vertical slice: a maintainer runs `/triage https://github.com/org/repo/issues/123` in Claude Code. The skill fetches the issue, classifies it (showing the reasoning: "This looks like a change request to the sidekick-capture feature because it describes a modification to existing behavior"), and either scaffolds the artifact (if the issue has enough information) or posts a clarifying comment on the GitHub issue, adds the `specscore:awaiting-clarification` label, and records the pending state locally. When the maintainer re-runs `/triage <url>` after a reply, the skill evaluates the updated content and either scaffolds or asks follow-up questions. On scaffold, the skill asks the user to confirm or override the classification and Feature attribution, then writes the artifact with the GitHub issue URL in the source-link metadata and removes the label. Success: 10 real issues from specscore-cli or specstudio-skills triaged end-to-end, with ≥80% classification accuracy before user correction, every output artifact passing specscore spec lint, and at least 3 issues going through the clarification loop (question posted → reply received → artifact scaffolded).

## Not Doing (and Why)

- GitHub App auto-triage — separate Idea that consumes the triage logic as a library
- Batch triage of multiple issues in one invocation — clean follow-on once single-issue triage is validated
- Non-GitHub trackers (Linear, Jira, Plane) — GitHub first per issue-to-spec.md initiative; the skill's input interface is generic enough to extend
- Bidirectional sync between the local artifact and the GitHub issue — write-only at triage time; the clarification comment is a one-way write to GitHub, not a sync mechanism
- Auto-classification without user confirmation — the confirm-before-write gate is non-negotiable for MVP
- Webhook-driven re-triage on new comments — MVP requires the maintainer to re-run `/triage <url>` manually; the GitHub App Idea handles automated re-evaluation on webhook events
- More than one follow-up comment if the author ignores the first — matches issue-to-spec.md posture: do not repeat if ignored
- Full bug-artifact-and-rca pipeline — the skill writes a forward-compatible stub; the lifecycle and RCA workflow belong to the sibling Idea
- Semantic search or embedding-based Feature attribution — keyword/summary matching for MVP; sophistication follows demand

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | LLM classification into the 4-type taxonomy (idea, proposal, issue, bug) is ≥80% accurate on real GitHub issues without fine-tuning or few-shot examples beyond the skill prompt. | Manually classify 20 real issues from specscore-cli and specstudio-skills. Run the skill. Compare. If accuracy is <80%, add few-shot examples to the skill prompt and re-test before adding more complexity. |
| Must-be-true | Feature attribution via keyword/summary matching against spec/features/*/README.md is accurate enough that proposals route to the right Feature ≥70% of the time. | Run against 10 issues in a repo with 5+ Features. Count correct Feature attributions. If <70%, the confirm-before-write gate catches errors, but the skill's value proposition weakens. |
| Must-be-true | The `gh` CLI can fetch issue content (title, body, labels, comments) in a single command or two, with latency ≤2s for a typical issue. | Benchmark `gh issue view <url> --json title,body,labels,comments` on 10 issues of varying size. If latency is consistently >2s, consider fetching only title+body and making comments opt-in. |
| Must-be-true | Clarifying comments posted by the skill are specific enough that issue authors respond with useful information ≥50% of the time, rather than ignoring or giving generic replies. | Post clarifying comments on 10 real issues. Track response rate and response usefulness (did the reply contain the missing info?). If <50% useful responses, the comment template needs sharper, more specific questions. |
| Should-be-true | The confirm-before-write UX feels like a useful checkpoint rather than friction. Users override the classification or attribution in ≤20% of cases; when they do, the override is quick (one selection, not a freeform reclassification). | Dogfood for one week. Track override rate and time-to-override. If override rate exceeds 40%, the classification logic needs improvement. If override takes >30s, the UX needs simplification. |
| Should-be-true | The forward-compatible bug stub (spec/bugs/<slug>.md) doesn't cause confusion or lint noise before the bug-artifact-and-rca Idea ships. Users understand it's a placeholder, not a fully specified artifact. | After 5 bug-type triages, check whether users try to transition the bug through a lifecycle that doesn't exist yet. If they do, add a clear "This is a stub — full bug lifecycle coming in bug-artifact-and-rca" note to the generated artifact. |
| Should-be-true | The proposal output path works even though the canonical proposal Feature spec doesn't exist yet — the forward-referenced schema from feature/README.md and plan/README.md is sufficient to scaffold a valid proposal artifact. | Scaffold 3 proposals via the skill, then lint. If lint rejects them because the proposal spec doesn't exist, the skill must fall back to writing an Idea with a "proposed change to Feature X" framing. |
| Should-be-true | The manual re-run model (`/triage <url>` again after a reply) is acceptable for MVP — maintainers don't mind checking back manually. | Dogfood for two weeks. If maintainers consistently forget to re-run and issues go stale with the `specscore:awaiting-clarification` label, the webhook-driven re-evaluation (GitHub App scope) becomes urgent. |
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
- **Clarification loop state persistence.** The tracking file at `.specscore/triage/<slug>.json` needs to survive across sessions. Should it be git-tracked (visible to teammates but noisy in diffs) or gitignored (local-only, lost if the user switches machines)? Lean: gitignored — it's transient triage state, not a spec artifact. The GitHub issue label is the durable cross-machine signal.
- **Label creation permissions.** The skill needs to create the `specscore:awaiting-clarification` label if it doesn't exist. Does it auto-create on first use (requires label-create permission on the repo) or require the maintainer to pre-create it? Lean: auto-create with a sensible color and description; fail gracefully if permissions are insufficient (the comment still posts, only the label is skipped).
- **Who removes the label?** When the skill scaffolds the artifact after a reply, it removes the label automatically. But if the author replies directly without the maintainer re-running `/triage`, the label stays stale. Is this acceptable? Lean: yes for MVP — the webhook-driven GitHub App is the right fix for stale labels.
- **Participant role awareness.** The skill should distinguish between the issue submitter, external users commenting, and repo maintainers in the comment thread. Maintainer comments may contain triage context ("this is a duplicate of #12", "this affects the sidekick-capture feature") that should influence classification differently from submitter clarifications. How to detect roles: `gh` API exposes `author_association` on comments (OWNER, MEMBER, COLLABORATOR, CONTRIBUTOR, NONE). The skill should weight maintainer comments as authoritative context and submitter comments as clarification input.

---
*This document follows the https://specscore.md/idea-specification*
