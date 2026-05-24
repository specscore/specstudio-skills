# `init` skill

The `specstudio:init` Claude Code skill — bootstraps a SpecScore-managed project (creates `specscore.yaml`, scaffolds the `spec/` tree, installs the canonical Producer-shape instruction snippet) in one wizard-driven step.

- **Skill manifest:** [`SKILL.md`](./SKILL.md) — the canonical, machine-readable skill definition.
- **Feature specification:** [`spec/features/skills/init/`](../../spec/features/skills/init/README.md) — the SpecScore Feature this skill implements.
- **Skills index:** [`../README.md`](../README.md) — all SpecStudio skills with status and lifecycle position.

## What it does

Runs a 3–4-question wizard with defaults pre-filled from project-state detection, then idempotently scaffolds:

1. `specscore.yaml` (via `specscore init` when the CLI is on PATH; AI-agent fallback otherwise) — line 1 is the canonical schema-pointer comment, the `project:` block is populated from flags / prompts / git-remote inference.
2. `spec/{,ideas,features}/README.md` — lint-clean indexes per the canonical [Index Features](https://github.com/specscore/specscore/blob/main/spec/features/).
3. The canonical Producer-shape instruction snippet, defined by [`spec/features/third-party-integration/`](../../spec/features/third-party-integration/README.md), pasted into the right platform agent-instructions file (`CLAUDE.md` / `AGENTS.md` / `GEMINI.md` / `.cursor/rules/specstudio.md`) per the explicit platform-detection rule. Always prompts for explicit consent before writing outside `spec/`.

Triggers: `specstudio:init`, `/specstudio:init`, "set up specstudio", "bootstrap a spec repo".

## Two operational modes

- **Default — `specstudio:init`** — full wizard flow. Greenfield AND brownfield repos both flow through this; brownfield reuses defaults from detected state.
- **Update — `specstudio:init --update`** — drift-only reconciliation, no wizard. Diffs each managed artifact (snippet, schema header, indexes) against canonical and asks the user **replace** / **keep** / **abort** per artifact.

## Idempotence

Re-running `specstudio:init` is safe:

- Fully-initialized + no drift → no-op (skip condition; no wizard, no writes, no event).
- Partial state → resume; create missing pieces; preserve existing pieces byte-identical.
- Drift detected → diff-and-confirm; never silently update.

State detection is via filesystem inspection only — no hidden state file.

## CLI delegation

When `specscore` is missing from PATH, the skill delegates installation to the dedicated install skill:

- `specscore` missing → invoke [`specscore:install`](https://github.com/specscore/ai-plugin-specscore/blob/main/skills/install/SKILL.md) (with explicit user consent).

The skill does NOT run install commands itself. Install URLs, supported methods (curl-pipe-sh, brew, npm, manual), and signature verification all live in the install skill.

## Events

- `project.initialized` — emitted exactly once per project, on first successful greenfield init.
- `project.updated` — emitted on subsequent state-changing reruns.
- No event on no-op rerun.

Both events follow the canonical change-context shape (`changed_sections`, `previous_revision`, factual `change_summary` ≤2 sentences) used by `idea.updated` and `feature.updated`. See [`../shared/events.md`](../shared/events.md).
