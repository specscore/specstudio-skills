---
type: sidekick-seed
slug: specstudio-skill-md-changes-don-t-activate-until-plugin
captured_at: 2026-05-22T00:00:00Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# specstudio SKILL.md changes don't activate until plugin republish

**Observed:** changes to `skills/<name>/SKILL.md` in the `specstudio-skills` source repo do not take effect in the live skill that Claude Code invokes. The cached plugin (e.g., `~/.claude/plugins/cache/specscore/specstudio/0.0.5/skills/sidekick/SKILL.md`) is what gets loaded — frozen at the plugin's installed version.

**Worked example:** specstudio-skills commit `7120ba2` (today) shipped the destination-resolution flow in `skills/sidekick/SKILL.md`. A `/sidekick` invocation immediately after loaded the cached 0.0.5 SKILL.md (no destination-resolution flow) — the brand-new Feature is dark in live use until the plugin republishes. The seed about the CLI cascade bug had to be captured manually with manual destination routing (specscore-cli commit `9ac63ed`).

**Why it matters:** every SKILL.md iteration — wording tweaks, contract revisions, new sections — needs a plugin republish cycle to be testable in-context. That's slow feedback for a skill author iterating on prompt wording (the very iteration loop `REQ:helper-prompt-iteration` anticipates). Worst case it forces sequencing — wording changes can't be dogfooded until ship → republish → adopt.

**Suggested investigation:** check whether Claude Code supports a local-source override for plugin development (e.g., a config to symlink cache → source, or a `--local-skills` flag). If not, document the lag prominently in contributor docs, and consider a `specstudio:dev` helper that does the symlinking automatically for active development.
