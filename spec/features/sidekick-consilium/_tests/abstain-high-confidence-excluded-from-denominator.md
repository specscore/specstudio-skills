---
type: rehearse-stub
status: pending
ac: abstain-high-confidence-excluded-from-denominator
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: abstain-high-confidence-excluded-from-denominator

## Scenario (from AC)

**Given** a panel where one customer role returns `verdict: abstain, confidence: high` and the other two customers return `verdict: should-implement, confidence: medium`
**When** the arbiter computes the verdict with `require_all_customers: true`
**Then** the abstaining customer is excluded from `denominators.customers` (which is now 2); both remaining customers approve, so the customer gate passes; `excluded_votes` in the arbiter output contains the abstaining role's slug.

## Verification approach

Construct a fixture vote bundle matching the Given precisely; invoke `specscore consilium verdict` directly with `require_all_customers: true`; assert the stdout YAML shows `denominators.customers: 2`, the customer gate passes, and `excluded_votes` includes the abstaining role's slug.

---
*This document follows the https://specscore.md/scenario-specification*
