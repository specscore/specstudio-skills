# Idea: Add a manual /score command over reviewer-gates

**Status:** Implementing
**Date:** 2026-05-28
**Owner:** alex
**Promotes To:** score-command
**Supersedes:** —
**Related Ideas:** depends_on:reviewer-gates

## Problem Statement

How might we let an author or reviewer manually trigger reviewer-gates outside the producer-skill exit context — over one artifact, a set of artifacts, or a tree — and get a graded readiness assessment (`/score`) whose `Approved`/`Issues` verdict matches the one the automatic producer-exit gates apply?

## Context

The [Reviewer Gates Feature](../features/reviewer-gates/README.md) (Approved; Implementing) dispatches typed reviewers (`ai`, `human`) at producer-skill *exits* — when `specstudio:ideate`, `specstudio:specify`, or `specstudio:plan` finish. That covers "I just authored this; gate me." It does not cover the rest of an author's day:

- **Mid-iteration self-review** before re-running the producer skill (which has other side effects — status transitions, event emission, index updates).
- **Audit** of artifacts the user did not author (someone else's draft).
- **Triage pass** across the tree to find what needs attention next.
- **Pre-PR sanity check** on a draft the user paused on.

The reframed reviewer use case — *a BA or developer wants AI feedback on a draft before spending human review time on it* — is exactly the AI-reviewer slot in reviewer-gates, but with no current manual surface. The mechanism exists; the user-facing trigger does not.

Two readings of feedback surface in that day — *"did I just break something?"* (a quick check while iterating) and *"is this good enough?"* (a readiness assessment). The original direction split these into two commands (`/review` and `/score`). They are now **unified into one**: once a reviewer run yields an A–F **grade** and `Approved` is *defined* as `grade ≥ threshold` (configurable, default `B`), the quick check and the readiness assessment are the same output read against a cutoff. One command suffices — and because the grade + threshold live in the shared `reviewer-gates` layer, the manual and automatic paths can never disagree.

A literal `/score` also closes the "Score" half of the SpecScore brand: the score the name implies becomes something the tool returns.

## Recommended Direction

Ship **one** user-facing skill — `specstudio:score` — a thin wrapper over the `reviewer-gates` dispatch pipeline. There is no separate `/review`. The original two-command split bundled three orthogonal axes (verdict shape, persistence, diff-awareness) onto one naming line; those collapse to flags on the single command.

`specstudio:score [PATHS...] [-r|--recursive] [--against REF] [--save] [--badge] [--verbose] [--yes]`

- Invokes the configured `reviewer-gates` list for each artifact's stage.
- **Output: an A–F grade + findings per artifact.** `Approved` is *derived* — it means `grade ≥ threshold`, where the threshold is configurable and defaults to `B`. There is no separate binary-verdict concept.
- Ephemeral by default — writes nothing. `--save` persists a Markdown report at `spec/_score/<artifact-slug>-<sha>.md`; `--badge` injects an A–F badge (manual runs propose the diff and ask before writing).
- `-r` opts into recursive descent (with a confirm-at-threshold cost guard); `--against REF` sets a diff baseline (default `HEAD`) so reviewers can focus on what changed.

**The grade + threshold live in the shared `reviewer-gates` layer, not in this skill.** Both the event/workflow-triggered producer-exit gates and manual `/score` consume the same grade and threshold, so verdict parity between the automatic and manual paths is structural — not a contract this skill must re-establish. The remaining design task — the findings → A–F aggregation function and the threshold-config schema — is owned by [Reviewer Gates](../features/reviewer-gates/README.md).

**Reviewer shape: one multi-role-aware reviewer per stage by default.** A single AI reviewer evaluates each artifact through multiple lenses — BA (do the requirements address the stated Problem?), developer (is it implementable as written?), QA (are the ACs observable/testable?) — and emits per-lens sub-scores that aggregate into the grade. A true multi-agent panel (separate BA/developer/QA reviewers) stays **opt-in** by adding entries to `gates.<stage>.reviewers`; `consilium` remains the heavyweight escape hatch for high-stakes deliberation. This keeps `/score` cheap enough to run constantly and avoids an N-role fan-out on tree-wide sweeps.

Shared semantics:

- **Multi-artifact from day one.** `PATHS` is a positional list of files, directories, or globs; a directory resolves to its `README.md` and (with `-r`) to all sub-artifacts; empty `PATHS` defaults to `spec/`. No `-r` means current artifact only.
- **AI-only outside producer flow.** `type: human` entries are silently skipped — manual invocation cannot coherently suspend the user's session waiting for themselves.
- **Single source of truth.** All reviewer logic, grade aggregation, and the threshold come from `reviewer-gates`; this skill carries none of its own.
- **Output shape mirrors `eslint` / `pytest`.** Per-artifact section + footer summary. No surprises for adopters.

## Alternatives Considered

- **Two separate commands (`/review` + `/score`).** *Originally recommended; now rejected.* The two "mental models" (iteration feedback vs graded assessment) are two readings of one output, not two actions — once `Approved` means `grade ≥ threshold`, the binary verdict is just the grade read against a cutoff. Two commands re-introduce "which one do I run?" for no gain.
- **Single command with a `--score` toggle.** Rejected for the opposite reason: there is nothing to toggle. The grade is always the output; only persistence (`--save`) and badge (`--badge`) are genuinely optional.
- **No user-facing command; rely on producer-exit gates.** Rejected: misses mid-iteration self-review, audit of others' artifacts, triage passes, and pre-PR checks on paused drafts. The reviewer-gates mechanism solves dispatch; manual invocation is a separate surface.
- **Grade + threshold inside the `/score` skill.** Rejected: it would diverge from the producer-exit gates and break verdict parity. The grade is the shared verdict currency, single-sourced in `reviewer-gates`.
- **Build review/score logic from scratch instead of wrapping reviewer-gates.** Rejected: duplicates the reviewer registry, the dispatch contract, and the verdict shape — two configuration surfaces and two reviewer implementations to maintain.

## MVP Scope

Ship the single `specstudio:score` skill as the manual surface over `reviewer-gates`. Single- and multi-artifact modes (positional `PATHS`); `-r` recursive descent with a confirm-at-threshold guard; `--against REF` diff baseline (default `HEAD`); AI-only (`type: human` skipped). Output is the grade + findings per artifact with a footer summary across multiple artifacts; `Approved` = `grade ≥ threshold` (default `B`). Ephemeral by default; `--save` and `--badge` are opt-in.

The grade itself comes from `reviewer-gates`, so this MVP is **gated on the upstream `reviewer-gates` grade work** — the findings → A–F aggregation function and the configurable Approve threshold (a separate design pass). Until that lands, `/score` can surface the underlying findings and the threshold-derived pass/fail, but the letter grade is only as meaningful as the aggregation `reviewer-gates` defines. Badge and report persistence (`--badge`, `--save`) layer on the same output; manual runs require user approval before any write.

If `/score` cannot produce the same grade/verdict on the same artifact as the producer-exit gate does (no drift between manual and automatic invocation), the architecture is wrong — grade and threshold must be single-sourced in `reviewer-gates`.

## Not Doing (and Why)

- `/score` in MVP — deferred to Phase 2. The grade rubric and report-file shape need design that should not block `/score` shipping.
- A `specscore review` or `specscore score` CLI verb — out of MVP; skills are the shipping surface. The CLI verb is a likely follow-up once both skills stabilize.
- Cross-artifact reviewers (e.g., "does this Plan's task citations match its Feature's ACs") — a separate Feature in the reviewer-gates Idea family; out of scope for the manual command surface.
- README badge integration — depends on `/score` aggregate output; comes after Phase 2.
- Auto-running `/score` on every `specscore spec lint` — performance and noise risk. The commands are signal, not gate; users invoke them, the system does not run them silently.
- Reviewer-prompt changes inside reviewer-gates entries — reuse existing prompts in MVP. If diff-aware vs snapshot prompts need a `mode:` discriminator on AI entries, that is a reviewer-gates change, owned by [the Reviewer Gates Feature](../features/reviewer-gates/README.md).
- Manual human-reviewer invocation outside producer flow — `type: human` entries are silently skipped when the command is invoked manually. Handing a draft to a human is a hand-off, not a skill invocation.
- Score-based CI gating — `specstudio:verify` and `specstudio:recap` already serve as gates. The score is triage signal, not a gate.

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | Manual `/score` against an artifact produces the same verdict the producer-exit reviewer-gate does. No drift between the manual and the automatic invocation paths. | Cross-check on a sample of Ideas and Features: run the producer skill on each, capture the gate verdict, then run `/score` on the same artifact and diff the output. Any difference is a contract bug in this skill. |
| Must-be-true | `-r` recursion handles a 50+ artifact tree without runaway LLM cost; either a concurrency cap or a confirmation threshold ships in MVP. | Run `/score spec/ -r` on this repo and on `specscore-cli` at MVP completion. Measure wall-clock time and total subagent dispatches. If either exceeds a reasonable bound, the concurrency mechanism is not optional. |
| Should-be-true | Authors use `/score` between producer-skill runs rather than just re-running the producer. The iteration loop is real. | Watch first-month adopter behaviour after launch; look for `/score` invocation patterns between `/ideate` and `/specify`, or between `/specify` and `/plan`. |
| Should-be-true | AI-only default for manual invocation is acceptable; users do not expect manual `/score` to suspend waiting for human reviewers. | Solicit qualitative feedback from the first reviewers using the skill in the first month. If users are surprised that `type: human` entries are skipped, the silent-skip is wrong; a warning may be the right shape. |
| Might-be-true | `/score --badge` with `-r` becomes the README-badge mechanism once the `reviewer-gates` grade lands. | Defer; validate only after the grade aggregation ships and at least one rubric revision. Premature validation here biases the rubric toward badge optics over reviewer utility. |

## SpecScore Integration

- **New Features this would create:** a single `score-command` Feature (the manual `/score` surface). No second command — grade, persistence (`--save`), and badge (`--badge`) are flags on the one command, not a separate skill.
- **Existing Features affected:** depends on the [Reviewer Gates Feature](../features/reviewer-gates/README.md), which must add the **grade as verdict currency**: the findings → A–F aggregation function and the configurable Approve threshold (default `B`). `/score` consumes that and carries no grade logic of its own. The default reviewer shape — one multi-role-aware reviewer per stage, with a multi-agent panel opt-in via the reviewer list — is also a reviewer-gates concern.
- **Dependencies:** `reviewer-gates` must ship the grade + threshold before `/score`'s grade output is meaningful. No Idea-Idea dependencies beyond `depends_on:reviewer-gates`.

## Open Questions

- Concurrency cap for `-r` over large trees: sequential default with explicit `--parallel`, a `--max-concurrency` flag, or a confirm-at-threshold UX (`/score` refuses to run on >20 artifacts without `--yes`)?
- `/score --against` baseline default: working-tree-vs-`HEAD` (Git's default `diff`) or `HEAD`-vs-`HEAD~1` (last committed change)? Different defaults match different workflows.
- Multi-artifact output verbosity: full per-artifact detail by default, or a summary table with `--verbose` for detail-per-artifact? Affects both terminal noise and copy-paste-to-PR utility.
- `type: human` entries during manual invocation: silently skip, surface a warning, or refuse to run the gate at all? The silent-skip is least-surprising for solo authors, the warning is most-honest, the refusal is safest for shared-team configurations.
- `/score` report file location: `spec/_score/<artifact-slug>-<sha>.md` mirrors recap/verify per-Feature pattern, but artifact-slug collisions across types (idea vs feature with same slug) need a disambiguator (`spec/_score/idea-<slug>-<sha>.md`?).
- ~~Naming collision with the `review` pipeline-step~~ — **resolved** by naming the command `/score`: it no longer overloads `specstudio:review` (the separate lifecycle code-review step).
- Grade-aggregation function: how per-lens sub-scores (BA / developer / QA) and Blocker/Advisory counts combine into a single A–F letter. Owned by `reviewer-gates`.
- Approve-threshold config shape in `specscore.yaml`: a per-stage key under `gates:`, or a top-level `score:`/`grade:` block? Default value `B` is decided; the key location is not.
- Badge rendering: literal text line (`**Quality:** B+`), a Markdown image badge (`![Quality: B+](badge-url)`), or a shields.io-style remote SVG? Local-rendered text is portable and offline-safe; image badges are visually familiar but introduce a hosting dependency.
- Badge injection location inside an artifact: top of the body (above the first H1), inside the metadata block (alongside `**Status:**`), or in a dedicated `## Quality` section? Each choice has different lint implications and different visual prominence.
- Badge injection scope when running `/score` with `-r`: inject only into the explicitly named artifact, into every artifact visited during the recursive walk, or both? Different defaults match different mental models.
- Event-triggered configuration shape: `specscore.yaml` top-level `score:` block with per-artifact-type opt-ins (`score: { features: inject, ideas: report-only }`), or per-stage entries inside the existing `gates:` block?
- Re-injection on subsequent `/score` runs: replace the existing badge in-place (idempotent), append a history line, or refuse to overwrite without `--force`? Affects how cleanly the badge tracks the artifact's current grade.

---
*This document follows the https://specscore.md/idea-specification*
