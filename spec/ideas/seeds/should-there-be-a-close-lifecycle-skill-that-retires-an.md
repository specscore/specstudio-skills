---
captured_by: user
status: queued
---
# Should there be a close lifecycle skill that retires an Idea or seed via the CLI change-status verb instead of manual frontmatter edits

## Question

Should SpecStudio have a dedicated **close** lifecycle skill (parallel to ship, recap, plan, implement) that retires an artifact when its work is done?

Triggered while triaging the seed queue: several seeds were verified as implemented, but there was no clean way to mark them closed. Manual frontmatter edits (status: completed) violate the SpecScore tenet that ALL status transitions go through a CLI verb — see companion seeds audit-all-specscore-skills-for-manual-status-edit and update-ideate-and-specify-skills-to-transition-artifact.

## Hard requirement

A close skill MUST change status via the CLI (specscore idea|feature change-status --to=<status>), never by hand-editing the **Status:** line or frontmatter. It would orchestrate: pick the right artifact + target status, invoke the CLI verb, re-lint, emit the lifecycle event, apply publication policy.

## Decision (2026-06-16)

Yes — close MUST cover seeds, not just Ideas/Features: closing a seed moves it to implemented or archived. Requires changes in BOTH layers — the seed change-status/close verb in specscore-cli (tracked: specscore/specscore-cli#72) AND the close skill here that drives it.

## Open questions

- Is close a new skill, or just guidance folded into existing skills (ship already does Implementing -> Stable for Features)?
- Terminal vocabulary for 'closed because implemented' vs archived/deprecated/promoted.

## Provenance

Surfaced 2026-06-16 during seed-queue triage in specstudio-skills.
