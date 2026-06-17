---
type: rehearse-stub
stub_status: pending
ac: lint-rejects-malformed-seed
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: lint-rejects-malformed-seed

## Scenario (from AC)

**Given** a seed file with any of: (a) an unknown frontmatter key, (b) a missing required key, (c) `type` other than `sidekick-seed`, (d) `trigger` outside the enumerated set, (e) a body whose first non-blank line is not an H1 heading, (f) a `queued` body exceeding 3000 characters or a closed/terminal-status body exceeding 5000 characters
**When** `specscore spec lint` is run on the project
**Then** lint reports a violation pointing at the offending file and the specific rule fired; exit code is non-zero (per the SpecScore CLI exit-code contract).

## Verification approach

Write six fixture seeds, each triggering one of the rejection conditions (a–f from REQ `seed-lint-rule`); run `specscore spec lint`; assert non-zero exit and one violation per fixture; assert the violation message references the specific rule.

---
*This document follows the https://specscore.md/scenario-specification*
