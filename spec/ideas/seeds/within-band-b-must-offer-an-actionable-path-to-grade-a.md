---
type: sidekick-seed
slug: within-band-b-must-offer-an-actionable-path-to-grade-a
captured_at: 2026-06-02T19:48:28Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# Within-band B must offer an actionable path to grade A

Amend the within-band rubric in `reviewer-gates` (`multi-role-reviewer` REQ) and `skills/specify/references/reviewer-prompt.md`: assigning within-band **B** MUST include ≥1 concrete, actionable improvement that would raise the artifact to **A**; if the reviewer cannot name one, it MUST award **A**. This makes the pass band symmetric with the actionable fail band (every `Blocker` is a fix) and eliminates non-actionable "dense, bar met not exceeded" B verdicts. Surfaced while re-grading the reviewer-gates event-keyed revision, which returned B with zero advisories and no stated path to A.
