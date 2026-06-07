---
captured_by: user
status: queued
---

# improve specstudio-skills to instruct Ai agent that --bg is undocummented argument and agent should try to run it instead of relying on --help.

Observed in a specscore-cli session: `claude --help` did not list `--bg`, so the agent wrongly concluded background launch was unsupported and nearly downgraded the user's choice — but the flag works. Skills that shell out to optional or undocumented CLI flags (e.g. the `specstudio:plan` Option-3 `claude --bg` launch) should instruct agents to ATTEMPT the documented command and treat a missing `--help` entry as non-authoritative, not as proof the flag is absent.
