# Idea: Add manual /review and /score commands over reviewer-gates

**Status:** Implementing
**Date:** 2026-05-28
**Owner:** alex
**Promotes To:** manual-review-command
**Supersedes:** —
**Related Ideas:** depends_on:reviewer-gates

## Problem Statement

How might we let an author or reviewer manually trigger reviewer-gates outside the producer-skill exit context — over one artifact, a set of artifacts, or a tree — and choose between fast iteration feedback (`/review`) and a graded readiness assessment (`/score`)?

## Context

The [Reviewer Gates Feature](../features/reviewer-gates/README.md) (Approved; Implementing) dispatches typed reviewers (`ai`, `human`) at producer-skill *exits* — when `specstudio:ideate`, `specstudio:specify`, or `specstudio:plan` finish. That covers "I just authored this; gate me." It does not cover the rest of an author's day:

- **Mid-iteration self-review** before re-running the producer skill (which has other side effects — status transitions, event emission, index updates).
- **Audit** of artifacts the user did not author (someone else's draft).
- **Triage pass** across the tree to find what needs attention next.
- **Pre-PR sanity check** on a draft the user paused on.

The reframed reviewer use case — *a BA or developer wants AI feedback on a draft before spending human review time on it* — is exactly the AI-reviewer slot in reviewer-gates, but with no current manual surface. The mechanism exists; the user-facing trigger does not.

Two distinct mental models surface in that day:

- *"Did I just do something wrong?"* — diff-aware iteration feedback. Answer is `Approved` or `Issues Found`; output is ephemeral; runs often.
- *"Is this draft good enough?"* — snapshot quality assessment. Answer is a letter grade + per-axis breakdown; output is persisted; runs at triage moments.

Folding both into one command loses signal. Splitting them, both as thin wrappers over the same reviewer-gates pipeline, keeps reviewer logic single-sourced and lets each command optimize for its own mental model.

A literal `/score` also closes the "Score" half of the SpecScore brand: the score the name implies becomes something the tool returns.

## Recommended Direction

Ship two thin user-facing skills as a single Idea — both wrap the reviewer-gates dispatch pipeline; neither duplicates reviewer logic.

`specstudio:review [PATHS...] [-r|--recursive] [--against REF]`

- Manually invokes the configured reviewer-gates list for each artifact's stage.
- Output: `Approved | Issues Found` verdict per artifact, plus inline feedback. No file written.
- `-r` opts into descending into sub-artifacts; default off so users cannot accidentally fan a 50-LLM-call sweep.
- `--against REF` sets a diff baseline (defaults to `HEAD`); reviewers receive both the artifact and the diff so feedback can focus on what changed.

`specstudio:score [PATHS...] [-r|--recursive]`

- Runs `/review` against each artifact, computes a weighted A–F letter grade from the verdicts and signals, persists a Markdown report at `spec/_score/<artifact-slug>-<sha>.md`, and writes an aggregate report when multiple artifacts are in scope.
- Output: grade + verdict + feedback summary per artifact; aggregate grade + distribution as the footer; persisted reports.
- Optionally **injects an A–F quality badge** into the scored artifact (per-node when scoring a leaf, per-sub-artifact when scoring with `-r`, and into the root index `spec/README.md` when scoring the tree). Injection behaviour depends on how the command was invoked:
  - **Manual invocation** (user runs `/score`): the skill proposes the injection diff and asks for approval before writing it. No badge is injected silently.
  - **Event-triggered invocation** (e.g., scheduled run, post-`recap` hook): injection is **configured** — opt-in per artifact type or per stage in `specscore.yaml`. Configured-auto-inject lets the badge stay fresh in repos that want it; the default for unconfigured event-triggered runs is do-not-inject.

Shared semantics for both commands:

- **Multi-artifact in scope from day one.** `PATHS` is a positional list of files, directories, or globs. A directory resolves to its `README.md` and (with `-r`) to all sub-artifacts; the spec/ root with `-r` is a tree-wide audit. Empty `PATHS` defaults to `spec/`. No `-r` means current artifact only.
- **AI-only by default outside producer flow.** Reviewer entries with `type: human` are silently skipped (or surfaced as *"this stage also requires human review — not invokable from `/review`"*). Manual invocation cannot coherently suspend the user's session waiting for themselves.
- **Single source of truth for reviewer prompts.** Both commands invoke the existing reviewer-gates dispatch. If reviewer-gates needs a `mode:` discriminator on AI entries to distinguish diff-aware from snapshot prompts, that is a [Reviewer Gates](../features/reviewer-gates/README.md) change — not a change in these skills.
- **Output shape mirrors `eslint` / `pytest`.** Per-artifact section + footer summary. No surprises for adopters.

`/score` is structurally `/review` + grade computation + report persistence + aggregation. It is split because the mental model and output destination differ; the implementation reuses everything from `/review`.

## Alternatives Considered

- **Single combined command (`/review --score`).** Rejected: the two mental models (iteration feedback vs snapshot grade) deserve different commands. Conflating them confuses adopters about which output they are reading and forces every author who just wants a quick check to also pay the grade-rubric cost.
- **No user-facing commands; rely on producer-exit gates.** Rejected: misses mid-iteration self-review, audit of others' artifacts, triage passes, and pre-PR checks on paused drafts. The reviewer-gates mechanism solves dispatch; manual invocation is a separate surface.
- **Build review/score logic from scratch instead of wrapping reviewer-gates.** Rejected: duplicates the reviewer registry, the dispatch contract, and the verdict shape. Adopters get two configuration surfaces for the same job; the system grows two parallel reviewer implementations to maintain.
- **Tree-wide score as a separate later phase.** Rejected: with the `-r` flag the multi-artifact path is essentially free. Running `/score spec/ -r` on the tree IS the tree-wide audit. One less phase to scope, and the public README-badge use case becomes achievable in Phase 2 rather than Phase 3.

## MVP Scope

A two-week build that ships `specstudio:review` only. Single-artifact and multi-artifact modes (positional `PATHS` list). The `-r` flag opts into recursive descent. The `--against REF` flag sets a diff baseline (default `HEAD`). AI-only by default; `type: human` entries silently skipped. Dispatch goes through the reviewer-gates pipeline; no new reviewer infrastructure. Output is inline per-artifact verdict + feedback, with a footer summary when multiple artifacts are in scope. No persistence. No grade. No `/score`.

`/score` lands in Phase 2 once `/review` is dogfooded, the persisted-report file shape is designed, and the weighted-grade rubric is defensible. Badge injection (per-artifact, per-sub-artifact with `-r`, or into the root index for tree-wide scores) is part of the Phase 2 `/score` deliverable, with manual runs requiring user approval before any write and event-triggered runs honouring `specscore.yaml` configuration. The Phase 1 deliverable is enough on its own — adopters can `/review spec/ -r` for tree-wide audit on day one.

If `/review` cannot produce the same verdict on the same artifact as the producer-exit gate does (no drift between manual and auto invocation), the MVP is wrong on architecture, not on scope.

## Not Doing (and Why)

- `/score` in MVP — deferred to Phase 2. The grade rubric and report-file shape need design that should not block `/review` shipping.
- A `specscore review` or `specscore score` CLI verb — out of MVP; skills are the shipping surface. The CLI verb is a likely follow-up once both skills stabilize.
- Cross-artifact reviewers (e.g., "does this Plan's task citations match its Feature's ACs") — a separate Feature in the reviewer-gates Idea family; out of scope for the manual command surface.
- README badge integration — depends on `/score` aggregate output; comes after Phase 2.
- Auto-running `/review` on every `specscore spec lint` — performance and noise risk. The commands are signal, not gate; users invoke them, the system does not run them silently.
- Reviewer-prompt changes inside reviewer-gates entries — reuse existing prompts in MVP. If diff-aware vs snapshot prompts need a `mode:` discriminator on AI entries, that is a reviewer-gates change, owned by [the Reviewer Gates Feature](../features/reviewer-gates/README.md).
- Manual human-reviewer invocation outside producer flow — `type: human` entries are silently skipped when the command is invoked manually. Handing a draft to a human is a hand-off, not a skill invocation.
- Score-based CI gating — `specstudio:verify` and `specstudio:recap` already serve as gates. The score is triage signal, not a gate.

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | Manual `/review` against an artifact produces the same verdict the producer-exit reviewer-gate does. No drift between the manual and the automatic invocation paths. | Cross-check on a sample of Ideas and Features: run the producer skill on each, capture the gate verdict, then run `/review` on the same artifact and diff the output. Any difference is a contract bug in this skill. |
| Must-be-true | `-r` recursion handles a 50+ artifact tree without runaway LLM cost; either a concurrency cap or a confirmation threshold ships in MVP. | Run `/review spec/ -r` on this repo and on `specscore-cli` at MVP completion. Measure wall-clock time and total subagent dispatches. If either exceeds a reasonable bound, the concurrency mechanism is not optional. |
| Should-be-true | Authors use `/review` between producer-skill runs rather than just re-running the producer. The iteration loop is real. | Watch first-month adopter behaviour after launch; look for `/review` invocation patterns between `/ideate` and `/specify`, or between `/specify` and `/plan`. |
| Should-be-true | AI-only default for manual invocation is acceptable; users do not expect manual `/review` to suspend waiting for human reviewers. | Solicit qualitative feedback from the first reviewers using the skill in the first month. If users are surprised that `type: human` entries are skipped, the silent-skip is wrong; a warning may be the right shape. |
| Might-be-true | `/score` with `-r` aggregate becomes the README-badge mechanism once Phase 2 lands. | Defer; validate only after Phase 2 and at least one rubric revision. Premature validation here biases the rubric toward badge optics over reviewer utility. |

## SpecScore Integration

- **New Features this would create:** a top-level `manual-review-command` Feature for the Phase 1 deliverable, with a sub-Feature for `/score` in Phase 2. The combined Idea may also be specified as a single parent Feature with two sub-Features.
- **Existing Features affected:** depends on the [Reviewer Gates Feature](../features/reviewer-gates/README.md) being implemented; the manual commands consume its dispatch pipeline. If diff-aware reviewer prompts need a `mode:` discriminator on AI reviewer entries, the change is owned by reviewer-gates and pulled in here as a dependency.
- **Dependencies:** depends on reviewer-gates completing its Implementing phase before either skill can ship. No Idea-Idea dependencies beyond that.

## Open Questions

- Concurrency cap for `-r` over large trees: sequential default with explicit `--parallel`, a `--max-concurrency` flag, or a confirm-at-threshold UX (`/review` refuses to run on >20 artifacts without `--yes`)?
- `/review --against` baseline default: working-tree-vs-`HEAD` (Git's default `diff`) or `HEAD`-vs-`HEAD~1` (last committed change)? Different defaults match different workflows.
- Multi-artifact output verbosity: full per-artifact detail by default, or a summary table with `--verbose` for detail-per-artifact? Affects both terminal noise and copy-paste-to-PR utility.
- `type: human` entries during manual invocation: silently skip, surface a warning, or refuse to run the gate at all? The silent-skip is least-surprising for solo authors, the warning is most-honest, the refusal is safest for shared-team configurations.
- `/score` report file location: `spec/_score/<artifact-slug>-<sha>.md` mirrors recap/verify per-Feature pattern, but artifact-slug collisions across types (idea vs feature with same slug) need a disambiguator (`spec/_score/idea-<slug>-<sha>.md`?).
- Naming collision: the old `review` pipeline-step Feature is being archived per reviewer-gates' Idea. Reusing `specstudio:review` as the manual skill name is the natural verb but may confuse adopters who remember the old step.
- Badge rendering: literal text line (`**Quality:** B+`), a Markdown image badge (`![Quality: B+](badge-url)`), or a shields.io-style remote SVG? Local-rendered text is portable and offline-safe; image badges are visually familiar but introduce a hosting dependency.
- Badge injection location inside an artifact: top of the body (above the first H1), inside the metadata block (alongside `**Status:**`), or in a dedicated `## Quality` section? Each choice has different lint implications and different visual prominence.
- Badge injection scope when running `/score` with `-r`: inject only into the explicitly named artifact, into every artifact visited during the recursive walk, or both? Different defaults match different mental models.
- Event-triggered configuration shape: `specscore.yaml` top-level `score:` block with per-artifact-type opt-ins (`score: { features: inject, ideas: report-only }`), or per-stage entries inside the existing `gates:` block?
- Re-injection on subsequent `/score` runs: replace the existing badge in-place (idempotent), append a history line, or refuse to overwrite without `--force`? Affects how cleanly the badge tracks the artifact's current grade.

---
*This document follows the https://specscore.md/idea-specification*
