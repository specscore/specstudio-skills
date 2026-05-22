# Scenarios — `specify` skill

Scenarios validating the [Specify Skill feature](../README.md). Each scenario references the AC or REQ it validates and uses `Given / When / Then` form per the [SpecScore Scenario specification](https://specscore.md/scenario-specification).

## Index

| Scenario | Validates |
|---|---|
| [hard-gate-blocks-writing-plans-without-approval.md](hard-gate-blocks-writing-plans-without-approval.md) | `specify#ac:hard-gate-enforced` |
| [auto-create-spec-features-dir.md](auto-create-spec-features-dir.md) | `specify#req:auto-create-features-dir` |
| [auto-stage-feature-files-in-git.md](auto-stage-feature-files-in-git.md) | `specify#req:auto-stage-on-create` |
| [source-ideas-references-must-resolve.md](source-ideas-references-must-resolve.md) | `specify#ac:source-idea-linkage` |
| [related-idea-surfaced-on-clear-intent.md](related-idea-surfaced-on-clear-intent.md) | `specify#req:related-idea-surfacing` |
| [requirements-need-given-when-then-acs.md](requirements-need-given-when-then-acs.md) | `specify#req:ac-format` |
| [reviewer-subagent-must-approve-before-user.md](reviewer-subagent-must-approve-before-user.md) | `specify#ac:reviewer-then-user` |
| [baseline-reviewer-enforces-default-blockers.md](baseline-reviewer-enforces-default-blockers.md) | `specify#req:reviewer-baseline-blockers` |
| [additional-reviewer-veto-blocks-approval.md](additional-reviewer-veto-blocks-approval.md) | `specify#req:reviewer-composition` |
| [rehearse-decision-recorded-per-ac.md](rehearse-decision-recorded-per-ac.md) | `specify#ac:rehearse-coverage` |
| [feature-updated-event-on-post-approval-edit.md](feature-updated-event-on-post-approval-edit.md) | `specify#ac:lifecycle-events` |
| [transition-only-to-writing-plans.md](transition-only-to-writing-plans.md) | `specify#ac:promotion-boundary-held` |
| [revise-in-place-vs-supersede.md](revise-in-place-vs-supersede.md) | `specify#req:revise-vs-supersede` |

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/scenarios-index-specification*
