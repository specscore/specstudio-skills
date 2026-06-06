---
format: https://specscore.md/scenario-specification
---

# Scenario: a gate on a multi-occurrence gate-point event is evaluated independently per occurrence

**Status:** pending
**Validates:** [reviewer-gates#ac:pre-commit-gate-fires-per-occurrence](../README.md#ac-pre-commit-gate-fires-per-occurrence)

## Steps

GIVEN a gate keyed on the multi-occurrence gate-point event `implementation.pre_commit` and a run in which that event occurs three times (simulated via the gate-runner harness)
WHEN the runner processes the run (runner Step 7 — each occurrence is a fresh gate run from Step 0 with an empty previous-pass verdict map)
THEN the gate is evaluated three times — once per occurrence
AND each evaluation dispatches the gate's reviewers and produces its own independent verdict
AND there is no single-shot-per-run caching (no verdict is memoized and reused across occurrences)

---
*This document follows the https://specscore.md/scenario-specification*
