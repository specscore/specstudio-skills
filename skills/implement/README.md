# `implement` skill

The `specstudio:implement` Claude Code skill — turns an approved SpecScore Plan into focused, AC-traceable source-code changes by dispatching one subagent per task in parallel batches computed from the Plan's `**Depends-On:**` dependency graph.

- **Skill manifest:** [`SKILL.md`](./SKILL.md) — the canonical, machine-readable skill definition.
- **Feature specification:** [`spec/features/skills/implement/`](../../spec/features/skills/implement/README.md) — the SpecScore Feature this skill implements.
- **Skills index:** [`../README.md`](../README.md) — all SpecStudio skills with status and lifecycle position.

## What it does

For each invocation, against one Plan:

1. Reads the dependency graph from the Plan's `**Depends-On:**` task fields.
2. Computes the next executable batch (tasks with all predecessors `**Status:** done`, own status `pending`).
3. Dispatches one subagent per batch task (max 5 concurrent, MVP), with isolated prompts that differ by Plan posture (`full` passes the authored task body; `stub` passes the Plan's `## Approach` and asks the subagent to infer).
4. Aggregates subagent returns per the SDD-style four-status protocol (`DONE` / `DONE_WITH_CONCERNS` / `NEEDS_CONTEXT` / `BLOCKED`).
5. Runs line-overlap conflict detection on the staged diffs.
6. Presents a consolidated batch diff plus a `Verifies: <feature-slug>#ac:<ac-slug>, ...` commit-message template to the user.
7. **Stages only — never commits.** User owns commit shape.
8. After user-approval and commit, advances to the next batch. Repeats until all tasks are `**Status:** done`.

The central guarantee: **every commit references the AC IDs it satisfies** via the `Verifies:` trailer, closing the spec↔code coherence loop at the implementation handoff.

## Status

**Shipped.** Skill exists at `skills/implement/` and is usable today via Claude Code.

## Triggers

`implement`, `/implement`, `specstudio:implement`, "implement this plan", "implement this task", or the event `plan.approved`.

## What it does NOT do

- **Commit on the user's behalf.** Stages via `git add`; user runs `git commit` with the suggested template.
- **Plan against unapproved Plans.** Refuses if Plan Status ∉ {Approved, Implementing}.
- **Run downstream skills directly.** Promotion boundary is `specstudio:verify` only.
- **Run a code-quality reviewer subagent inside the loop.** Code review is `specstudio:review`'s job downstream.
- **Switch posture mid-flight.** Posture is the planner's call, declared at plan time; one-way.
- **Cross-machine dispatch.** MVP uses local subagent dispatch via the Claude Code Agent tool only; remote runner dispatch is a follow-on Idea.
- **Auto-resolve conflicts.** Line-overlap conflicts roll back the batch and prompt the user.
