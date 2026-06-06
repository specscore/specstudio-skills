---
format: https://specscore.md/scenarios-index-specification
---

# Scenarios — `score-command`

Scenarios validating the [Score Command feature](../README.md). Each scenario references the AC it validates and uses `Given / When / Then` form per the [SpecScore Scenario specification](https://specscore.md/scenario-specification).

All scenarios are scaffolded with `**Status:** pending`; their concrete Given/When/Then steps are authored during the implementation plan, not at spec time.

## Index

| Scenario | Validates |
|---|---|
| [invocation-triggers-respond.md](invocation-triggers-respond.md) | `score-command#ac:invocation-triggers-respond` |
| [unknown-flag-refused.md](unknown-flag-refused.md) | `score-command#ac:unknown-flag-refused` |
| [empty-paths-defaults-to-spec.md](empty-paths-defaults-to-spec.md) | `score-command#ac:empty-paths-defaults-to-spec` |
| [empty-paths-recursive-walks-tree.md](empty-paths-recursive-walks-tree.md) | `score-command#ac:empty-paths-recursive-walks-tree` |
| [directory-uses-readme.md](directory-uses-readme.md) | `score-command#ac:directory-uses-readme` |
| [directory-without-readme-skipped.md](directory-without-readme-skipped.md) | `score-command#ac:directory-without-readme-skipped` |
| [glob-unmatched-refused.md](glob-unmatched-refused.md) | `score-command#ac:glob-unmatched-refused` |
| [deduplication-preserves-first-order.md](deduplication-preserves-first-order.md) | `score-command#ac:deduplication-preserves-first-order` |
| [stage-mapping-resolves-correctly.md](stage-mapping-resolves-correctly.md) | `score-command#ac:stage-mapping-resolves-correctly` |
| [index-readme-skipped.md](index-readme-skipped.md) | `score-command#ac:index-readme-skipped` |
| [stage-without-gate-skipped.md](stage-without-gate-skipped.md) | `score-command#ac:stage-without-gate-skipped` |
| [human-reviewers-silently-omitted.md](human-reviewers-silently-omitted.md) | `score-command#ac:human-reviewers-silently-omitted` |
| [all-human-list-reports-no-verdict.md](all-human-list-reports-no-verdict.md) | `score-command#ac:all-human-list-reports-no-verdict` |
| [diff-supplied-to-ai-with-default-ref.md](diff-supplied-to-ai-with-default-ref.md) | `score-command#ac:diff-supplied-to-ai-with-default-ref` |
| [against-custom-ref-applied.md](against-custom-ref-applied.md) | `score-command#ac:against-custom-ref-applied` |
| [invalid-ref-falls-back-to-empty-diff.md](invalid-ref-falls-back-to-empty-diff.md) | `score-command#ac:invalid-ref-falls-back-to-empty-diff` |
| [verdict-parity-with-specify.md](verdict-parity-with-specify.md) | `score-command#ac:verdict-parity-with-specify` |
| [confirm-at-threshold-prompts.md](confirm-at-threshold-prompts.md) | `score-command#ac:confirm-at-threshold-prompts` |
| [confirm-at-threshold-below-bound.md](confirm-at-threshold-below-bound.md) | `score-command#ac:confirm-at-threshold-below-bound` |
| [yes-flag-skips-confirm.md](yes-flag-skips-confirm.md) | `score-command#ac:yes-flag-skips-confirm` |
| [serial-across-artifacts.md](serial-across-artifacts.md) | `score-command#ac:serial-across-artifacts` |
| [single-artifact-detailed-output.md](single-artifact-detailed-output.md) | `score-command#ac:single-artifact-detailed-output` |
| [multi-artifact-default-summary-mode.md](multi-artifact-default-summary-mode.md) | `score-command#ac:multi-artifact-default-summary-mode` |
| [multi-artifact-verbose-mode.md](multi-artifact-verbose-mode.md) | `score-command#ac:multi-artifact-verbose-mode` |
| [skipped-artifacts-in-table.md](skipped-artifacts-in-table.md) | `score-command#ac:skipped-artifacts-in-table` |
| [exit-code-success.md](exit-code-success.md) | `score-command#ac:exit-code-success` |
| [exit-code-failure.md](exit-code-failure.md) | `score-command#ac:exit-code-failure` |
| [exit-code-all-skipped.md](exit-code-all-skipped.md) | `score-command#ac:exit-code-all-skipped` |
| [no-files-written.md](no-files-written.md) | `score-command#ac:no-files-written` |
| [no-new-config-keys.md](no-new-config-keys.md) | `score-command#ac:no-new-config-keys` |
| [missing-gates-block-graceful.md](missing-gates-block-graceful.md) | `score-command#ac:missing-gates-block-graceful` |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/scenarios-index-specification*
