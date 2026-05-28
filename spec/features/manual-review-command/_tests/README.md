# Scenarios — `manual-review-command`

Scenarios validating the [Manual Review Command feature](../README.md). Each scenario references the AC it validates and uses `Given / When / Then` form per the [SpecScore Scenario specification](https://specscore.md/scenario-specification).

All scenarios are scaffolded with `**Status:** pending`; their concrete Given/When/Then steps are authored during the implementation plan, not at spec time.

## Index

| Scenario | Validates |
|---|---|
| [invocation-triggers-respond.md](invocation-triggers-respond.md) | `manual-review-command#ac:invocation-triggers-respond` |
| [unknown-flag-refused.md](unknown-flag-refused.md) | `manual-review-command#ac:unknown-flag-refused` |
| [empty-paths-defaults-to-spec.md](empty-paths-defaults-to-spec.md) | `manual-review-command#ac:empty-paths-defaults-to-spec` |
| [empty-paths-recursive-walks-tree.md](empty-paths-recursive-walks-tree.md) | `manual-review-command#ac:empty-paths-recursive-walks-tree` |
| [directory-uses-readme.md](directory-uses-readme.md) | `manual-review-command#ac:directory-uses-readme` |
| [directory-without-readme-skipped.md](directory-without-readme-skipped.md) | `manual-review-command#ac:directory-without-readme-skipped` |
| [glob-unmatched-refused.md](glob-unmatched-refused.md) | `manual-review-command#ac:glob-unmatched-refused` |
| [deduplication-preserves-first-order.md](deduplication-preserves-first-order.md) | `manual-review-command#ac:deduplication-preserves-first-order` |
| [stage-mapping-resolves-correctly.md](stage-mapping-resolves-correctly.md) | `manual-review-command#ac:stage-mapping-resolves-correctly` |
| [index-readme-skipped.md](index-readme-skipped.md) | `manual-review-command#ac:index-readme-skipped` |
| [stage-without-gate-skipped.md](stage-without-gate-skipped.md) | `manual-review-command#ac:stage-without-gate-skipped` |
| [human-reviewers-silently-omitted.md](human-reviewers-silently-omitted.md) | `manual-review-command#ac:human-reviewers-silently-omitted` |
| [all-human-list-reports-no-verdict.md](all-human-list-reports-no-verdict.md) | `manual-review-command#ac:all-human-list-reports-no-verdict` |
| [diff-supplied-to-ai-with-default-ref.md](diff-supplied-to-ai-with-default-ref.md) | `manual-review-command#ac:diff-supplied-to-ai-with-default-ref` |
| [against-custom-ref-applied.md](against-custom-ref-applied.md) | `manual-review-command#ac:against-custom-ref-applied` |
| [invalid-ref-falls-back-to-empty-diff.md](invalid-ref-falls-back-to-empty-diff.md) | `manual-review-command#ac:invalid-ref-falls-back-to-empty-diff` |
| [verdict-parity-with-specify.md](verdict-parity-with-specify.md) | `manual-review-command#ac:verdict-parity-with-specify` |
| [confirm-at-threshold-prompts.md](confirm-at-threshold-prompts.md) | `manual-review-command#ac:confirm-at-threshold-prompts` |
| [confirm-at-threshold-below-bound.md](confirm-at-threshold-below-bound.md) | `manual-review-command#ac:confirm-at-threshold-below-bound` |
| [yes-flag-skips-confirm.md](yes-flag-skips-confirm.md) | `manual-review-command#ac:yes-flag-skips-confirm` |
| [serial-across-artifacts.md](serial-across-artifacts.md) | `manual-review-command#ac:serial-across-artifacts` |
| [single-artifact-detailed-output.md](single-artifact-detailed-output.md) | `manual-review-command#ac:single-artifact-detailed-output` |
| [multi-artifact-default-summary-mode.md](multi-artifact-default-summary-mode.md) | `manual-review-command#ac:multi-artifact-default-summary-mode` |
| [multi-artifact-verbose-mode.md](multi-artifact-verbose-mode.md) | `manual-review-command#ac:multi-artifact-verbose-mode` |
| [skipped-artifacts-in-table.md](skipped-artifacts-in-table.md) | `manual-review-command#ac:skipped-artifacts-in-table` |
| [exit-code-success.md](exit-code-success.md) | `manual-review-command#ac:exit-code-success` |
| [exit-code-failure.md](exit-code-failure.md) | `manual-review-command#ac:exit-code-failure` |
| [exit-code-all-skipped.md](exit-code-all-skipped.md) | `manual-review-command#ac:exit-code-all-skipped` |
| [no-files-written.md](no-files-written.md) | `manual-review-command#ac:no-files-written` |
| [no-new-config-keys.md](no-new-config-keys.md) | `manual-review-command#ac:no-new-config-keys` |
| [missing-gates-block-graceful.md](missing-gates-block-graceful.md) | `manual-review-command#ac:missing-gates-block-graceful` |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/scenarios-index-specification*
