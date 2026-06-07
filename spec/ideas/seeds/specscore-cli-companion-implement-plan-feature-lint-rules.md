---
captured_by: specstudio:specify
status: completed
---

# specscore CLI companion: implement plan-Feature lint rules P-001..P-004 and parser support for Mode/Status/Depends-On task fields

**Hard dependency** for shipping the `plan` Feature revision (`spec/features/skills/plan/`) and the downstream `implement` skill. The Feature defines the contract; the `specscore` CLI must implement it.

## New lint rules

- **P-001** (AC coverage gap) — reserved in the original plan Feature; not yet implemented in the CLI.
- **P-002** (stale AC reference) — reserved; not yet implemented.
- **P-003** (Depends-On cycle / dangling-reference / self-reference) — NEW. Must cite the cycle path on failure.
- **P-004** (placeholder body on `done`-status task in `stub` Plan) — NEW. Must cite the offending task number and reference both the placeholder rule and the writeback contract.

## Parser extensions

- `**Mode:** <full|stub>` Plan body-metadata line (default `full` when absent).
- `**Status:** <pending|in-progress|done|blocked>` task body field (default `pending`).
- `**Depends-On:** — | <task-number>, ...` task body field (default `—`).
- Recognized placeholder body token in `stub` Plans (token TBD; tracked in plan Feature Outstanding Questions).

## Posture-aware lint logic

- `full` Plans: task bodies MUST be non-empty prose.
- `stub` Plans: placeholder body permitted when `**Status:** ≠ done`; prose required when `done`.

Captured during the revise-in-place revision of the plan Feature driven by `spec/ideas/specstudio-implement-skill.md`.

## Resolution

Shipped 2026-05-19 as `specscore-cli` Feature `cli/spec/lint/plan-rules`. Placeholder token chosen: `<!-- implement: pending -->`.
