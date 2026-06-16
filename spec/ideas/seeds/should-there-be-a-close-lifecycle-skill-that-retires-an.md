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

## Open questions

- Scope: Ideas + Features only (CLI verbs exist), or also seeds? Seeds currently have NO CLI change-status verb (see companion seed cli-and-sidekick-skill-need-a-seed-change-status-close-verb) — a close skill for seeds is blocked until that ships.
- Is close a new skill, or just guidance folded into existing skills (ship already does Implementing -> Stable for Features)?
- Terminal vocabulary for 'closed because implemented' vs archived/deprecated/promoted.

## Provenance

Surfaced 2026-06-16 during seed-queue triage in specstudio-skills.
