---
captured_by: user
status: queued
---

# Prompt users to configure reviewer gates when missing

When a producer skill needs `gates.<stage>.reviewers` but the repo has no configured gate, the agent should ask the user how to proceed and offer available setup options with a default preselected instead of stopping on a raw configuration error.
