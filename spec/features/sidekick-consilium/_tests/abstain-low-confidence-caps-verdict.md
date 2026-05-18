---
type: rehearse-stub
status: pending
ac: abstain-low-confidence-caps-verdict
feature: sidekick-consilium
---

# Rehearse: abstain-low-confidence-caps-verdict

## Scenario (from AC)

**Given** a panel where one role returns `verdict: abstain, confidence: low` and all other votes are `should-implement` with high confidence
**When** the arbiter computes the verdict
**Then** the verdict is `needs-human-review` (rule `low-abstain-veto` fires); the rule_trace records this; the strict-gate path is not evaluated.

## Verification approach

Build a fixture vote bundle where exactly one vote is `verdict: abstain, confidence: low` and the rest are `should-implement` at high confidence; invoke `specscore consilium verdict`; assert the stdout verdict is `needs-human-review`, `rule_trace` lists `low-abstain-veto`, and the trace shows the strict-gate evaluation was short-circuited.

---
*This document follows the https://specscore.md/scenario-specification*
