---
type: rehearse-stub
stub_status: pending
ac: back-link-write-failure-does-not-roll-back-seed
feature: sidekick-capture
format: https://specscore.md/scenario-specification
---

# Rehearse: back-link-write-failure-does-not-roll-back-seed

## Scenario (from AC)

**Given** a source artifact at the resolved `captured_during` path that exists but cannot be written (e.g., read-only filesystem or insufficient permission on that single file)
**When** the skill is invoked with a valid one-liner
**Then** the seed file is still written at `spec/ideas/seeds/<slug>.md`; the `sidekick-idea.captured` event is still emitted; the skill reports the back-link write failure to the caller as a warning; the skill's exit code is 0 (success), because the seed and event — the load-bearing artifacts — are correctly written.

## Verification approach

Pre-create a fixture Feature and `chmod 0444` its README.md (or otherwise make it unwritable while the parent dir remains writable). Invoke the skill pointing at it. Assert: seed file exists, event emitted, warning surfaced about back-link failure (substring "back-link write to <source-path> failed"), exit code 0. Reset permissions after the test.

---
*This document follows the https://specscore.md/scenario-specification*
