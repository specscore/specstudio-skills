---
type: sidekick-seed
slug: specstudio-implement-should-detect-and-handle-concurrent
captured_at: 2026-05-23T08:00:00Z
captured_by: user:alexander.trakhimenok
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# specstudio:implement should detect and handle concurrent sessions writing to the same working tree

Observed during `cli-event-emit` implementation: a second AI session was running in parallel in the same repo, committing `issue-rules` work. The two sessions raced on `git add` / `git commit` operations multiple times:

- **Misattributed commit `7136551`**: between my `git add internal/cli/event.go event_test.go root.go cli-event-emit.md` and the immediately-following `git commit` (one Bash invocation, one second apart), the parallel session re-modified the index. The commit captured `spec/plans/issue-rules-implementation.md` (parallel session's Plan-status flip) instead of my four files, with MY commit message. Recovery required `git commit --amend` to rewrite the misattributed message + a second atomic stage+commit for my work.
- **Plan-file sweep at Batch 1**: my `cli-event-emit.md` schema-catchup edit got swept into the parallel session's commit `6ceaf02` (described as "Task 6 — IssueSlug helper + rules I-010/I-011") rather than landing in my own commit.
- **Stash conflict on pop**: at session end, `git stash pop` hit a 3-way merge conflict in `issue-rules-implementation.md` because the parallel session had committed updates to the same file while my stash sat preserved.

The implement skill's documented conflict detection is "line-overlap only (post-batch)" via `git diff --staged`. That contract assumes a SINGLE writer to the working tree. Concurrent sessions break the assumption silently. The skill needs either (a) a pre-stage probe (`git status --porcelain` immediately before `git add`, and again before `git commit`, refusing to advance if the index has unexpected entries), or (b) an explicit lock file at `.specscore/implement.lock` that the skill takes during a batch and refuses to start while another holds it.

Workaround for now: chain `git add` and `git commit` into a single shell pipeline with `&&` so there's no observable window for interleaving — but this only narrows the race; it doesn't eliminate it.
