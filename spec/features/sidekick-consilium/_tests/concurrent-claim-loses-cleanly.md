---
type: rehearse-stub
status: pending
ac: concurrent-claim-loses-cleanly
feature: sidekick-consilium
---

# Rehearse: concurrent-claim-loses-cleanly

## Scenario (from AC)

**Given** two concurrent `/consilium` invocations claiming the same task
**When** both reach the claim step
**Then** exactly one invocation transitions the task `queued → claimed` and proceeds; the other observes `claimed` and skips the task without retry or error.

## Verification approach

Launch two `/consilium` invocations concurrently against a fixture project containing exactly one queued task; assert exactly one invocation proceeds to run the pipeline and the other exits cleanly (zero exit code, no error) after observing the task as already claimed. Confirm via `synchestra:task` that the task transitioned through `claimed → complete` (or `failed`) once and only once.

---
*This document follows the https://specscore.md/scenario-specification*
