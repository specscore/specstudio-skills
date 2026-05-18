---
type: rehearse-stub
status: pending
ac: seed-gets-consilium-verdict-section
feature: sidekick-consilium
---

# Rehearse: seed-gets-consilium-verdict-section

## Scenario (from AC)

**Given** a seed at `spec/ideas/seeds/<slug>.md` and a successful pipeline completion against it
**When** the seed file is read after the pipeline
**Then** the seed contains a `## Consilium Verdict` section positioned immediately before the SpecScore footer line; the section's first line is `**Verdict:** <verdict-enum> (<YYYY-MM-DD>)`; the second line links to the synchestra task; the rest is the scribe's prose paragraph.

## Verification approach

Place a fixture seed with a SpecScore footer line into `spec/ideas/seeds/`; run the pipeline against the matching queued task; re-read the seed file and assert a `## Consilium Verdict` section appears immediately before the footer. Assert the section's first line matches `^**Verdict:** (should-implement|should-not-implement|needs-human-review) \(\d{4}-\d{2}-\d{2}\)$`, the second line is a markdown link to the synchestra task, and the remaining lines equal the task's `scribe_summary`.

---
*This document follows the https://specscore.md/scenario-specification*
