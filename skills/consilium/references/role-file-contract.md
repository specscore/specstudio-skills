# Role File Contract

Every role file used by the consilium panel — default or custom — MUST follow this contract. This is the canonical spec for REQ `custom-role-markdown-contract` in the [`sidekick-consilium` Feature](../../../spec/features/sidekick-consilium/README.md).

## Required structure

A role file is a single markdown file with three mandatory parts.

### 1. Body metadata header (immediately after the H1 title)

```markdown
# Role: <Display Name>

**Name:** <kebab-case-slug>
**Group:** builders | customers | adversaries
**Output Schema Version:** 1
```

- `**Name:**` MUST equal the filename without `.md`.
- `**Group:**` MUST be one of the three literal values.
- `**Output Schema Version:**` MUST be the literal `1` in Phase 1 (reserved for future schema migrations).

### 2. `## Role Prompt` section

The prompt body the agent receives. Should establish:
- The role's identity ("You are the Engineer on a consilium reviewing a captured sidekick idea")
- What the role looks for in a seed
- The output format expectation (cross-reference REQ `vote-schema` and the example below)
- Tone and length guidance

### 3. `## Example Vote` section

One fully-formed YAML vote matching REQ `vote-schema`:

```yaml
verdict: should-implement | should-not-implement | no-opinion | abstain
confidence: low | medium | high
cost: 🟢 | 🟡 | 🔴
complexity: 🟢 | 🟡 | 🔴
argument: <one-sentence strongest argument, ≤ 280 characters>
```

The example MUST parse as valid YAML and the values MUST match the role's typical perspective (e.g., the Skeptic's example shows a `should-not-implement` with high confidence).

## Optional sections

A role file MAY include additional sections (e.g., `## Heuristics`, `## Common Failure Modes`) but they are advisory — the agent reads the whole file but votes on what the seed presents.

## Loading

The skill loads role files at run-time per the active roster (REQ `roster-validation`). A file that fails the contract above (missing `**Name:**`, wrong group enum, no `## Role Prompt` section, no `## Example Vote` section) causes a load-error per REQ `roster-validation` and the skill exits before claiming any task.
