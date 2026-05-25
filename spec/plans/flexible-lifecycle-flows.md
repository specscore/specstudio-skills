# Plan: Flexible Lifecycle Flows

**Status:** Approved
**Source Feature:** flexible-lifecycle-flows
**Date:** 2026-05-25
**Owner:** alexander.trakhimenok
**Supersedes:** —

## Summary

Decomposes the flexible-lifecycle-flows Feature into 5 ordered tasks: event vocabulary first (shared dependency), then the two transition menus (ideate, specify), then the two entry-gate relaxations (plan, implement). Each task modifies one skill's `skill.md` file plus the shared event reference.

## Approach

Ordered by dependency: the event schema must exist before any skill can emit it; the transition menus must exist before downstream skills can accept non-standard sources (plan can't accept an Idea handoff until ideate offers that path). The suggestion heuristics and config key are bundled into the menu tasks since they're part of the same `AskUserQuestion` presentation logic.

## Tasks

### Task 1: Add lifecycle.phase-skipped event to shared/events.md

**Verifies:** flexible-lifecycle-flows#ac:skip-event-emitted, flexible-lifecycle-flows#ac:skip-event-not-emitted-on-default

Add the `lifecycle.phase-skipped` event definition to `skills/shared/events.md` following the existing event envelope format. Document payload fields (`from_phase`, `skipped_phases`, `to_phase`, `reason`, `source_artifact`), emission rules (emitted by the menu-presenting skill, before downstream invocation, only on non-default choices), and add it to the Reference table.

### Task 2: Add transition menu to ideate skill

**Verifies:** flexible-lifecycle-flows#ac:ideate-presents-three-options, flexible-lifecycle-flows#ac:ideate-routes-to-plan, flexible-lifecycle-flows#ac:ideate-routes-to-implement

Modify `skills/ideate/skill.md`: replace the single-path "Promotion to Feature(s)" / transition section with a multi-path transition menu presented via `AskUserQuestion` (Specify default, Plan, Implement). Add the keyword-based heuristic that suggests "Implement" when the user's language indicates trivial scope. Emit `lifecycle.phase-skipped` on non-default choices. Update the Hard Gate to allow invoking `specstudio:plan` and `specstudio:implement` in addition to `specstudio:specify`.

### Task 3: Add transition menu to specify skill

**Verifies:** flexible-lifecycle-flows#ac:specify-presents-two-options, flexible-lifecycle-flows#ac:specify-routes-to-implement, flexible-lifecycle-flows#ac:suggest-skip-plan-on-small-feature, flexible-lifecycle-flows#ac:suggestion-suppressed-by-config

Modify `skills/specify/skill.md`: replace the single-path "Transition to Implementation" section with a two-option menu (Plan default, Implement). Add the AC-count heuristic (≤2 ACs → suggest skip-plan). Implement the `lifecycle.suggest_skips` config check from `specscore.yaml`. Emit `lifecycle.phase-skipped` on non-default choice. Update the Hard Gate to allow invoking `specstudio:implement` in addition to `writing-plans`.

### Task 4: Relax plan skill entry gate to accept Ideas

**Verifies:** flexible-lifecycle-flows#ac:plan-accepts-idea-source, flexible-lifecycle-flows#ac:plan-skips-ac-coverage

Modify `skills/plan/skill.md`: update Pre-Flight step 1 to accept either `spec/features/<slug>/README.md` (existing) OR `spec/ideas/<slug>.md` with Status Approved. When sourced from an Idea, skip AC-list parsing (step 2), skip P-001 enforcement, and adjust the Plan schema to use `**Source:** idea:<slug>` instead of `**Source Feature:**`. Tasks use `**Source:**` instead of `**Verifies:**` when referencing an Idea.

### Task 5: Relax implement skill entry gate to accept Feature or Idea directly

**Verifies:** flexible-lifecycle-flows#ac:implement-works-without-plan, flexible-lifecycle-flows#ac:implement-works-with-idea-only

Modify `skills/implement/skill.md`: update the Pre-Flight / Hard Gate to accept three entry modes: (a) Plan-sourced (existing), (b) Feature-sourced (no Plan, `Verifies:` trailer uses Feature AC IDs, single-pass mode), (c) Idea-sourced (no Feature or Plan, `Verifies: idea:<slug>` trailer, single-pass mode). In modes (b) and (c), the per-batch dispatch model is disabled — implement operates as a single-pass conversation.

## Open Questions

- Should the `lifecycle.suggest_skips` config key live under a top-level `lifecycle:` section in `specscore.yaml`, or under a per-skill `gates.ideate` / `gates.specify` section?
- When plan accepts an Idea, should it still require the reviewer subagent, or is that overkill for trivial plans?

---
*This document follows the https://specscore.md/plan-specification*
