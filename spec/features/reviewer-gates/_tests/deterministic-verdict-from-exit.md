# Scenario: a `type: deterministic` reviewer's verdict is derived from its command's exit code

**Status:** pending
**Validates:** [reviewer-gates#ac:deterministic-verdict-from-exit](../README.md#ac-deterministic-verdict-from-exit)

## Steps

GIVEN a gate `reviewers:` list with a `type: deterministic` entry whose `run:` command exits non-zero, gated at the default threshold `B`
WHEN the gate is evaluated (runner Step 2-det runs the command and maps the exit code to a verdict)
THEN that reviewer's verdict is `Issues Found`
AND the command's diagnostic output is captured as at least one `Blocker` finding
AND the gate does NOT release at the default threshold `B` (the `Blocker` yields grade ≤ `C` < `B`)
AND GIVEN the same command exits zero
WHEN the gate is re-evaluated
THEN that reviewer contributes `Approved` with no findings

---
*This document follows the https://specscore.md/scenario-specification*
