---
type: sidekick-seed
slug: plan-skill-should-evaluate-sequential-vs-subagent-execution
captured_at: 2026-06-03T13:06:04Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---
# Plan skill should evaluate sequential vs subagent execution

The `plan` skill should evaluate the task decomposition and decide whether
subagents should be used at all. When tasks are **sequential** (a linear
dependency chain) and **not too big**, and they touch the **same files/functions**
(so isolated parallel subagents would only conflict and re-derive shared
context), it often makes more sense for the main agent/session to implement them
directly — no subagent dispatch.

When that's the case, the plan SHOULD declare it **explicitly** — e.g. an
execution-mode field on the plan (or per-task) such as `Execution: sequential-main-agent`
vs `Execution: parallel-subagents` — **with reasoning** recorded in the plan, so
`implement` honors it instead of defaulting to parallel subagent fan-out.

Heuristics worth encoding for "skip subagents":
- linear `Depends-On` chain with no real parallelism to gain, AND
- small/tightly-coupled tasks editing the same files (high conflict risk), AND
- shared context that isolated subagents would each have to rebuild.

Provenance: surfaced while running `specstudio:implement` on
`ingitdb-cli` plan `spec/plans/2026-06-03-tui-lazy-computed-cells.md` — three
"linear" tasks all editing the same two TUI files. Dispatching them as parallel
subagents would have guaranteed line-overlap conflicts and redundant
re-reading; the main agent implementing them in sequence was clearly better.
The plan lacked machine-readable execution-mode metadata to express this, so the
decision had to be made ad hoc at implement time.
