# Design: Grade as the reviewer-gates verdict currency

**Date:** 2026-05-29
**Owner:** alex
**Status:** Designed (pending spec revision of `reviewer-gates`)
**Affects:** Feature `reviewer-gates` (Implementing); consumed by `score-command` and the producer-exit gates
**Source:** the unified `/score` decision — [`review-vs-score-command-merge.md`](review-vs-score-command-merge.md)

## Problem

The unified `/score` decision makes the **grade** the shared verdict currency:
`Approved` is defined as `grade ≥ threshold`, and the grade + threshold live in
the shared `reviewer-gates` layer so the producer-exit gates and manual `/score`
agree by construction. `reviewer-gates` today returns only a binary
`Approved | Issues Found` under AND-composition. This document designs the grade
that replaces it — without breaking the current gate behavior at the default.

Two constraints shape every choice below:

1. **The gate decision must be deterministic** (reproducible; verdict parity
   between manual and automatic runs is structural, not best-effort).
2. **Default behavior must not change** for existing repos — the new model must
   reduce to today's "any Blocker fails" at the default threshold.

## 1. Grade model — hybrid (Blockers cap, judgment refines)

A reviewer classifies each finding as `Blocker` or `Advisory` (unchanged) and
adds a per-lens within-band assessment. The **runner** computes the grade:

| Grade | Condition | Gate at default `B` |
|-------|-----------|---------------------|
| **A** | zero Blockers, negligible Advisories | pass |
| **B** | zero Blockers, some Advisories | pass |
| **C** | 1 Blocker | fail |
| **D** | several Blockers | fail |
| **F** | pervasive / foundational Blockers | fail |

**Determinism split — the load-bearing rule:**

- **Pass/fail hinges only on Blocker presence.** Zero Blockers → band `{A, B}`;
  ≥1 Blocker → band `{C, D, F}`. Blocker *count* selects C/D/F deterministically.
- **Judgment is confined to where it cannot flip a default gate:** A-vs-B (both
  pass at default `B`) and the failing letter (all fail). The reviewer's per-lens
  nuance is **clamped to the band** the Blockers already fixed.
- **Advisories never cross the pass/fail line** — they only move A↔B. This
  preserves the current "Advisory is non-gate-failing" contract.

Consequence: **default threshold `B` reproduces today's behavior exactly**
("any Blocker fails; advisories don't"). The letter adds resolution above and
below the line without changing the line.

## 2. Threshold — `Approved` ≡ `grade ≥ threshold`

- `B` (default) → requires zero Blockers → identical to today.
- `A` → stricter; demands an exemplary spec (opts into letting the A-vs-B
  judgment affect the gate — an explicit choice).
- `C` → lenient; tolerates a single Blocker (the C band).

Lowering the threshold *generalizes* the current hard "Blocker always blocks"
rule into "Blockers cost grade; the threshold decides how much you tolerate,"
while the default changes nothing.

## 3. Aggregation — worst-wins (min)

Across the BA/dev/QA lenses of one reviewer **and** across multiple reviewers in
a panel:

- The **Blocker set is the union** across all lenses and reviewers — any Blocker
  anywhere lands the artifact in the fail band.
- Total Blocker count drives C/D/F.
- In the pass band, the **lowest letter wins**.

This is today's AND-composition (any `Issues Found` → fail) generalized: a spec
is only as ready as its weakest dimension.

## 4. Configuration schema (`specscore.yaml`)

```yaml
grade:
  threshold: B          # top-level default, applied when a stage omits its own

gates:
  specify:
    threshold: B        # per-stage override (optional)
    reviewers: [ ... ]
  ideate:
    threshold: C        # e.g. a more lenient bar for early-stage ideas
    reviewers: [ ... ]
```

Resolution order per stage: `gates.<stage>.threshold` → `grade.threshold` →
built-in default `B`.

## 5. Reviewer output contract

Each reviewer returns:

- The **findings list** with `Blocker` / `Advisory` severities (unchanged shape).
- A **per-lens line** — BA, developer, QA — naming what each lens checked. The
  **BA lens explicitly verifies "do the requirements address the stated
  `## Problem`?"**, which closes the problem→requirements traceability gap noted
  in the merge decision (it becomes a Blocker category, not a missing check).
- A **within-band letter suggestion** used only to refine A-vs-B (or the failing
  letter); the runner clamps it to the Blocker-determined band.

**Default reviewer shape: one multi-role-aware reviewer per stage.** A single AI
reviewer carries all lenses and emits per-lens sub-assessments. A true
multi-agent panel (separate BA/dev/QA reviewers) stays **opt-in** by adding
entries to `gates.<stage>.reviewers`; `consilium` remains the heavyweight escape
hatch for high-stakes deliberation. This keeps routine and tree-wide `/score`
runs cheap.

## 6. What this changes in the `reviewer-gates` Feature

A spec revision (its own ideate → specify pass) will touch:

- **`verdict-contract`** — the runner returns a **grade + derived verdict**
  (`grade ≥ threshold`), not a bare `Approved | Issues Found`.
- **`and-composition`** → **Blocker-union + worst-wins** aggregation across lenses
  and reviewers (a generalization, not a reversal — default behavior identical).
- **New REQ: threshold config** — per-stage `gates.<stage>.threshold` + top-level
  `grade.threshold`, default `B`, with the resolution order in §4.
- **Reviewer-prompt guidance** — multi-role lenses, per-lens sub-assessment, the
  BA problem-traceability Blocker category, and the within-band letter output.
- **New ACs** for each, plus an explicit **migration AC**: a repo with no
  `threshold` config and the existing reviewer prompts produces the *same*
  pass/fail verdicts as today.

## 7. Out of scope (deliberately)

- `/score`'s `--save` (report persistence) and `--badge` (badge injection) — they
  *consume* this grade; specified in [`score-command`](../features/score-command/README.md).
- Badge rendering, location, and idempotency — open questions in the source Idea.
- Numeric sub-scores beyond the per-lens letter — the A–F letter is the currency;
  finer scoring is not justified yet.

## Migration / back-compat

At the default threshold `B` with the existing reviewer prompts, every gate
returns the same pass/fail it does today. The grade is additive: existing repos
see a letter where they saw `Approved | Issues Found`, and nothing they gated on
changes until they explicitly set a non-default threshold.

## Open questions (for the spec revision)

- Exact Blocker-count cut points for C vs D vs F (1 / 2–3 / 4+? or severity-weighted?).
- Whether the per-lens set (BA / dev / QA) is fixed or itself configurable per stage.
- Whether `grade.threshold` and per-stage overrides accept `+`/`-` letters
  (B+ / B-) or whole letters only.
