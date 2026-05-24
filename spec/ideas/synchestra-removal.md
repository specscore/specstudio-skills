# Idea: Remove "Synchestra" from SpecScore Repos

**Status:** Approved
**Date:** 2026-05-24
**Owner:** alexander.trakhimenok
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we remove every reference to "synchestra" from the specscore-cli and specstudio-skills repos — covering GitHub-org URLs, the `synchestra-events.md` filename and its content, concept references to the workflow orchestrator, and the in-flight winget rebrand — without breaking install instructions, event emission, or referrer integrity?

## Context

A meta-decision was made to remove the Synchestra brand from SpecScore: the GitHub org was renamed `synchestra-io → specscore` (repos already migrated to the new org), the winget rebrand `Synchestra.SpecScore → SpecScore.CLI` was started on 2026-05-19 (per the `.goreleaser.yml` cutover comment), and a one-off scrub of a single Idea (`event-emit-dispatcher`) was done on 2026-05-22 as part of that day's work. The empty seed `outline-idea-of-synchestra-removal` was captured 2026-05-22 marking the broader intent. Today the doc/spec surface across both repos still carries roughly 1,225 occurrences of "synchestra" / "Synchestra" / "synchestra-io" across 60+ markdown files plus one filename rename, and the inconsistency now actively misleads readers: install URLs look stale, event-emission docs reference a tool that's no longer in scope, and the cross-repo event contract document uses a name (`synchestra-events.md`) that no longer matches the project's identity.

## Recommended Direction

Treat the scrub as four mechanical workstreams executed in one Plan per repo: (1) **GitHub org URLs** — find/replace `synchestra-io` → `specscore` in all markdown and YAML across both repos. GitHub's automatic redirect keeps any third-party-held old URLs working; we only own what's in our trees. (2) **File rename** — `specstudio-skills/skills/shared/synchestra-events.md` → `skills/shared/events.md`, then sweep every referrer in both repos (the doc is referenced from `specstudio:ideate`, `specstudio:specify`, `specstudio:sidekick`, `specstudio:consilium`, `specstudio:plan`, plus shared docs and several Features). (3) **Concept-noun scrub** — replace prose mentions of "Synchestra" (the workflow orchestrator) with role-based phrasing ("workflow orchestrator", "downstream automation", "SDD event consumer") or simply drop the attribution where the sentence reads cleanly without it. The cli/lifecycle-transitions Feature explicitly mentions "the architectural positioning vs Synchestra" — that contrast loses its referent once Synchestra is no longer named; rewrite to describe the contract on its own terms. (4) **Winget identifier** — finish the `.goreleaser.yml` cutover by removing residual `Synchestra.SpecScore` comments and confirming the manifest publishes only as `SpecScore.CLI`. **What's deliberately preserved**: the historical rebrand docs at `docs/superpowers/2026-04-01-specscore-synchestra-restructuring*.md` stay verbatim because they ARE the record of why the name changed — scrubbing them would erase the audit trail. The seed `outline-idea-of-synchestra-removal.md` stays through the consilium phase per Phase 1 policy. Other repos in the workspace (`specscore`, `specscore-studio`, `ai-plugin-specscore`, `ai-marketplace`, the homebrew/scoop taps) are explicitly out of MVP — listed as Outstanding Questions for a follow-up sweep.

## Alternatives Considered

- **Do nothing, accept brand drift.** Loses: leaves 1,225 stale brand references in user-facing docs across two repos. Every new reader encounters install URLs that don't reflect the real org, event-emission docs that name a tool no longer in scope, and a shared contract file (`synchestra-events.md`) whose name disagrees with its content. Drift compounds — every new Feature inherits the old vocabulary.
- **Big-bang sed `s/synchestra/specscore/gi` across both repos.** Loses: corrupts edge cases. The winget identifier is `Synchestra.SpecScore → SpecScore.CLI` (NOT `SpecScore.SpecScore`); the historical rebrand docs at `docs/superpowers/2026-04-01-specscore-synchestra-restructuring*.md` deliberately preserve the original brand for audit; the seed `outline-idea-of-synchestra-removal.md` filename intentionally names the thing being removed. A naive global replace mangles all three.
- **Spec it per repo as two independent Ideas.** Loses: doubles the ideation ceremony for one cohesive decision. The four workstreams (URLs, file rename, concept scrub, winget finish) are conceptually identical across both repos; only the file inventories differ. One Idea → one Feature → two Plans (one per repo) is the natural decomposition.
- **Defer indefinitely; treat as polish.** Loses: every new doc written today perpetuates the inconsistency. The cost of the scrub grows linearly with the doc surface; the right time is *before* the next major Feature lands, not after.

## MVP Scope

Two Plans, one per repo (`specscore-cli`, `specstudio-skills`), executed sequentially over a single working session. Each Plan covers the four workstreams above against its own repo. Definition of done: `grep -ri synchestra` across each repo returns zero hits outside the preserved-historical paths (`docs/superpowers/2026-04-01-...`) and the sidekick seed file; `go test ./...` (specscore-cli) and `specscore spec lint` (both repos) both pass; the renamed `events.md` is referenced by every prior referrer of `synchestra-events.md`. Timeboxed: ship within the working session that approves this Idea.

## Not Doing (and Why)

- Other workspace repos (`specscore`, `specscore-studio`, `ai-plugin-specscore`, `ai-marketplace`, `homebrew-tap`, `scoop-bucket`) — out of MVP scope; tracked as Outstanding Questions for a follow-up sweep
- Historical rebrand docs at `docs/superpowers/2026-04-01-specscore-synchestra-restructuring*.md` — these ARE the audit trail of the rebrand decision; scrubbing them erases the historical record
- The empty seed `outline-idea-of-synchestra-removal.md` — Phase 1 consilium policy says seeds are durable until reviewed; this Idea supersedes it but the seed stays through the consilium queue
- The `synchestra_task` field in sidekick-seed YAML frontmatter — that's an event-payload contract change distinct from the doc/brand scrub; separate Idea if/when needed
- Renaming any CLI verb, package, or runtime identifier in specscore-cli — the user-facing CLI surface is already `specscore event emit` etc.; this Idea is doc/spec/brand scrub only
- The `.synchestra/` directory references in some legacy skill docs (e.g., the sidekick SKILL.md mentions `.synchestra/events.jsonl` while the cross-repo contract uses `.specscore/events.jsonl`) — fold into workstream (3) concept-scrub if encountered; do NOT introduce a new directory

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | The GitHub org rename `synchestra-io → specscore` has actually completed on github.com — URLs like `https://github.com/specscore/specscore-cli` resolve to real repos today. | Run `curl -fsI https://github.com/specscore/specscore-cli` and confirm `HTTP/2 200`. Run `git -C specscore-cli remote get-url origin` and assert the URL contains `specscore/`, not `synchestra-io/`. |
| Must-be-true | The winget rebrand from `Synchestra.SpecScore` → `SpecScore.CLI` (started 2026-05-19 per `.goreleaser.yml` comment) is in a state where finishing the documentation cleanup doesn't break the current package publish. | Inspect `.goreleaser.yml` for active winget `package_identifier` lines; confirm the manifest writes only `SpecScore.CLI`. Cross-check against `winget search specscore` output. |
| Must-be-true | The file rename `synchestra-events.md → events.md` won't break any link beyond the two repos' control. The doc is internal to SpecStudio; no external doc/marketplace deep-links to it. | `grep -r "synchestra-events.md"` across all sibling repos in the workspace; confirm no third-party doc references the path. |
| Should-be-true | All referrers of `synchestra-events.md` can be located by `grep` (no obfuscated link via redirect, no link via slug-only reference). | Sweep both repos with `grep -rln "synchestra-events"` before rename; spot-check the count against ack/ripgrep equivalent for confirmation. |
| Should-be-true | Concept-noun scrubs (replacing "Synchestra" with "workflow orchestrator" / "downstream automation") read naturally in the existing prose without rewriting whole sections. | Sample 5 representative passages from the highest-traffic files (cli/lifecycle-transitions, shared/synchestra-events.md, README.md per repo). If any passage requires more than a one-sentence rewrite, escalate to /specify-time discussion. |
| Should-be-true | The historical rebrand docs at `docs/superpowers/2026-04-01-specscore-synchestra-restructuring*.md` are PROCESS history and don't drive any active build/test/release flow. | Confirm none are referenced from `Makefile`, CI config, `.goreleaser.yml`, or skill prompts. If any are referenced, decide at /specify time. |
| Might-be-true | After the scrub, the only remaining "synchestra" references in the working tree will be in (a) the preserved historical docs and (b) the sidekick seed `outline-idea-of-synchestra-removal.md`. | Final acceptance gate: `grep -rli synchestra --include="*.md" --include="*.go" --include="*.yaml" --include="*.yml" .` returns ONLY those two paths per repo. |
| Might-be-true | The other workspace repos (`specscore`, `specscore-studio`, `ai-plugin-specscore`, etc.) will get the same scrub treatment in a follow-up Idea/Feature pair, soon. | Track follow-up in Outstanding Questions; revisit after this Idea ships. |


## SpecScore Integration

- **New Features this would create:** Likely one Feature per repo — `synchestra-scrub` in `specstudio-skills` and a sibling in `specscore-cli` (final slug to be confirmed at `/specstudio:specify` time). The work is the same shape in both repos but the file inventories differ; sharing one Feature across repos is awkward, so one Feature per repo is the natural split. Two Plans (one per Feature). Alternative: a single shared Feature documented in one repo with the second referencing it; cleaner narrative, but breaks the per-repo Plan locality.
- **Existing Features affected:** Indirectly, every Feature in both repos that contains a "synchestra" mention — the scrub touches their prose but does NOT change their behavioral contracts. The `cli/lifecycle-transitions` Feature in `specscore-cli` explicitly names "the architectural positioning vs Synchestra" and will need its description rewritten on its own terms. The five emitter skill Features in `specstudio-skills` (`ideate`, `specify`, `sidekick`, `consilium`, `plan`) all reference `shared/synchestra-events.md` and follow the file rename.
- **Dependencies:** The GitHub org rename `synchestra-io → specscore` must be live (validate as the first Must-be-true assumption). The winget rebrand in `.goreleaser.yml` must be in a state where finishing the docs doesn't conflict with the binary publish pipeline (validate as the second Must-be-true assumption).

## Open Questions

- **Follow-up sweep for other workspace repos.** The other SpecScore-managed repos in the workspace (`specscore`, `specscore-studio`, `ai-plugin-specscore`, `ai-marketplace`, `homebrew-tap`, `scoop-bucket`) likely carry similar brand references. Out of scope for this Idea's MVP; settle the follow-up cadence after this Idea ships (one combined sweep Idea, or per-repo Ideas).
- **The `synchestra_task` field in sidekick-seed YAML frontmatter.** Renaming this field is an event-payload contract change distinct from doc/brand scrub — it affects consumers parsing the seed. Defer to a separate Idea once a consumer of the field exists.
- **The `.synchestra/` directory convention in legacy skill docs.** The cross-repo event contract has already pivoted to `.specscore/events.jsonl`. Some skill SKILL.md files (notably the sidekick skill) still mention `.synchestra/events.jsonl`. Audit and fold these into workstream (3) at `/specstudio:specify` time; do NOT create a new directory convention.
- **Feature naming.** `synchestra-scrub` vs `brand-cleanup` vs `synchestra-removal-<repo>` — pick at `/specstudio:specify` time.

---
*This document follows the https://specscore.md/idea-specification*
