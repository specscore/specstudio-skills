---
captured_by: user
status: queued
---
# Update ideate and specify skills to transition artifact status via the change-status CLI instead of manual file edits

Surfaced while running specstudio:ideate then specify on an external project. Both skills in THIS repo (skills/ideate/SKILL.md, skills/specify/SKILL.md) instruct the agent to change artifact Status by hand-editing the body **Status:** line (e.g. ideate: 'Update Status: Draft -> Approved in the body metadata'). That conflicts with the skills' own 'prefer stable CLI contracts over ad-hoc file writes' tenet and causes frontmatter/body/index drift -> lint violations (status-mirror, idea-index-row-sync) needing migrate/--fix to recover. Observed twice in one session.

Fix: replace manual-status-edit guidance with the dedicated verbs 'specscore idea change-status <slug> --to=<status>' and 'specscore feature change-status <id> --to=<status>'. Companion seed in ai-plugin-specscore covers the idea skill there.
