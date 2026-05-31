---
type: sidekick-seed
slug: prompt-users-to-configure-reviewer-gates-when-missing
captured_at: 2026-05-30T15:13:08Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# Prompt users to configure reviewer gates when missing

When a producer skill needs `gates.<stage>.reviewers` but the repo has no configured gate, the agent should ask the user how to proceed and offer available setup options with a default preselected instead of stopping on a raw configuration error.
