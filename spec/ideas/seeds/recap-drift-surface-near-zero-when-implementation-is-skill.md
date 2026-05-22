---
type: sidekick-seed
slug: recap-drift-surface-near-zero-when-implementation-is-skill
captured_at: 2026-05-22T13:01:39Z
captured_by: user
captured_during: spec/features/skills/recap
trigger: explicit
status: queued
synchestra_task: null
---

# Recap drift surface near-zero when implementation is SKILL.md authored from ACs — add confidence or coverage signal

Observed during first meta-recap run on `skills/recap` (commit `bad8797`, report `_recap/81143e4.md`): drift-narrator returned `no-drift` on all 15 ACs. When the implementation artifact is itself a SKILL.md authored from the Feature's ACs, the SKILL.md's prose mechanically mirrors each AC's `Then` clause; spec and implementation are the same prose graph at two indirections. Drift surface collapses to near-zero by construction.

A reviewer reading a 15/15 no-drift report can't tell whether (a) the implementation is genuinely faithful, or (b) the implementation simply restated the spec and the detector had nothing to bite on. Both produce identical reports.

Possible directions to explore:

- **Regime classifier** that names the input type (skill-prose vs behavioral-code) and tags the report's prior on detected drift.
- **Coverage signal** that measures how much of the commit is novel mechanism vs restated AC language (n-gram overlap, semantic similarity); surface as `coverage_signal: high|medium|low|skill-prose` in the YAML.
- **Confidence dimension** per drift verdict: low-confidence `no-drift` flagged as "low-information run, consider supplemental review".
- **Separate skill** (`specstudio:meta-review`?) for skill-on-skill recap, since the 4-bucket taxonomy is overfit to behavioral-code drift.

Not blocking for recap MVP — current behavior is correct for the behavioral-code case it was designed around. Worth a dedicated Idea after recap runs against non-meta Features in production and we have data on whether real runs commonly hit this regime.
