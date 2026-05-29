# Decision: Merge `/review` into a single `/score` command

**Date:** 2026-05-29
**Owner:** alex
**Status:** Decided (pending propagation into Idea + Feature)
**Affects:** Idea `manual-review-and-score-commands` (Implementing), Feature `manual-review-command` (Implementing)

## Question

The in-flight design ships **two** manual commands — `/review` (fast iteration
feedback) and `/score` (graded readiness assessment). Having both is confusing.
Do we keep two with a clear justification, or collapse to one?

**Decision: collapse to one command, `/score`. There is no `/review`.**

## Why one command

`/score` was always defined as a strict **superset** of `/review`. From the
source Idea: *"`/score` is structurally `/review` + grade computation + report
persistence + aggregation."* `/score` runs `/review` internally and adds output
on top. A superset relationship is the signature of "one command with options,"
not "two commands."

The original split bundled **three orthogonal axes** onto a single
`/review`-vs-`/score` line:

| Axis | `/review` end | `/score` end |
|------|---------------|--------------|
| Verdict shape | binary `Approved` / `Issues Found` | A–F letter grade |
| Persistence | ephemeral | writes `spec/_score/<slug>-<sha>.md` |
| Diff-awareness | `--against REF` (what changed) | snapshot of whole artifact |

These axes are independent — an author could legitimately want an *ephemeral
grade* or a *persisted binary verdict*. Carving two commands along the bundle
forces unrelated choices to move together and is exactly what reads as
confusing. Unbundling: the command names the action ("give me feedback on this
artifact"); the other two axes become flags.

`/score` (not `/review`) survives because:

- It is the superset; the binary verdict is *derivable* from a grade
  (any `Blocker` finding, or grade below threshold → "not passing").
- It closes the "Score" half of the **SpecScore** brand — the score the name
  implies becomes something the tool returns.
- One command to learn instead of "which one do I run?"

### Grade is the verdict currency (refinement)

There is no separate binary-verdict concept. A reviewer run produces an **A–F
grade**; `Approved` is *defined* as `grade ≥ threshold`, where the threshold is
**configurable** and defaults to **`B`**. This collapses the last remnant of the
`/review`-vs-`/score` split: "fast verdict" and "graded assessment" are the same
output read two ways.

Crucially, the grade + threshold live in the **shared `reviewer-gates` layer**,
not in the `/score` skill. Both consumers of that layer —

- **event/workflow-triggered producer-exit gates** (`specstudio:specify`, etc.), and
- **manual invocation** via `/score` —

therefore emit the same grade and apply the same threshold. Verdict parity
between the automatic and manual paths is then a *structural* property, not a
contract the `/score` skill has to re-establish.

## Design

### `/score [PATHS...]` — defaults

- Dispatches the configured `reviewer-gates` list for each artifact's stage.
  Still a **thin wrapper**: carries no reviewer logic of its own (single source
  of truth preserved).
- Prints verdict/grade + findings to the terminal. **Ephemeral — no file
  writes, no events.**

### Flags (the unbundled axes)

| Flag | Effect |
|------|--------|
| `--save` | Persist `spec/_score/<artifact-slug>-<sha>.md` |
| `--badge` | Inject A–F badge into the artifact (manual run: propose diff + ask approval) |
| `--against REF` | Diff focus for the "about to commit" loop (default `HEAD`); opt-in |
| `-r` / `--recursive` | Descend into sub-artifacts; keeps the `confirm-at-threshold` cost guard |
| `--verbose` | Full per-artifact detail in multi-artifact mode |
| `--yes` | Skip the recursion threshold confirmation |

### CI / exit code

Non-zero when any reviewed artifact fails the pass threshold (default: any
`Blocker` finding fails; threshold can be made configurable later). Skip and
exit-code semantics already written for the in-flight Feature carry over
unchanged.

## Sequencing — grade is the model; the dependency is `reviewer-gates`

The earlier "rename now, grade later" phasing is **superseded**: grade is the
verdict currency from the start, so it is not deferred to a later command phase.
What *is* sequenced is the dependency direction, because the grade lives in the
shared layer:

1. **`reviewer-gates` (upstream, needs its own design pass).** Introduce the
   grade as the reviewer/gate output currency, the **findings → A–F aggregation
   function**, and the **configurable Approve threshold** (default `B`) in
   `specscore.yaml`. This is the only genuinely undesigned piece and owns the
   "what does an A mean" question. It is a change to the `reviewer-gates`
   Feature, not to `/score`.
2. **`score-command` (downstream, mostly done here).** Rename `/review` →
   `/score`; the skill *consumes* the shared grade + threshold and surfaces
   grade + findings. The in-flight Feature already specifies path resolution,
   stage mapping, dispatch, human-reviewer skip, diff context, recursion
   threshold guard, and no-writes — all carry over unchanged. `--save` and
   `--badge` remain opt-in flags layered on the same output.

Net: the `/score` propagation is a rename + reframe (this work); the rubric +
threshold schema is a focused upstream design task in `reviewer-gates`.

## The grade rubric is *not* a blank page

A concern when leading with `/score` was "the A–F rubric is now on the critical
path." It is smaller than it looks: the **quality criteria already exist** as
per-stage reviewer prompts that `/score` dispatches unchanged —

- `skills/specify/references/reviewer-prompt.md` — six Blocker categories +
  check table (completeness, schema, consistency, clarity, scope, YAGNI,
  assumption-carryover, Rehearse, body-metadata).
- `skills/plan/references/reviewer-prompt.md` — its own taxonomy.
- `skills/ideate/references/refinement-criteria.md` — user value / feasibility /
  differentiation (currently a stress-test rubric, not yet a wired gate; only
  `gates.specify` is configured in `specscore.yaml`).

The `reviewer-gates` grade work therefore needs only a **grade-aggregation
function** mapping existing reviewer output (Blocker / Advisory counts,
per-category) to a letter — a far more tractable problem than inventing criteria
from scratch. Design of that function (plus the threshold-config schema) is the
focused upstream task and is out of scope for this decision.

## Noted gap (separate from this decision)

The `specify` reviewer checks internal consistency (requirements ↔ ACs) and
source-Idea assumption carryover, but has **no first-class check that the
requirements actually address the stated `## Problem`.** A Feature with
well-formed, internally-consistent REQs that solve a *different* problem than
stated would pass today. Closing this is a `reviewer-gates` change (a new Blocker
category), owned separately — recorded here so it is not lost.

## Alternatives (reversed from the source Idea)

The source Idea argued *for* two commands and rejected the single-command option.
This decision reverses that:

- **Single combined command (now chosen).** Previously rejected on "two mental
  models deserve two commands." Re-evaluated: the two "mental models" are two
  *outputs* of one action (feedback on an artifact), separated by flags, not two
  actions. The superset relationship makes one command the simpler model.
- **Keep both `/review` and `/score`.** Rejected: confusing, and the
  distinction collapses to flags on one command.
- **Split differently (ephemeral-vs-persisted as the command line).** Rejected:
  same unbundling applies — persistence is one flag among three, not a command
  boundary.

## Open questions (deferred)

- Grade-aggregation function shape (Blocker/Advisory weighting → letter) —
  owned by `reviewer-gates`.
- Approve threshold: default `B` decided; the config key shape in
  `specscore.yaml` is part of the `reviewer-gates` grade work.
- Report-file shape, badge rendering/location, re-injection idempotency for the
  `--save` / `--badge` flags (unchanged from the source Idea's open questions).

## Propagation

**Done in this pass:**

1. Renamed `skills/review/` → `skills/score/` and Feature
   `spec/features/manual-review-command/` → `spec/features/score-command/`;
   triggers `/review` → `/score`; lifecycle-`review` references left untouched.
2. Reframed the Idea, the `score-command` Feature, and the skill to the
   one-command grade-as-currency model; indexes and `_tests/` back-references
   reconciled.
3. Recorded the grade+threshold ownership decision in `reviewer-gates` as a
   scoped dependency note.

**Next (own design pass, not in this propagation):**

- `reviewer-gates` grade-as-currency: the findings → A–F aggregation function
  and the `specscore.yaml` Approve-threshold config (default `B`).
