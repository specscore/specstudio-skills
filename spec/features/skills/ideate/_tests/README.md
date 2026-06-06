---
format: https://specscore.md/scenarios-index-specification
---

# Scenarios — `ideate` skill

Scenarios validating the [Ideate Skill feature](../README.md). Each scenario references the AC or REQ it validates and uses `Given / When / Then` form per the [SpecScore Scenario specification](https://specscore.md/scenario-specification).

## Index

| Scenario | Validates |
|---|---|
| [hard-gate-blocks-specify-without-approval.md](hard-gate-blocks-specify-without-approval.md) | `ideate#ac:hard-gate-enforced` |
| [empty-not-doing-rejected.md](empty-not-doing-rejected.md) | `ideate#req:not-doing-required` |
| [docs-ideas-path-rejected.md](docs-ideas-path-rejected.md) | `ideate#req:no-docs-path` |
| [auto-create-spec-ideas-dir.md](auto-create-spec-ideas-dir.md) | `ideate#req:auto-create-ideas-dir` |
| [auto-stage-created-files-in-git.md](auto-stage-created-files-in-git.md) | `ideate#req:auto-stage-on-create` |
| [cli-path-preferred-when-available.md](cli-path-preferred-when-available.md) | `ideate#ac:cli-vs-fallback` |
| [fallback-when-cli-missing.md](fallback-when-cli-missing.md) | `ideate#ac:cli-vs-fallback` |
| [approval-emits-idea-approved-event.md](approval-emits-idea-approved-event.md) | `ideate#ac:lifecycle-events` |
| [post-approval-edit-emits-idea-updated.md](post-approval-edit-emits-idea-updated.md) | `ideate#ac:lifecycle-events` |
| [event-payload-carries-change-context.md](event-payload-carries-change-context.md) | `ideate#req:event-payload-change-context` |
| [change-summary-stays-factual.md](change-summary-stays-factual.md) | `ideate#req:change-summary-discipline` |
| [explicit-approve-phrase-transitions-without-confirmation.md](explicit-approve-phrase-transitions-without-confirmation.md) | `ideate#ac:approval-detection` |
| [non-english-approval-recognized.md](non-english-approval-recognized.md) | `ideate#req:approval-explicit-phrase` |
| [vague-positive-signal-requires-confirmation.md](vague-positive-signal-requires-confirmation.md) | `ideate#ac:approval-detection` |
| [lint-fix-attempted-then-surfaces-remaining.md](lint-fix-attempted-then-surfaces-remaining.md) | `ideate#ac:lint-failure-recovery` |
| [clear-intent-hands-off-to-specify.md](clear-intent-hands-off-to-specify.md) | `ideate#ac:skip-condition-respected` |
| [manual-promotes-to-edit-rejected.md](manual-promotes-to-edit-rejected.md) | `ideate#ac:promotion-boundary-held` |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/scenarios-index-specification*
