---
type: rehearse-stub
status: pending
ac: back-link-appended-on-capture
feature: sidekick-capture
---

# Rehearse: back-link-appended-on-capture

## Scenario (from AC)

**Given** an existing Feature at `spec/features/foo/README.md` containing a SpecScore footer line, and a `/sidekick` invocation with `captured_during: spec/features/foo`
**When** the skill writes the seed successfully
**Then** the Feature's README is modified to contain a `## Sidekick Seeds Generated` section (created if absent) positioned immediately before the footer line; the section contains a new entry referencing the new seed in the format `- [<slug>](<relative path>) — captured <ISO-8601 date> by <captured_by>`; no other content in the source artifact is modified.

## Verification approach

Pre-create a fixture Feature at `spec/features/foo/README.md` containing a SpecScore footer line and no existing back-link section. Invoke the skill with `captured_during=spec/features/foo` and a known one-liner. Assert: (a) the seed file exists, (b) the Feature's README now contains a `## Sidekick Seeds Generated` section positioned immediately before the footer line, (c) the section contains exactly one entry in the format `- [<slug>](../../ideas/seeds/<slug>.md) — captured <YYYY-MM-DD> by <captured_by>`, (d) no other content in the Feature's README has changed (diff outside the new section is empty).

---
*This document follows the https://specscore.md/scenario-specification*
