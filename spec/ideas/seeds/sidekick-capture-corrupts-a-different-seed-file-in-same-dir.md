---
type: sidekick-seed
slug: sidekick-capture-corrupts-a-different-seed-file-in-same-dir
captured_at: 2026-05-22T09:00:00Z
captured_by: user
captured_during: null
trigger: explicit
status: queued
synchestra_task: null
---

# sidekick capture corrupts a different seed file in same dir

**Observed** during dogfooding the destination-resolution flow (specstudio-skills `b62457e`, cache fast-forwarded). The live `/sidekick` wrote a NEW seed correctly at `specscore-cli/spec/ideas/seeds/specscore-feature-change-status-should-print-source-idea.md`. But a PRE-EXISTING seed in the same dir (`specscore-feature-change-status-to-stable-backward.md` from `9ac63ed`) had its first line corrupted from `---` to `Captured---` — the word `Captured` prepended to the YAML frontmatter delimiter with no newline, breaking the frontmatter.

**Visible transcript shows no tool call that should have touched the old file.** Agent's recorded calls: collision-check (`ls | grep`), Bash (compute timestamp/hash/uuid), Write (new file at correct path), Bash (synchestra probe), Bash (events.jsonl append). None should touch the existing seed.

**Suspected causes (descending plausibility):**
1. Parallel-agent race — another concurrent agent in the workspace did an Edit on the file. Old file's mtime sits ~1 min before the new seed's `captured_at`.
2. LLM tool misuse inside the live agent's autonomy pass — SKILL.md "Output (success)" says "the skill prints `Captured: <slug> at ...`". An agent could interpret "prints" as "writes" against the wrong file.
3. Stale editor buffer flushing a partial write mid-dogfood.

**Workaround applied:** corrupted file restored by hand (specscore-cli commit alongside the dogfood seed). Captured here directly via Write rather than re-invoking `/sidekick`, to avoid risking another side-effect on yet another existing seed.
