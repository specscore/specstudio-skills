---
format: https://specscore.md/scenarios-index-specification
---

# Scenarios — `reviewer-gates`

Scenarios validating the [Reviewer Gates feature](../README.md). Each scenario references the AC it validates and uses `Given / When / Then` form per the [SpecScore Scenario specification](https://specscore.md/scenario-specification).

All scenarios are scaffolded with `**Status:** pending`; their concrete Given/When/Then steps are authored during the implementation plan, not at spec time.

## Index

| Scenario | Validates |
|---|---|
| [gates-block-preserved.md](gates-block-preserved.md) | `reviewer-gates#ac:gates-block-preserved` |
| [untyped-entry-refused.md](untyped-entry-refused.md) | `reviewer-gates#ac:untyped-entry-refused` |
| [unknown-type-refused.md](unknown-type-refused.md) | `reviewer-gates#ac:unknown-type-refused` |
| [name-defaults-to-type.md](name-defaults-to-type.md) | `reviewer-gates#ac:name-defaults-to-type` |
| [duplicate-effective-name-refused.md](duplicate-effective-name-refused.md) | `reviewer-gates#ac:duplicate-effective-name-refused` |
| [legacy-command-key-rejected.md](legacy-command-key-rejected.md) | `reviewer-gates#ac:legacy-command-key-rejected` |
| [deterministic-verdict-from-exit.md](deterministic-verdict-from-exit.md) | `reviewer-gates#ac:deterministic-verdict-from-exit` |
| [auto-approve-always-approves.md](auto-approve-always-approves.md) | `reviewer-gates#ac:auto-approve-always-approves` |
| [pre-commit-gate-fires-per-occurrence.md](pre-commit-gate-fires-per-occurrence.md) | `reviewer-gates#ac:pre-commit-gate-fires-per-occurrence` |
| [ai-entry-shape-violations-refused.md](ai-entry-shape-violations-refused.md) | `reviewer-gates#ac:ai-entry-shape-violations-refused` |
| [human-entry-min-approvers-cap.md](human-entry-min-approvers-cap.md) | `reviewer-gates#ac:human-entry-min-approvers-cap` |
| [human-entry-rejects-prompt.md](human-entry-rejects-prompt.md) | `reviewer-gates#ac:human-entry-rejects-prompt` |
| [serial-dispatch-observed.md](serial-dispatch-observed.md) | `reviewer-gates#ac:serial-dispatch-observed` |
| [and-composition-blocks-on-any-issues-found.md](and-composition-blocks-on-any-issues-found.md) | `reviewer-gates#ac:and-composition-blocks-on-any-issues-found` |
| [rerun-policy-applies-on-structural-fix.md](rerun-policy-applies-on-structural-fix.md) | `reviewer-gates#ac:rerun-policy-applies-on-structural-fix` |
| [specify-loads-gate-not-builtin.md](specify-loads-gate-not-builtin.md) | `reviewer-gates#ac:specify-loads-gate-not-builtin` |
| [missing-gates-block-refuses-with-error.md](missing-gates-block-refuses-with-error.md) | `reviewer-gates#ac:missing-gates-block-refuses-with-error` |
| [third-party-integration-revised.md](third-party-integration-revised.md) | `reviewer-gates#ac:third-party-integration-revised` |
| [specify-feature-revised.md](specify-feature-revised.md) | `reviewer-gates#ac:specify-feature-revised` |
| [review-feature-archived.md](review-feature-archived.md) | `reviewer-gates#ac:review-feature-archived` |
| [root-readme-link-present.md](root-readme-link-present.md) | `reviewer-gates#ac:root-readme-link-present` |
| [skill-doc-cross-links-present.md](skill-doc-cross-links-present.md) | `reviewer-gates#ac:skill-doc-cross-links-present` |
| [grade-band-by-blocker-count.md](grade-band-by-blocker-count.md) | `reviewer-gates#ac:grade-band-by-blocker-count` |
| [threshold-default-reproduces-today.md](threshold-default-reproduces-today.md) | `reviewer-gates#ac:threshold-default-reproduces-today` |
| [threshold-resolution-order.md](threshold-resolution-order.md) | `reviewer-gates#ac:threshold-resolution-order` |
| [invalid-threshold-refused.md](invalid-threshold-refused.md) | `reviewer-gates#ac:invalid-threshold-refused` |
| [lenient-threshold-tolerates-blocker.md](lenient-threshold-tolerates-blocker.md) | `reviewer-gates#ac:lenient-threshold-tolerates-blocker` |
| [worst-wins-union-across-reviewers.md](worst-wins-union-across-reviewers.md) | `reviewer-gates#ac:worst-wins-union-across-reviewers` |
| [within-band-letter-derivation.md](within-band-letter-derivation.md) | `reviewer-gates#ac:within-band-letter-derivation` |
| [ba-lens-problem-traceability-blocker.md](ba-lens-problem-traceability-blocker.md) | `reviewer-gates#ac:ba-lens-problem-traceability-blocker` |
| [grade-recorded-on-release.md](grade-recorded-on-release.md) | `reviewer-gates#ac:grade-recorded-on-release` |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/scenarios-index-specification*
