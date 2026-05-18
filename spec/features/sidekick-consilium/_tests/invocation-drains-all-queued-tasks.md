---
type: rehearse-stub
status: pending
ac: invocation-drains-all-queued-tasks
feature: sidekick-consilium
---

# Rehearse: invocation-drains-all-queued-tasks

## Scenario (from AC)

**Given** a project with N ≥ 2 `consilium-review` tasks in status `queued`
**When** the user invokes `/consilium`
**Then** all N tasks transition through `claimed → in_review → complete` (or `failed`); the skill returns one verdict per task; no `queued` tasks remain at the end of the run.

## Verification approach

Seed a fixture project with two or more `consilium-review` tasks in `queued` status and matching seed files; invoke `/consilium`; after the run, assert via `synchestra:task` listing that zero queued `consilium-review` tasks remain and each prior task has reached `complete` or `failed`. Capture the skill's stdout to confirm one verdict line per task.

---
*This document follows the https://specscore.md/scenario-specification*
