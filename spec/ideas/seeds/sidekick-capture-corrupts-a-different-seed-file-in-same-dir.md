---
captured_by: user
status: queued
---

# sidekick capture corrupts a different seed file in same dir

**Observed** during dogfooding the destination-resolution flow (specstudio-skills `b62457e`, cache fast-forwarded). The live `/sidekick` wrote a NEW seed correctly at `specscore-cli/spec/ideas/seeds/specscore-feature-change-status-should-print-source-idea.md`. A PRE-EXISTING seed in the same dir (`specscore-feature-change-status-to-stable-backward.md` from `9ac63ed`) had its first line corrupted from `---` to `Captured---`, breaking the YAML frontmatter.

**Post-capture investigation:**

- **Git history is clean.** The corrupted file has exactly one touching commit (`9ac63ed`, the original write). Corruption was working-tree-only.
- **No active git hooks.** `.git/hooks/` is samples-only.
- **events.jsonl is clean.** The live capture emitted one well-formed event (uuid `2cbd5bc0-71d2-4a0d-991f-03363a7e3aae`) referencing only the new slug.
- **Mtime ordering rules out the live agent.** Corrupted file's mtime is ~1 min BEFORE the new seed's `captured_at` — a capture cannot modify another file before its own write.

**Revised cause ranking:**

1. **Parallel-agent interference (top).** Multiple agents were active in the workspace (org-rename migrator, telemetry sub-Features, cache sync). Any could have issued `Edit(file, old="---", new="Captured---")`. The mtime-before-capture ordering is consistent only with this.
2. ~~LLM tool misuse inside the sidekick agent~~ — downgraded; events.jsonl, transcript, and mtime all argue against.
3. ~~Stale editor buffer~~ — possible but unlikely.

**Implication:** the destination-resolution Feature is clean. 0.0.6 (`specstudio--v0.0.6`) ships uncontaminated. Seed kept for consilium triage; likely no-repro race condition.

**Next diagnostic if recurrence:** controlled re-dogfood from an isolated session with no other agents active. If corruption recurs, bug is in the skill; if not, archive as no-repro.
