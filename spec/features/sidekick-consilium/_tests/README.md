---
format: https://specscore.md/scenarios-index-specification
---

# Rehearse stubs — `sidekick-consilium` Feature

Rehearse stubs validating the [Sidekick Consilium feature](../README.md). Each stub references an Acceptance Criterion from the Feature spec, copies its Given/When/Then verbatim, and sketches a 2–4-sentence verification approach. Stubs land with `status: pending`; authoring the actual scenario steps follows the implementation plan.

## Index

| Stub | Validates |
|---|---|
| [invocation-drains-all-queued-tasks.md](invocation-drains-all-queued-tasks.md) | `sidekick-consilium#ac:invocation-drains-all-queued-tasks` |
| [invocation-rejects-per-slug-argument.md](invocation-rejects-per-slug-argument.md) | `sidekick-consilium#ac:invocation-rejects-per-slug-argument` |
| [pipeline-runs-five-stages-in-order.md](pipeline-runs-five-stages-in-order.md) | `sidekick-consilium#ac:pipeline-runs-five-stages-in-order` |
| [token-usage-recorded-on-task.md](token-usage-recorded-on-task.md) | `sidekick-consilium#ac:token-usage-recorded-on-task` |
| [seed-mutation-blocks-review.md](seed-mutation-blocks-review.md) | `sidekick-consilium#ac:seed-mutation-blocks-review` |
| [researcher-briefing-contains-no-judgment.md](researcher-briefing-contains-no-judgment.md) | `sidekick-consilium#ac:researcher-briefing-contains-no-judgment` |
| [every-expert-receives-briefing-and-may-research-deeper.md](every-expert-receives-briefing-and-may-research-deeper.md) | `sidekick-consilium#ac:every-expert-receives-briefing-and-may-research-deeper` |
| [scribe-summary-respects-flavor-and-length.md](scribe-summary-respects-flavor-and-length.md) | `sidekick-consilium#ac:scribe-summary-respects-flavor-and-length` |
| [scribe-verdict-field-ignored.md](scribe-verdict-field-ignored.md) | `sidekick-consilium#ac:scribe-verdict-field-ignored` |
| [default-roster-is-9-roles-in-three-groups.md](default-roster-is-9-roles-in-three-groups.md) | `sidekick-consilium#ac:default-roster-is-9-roles-in-three-groups` |
| [panel-fans-out-in-parallel.md](panel-fans-out-in-parallel.md) | `sidekick-consilium#ac:panel-fans-out-in-parallel` |
| [malformed-vote-fails-pipeline.md](malformed-vote-fails-pipeline.md) | `sidekick-consilium#ac:malformed-vote-fails-pipeline` |
| [abstain-high-confidence-excluded-from-denominator.md](abstain-high-confidence-excluded-from-denominator.md) | `sidekick-consilium#ac:abstain-high-confidence-excluded-from-denominator` |
| [abstain-low-confidence-caps-verdict.md](abstain-low-confidence-caps-verdict.md) | `sidekick-consilium#ac:abstain-low-confidence-caps-verdict` |
| [custom-role-loads-and-votes.md](custom-role-loads-and-votes.md) | `sidekick-consilium#ac:custom-role-loads-and-votes` |
| [roster-with-malformed-custom-role-rejected.md](roster-with-malformed-custom-role-rejected.md) | `sidekick-consilium#ac:roster-with-malformed-custom-role-rejected` |
| [roster-violating-group-floor-rejected.md](roster-violating-group-floor-rejected.md) | `sidekick-consilium#ac:roster-violating-group-floor-rejected` |
| [roster-snapshot-stored-on-task.md](roster-snapshot-stored-on-task.md) | `sidekick-consilium#ac:roster-snapshot-stored-on-task` |
| [pipeline-transcript-payload-shape.md](pipeline-transcript-payload-shape.md) | `sidekick-consilium#ac:pipeline-transcript-payload-shape` |
| [verdict-task-payload-completeness.md](verdict-task-payload-completeness.md) | `sidekick-consilium#ac:verdict-task-payload-completeness` |
| [seed-gets-consilium-verdict-section.md](seed-gets-consilium-verdict-section.md) | `sidekick-consilium#ac:seed-gets-consilium-verdict-section` |
| [reviewed-event-emitted-on-success.md](reviewed-event-emitted-on-success.md) | `sidekick-consilium#ac:reviewed-event-emitted-on-success` |
| [arbiter-reproducibility-snapshot.md](arbiter-reproducibility-snapshot.md) | `sidekick-consilium#ac:arbiter-reproducibility-snapshot` |
| [idempotent-task-creation-on-duplicate-hash.md](idempotent-task-creation-on-duplicate-hash.md) | `sidekick-consilium#ac:idempotent-task-creation-on-duplicate-hash` |
| [concurrent-claim-loses-cleanly.md](concurrent-claim-loses-cleanly.md) | `sidekick-consilium#ac:concurrent-claim-loses-cleanly` |

## Skipped ACs

See [_skipped.md](_skipped.md) for the `calibration-set-passes-95-percent` AC, which is a manual quality gate on the Phase 1 ship decision rather than a runtime observable.

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/scenarios-index-specification*
