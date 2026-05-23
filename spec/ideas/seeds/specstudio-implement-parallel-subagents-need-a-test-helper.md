---
type: sidekick-seed
slug: specstudio-implement-parallel-subagents-need-a-test-helper
captured_at: 2026-05-23T08:00:00Z
captured_by: user:alexander.trakhimenok
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# specstudio:implement parallel subagents need a test-helper namespacing convention to avoid Go package-scope identifier collisions

Observed during `cli-event` Batch 2 (T2 / T3 / T4 / T5 dispatched in parallel, each writing a separate file in `pkg/event/`). Four subagents independently chose the same name `sampleEvent` for their test helper function — different file paths, different signatures (`sampleEvent(t *testing.T) Event` vs. `sampleEvent(name string) Event`), but the same package-scope Go identifier. Only the Exec subagent (T4) noticed mid-flight and renamed its helper to `execSampleEvent`; the others would have collided on compile if not for that defensive rename. A similar collision surfaced again in `cli-event` Batch 4 where the dispatcher subagent's `validEvent` helper collided with the validator subagent's existing `validEvent`.

The skill's documented conflict detection is "line-overlap only" — it inspects `git diff --staged` for overlapping line ranges in the same file. Cross-file identifier conflicts (two subagents declaring the same package-scope name in different files within the same Go package) slip through entirely. The skill explicitly says semantic conflicts "are explicitly out of MVP scope," but this is a narrower mechanical issue: it's deterministically detectable by parsing the staged diff for new top-level declarations and checking for name collisions within the same package.

Two possible fixes: (a) instruct each parallel subagent in its prompt to namespace its test helpers with the task number or file prefix (`task3SampleEvent`, `jsonlSampleEvent`); (b) add a post-batch lint step that runs `go vet` on the staged combined state and refuses to advance if the build is broken. (b) is more robust because it catches any cross-file collision, not just test helpers — but (a) is one prompt sentence away.
