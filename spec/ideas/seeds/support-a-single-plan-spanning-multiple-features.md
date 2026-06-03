---
type: sidekick-seed
slug: support-a-single-plan-spanning-multiple-features
captured_at: 2026-06-03T12:00:00Z
captured_by: specstudio:plan
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---
# Support a single Feature-sourced Plan spanning multiple Features

**Problem.** A SpecScore Plan is single-source (`**Source Feature:**` = one Feature). Tightly-coupled clusters — e.g. a parent engine Feature plus thin verb child Features that ship together and share one package — must be split into N Plans, losing one coherent build sequence. Surfaced while planning the `cli/consilium` group (parent + verdict/roster/config) in specscore-cli.

**What blocks it:** (1) single-valued `**Source Feature:**` field; (2) lint `P-001` AC-coverage validates against one source — would need to union ACs across all declared sources; (3) reviewer blocker #5 "hidden multi-source scope"; (4) 1:1 Feature↔Plan lifecycle/status sync; (5) plans index renders one source.

**Already supports it:** AC IDs are Feature-namespaced (`<feature-slug>#ac:<slug>`), explicitly "so they are unambiguous when planning across Features in the future"; per-task `**Depends-On:**` already exists for cross-task ordering.

**Effort:** contained — schema + `P-001` + source-resolution + lifecycle-sync in specscore-cli; relax the single-source refusal + reviewer scope in specstudio-skills. Small Feature, not an epic.

**Alternative that already exists:** Idea-sourced Plans span multiple Features (via the promoting Idea) with no AC-coverage contract — the current intended mechanism, trading coverage rigor for grouping ergonomics.

Needs ideate with options + recommendation.
