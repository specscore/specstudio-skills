---
format: https://specscore.md/scenario-specification
---

# Scenario: Clear-intent path surfaces related Approved Ideas before writing the Feature

**Validates:** [specify#req:related-idea-surfacing](../README.md#req-related-idea-surfacing)

## Steps

GIVEN the project has an Approved Idea at `spec/ideas/payment-fraud.md` with HMW "How might we detect payment fraud signals during checkout?"
AND no other Ideas in the project are topically related
AND the user invokes `specstudio:specify` on the clear-intent path with intent "I want a Feature that scores card transactions for fraud risk and rejects high-risk ones"
WHEN the skill processes the invocation
THEN the skill scans `spec/ideas/` for Approved Ideas
AND the skill surfaces `payment-fraud` to the user with its slug and HMW
AND the skill asks the user whether to link, waive, or revise
AND the skill MUST NOT write the Feature until the user responds

GIVEN the user chooses to link by responding "link payment-fraud"
WHEN the skill writes the Feature
THEN the Feature `README.md` contains `**Source Ideas:** payment-fraud` immediately after the Status line
AND the skill acknowledges the link in its response

GIVEN the user chooses to waive by responding "skip linking; this is a different angle"
WHEN the skill writes the Feature
THEN the Feature is written with NO `**Source Ideas:**` line
AND the skill acknowledges the waiver in its response (e.g., "Proceeding without linking payment-fraud")
AND the existing Approved Idea is left unchanged (no orphan side-effect)

GIVEN the project has zero Approved Ideas semantically related to the user's intent
WHEN the skill processes the invocation
THEN the skill MUST NOT prompt the user with empty candidate lists
AND the skill proceeds to write the Feature directly on the clear-intent path

GIVEN the user invokes `specstudio:specify` with an explicit Idea path (e.g., points the skill at `spec/ideas/payment-fraud.md`)
WHEN the skill processes the invocation
THEN the skill MUST NOT run the related-Idea scan
AND the skill MUST link the named Idea directly via `**Source Ideas:**`
AND the related-idea-surfacing flow is bypassed because the user has already declared intent

---
*This document follows the https://specscore.md/scenario-specification*
