---
captured_by: user
status: queued
---

# Plan, verify, and recap skills should be proposal-aware (change-request ideas update Feature definitions)

When a change-request idea (proposal) reaches Implemented, the target Feature README must be updated to incorporate the behavioral changes described by the proposal (REQ: proposal-merge-on-implementation in the idea Feature spec). This has implications for three SpecStudio skills:

1. **specstudio:plan** — When planning implementation of an approved change-request idea, the planner MUST include a task for updating the target Feature's README with the new behavior. This task should be sequenced after implementation and before verification.

2. **specstudio:verify** — When verifying a change-request idea's implementation, the verify skill MUST check that the target Feature README has been updated to reflect the proposal's changes. A proposal at Implemented without corresponding Feature README updates is a verification gap.

3. **specstudio:recap** — When recapping a change-request idea, the recap skill MUST compare both the proposal's AC against the implementation AND the updated Feature README against the proposal. Drift between the Feature README and the implemented proposal content is a recap finding.

This ensures the full lifecycle closes: proposal describes desired change → implementation lands → Feature README is updated → verify/recap confirm both the code and the spec are consistent.
