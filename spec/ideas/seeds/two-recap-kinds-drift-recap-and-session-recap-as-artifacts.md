---
type: sidekick-seed
slug: two-recap-kinds-drift-recap-and-session-recap-as-artifacts
captured_at: 2026-06-01T13:38:48Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# Two distinctly-named recap kinds — a code↔spec "drift recap" and a new "session recap" — saved as SpecScore artifacts

`specstudio:recap` today produces one kind: a per-AC **drift** analysis (spec vs implemented code; verdicts no-drift / spec-tighter / code-tighter / contradiction) at `spec/features/<slug>/_recap/<sha>.md`.

Users want a second, distinct kind with no home today: a **session recap** — a curated narrative of a working session (goal, decisions, scope changes, what shipped, commits, verification, follow-ups). Surfaced after a long synchestra-ws static→Astro migration where a useful chat recap was produced but never persisted.

Two artifacts, different inputs/audiences → they need **distinctive names**:

- **Drift recap** (existing): spec↔code at a SHA; input = Feature + verify report. Candidate: keep `recap` or `recap:drift` / "conformance recap".
- **Session recap** (new): narrative of a session, not tied to one Feature's ACs; input = commits/events/decisions. Candidate: `recap:session` / "session recap" / "worklog".

**Where/how saved?** Likely first-class SpecScore artifacts. Open questions:

- Keep drift recap at `spec/features/<slug>/_recap/<sha>.md`.
- Session recap: standalone (`spec/recaps/<date>-<slug>.md`) or a curated rollup over the event journal? Decide: new artifact type, special journal summary, or both.
- Attach session recap to a Plan/Feature, or standalone (a session may span many features or none)?
- CLI/skill surface: one `recap` with subcommands vs two skills.

**See also:** specscore-core `journal-and-summary.md` (overlapping storage: event journal + rollups — decide session-recap alongside it); seeds `proposal-aware-plan-verify-recap-skills.md`, `recap-drift-surface-near-zero...`.

Note: captured from a non-SpecScore repo — session-recap input may span repos that aren't SpecScore-managed.
