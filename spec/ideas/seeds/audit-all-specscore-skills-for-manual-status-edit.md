---
captured_by: user
status: queued
---
# Audit all SpecScore skills for manual status-edit instructions and route every status change through the change-status CLI

Companion to the idea/ideate change-status seed. Sweep every skill in the SpecScore skill repos (ai-plugin-specscore, specstudio-skills: ideate, specify, plan, implement, ship, recap, verify, etc.) for any guidance that changes an artifact's Status by hand-editing the file. Replace each with the dedicated CLI verb (specscore idea|feature change-status --to=<status>).

Establish the invariant: ALL SpecScore artifact status transitions go through the dedicated skill + CLI, never ad-hoc file writes. Rationale: manual edits desync the frontmatter mirror, body **Status:** line, and parent index rows, producing lint errors and forcing migrate/--fix recovery.
