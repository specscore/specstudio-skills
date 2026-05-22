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

**Observed during dogfooding** the just-shipped destination-resolution flow (specstudio-skills commit `b62457e`, cache fast-forwarded via local pull).

When the live `/sidekick` wrote a NEW seed at `specscore-cli/spec/ideas/seeds/specscore-feature-change-status-should-print-source-idea.md`, a PRE-EXISTING seed in the same directory (`specscore-feature-change-status-to-stable-backward.md`, captured manually two captures earlier in commit `9ac63ed`) had its first line corrupted from `---` to `Captured---` (the word `Captured` prepended to the YAML frontmatter delimiter with no newline). The pre-existing file's frontmatter became malformed YAML; the rest of its content was untouched.

**The visible transcript does not show the cause.** The shipped sidekick agent's listed tool calls during the capture were: collision-check (`ls | grep`), Bash (compute timestamp/hash/uuid), Write (new file at correct path), Bash (synchestra binary probe), Bash (events.jsonl append). None of those should touch the pre-existing seed file.

**Suspected causes (descending plausibility):**
1. **Parallel-agent interference** — another concurrent agent in the workspace did an Edit on the file, racing with the dogfood. The old file's on-disk mtime sits ~1 min before the new seed's `captured_at`, plausible for a race.
2. **LLM tool misuse inside the live agent's autonomy pass** — the SKILL.md's "Output (success)" section says "the skill prints one short line ... `Captured: <slug> at ...`". An agent could plausibly interpret "prints" as "writes" and target the wrong file. Not visible in the surfaced transcript but possible inside a non-displayed sub-step.
3. **Stale editor buffer** — some editor held the file open with a partial-write state and flushed it mid-dogfood.

**Workaround applied:** the corrupted file was restored by hand (specscore-cli commit alongside this seed). The new seed itself was unaffected and landed correctly.

**Investigation:**
- Reproduce by running another `/sidekick` capture in the same dir; check whether the side-effect recurs.
- Audit the shipped sidekick `SKILL.md`'s "Output (success)" section for wording that could lead an agent to write a file instead of printing a stdout line.
- Check filesystem audit logs (if available) to identify the exact write source.

**Why captured manually:** invoking the live `/sidekick` again risks reproducing the bug on yet another existing seed file in the destination dir. Writing this seed directly via Write avoids that.
