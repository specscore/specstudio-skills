---
type: rehearse-stub
status: pending
ac: arbiter-reproducibility-snapshot
feature: sidekick-consilium
format: https://specscore.md/scenario-specification
---

# Rehearse: arbiter-reproducibility-snapshot

## Scenario (from AC)

**Given** a fixture vote bundle + roster + gate config that the gate rules deterministically produce `should-implement` for
**When** `specscore consilium verdict --votes votes.yaml --roster roster.yaml --gate gate.yaml --seed seed.md` is invoked twice
**Then** both invocations produce identical stdout YAML (verdict, rule_trace, excluded_votes, denominators all byte-identical); exit code 0 both times.

## Verification approach

Build a fixture set of `votes.yaml`, `roster.yaml`, `gate.yaml`, and `seed.md` known to resolve to `should-implement`; invoke `specscore consilium verdict ...` twice and capture stdout; assert exit code 0 both times and byte-equal stdout. Diff the two outputs explicitly to surface any non-deterministic ordering.

---
*This document follows the https://specscore.md/scenario-specification*
