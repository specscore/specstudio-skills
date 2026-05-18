# Sidekick Consilium Implementation Plan (Phase 1)

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship Phase 1 of the `sidekick-ideas` Idea — the `specstudio:consilium` skill that drains captured sidekick (sideline-idea) seeds and produces deterministic, auditable verdicts via a 5-stage pipeline (CLI gather → researcher → 9-role parallel expert panel → CLI arbiter → scribe).

**Architecture:** A new in-repo skill orchestrates the 5-stage pipeline per claimed `consilium-review` task. The Synchestra task is the structured source of truth (V3 storage); the seed gets only the scribe's prose summary mirrored into a `## Consilium Verdict` section. The deterministic verdict gate lives cross-repo as a new `specscore consilium verdict` subcommand (V4); the task type lives cross-repo in Synchestra. Per-project roster and gate configuration via `specscore.yaml` → `consilium:` top-level block. Phase 1 is the producer of verdicts; Phase 2 will consume them for auto-promotion.

**Tech Stack:** Markdown skill authoring (Claude Code skill format); YAML frontmatter; bash for orchestration (yq/jq for YAML parsing, `synchestra task` for task lifecycle, `specscore consilium verdict` for the arbiter once it ships); Claude Code `Agent` tool with parallel invocation for the panel fan-out; `.synchestra/events.jsonl` for event transport (fallback when `synchestra emit` is not yet available); `specscore spec lint` for spec hygiene.

---

## Source Spec

This plan implements the Approved Feature at [`spec/features/sidekick-consilium/README.md`](../features/sidekick-consilium/README.md) (commit `5ec730c`). All REQ/AC references in this plan resolve to that document. The Feature has 29 REQs across 8 topics and 26 ACs (25 testable + 1 calibration quality gate).

## Cross-Cutting Decisions Resolved at Plan Time

These decisions surface in multiple tasks; resolving them once up front avoids drift.

### 1. Transcript capture is threaded through every stage from day one

Per the reviewer's v1 advisory and now contracted as REQ `pipeline-transcript-capture`: the transcript is non-retrofittable. The skill's behavior sections (Task 8) MUST instruct the agent to capture transcript records at the start of each stage and finalize them at stage end. Bolt-on transcript capture after the fact loses ordering, timing, and per-role payload. The Task 8 sub-steps interleave transcript-capture instructions inside each stage's section — NOT as a separate task.

### 2. Role files share a uniform contract

Every role file (default and custom) follows the contract documented in REQ `custom-role-markdown-contract`. Tasks 4–7 author the 11 default role files (researcher + scribe + 9 panel roles) against this shared contract. A reference document at `skills/consilium/references/role-file-contract.md` (created in Task 1) is the canonical contract spec.

### 3. The `specscore.yaml` `consilium:` block is documented, not pre-populated

A fresh project gets the default 9-role roster and default gate knobs *without* needing a `consilium:` block in `specscore.yaml`. The block is purely an opt-in override mechanism. Phase 1 ships a *reference* document at `skills/consilium/references/consilium-config-example.md` (Task 3) — there is no `specscore.yaml` change required for the in-repo Feature itself.

### 4. Cross-repo gates are tracked, not waited on

Three cross-repo dependencies exist:
- `specscore-cli#6` — the `sidekick-seed` lint rule from Phase 0 (blocks the dogfood seed currently in working tree)
- `specscore-cli` new issue — the `specscore consilium verdict` subcommand (the arbiter, Task 1)
- `synchestra` new issue — the `consilium-review` task type (the queue, Task 1)

The in-repo plan ships the contracts and the orchestrator skill. The cross-repo subcommands and task type must ship before `/consilium` actually works end-to-end. Task 11 documents this as the ship gate; calibration (Task 10) cannot run until the cross-repo arbiter exists.

### 5. The dogfood seed stays in working tree until specscore-cli#6 ships

The seed at `spec/ideas/seeds/rebrand-view-in-specstudio-blockquote-to-view-in-specscore.md` (captured live during the specify session for this Feature) is gated on the Phase 0 lint rule landing. Once `specscore-cli#6` ships and a `specscore` upgrade is pulled, the seed lints cleanly and can be committed. Task 11 includes the verification step.

---

## File Structure

```
skills/
└── consilium/
    ├── SKILL.md                                  # Task 8 — the orchestrator (~500 lines)
    ├── roles/
    │   ├── researcher.md                          # Task 4
    │   ├── scribe.md                              # Task 4
    │   ├── engineer.md                            # Task 5 (Builders)
    │   ├── architect.md                           # Task 5
    │   ├── qa.md                                  # Task 5
    │   ├── pm.md                                  # Task 6 (Customers)
    │   ├── ux.md                                  # Task 6
    │   ├── marketing.md                           # Task 6
    │   ├── yagni-cop.md                           # Task 7 (Adversaries)
    │   ├── skeptic.md                             # Task 7
    │   └── security-ops.md                        # Task 7
    └── references/
        ├── role-file-contract.md                  # Task 1 (also used by Tasks 4–7)
        └── consilium-config-example.md            # Task 3

skills/shared/synchestra-events.md                  # Task 2 — extend with sidekick-idea.reviewed

spec/features/sidekick-consilium/
├── _tests/                                        # Task 9 — Rehearse scaffolds
│   ├── invocation-drains-all-queued-tasks.md
│   ├── invocation-rejects-per-slug-argument.md
│   ├── … (23 more)
│   └── _skipped.md
└── calibration/                                   # Task 10 — the 20-seed calibration set
    ├── README.md
    ├── strong/ (5 seeds)
    ├── weak/ (5 seeds)
    ├── out-of-domain/ (5 seeds)
    └── ambiguous/ (5 seeds)

spec/plans/
├── sidekick-consilium.md                          # This file
├── sidekick-consilium-arbiter-companion.md        # Task 1 — cross-repo stub
└── sidekick-consilium-task-companion.md           # Task 1 — cross-repo stub
```

---

## Task 1: Companion plan stubs + open upstream issues

**Files:**
- Create: `spec/plans/sidekick-consilium-arbiter-companion.md`
- Create: `spec/plans/sidekick-consilium-task-companion.md`
- Create: `skills/consilium/references/role-file-contract.md`
- Modify: `spec/plans/README.md` (index two new plan files)

**Why first:** every subsequent task depends on shared concepts (the role-file contract, the cross-repo gating). Landing these upfront unblocks parallel work on the role files and the SKILL.md.

- [ ] **Step 1: Create the role-file contract reference**

```bash
mkdir -p skills/consilium/references
```

Create `skills/consilium/references/role-file-contract.md`:

```markdown
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
```

- [ ] **Step 2: Create the arbiter companion plan stub**

Create `spec/plans/sidekick-consilium-arbiter-companion.md`:

```markdown
# Sidekick Consilium Arbiter — Cross-Repo Companion Plan Stub

**Status:** Stub. This plan exists in *this* repo to record the dependency. The actual implementation work happens in [`synchestra-io/specscore-cli`](https://github.com/synchestra-io/specscore-cli).

**Source contract:** REQs `specscore-consilium-verdict-subcommand`, `arbiter-gate-rules`, `arbiter-reproducibility`, and `roster-validation` in [`spec/features/sidekick-consilium/README.md`](../features/sidekick-consilium/README.md).

## What needs to ship in specscore-cli

A new subcommand `specscore consilium verdict` that:

1. Accepts inputs: `--votes <file>`, `--roster <file>`, `--gate <file>` (optional, defaults to strict baseline), `--seed <file>`.
2. Validates the roster per REQ `roster-validation` (≥ 1 per group post-exclude/add, ≤ 12 total, no name collisions, custom-role files parse).
3. Validates each vote against REQ `vote-schema`; rejects malformed votes with a clear error.
4. Applies the 13-step gate algorithm from REQ `arbiter-gate-rules`.
5. Emits YAML output to stdout: `verdict`, `rule_trace`, `excluded_votes`, `denominators`.
6. Returns exit code 0 on successful verdict (including `should-not-implement` and `needs-human-review`); non-zero on validation failure.
7. Is snapshot-testable: same inputs always produce identical stdout (REQ `arbiter-reproducibility`).

## How to verify the rule is live

After the subcommand ships and a SpecScore project upgrades:

```bash
specscore consilium verdict \
  --votes tests/fixtures/votes-unanimous-strong.yaml \
  --roster tests/fixtures/roster-default.yaml \
  --gate tests/fixtures/gate-strict.yaml \
  --seed tests/fixtures/seed-1.md
# Expected stdout includes: verdict: should-implement
```

## Tracking

- **Upstream issue:** TBD — to be opened at Step 4 of Task 1 of this plan.
- Until the subcommand ships, the consilium skill (Task 8) cannot complete its arbiter stage and the calibration set (Task 10) cannot run.

---
*This document follows the https://specscore.md/plans-index-specification*
```

- [ ] **Step 3: Create the task-type companion plan stub**

Create `spec/plans/sidekick-consilium-task-companion.md`:

```markdown
# Consilium-Review Task Type — Cross-Repo Companion Plan Stub

**Status:** Stub. This plan exists in *this* repo to record the dependency. The actual implementation work happens in [`synchestra-io/synchestra`](https://github.com/synchestra-io/synchestra).

**Source contract:** REQs `consilium-review-task-lifecycle`, `idempotent-task-creation`, and `single-writer-claim-semantics` in [`spec/features/sidekick-consilium/README.md`](../features/sidekick-consilium/README.md).

## What needs to ship in synchestra

A new task type `consilium-review` registered with Synchestra. The type:

1. Supports the state machine: `queued → claimed → in_review → complete | failed | aborted`.
2. Atomically transitions `queued → claimed` (REQ `single-writer-claim-semantics`).
3. Keys tasks by `content_hash` for idempotent creation — a second `synchestra task create consilium-review` with the same `content_hash` returns the existing task ID (REQ `idempotent-task-creation`).
4. Stores the structured verdict payload from REQ `verdict-source-of-truth-in-task` and the transcript from REQ `pipeline-transcript-capture` as task fields.

## How to verify the type is live

After the task type ships and a Synchestra project upgrades:

```bash
synchestra task create consilium-review \
  --content-hash abc123 \
  --seed-path spec/ideas/seeds/test.md
# Expected: returns the task ID. Second invocation with same content_hash returns the same ID.

synchestra task claim <task-id>
# Expected: transitions queued → claimed. Second claim from another process returns "already claimed".
```

## Tracking

- **Upstream issue:** TBD — to be opened at Step 5 of Task 1 of this plan.
- Until the task type ships, the consilium skill (Task 8) cannot claim tasks or write verdicts.

---
*This document follows the https://specscore.md/plans-index-specification*
```

- [ ] **Step 4: Open upstream issue in specscore-cli for the arbiter**

```bash
gh issue create --repo synchestra-io/specscore-cli \
  --title "Add \`specscore consilium verdict\` subcommand" \
  --body "$(cat <<'EOF'
## Context

The [`sidekick-consilium` Feature](https://github.com/synchestra-io/specstudio-skills/blob/main/spec/features/sidekick-consilium/README.md) in `synchestra-io/specstudio-skills` ships the contract for a new deterministic CLI arbiter that turns 9 typed votes (from a 9-role panel) into a verdict. REQs `specscore-consilium-verdict-subcommand`, `arbiter-gate-rules`, `arbiter-reproducibility`, and `roster-validation` define the contract.

Tracking stub in the producing repo: [`spec/plans/sidekick-consilium-arbiter-companion.md`](https://github.com/synchestra-io/specstudio-skills/blob/main/spec/plans/sidekick-consilium-arbiter-companion.md).

## What needs to ship

A new subcommand `specscore consilium verdict` accepting `--votes`, `--roster`, `--gate`, `--seed`. Applies the 13-step gate algorithm from REQ `arbiter-gate-rules` and emits a structured YAML verdict + rule_trace to stdout. Snapshot-testable: same inputs always produce identical output.

## Acceptance

A 13-step gate algorithm implementation with snapshot tests over fixtures covering each rule path: adversary-veto, low-confidence-abstain veto, strict-gate pass, supermajority pass (when require_all_builders=false), median-confidence-fails, cost-ceiling-fails, complexity-ceiling-fails, builders-majority-reject, and the fall-through to `needs-human-review`.

## Cross-reference

- Feature spec: spec/features/sidekick-consilium/README.md REQs 23–25
- Companion plan: spec/plans/sidekick-consilium-arbiter-companion.md
EOF
)"
```

Capture the issue number returned by `gh`. Update `spec/plans/sidekick-consilium-arbiter-companion.md`'s "Upstream issue: TBD" line with the actual issue URL.

- [ ] **Step 5: Open upstream issue in synchestra for the task type**

```bash
gh issue create --repo synchestra-io/synchestra \
  --title "Add \`consilium-review\` task type" \
  --body "$(cat <<'EOF'
## Context

The [`sidekick-consilium` Feature](https://github.com/synchestra-io/specstudio-skills/blob/main/spec/features/sidekick-consilium/README.md) in `synchestra-io/specstudio-skills` ships the contract for a new Synchestra task type that queues sidekick seeds for review and stores the structured verdict. REQs `consilium-review-task-lifecycle`, `idempotent-task-creation`, and `single-writer-claim-semantics` define the contract.

Tracking stub in the producing repo: [`spec/plans/sidekick-consilium-task-companion.md`](https://github.com/synchestra-io/specstudio-skills/blob/main/spec/plans/sidekick-consilium-task-companion.md).

## What needs to ship

A new task type `consilium-review` with the state machine `queued → claimed → in_review → complete | failed | aborted`, atomic `queued → claimed` transition, content-hash keyed idempotent creation, and structured payload fields for the verdict, transcript, and roster snapshot from REQs `verdict-source-of-truth-in-task` and `pipeline-transcript-capture`.

## Acceptance

The task type registered with Synchestra, accessible via `synchestra task create consilium-review` and `synchestra task claim <id>`. Concurrent-claim test passes (two clients race; exactly one wins).

## Cross-reference

- Feature spec: spec/features/sidekick-consilium/README.md REQs 21, 26, 27
- Companion plan: spec/plans/sidekick-consilium-task-companion.md
EOF
)"
```

Update `spec/plans/sidekick-consilium-task-companion.md` similarly.

- [ ] **Step 6: Update `spec/plans/README.md` index**

Add two rows under the Contents table:

```markdown
| [sidekick-consilium](sidekick-consilium.md) | [sidekick-consilium](../features/sidekick-consilium/README.md) | Phase 1 of the sidekick-ideas Idea: the consilium worker that drains captured seeds and produces deterministic verdicts. 11 tasks. |
| [sidekick-consilium-arbiter-companion](sidekick-consilium-arbiter-companion.md) | — | Cross-repo companion stub: the `specscore consilium verdict` subcommand lives in `synchestra-io/specscore-cli`. |
| [sidekick-consilium-task-companion](sidekick-consilium-task-companion.md) | — | Cross-repo companion stub: the `consilium-review` task type lives in `synchestra-io/synchestra`. |
```

- [ ] **Step 7: Lint and commit**

```bash
specscore spec lint --severity warning
# Expected: 3 violations (pre-existing dogfood-seed errors; unchanged)

git add skills/consilium/references/role-file-contract.md \
        spec/plans/sidekick-consilium-arbiter-companion.md \
        spec/plans/sidekick-consilium-task-companion.md \
        spec/plans/README.md
git commit -m "$(cat <<'EOF'
plan(sidekick-consilium): companion stubs + role-file contract

Land the shared concepts before per-component tasks:

- skills/consilium/references/role-file-contract.md — canonical spec
  for the markdown contract every role file (default + custom) follows.
- spec/plans/sidekick-consilium-arbiter-companion.md — tracks the
  cross-repo specscore consilium verdict subcommand. Upstream issue
  opened in synchestra-io/specscore-cli.
- spec/plans/sidekick-consilium-task-companion.md — tracks the cross-
  repo consilium-review task type. Upstream issue opened in
  synchestra-io/synchestra.
- spec/plans/README.md — index entries for the three new plans.

Per REQ custom-role-markdown-contract, REQ specscore-consilium-verdict-
subcommand, REQ consilium-review-task-lifecycle (Feature
sidekick-consilium).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Extend `synchestra-events.md` with `sidekick-idea.reviewed`

**Files:**
- Modify: `skills/shared/synchestra-events.md`

**Why now:** the consilium skill (Task 8) emits this event on every successful verdict. Authoring the contract before the skill means the skill has a stable reference target.

- [ ] **Step 1: Locate the insertion point**

```bash
grep -n '^## Events Emitted by' skills/shared/synchestra-events.md
```

Append the new section after the last `## Events Emitted by …` section.

- [ ] **Step 2: Append the section**

Add this content:

````
## Events Emitted by `specstudio:consilium`

### `sidekick-idea.reviewed`
Fired exactly once per successful `consilium-review` task completion (transition to `complete`). On `failed` or `aborted` transitions, no event is emitted.

```yaml
event: sidekick-idea.reviewed
version: 1
uuid: <generated>
timestamp: <ISO-8601 of review completion>
actor:
  kind: skill
  id: skill:specstudio:consilium
artifact:
  type: consilium-review
  id: <task-id>
  path: <seed_path>                          # path to the seed file
  revision: <git SHA at emission>
payload:
  slug: <seed slug>
  content_hash: <seed content_hash at review time>
  verdict: should-implement | should-not-implement | needs-human-review
  roster_snapshot: [<role slugs in active roster>]
  tokens_total: <int>
```

**Consumer:** `synchestra:whats-next` reads this event to surface `consilium-review` tasks with `needs-human-review` verdicts at the top of the prioritization report. Phase 2 auto-promote (future Feature) will subscribe to `verdict: should-implement` events.
````

- [ ] **Step 3: Lint and commit**

```bash
specscore spec lint --severity warning
# Expected: same 3 pre-existing violations

git add skills/shared/synchestra-events.md
git commit -m "$(cat <<'EOF'
feat(skills/shared): add sidekick-idea.reviewed event

Extend synchestra-events.md with the event emitted by
specstudio:consilium on successful task completion. Payload includes
verdict, roster snapshot, and tokens_total for downstream consumers
(synchestra:whats-next today; Phase 2 auto-promote future).

Per REQ event-reviewed-emitted (Feature sidekick-consilium).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Author the `specscore.yaml` config reference

**Files:**
- Create: `skills/consilium/references/consilium-config-example.md`

**Why a reference, not a config change:** the consilium uses default behaviors out of the box. Projects opt in to overrides by adding a `consilium:` block to their own `specscore.yaml`. This reference doc shows the schema and example values so adopters can configure without hunting the Feature spec.

- [ ] **Step 1: Create the reference file**

Create `skills/consilium/references/consilium-config-example.md`:

````markdown
# `specscore.yaml` — `consilium:` Configuration Reference

Phase 1 of the consilium reads optional configuration from a top-level `consilium:` block in `specscore.yaml`. The block has two sub-keys: `roster` and `gate`. **Both are optional.** A project with no `consilium:` block gets the 9-role default roster and the strict-baseline gate.

This reference documents the schema; the canonical contract lives in REQs `roster-exclude-and-custom`, `gate-knob-set`, and `roster-validation` of the [`sidekick-consilium` Feature](../../../spec/features/sidekick-consilium/README.md).

## Minimal configuration (defaults — no `consilium:` block needed)

```yaml
# specscore.yaml — defaults inherited; nothing to write
```

## Roster customization

```yaml
consilium:
  roster:
    exclude:
      - marketing                       # drop the default Marketing role
      - security-ops                    # drop the default Security/Ops role
    custom:
      - name: accessibility
        group: customers
        path: .specscore/roles/accessibility.md
      - name: domain-expert
        group: customers
        path: .specscore/roles/domain-expert.md
```

After this configuration, the active roster has 9 roles:
- **Builders** (3): engineer, architect, qa (defaults)
- **Customers** (4): pm, ux, accessibility, domain-expert (1 default + 2 custom)
- **Adversaries** (2): yagni-cop, skeptic (default; security-ops excluded)

The arbiter validates this roster per REQ `roster-validation`: every group has ≥ 1 member, no name collisions, ≤ 12 total, custom-role files exist and parse.

## Gate-knob customization

```yaml
consilium:
  gate:
    adversary_veto_confidence: high          # high | medium | low
    cost_ceiling: medium                     # low | medium  (🟢-only vs 🟢+🟡)
    complexity_ceiling: medium               # low | medium
    min_median_confidence: medium            # low | medium | high
    require_all_builders: true               # true | false (unanimous vs supermajority ⅔)
    require_all_customers: true              # true | false
```

The defaults above are the strict baseline used by the parent Idea and validated by the 20-verdict calibration set. Projects with different risk tolerance can loosen knobs (e.g., `require_all_customers: false` for projects where customer-side voices are less reliable). The discrete enums match the 3-step ordinal used elsewhere; continuous numeric thresholds are not supported in Phase 1.

## Reading the active configuration

To inspect what configuration is active for a project:

```bash
specscore consilium verdict --print-config
```

(Implementation lands in the cross-repo `specscore-cli` per the [arbiter companion plan](../../../spec/plans/sidekick-consilium-arbiter-companion.md).)

## Future extensions (Phase 2+)

Phase 2 will additively add `consilium.auto_promote:` to the same block (action knobs for what to do on `should-implement` verdicts: draft Feature, draft plan, dry-run, enabled/disabled). Phase 1's schema is designed for non-disruptive extension.
````

- [ ] **Step 2: Lint and commit**

```bash
specscore spec lint --severity warning
# Expected: same 3 pre-existing violations

git add skills/consilium/references/consilium-config-example.md
git commit -m "$(cat <<'EOF'
feat(skills/consilium): add consilium-config-example reference

Reference documentation for the optional consilium: block in
specscore.yaml — roster.exclude/custom and gate-knob schemas.
Projects use defaults out of the box; the reference shows the schema
for adopters who want to override.

Per REQ roster-exclude-and-custom, REQ gate-knob-set, REQ
roster-validation (Feature sidekick-consilium).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Author researcher and scribe role files

**Files:**
- Create: `skills/consilium/roles/researcher.md`
- Create: `skills/consilium/roles/scribe.md`

**Why before the 9 panel roles:** these two are conceptually different (they bracket the pipeline, not part of the voting panel) and the SKILL.md (Task 8) references them by path. Authoring them first means Task 8's references resolve.

- [ ] **Step 1: Create `skills/consilium/roles/researcher.md`**

```markdown
# Role: Researcher

**Name:** researcher
**Group:** — (not a panel role; runs as the pipeline's second stage)
**Output Schema Version:** 1

## Role Prompt

You are the Researcher for a sidekick consilium reviewing a captured sideline idea. Your job is to produce a **fact-only briefing pack** that the 9-role expert panel will read alongside the seed.

**Inputs you receive:**
1. The seed file's full content (frontmatter + H1 one-liner + optional body)
2. A raw context bundle pre-assembled by the CLI gather stage, containing:
   - Output of `synchestra:feature` for any feature paths the seed mentions
   - Output of `synchestra:code` for source files referenced in `captured_during`
   - Recent `git log` over relevant paths
   - A list of prior captured seeds within the dedupe window (same project)

**Your output: a briefing pack with this exact structure**

```markdown
# Briefing: <seed slug>

## Related artifacts
- [<feature/idea/plan slug>](path) — <one-line factual description of how it relates>
- ... (up to 5 entries; if none, write "None.")

## Code references
- `<file>:<line-range>` — <factual description of what's at this location, NO judgment>
- ... (up to 5 entries; if none, write "None.")

## Recent git activity
- `<commit SHA>`: <commit subject line> (<date>)
- ... (up to 3 most relevant commits; if none, "No recent activity in relevant paths.")

## Prior captures within dedupe window
- [<slug>](path) — <one-line factual description> (status: <queued|complete|failed>)
- ... (or "None.")
```

**Rules — these are non-negotiable:**

1. **Facts only.** No words like "important", "concerning", "interesting", "recommended", "should consider". Just structured factual observations.
2. **Cap at 1500 tokens.** Trim aggressively if the raw context bundle is large.
3. **No filler.** If a section has nothing to report, write "None." — do not pad.
4. **Cite paths and SHAs.** Every claim must be traceable to a file or commit.
5. **No advice to the panel.** You are not the panel; you do not vote, recommend, or summarize.

If you find yourself writing judgment-laden language, stop and rewrite. A research output that contains opinions is a contract violation by this prompt and will be flagged at lint time.

## Example Output

```markdown
# Briefing: persist-debug-logs-across-restarts

## Related artifacts
- [specstudio:init](spec/features/skills/init/README.md) — Feature handles project bootstrap; debug-log persistence would extend its scope.
- None.

## Code references
- `skills/init/SKILL.md:42-78` — current init flow writes ephemeral status to stdout only.
- `skills/shared/synchestra-events.md:30-45` — event-bus convention uses `.synchestra/events.jsonl` (gitignored, ephemeral by convention).

## Recent git activity
- `1640824`: feat(skills/init): implement specstudio:init skill (2026-05-08)

## Prior captures within dedupe window
- None.
```

Note this example contains NO words like "should", "could", "interesting" — only factual observations.
```

- [ ] **Step 2: Create `skills/consilium/roles/scribe.md`**

```markdown
# Role: Scribe

**Name:** scribe
**Group:** — (not a panel role; runs as the pipeline's fifth and final stage)
**Output Schema Version:** 1

## Role Prompt

You are the Scribe for a sidekick consilium that has just produced a verdict. Your job is to synthesize a **Panel summary paragraph** that humans will read on the seed file.

**Inputs you receive:**
1. The seed file's content
2. The full vote bundle (one YAML vote per active roster role)
3. The deterministic verdict from the CLI arbiter (`should-implement`, `should-not-implement`, or `needs-human-review`)
4. The arbiter's rule trace and excluded-votes list (so you know *why* the verdict landed)

**Your output: exactly one paragraph, in one of three flavors based on the verdict.**

### Flavor: converged (when verdict is `should-implement`)

Synthesize the **shared reasoning across roles** that produced consensus. Pattern:

> "All three builders agreed [shared engineering observation]. PM and Marketing both anchored on [shared user-facing observation]. Adversaries stood down because [reason adversaries did not block]."

### Flavor: rejected (when verdict is `should-not-implement`)

Synthesize the **shared reasoning against**:

> "YAGNI Cop's argument that [specific argument] was uncontested. Engineer flagged [specific concern] which Architect confirmed. Customer roles agreed [shared user-facing concern]."

### Flavor: split (when verdict is `needs-human-review`)

Quote the **dissenting arguments side-by-side**:

> "Builders all approved (low complexity). Marketing dissented with high confidence: '[verbatim quote from Marketing's vote argument]'. YAGNI Cop vetoed: '[verbatim quote]'."

**Rules:**

1. **Length cap: 500 characters.** Count strictly — verdicts longer than 500 chars are a contract violation.
2. **Paragraph form only.** No markdown headers, no bullet lists, no code blocks.
3. **You cannot change the verdict.** The arbiter has already set it. Any `verdict:` field you emit is ignored. Focus on synthesizing the *reasoning*, not the outcome.
4. **Quote arguments verbatim when used.** If you cite a role's argument, copy it word-for-word. Don't paraphrase. (Wrap in single quotes.)
5. **Mention high-confidence abstainers when relevant.** If a role abstained with high confidence ("not my domain"), the summary may note "Marketing abstained as out-of-domain."

## Example Outputs

**Converged example:**

> "All three builders agreed the change is local to `skills/init/SKILL.md` and existing tests cover the affected surface. PM and Marketing anchored on the same user pain — post-mortems lose context after `/clear`. Adversaries stood down: YAGNI Cop noted the work is bounded, Skeptic agreed the value is concrete."

**Rejected example:**

> "YAGNI Cop's argument that 'three higher-value items sit in the queue' was uncontested. Engineer flagged a coupling concern between the proposed log persistence and the init skill's stateless design; Architect confirmed the coupling would require an init redesign. Customer roles agreed the user value is real but not urgent."

**Split example:**

> "Builders all approved (low complexity, isolated change). Marketing dissented with high confidence: 'unclear who asks for this feature; no user-research signal.' YAGNI Cop vetoed: 'persistence-across-restarts is a wish, not a problem report.' Verdict downgraded to needs-human-review."
```

- [ ] **Step 3: Lint and commit**

```bash
specscore spec lint --severity warning
# Expected: same 3 pre-existing violations

git add skills/consilium/roles/researcher.md skills/consilium/roles/scribe.md
git commit -m "$(cat <<'EOF'
feat(skills/consilium): add researcher and scribe role files

Two pipeline-end agents: the researcher produces a fact-only briefing
pack before the panel votes; the scribe synthesizes the Panel summary
prose after the arbiter sets the verdict. Both follow the role-file
contract.

Per REQ researcher-fact-only-briefing, REQ briefing-floor-not-ceiling,
REQ scribe-summary-paragraph, REQ scribe-cannot-change-verdict (Feature
sidekick-consilium).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Author Builder role files (Engineer, Architect, QA)

**Files:**
- Create: `skills/consilium/roles/engineer.md`
- Create: `skills/consilium/roles/architect.md`
- Create: `skills/consilium/roles/qa.md`

**Why batched:** the three Builder roles share the same contract structure and are conceptually parallel. Implementer writes Engineer in detail (template) and the other two by analogy.

- [ ] **Step 1: Create `skills/consilium/roles/engineer.md` (worked example)**

```markdown
# Role: Engineer

**Name:** engineer
**Group:** builders
**Output Schema Version:** 1

## Role Prompt

You are the Engineer on a consilium panel reviewing a captured sidekick (sideline-idea) seed. Your perspective: **what would I have to build, what breaks, what does this couple to?**

You will receive the seed file and the researcher's briefing pack. You may use `Read`, `Grep`, and `Glob` to dig deeper into code paths the briefing references when your role demands it (the briefing is a floor, not a ceiling).

**Your job is to return exactly one YAML vote** matching the contract in REQ `vote-schema`. Focus on:

- **Build effort**: how much code does this change? Lines? Files? New tests?
- **Coupling**: does this couple things that should stay separate? Does it cross a module boundary cleanly?
- **Breakage surface**: what existing tests or behaviors could regress?
- **Reversibility**: if we ship this and it's wrong, how hard is it to roll back?

**Vote in this exact shape:**

```yaml
verdict: should-implement | should-not-implement | no-opinion | abstain
confidence: low | medium | high
cost: 🟢 | 🟡 | 🔴
complexity: 🟢 | 🟡 | 🔴
argument: <one-sentence strongest argument, ≤ 280 characters>
```

**Calibration:**

- `cost: 🟢` = ≤ 1 day; `🟡` = a few days; `🔴` = a week+
- `complexity: 🟢` = local change, no new abstractions; `🟡` = touches multiple modules cleanly; `🔴` = requires architectural rework or new patterns
- `abstain` only if the seed is *clearly* outside engineering concerns (a pure marketing-copy change, a docs-only fix). Use `abstain` sparingly — most sidekick ideas have an engineering dimension.

## Example Vote

```yaml
verdict: should-implement
confidence: high
cost: 🟢
complexity: 🟢
argument: "Localizable to `skills/init/SKILL.md`; existing test surface covers the change; rollback is a one-line revert."
```
```

- [ ] **Step 2: Create `skills/consilium/roles/architect.md` (by analogy)**

Following the same template as Engineer, swap the perspective:

```markdown
# Role: Architect

**Name:** architect
**Group:** builders
**Output Schema Version:** 1

## Role Prompt

You are the Architect on a consilium panel reviewing a captured sidekick seed. Your perspective: **where does this fit in the system, does it create new boundaries or violate existing ones?**

You will receive the seed file and the researcher's briefing pack. You may use `Read`, `Grep`, and `Glob` to inspect existing boundaries when your role demands it.

**Focus on:**

- **System fit**: does the change have an obvious home, or does it cross-cut?
- **Boundaries**: does it create a new abstraction that earns its complexity, or does it leak responsibilities?
- **Long-term shape**: would a senior architect 6 months from now applaud or wince?
- **Composability**: does the change compose with existing patterns or fight them?

**Vote shape:** same as Engineer (per REQ `vote-schema`).

**Calibration:**

- `complexity: 🟢` = aligns with existing patterns; `🟡` = introduces a new pattern with clear boundaries; `🔴` = pattern conflict or unclear ownership
- `cost: 🟢` = bounded design surface; `🟡` = moderate design work; `🔴` = redesign required
- `abstain` if the seed is *clearly* outside the system's design surface (e.g., a docs-only fix).

## Example Vote

```yaml
verdict: should-implement
confidence: medium
cost: 🟡
complexity: 🟡
argument: "Cleanly extends the existing event-bus convention; adds one new event type, no boundary violation."
```
```

- [ ] **Step 3: Create `skills/consilium/roles/qa.md` (by analogy)**

```markdown
# Role: QA

**Name:** qa
**Group:** builders
**Output Schema Version:** 1

## Role Prompt

You are the QA engineer on a consilium panel reviewing a captured sidekick seed. Your perspective: **how do I prove this works, what's the test surface, what edge cases exist?**

You will receive the seed file and the researcher's briefing pack. You may use `Read`, `Grep`, and `Glob` to inspect existing tests when your role demands it.

**Focus on:**

- **Test surface**: is the change testable? With what kind of test (unit/integration/manual)?
- **Edge cases**: what failure modes does the change introduce or fix?
- **Regression risk**: what existing behaviors could the change break that aren't currently tested?
- **Test maintenance**: would the test for this change be brittle (flaky, slow, hard to maintain)?

**Vote shape:** same as Engineer (per REQ `vote-schema`).

**Calibration:**

- `cost: 🟢` = obvious test, one fixture; `🟡` = several test scenarios; `🔴` = test infrastructure needs to be built first
- `complexity: 🟢` = test is pure-function; `🟡` = test needs fixtures or mocks; `🔴` = test requires a real environment (browser, server, etc.)
- `abstain` if the seed is *clearly* not testable in the deterministic-observable way Rehearse stubs expect (e.g., subjective UX polish).

## Example Vote

```yaml
verdict: should-implement
confidence: high
cost: 🟢
complexity: 🟢
argument: "Filesystem-observable; one fixture seed + one assertion that the back-link section appears in the source artifact."
```
```

- [ ] **Step 4: Lint and commit**

```bash
specscore spec lint --severity warning
git add skills/consilium/roles/engineer.md skills/consilium/roles/architect.md skills/consilium/roles/qa.md
git commit -m "$(cat <<'EOF'
feat(skills/consilium): add Builder role files (Engineer, Architect, QA)

Three Builders for the default 9-role panel. Engineer voted from the
build-effort/coupling/reversibility lens. Architect from the system-
boundary/composability lens. QA from the test-surface/regression-risk
lens. Each follows the role-file contract.

Per REQ default-roster-9-roles (Feature sidekick-consilium).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Author Customer role files (PM, UX, Marketing)

**Files:**
- Create: `skills/consilium/roles/pm.md`
- Create: `skills/consilium/roles/ux.md`
- Create: `skills/consilium/roles/marketing.md`

**Why batched:** same as Task 5 — three Customer roles share the contract.

- [ ] **Step 1: Create `skills/consilium/roles/pm.md`**

```markdown
# Role: PM

**Name:** pm
**Group:** customers
**Output Schema Version:** 1

## Role Prompt

You are the Product Manager on a consilium panel reviewing a captured sidekick seed. Your perspective: **what user problem does this solve, what's the success metric, who asked for it?**

**Focus on:**

- **User problem**: is there a real user job-to-be-done, or is this engineer-internal polish?
- **Success metric**: how would we know shipping this helped? Is there a measurable signal?
- **Demand signal**: did users actually report this, or are we anticipating?
- **Priority versus other work**: is this displacing higher-value work, or filling slack?

**Vote shape:** same as Engineer (per REQ `vote-schema`).

**Calibration:**

- `abstain (high confidence)` if the seed is purely engineer-internal (refactor, dead-code removal, dependency bump) with no user-facing surface.
- `abstain (low confidence)` if you can't tell — the seed is too technical for you to assess user impact.
- `should-implement` requires either an explicit user demand signal OR a clear job-to-be-done you can articulate.

## Example Vote

```yaml
verdict: abstain
confidence: high
cost: 🟢
complexity: 🟢
argument: "Pure internal refactor; no user-visible surface I can see. Engineer and Architect own the call."
```
```

- [ ] **Step 2: Create `skills/consilium/roles/ux.md`**

```markdown
# Role: UX

**Name:** ux
**Group:** customers
**Output Schema Version:** 1

## Role Prompt

You are the UX designer on a consilium panel reviewing a captured sidekick seed. Your perspective: **what does this feel like to use, is the interaction model coherent with the rest of the product?**

**Focus on:**

- **Interaction coherence**: does the change feel like it belongs to the same product, or does it introduce a new way of doing things?
- **Friction**: does it add steps, options, or cognitive load that wasn't there before?
- **Discoverability**: would a user find this feature/change when they need it?
- **Error states**: what happens when this goes wrong from the user's view?

**Vote shape:** same as Engineer (per REQ `vote-schema`).

**Calibration:**

- `abstain` if the seed has no user-facing interaction (pure backend, pure infrastructure).
- `should-not-implement` if the change adds friction or fragments the interaction model without proportional value.

## Example Vote

```yaml
verdict: should-implement
confidence: medium
cost: 🟢
complexity: 🟢
argument: "Reuses the existing acknowledgement line pattern; no new interaction model introduced."
```
```

- [ ] **Step 3: Create `skills/consilium/roles/marketing.md`**

```markdown
# Role: Marketing

**Name:** marketing
**Group:** customers
**Output Schema Version:** 1

## Role Prompt

You are the Marketing voice (think VP of Marketing or Growth) on a consilium panel reviewing a captured sidekick seed. Your perspective: **can we explain this in one sentence to someone who's never seen the product? Does it move a needle anyone cares about?**

**Focus on:**

- **Story**: is there a one-sentence narrative we could tell about this change?
- **Differentiation**: does it strengthen what makes the product distinctive, or is it parity work?
- **Audience**: which segment cares about this? Adopters? Power users? New users?
- **Public-API surface**: does the change affect anything externally visible (docs, demos, screenshots, public contracts)?

**Vote shape:** same as Engineer (per REQ `vote-schema`).

**Calibration:**

- `abstain (high confidence)` if the seed is purely internal with no externally-visible surface — most refactors, most dependency bumps.
- **BUT**: pay attention to subtle external surfaces. An "internal refactor" that secretly changes a public API contract should NOT be abstained. The safety catch is your job as the constant-panel member who reads everything.

## Example Vote

```yaml
verdict: abstain
confidence: high
cost: 🟢
complexity: 🟢
argument: "Internal codepath refactor with no user-visible surface. No story to tell."
```
```

- [ ] **Step 4: Lint and commit**

```bash
specscore spec lint --severity warning
git add skills/consilium/roles/pm.md skills/consilium/roles/ux.md skills/consilium/roles/marketing.md
git commit -m "$(cat <<'EOF'
feat(skills/consilium): add Customer role files (PM, UX, Marketing)

Three Customers for the default 9-role panel. PM voted from the user-
problem/success-metric/demand-signal lens. UX from the interaction-
coherence/friction lens. Marketing from the story/differentiation lens
— with explicit safety-catch behavior for "internal refactors that
secretly change a public API contract."

Per REQ default-roster-9-roles (Feature sidekick-consilium).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Author Adversary role files (YAGNI Cop, Skeptic, Security/Ops)

**Files:**
- Create: `skills/consilium/roles/yagni-cop.md`
- Create: `skills/consilium/roles/skeptic.md`
- Create: `skills/consilium/roles/security-ops.md`

**Why batched:** same as Tasks 5 and 6.

- [ ] **Step 1: Create `skills/consilium/roles/yagni-cop.md`**

```markdown
# Role: YAGNI Cop

**Name:** yagni-cop
**Group:** adversaries
**Output Schema Version:** 1

## Role Prompt

You are the YAGNI Cop on a consilium panel reviewing a captured sidekick seed. Your job is to push back on speculative work. Your perspective: **what's the smallest version that delivers the value? Is this a feature or a wish?**

**Focus on:**

- **Speculative scope**: does the seed include "while we're at it" expansions beyond the core idea?
- **Hypothetical users**: does the seed cite "users who might want…" instead of users who have asked?
- **Premature abstraction**: does the seed propose a generic mechanism for a single concrete need?
- **Opportunity cost**: are there already-queued or in-flight items with stronger evidence?

**Vote shape:** same as the panel (per REQ `vote-schema`).

**Calibration:**

- Default toward `should-not-implement` when the seed lacks concrete evidence of need. Adversaries earn their seat by saying "no" when the room is leaning "yes" on weak signal.
- `abstain` only if the seed is *clearly* well-bounded and concrete enough that there's nothing to push back on.
- High-confidence `should-not-implement` triggers the adversary-veto rule (per REQ `arbiter-gate-rules` step 4) — use it deliberately, not reflexively.

## Example Vote

```yaml
verdict: should-not-implement
confidence: high
cost: 🟡
complexity: 🟡
argument: "Solves a problem the user didn't report. Three smaller wins are already queued. Defer."
```
```

- [ ] **Step 2: Create `skills/consilium/roles/skeptic.md`**

```markdown
# Role: Skeptic

**Name:** skeptic
**Group:** adversaries
**Output Schema Version:** 1

## Role Prompt

You are the Skeptic (Devil's Advocate) on a consilium panel reviewing a captured sidekick seed. Your job is to argue against. **Assume the user is wrong. What's the strongest case for *not* building this?**

**Focus on:**

- **Hidden costs**: what costs aren't yet visible — maintenance, on-call, doc debt, training?
- **Wrong abstraction**: is the seed's framing of the problem actually the right framing?
- **Better alternatives**: is there a simpler or already-existing solution being overlooked?
- **Reversibility cost**: if we ship this and it's wrong, what's the cleanup?

**Vote shape:** same as the panel (per REQ `vote-schema`).

**Calibration:**

- Even on well-justified seeds, the Skeptic produces a substantive counter-argument. The argument may be wrong; that's the point — the panel reads the strongest case against and decides.
- `abstain` extremely rarely. The Skeptic role is constructed to engage.
- High-confidence `should-not-implement` is the veto signal — use when you genuinely believe the case against is strong, not as a default stance.

## Example Vote

```yaml
verdict: should-not-implement
confidence: medium
cost: 🟡
complexity: 🟡
argument: "Persistent logs that survive `/clear` defeat the user's own choice to clear context — they will accidentally see context they intended to erase."
```
```

- [ ] **Step 3: Create `skills/consilium/roles/security-ops.md`**

```markdown
# Role: Security/Ops

**Name:** security-ops
**Group:** adversaries
**Output Schema Version:** 1

## Role Prompt

You are the Security/Ops voice on a consilium panel reviewing a captured sidekick seed. Your perspective: **what attack surface does this add, who pages at 3am when it breaks, what does this cost to run at 100x?**

**Focus on:**

- **Attack surface**: does the change accept untrusted input, store data, expose new endpoints, or touch credentials?
- **Operational cost**: what does this cost to run? Storage? Network? Compute? Monitoring overhead?
- **Failure modes**: what happens at 100x current scale? What happens during a partial outage?
- **Data lifecycle**: does this introduce data that needs retention, encryption, or deletion policies?

**Vote shape:** same as the panel (per REQ `vote-schema`).

**Calibration:**

- `abstain (high confidence)` if the seed has no security or operational surface — pure documentation, pure local-developer-only behavior.
- `should-not-implement` when the seed introduces real risk that isn't compensated by clear value.
- The single Security/Ops adversary has substantial veto weight — use high-confidence `should-not-implement` when you genuinely believe the risk is unmitigated.

## Example Vote

```yaml
verdict: abstain
confidence: high
cost: 🟢
complexity: 🟢
argument: "Local-only debug-log persistence; no network exposure, no shared storage, no credentials handled."
```
```

- [ ] **Step 4: Lint and commit**

```bash
specscore spec lint --severity warning
git add skills/consilium/roles/yagni-cop.md skills/consilium/roles/skeptic.md skills/consilium/roles/security-ops.md
git commit -m "$(cat <<'EOF'
feat(skills/consilium): add Adversary role files (YAGNI Cop, Skeptic, Security/Ops)

Three Adversaries for the default 9-role panel. YAGNI Cop pushes back
on speculative scope. Skeptic constructs the strongest case against.
Security/Ops voices attack-surface and operational concerns. Each has
calibration guidance on when high-confidence should-not-implement
triggers the adversary-veto rule.

Per REQ default-roster-9-roles, REQ arbiter-gate-rules (Feature
sidekick-consilium).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Author the `specstudio:consilium` SKILL.md

**Files:**
- Create: `skills/consilium/SKILL.md`

**Why this task is big:** the skill orchestrates the 5-stage pipeline, threads transcript capture through every stage (non-retrofittable per Cross-Cutting Decision 1), and implements the verdict-write/back-link/event-emit flow. Steps split the file into independently verifiable sections.

- [ ] **Step 1: Frontmatter + intro**

Create `skills/consilium/SKILL.md` with this opening:

```markdown
---
name: consilium
description: |
  Drains captured sidekick (sideline-idea) seeds and produces deterministic
  verdicts via a 5-stage pipeline: CLI gather → researcher agent → 9-role
  parallel expert panel → CLI arbiter → scribe agent. The Synchestra task
  is the structured source of truth; the seed gets only the scribe's prose
  summary mirrored into a ## Consilium Verdict section. Per-project roster
  and gate configurable via specscore.yaml → consilium: block. No auto-
  promotion at this layer (Phase 2).
  Triggers: "specstudio:consilium", "/consilium", "run the consilium",
  "drain the sidekick queue", "review sidekick ideas".
aliases: [consilium]
---

# Consilium

The consilium drains queued `consilium-review` Synchestra tasks one at a time, running the full 5-stage pipeline per task. On every successful task, the verdict + scribe summary mirror onto the seed and the `sidekick-idea.reviewed` event fires. On any stage failure, the task transitions to `failed` and the next queued task continues.

For *what* a sidekick seed is and how it gets captured, read [Phase 0's `sidekick-capture` Feature](../../spec/features/sidekick-capture/README.md). For the verdict gate's full algorithm, read [REQ `arbiter-gate-rules`](../../spec/features/sidekick-consilium/README.md#req-arbiter-gate-rules) in this skill's source Feature.
```

- [ ] **Step 2: When-to-use + Pre-flight**

Append:

```markdown
## When to Use

- The user typed `/consilium` or "run the consilium" or any other trigger phrase.
- The user wants to drain queued sidekick captures into reviewed verdicts.

## Pre-flight

Before claiming any task, verify the cross-repo dependencies are present:

1. `command -v specscore` — the arbiter subcommand lives here (`specscore consilium verdict`).
2. `command -v synchestra` — the task lifecycle lives here (`synchestra task claim`, `synchestra task update`).
3. `specscore --version` — must include the `consilium verdict` subcommand. If absent, exit cleanly with a message: "Phase 1 requires `specscore` with the `consilium verdict` subcommand (cross-repo dependency, tracked in `spec/plans/sidekick-consilium-arbiter-companion.md`). Install or upgrade and re-run."
4. `synchestra task types` — must include `consilium-review`. If absent, exit cleanly with the analogous message referencing the task-type companion plan.

If either cross-repo dependency is missing, do NOT claim any task and do NOT modify any file. Exit with the actionable error.
```

- [ ] **Step 3: The 5-stage pipeline overview**

Append:

```markdown
## The pipeline (per claimed task)

Once pre-flight passes, claim all `queued` tasks and run this pipeline once per claim:

```
1. CLI gather       (deterministic, ~1K tokens of context)
2. Researcher       (one LLM call, produces briefing pack)
3. Expert panel     (N parallel LLM calls, one per active roster role)
4. CLI arbiter      (deterministic, applies the gate rules)
5. Scribe           (one LLM call, advisory only)
```

Each stage emits a transcript record (see "Transcript capture" below). Stage failures transition the task to `failed` with a typed reason; the next task continues.
```

- [ ] **Step 4: Transcript capture — threaded through every stage**

Append (this is the foundational concept the reviewer flagged as non-retrofittable):

````markdown
## Transcript capture (REQ `pipeline-transcript-capture`)

The skill MUST capture a structured transcript record at EACH stage's start and finalize at its end. This is NOT optional and NOT retrofittable — every stage's instructions below interleave transcript capture as integral to the stage, not a bolt-on.

**Transcript structure** (one entry per stage, plus per-role entries for the panel stage):

```yaml
pipeline_transcript:
  - stage: gather
    started_at: <ISO-8601>
    ended_at: <ISO-8601>
    outcome: ok | failed
    error: <string, only when outcome: failed>
    commands: [<bash commands run>]
    output_summary: <stdout/stderr truncated to 4KB>
  - stage: researcher
    started_at: …
    ended_at: …
    outcome: ok | failed
    input: { seed_path: <path>, raw_context_ref: <path> }
    output: { briefing_pack: <markdown string or ref> }
  - stage: panel
    started_at: …
    ended_at: …
    outcome: ok | failed
    roles:                                # one entry per active roster role
      - role: <role-slug>
        input_includes_briefing: true
        tool_calls: [<tool-name>, ...]    # empty list if none
        vote: { verdict: …, confidence: …, cost: …, complexity: …, argument: … }
  - stage: arbiter
    started_at: …
    ended_at: …
    outcome: ok | failed
    input: { votes_path: <path>, roster_path: <path>, gate_path: <path>, seed_path: <path> }
    output: { verdict: …, rule_trace: […], excluded_votes: […], denominators: {...} }
  - stage: scribe
    started_at: …
    ended_at: …
    outcome: ok | failed
    input: { verdict: …, votes: […], briefing_ref: <path> }
    output: { summary_paragraph: <≤500 chars> }
```

**The transcript is written to the synchestra task as `pipeline_transcript` before the task transitions to `complete`.** This is what makes the transcript-shape ACs (`pipeline-runs-five-stages-in-order`, `every-expert-receives-briefing-and-may-research-deeper`, `panel-fans-out-in-parallel`, `pipeline-transcript-payload-shape`) observable. If the transcript is missing or malformed, the task MUST transition to `failed` with reason `malformed-transcript` — even if the verdict itself is otherwise valid.
````

- [ ] **Step 5: Stage 1 — CLI gather**

Append:

````markdown
## Stage 1: CLI gather (deterministic)

**Transcript:** start a record with `stage: gather, started_at: <now>`.

Run these commands in order (capture stdout + stderr; truncate the summary to 4KB):

```bash
# 1. Related features
synchestra feature list --related "$SEED_SLUG" 2>&1 | head -50

# 2. Code-to-spec refs for captured_during
synchestra code refs "$CAPTURED_DURING" 2>&1 | head -50

# 3. Recent git activity in relevant paths
git log --oneline --since="30 days ago" -- "$CAPTURED_DURING" 2>&1 | head -20

# 4. Prior captures within the dedupe window (30 days)
ls -t spec/ideas/seeds/ 2>/dev/null | head -10
```

Assemble the outputs into a single context bundle file at `.synchestra/consilium/<task-id>/raw-context.md`. This file is the researcher's primary input.

**Transcript:** finalize with `ended_at: <now>, outcome: ok, commands: […], output_summary: <truncated combined output>`. On any command failure, set `outcome: failed, error: <stderr first line>` and transition the task to `failed` with reason `gather-failed`.
````

- [ ] **Step 6: Stage 2 — Researcher**

Append:

````markdown
## Stage 2: Researcher (one LLM call)

**Transcript:** start a record with `stage: researcher, started_at: <now>`.

Dispatch one Agent tool call with:

- `subagent_type: general-purpose`
- The seed file's full content + the raw context bundle from Stage 1
- The researcher prompt body from `skills/consilium/roles/researcher.md` (read it and include verbatim as the agent's instruction prompt)

Capture the agent's response. Write it to `.synchestra/consilium/<task-id>/briefing.md`.

**Validate the briefing**:
- Must contain the four `##` sections per the researcher's contract (Related artifacts, Code references, Recent git activity, Prior captures within dedupe window).
- Total length ≤ 1500 tokens (use `wc -c` and divide by ~4 for a quick approximation).

If validation fails, retry the Agent call ONCE with the same prompt. If it fails again, transition the task to `failed` with reason `researcher-malformed`.

**Transcript:** finalize with `ended_at: <now>, outcome: ok, input: {seed_path, raw_context_ref}, output: {briefing_pack: <file ref or inline ≤2KB>}`.
````

- [ ] **Step 7: Stage 3 — Expert panel (parallel)**

Append:

````markdown
## Stage 3: Expert panel (N parallel LLM calls)

**Transcript:** start a record with `stage: panel, started_at: <now>`.

### Load the active roster

```bash
# Check for project-level overrides in specscore.yaml
ROSTER=$(specscore consilium roster --resolve)
# Output is YAML: a list of {name, group, path} per active role
```

If `specscore.yaml` has no `consilium.roster` block, `--resolve` returns the 9-role default. Otherwise, defaults + customs - excludes per REQ `roster-exclude-and-custom`.

### Read all role files

For each role in the active roster, read its markdown file. Extract the `## Role Prompt` section's content as the agent's prompt for that role.

### Dispatch all role agents in parallel

This is the critical step for REQ `parallel-fan-out`. Use **a single message containing N Agent tool calls** so all role agents run concurrently:

```
Single message with N parallel Agent invocations:
  Agent (subagent_type: general-purpose, description: "engineer vote", prompt: <engineer role prompt + seed + briefing>)
  Agent (subagent_type: general-purpose, description: "architect vote", prompt: <architect role prompt + seed + briefing>)
  … (one per active role)
```

Each agent receives:
- The seed file's full content
- The briefing pack from Stage 2
- The role's prompt body

Each agent returns one YAML vote per REQ `vote-schema`.

**Per-role transcript entry**: as each agent returns, capture:
```yaml
- role: <role-slug>
  input_includes_briefing: true       # always true; assert this as a discipline check
  tool_calls: [<list of tools the agent invoked beyond the briefing>]
  vote: <parsed YAML vote>
```

**Validate each vote** against REQ `vote-schema` (5 required fields, enum values valid, argument ≤ 280 chars). Any malformed vote transitions the task to `failed` with reason `malformed-vote`. Do NOT discard a single malformed vote and continue — zero tolerance per REQ `vote-schema`.

**Transcript:** finalize the panel stage record with `ended_at: <now>, outcome: ok, roles: [<per-role entries>]`.
````

- [ ] **Step 8: Stage 4 — CLI arbiter**

Append:

````markdown
## Stage 4: CLI arbiter (deterministic)

**Transcript:** start a record with `stage: arbiter, started_at: <now>`.

Write the votes, roster snapshot, and active gate config to temporary YAML files:

```bash
mkdir -p .synchestra/consilium/<task-id>
echo "$VOTES_YAML" > .synchestra/consilium/<task-id>/votes.yaml
echo "$ROSTER_YAML" > .synchestra/consilium/<task-id>/roster.yaml
specscore consilium config --print-gate > .synchestra/consilium/<task-id>/gate.yaml
```

Invoke the arbiter:

```bash
specscore consilium verdict \
  --votes .synchestra/consilium/<task-id>/votes.yaml \
  --roster .synchestra/consilium/<task-id>/roster.yaml \
  --gate .synchestra/consilium/<task-id>/gate.yaml \
  --seed "$SEED_PATH" \
  > .synchestra/consilium/<task-id>/verdict.yaml
```

The arbiter's stdout YAML contains: `verdict`, `rule_trace`, `excluded_votes`, `denominators`.

On non-zero exit (validation failure: malformed vote, invalid roster, file not found), transition the task to `failed` with reason from the arbiter's stderr.

**Transcript:** finalize with `ended_at: <now>, outcome: ok, input: {<paths>}, output: <parsed verdict YAML>`.
````

- [ ] **Step 9: Stage 5 — Scribe**

Append:

````markdown
## Stage 5: Scribe (one LLM call, advisory only)

**Transcript:** start a record with `stage: scribe, started_at: <now>`.

Dispatch one Agent tool call with:

- `subagent_type: general-purpose`
- The seed + the votes (all of them) + the verdict + the rule trace
- The scribe prompt from `skills/consilium/roles/scribe.md` (read verbatim)

Capture the agent's response. **Ignore any `verdict:` field the scribe emits** — the arbiter has already set the verdict per REQ `scribe-cannot-change-verdict`. Only the prose paragraph is consumed.

**Validate the prose:**
- Length ≤ 500 characters.
- No markdown headers (no lines starting with `#`).
- No bullet lists (no lines starting with `- ` or `* `).
- No code blocks (no triple-backticks).

If validation fails, retry ONCE. If it fails again, transition to `failed` with reason `scribe-malformed`.

**Transcript:** finalize with `ended_at: <now>, outcome: ok, input: {verdict, votes_ref, briefing_ref}, output: {summary_paragraph: <prose>}`.
````

- [ ] **Step 10: Write-back: task payload + seed mirror + event emission**

Append:

````markdown
## Write-back (post-pipeline, before task transitions to complete)

### Update the task payload

Per REQ `verdict-source-of-truth-in-task`, the task carries the full structured payload:

```bash
synchestra task update <task-id> \
  --field roster_snapshot=@.synchestra/consilium/<task-id>/roster.yaml \
  --field votes=@.synchestra/consilium/<task-id>/votes.yaml \
  --field briefing_pack=@.synchestra/consilium/<task-id>/briefing.md \
  --field arbiter_output=@.synchestra/consilium/<task-id>/verdict.yaml \
  --field pipeline_transcript=@.synchestra/consilium/<task-id>/transcript.yaml \
  --field tokens_total=<computed_total> \
  --field scribe_summary=<scribe paragraph>
```

### Mirror the scribe summary to the seed

Per REQ `verdict-summary-in-seed`, append a `## Consilium Verdict` section to the seed file. Placement: immediately before the SpecScore footer line if present, else end-of-file. If a `## Consilium Verdict` section already exists (re-enqueue case), replace in place (do NOT duplicate).

Section format:

```markdown
## Consilium Verdict

**Verdict:** <verdict-enum> (<YYYY-MM-DD>)

Full payload: synchestra task <task-id>.

<scribe prose paragraph>
```

### Transition the task to complete

```bash
synchestra task update <task-id> --status complete
```

### Emit `sidekick-idea.reviewed`

Per REQ `event-reviewed-emitted`, emit the event with the envelope+payload from REQ:

```bash
# If synchestra emit is available, prefer it
if synchestra emit --help &>/dev/null 2>&1; then
  synchestra emit sidekick-idea.reviewed <event-yaml>
else
  # Fallback: append JSONL directly
  jq -c -n \
    --arg event "sidekick-idea.reviewed" \
    --arg uuid "$(uuidgen)" \
    --arg ts "$(date -u +%FT%TZ)" \
    --arg slug "$SEED_SLUG" \
    --arg verdict "$VERDICT" \
    '{event: $event, version: 1, uuid: $uuid, timestamp: $ts, …, payload: {slug: $slug, verdict: $verdict, …}}' \
    >> .synchestra/events.jsonl
fi
```

The full envelope+payload structure is in `skills/shared/synchestra-events.md` under the `sidekick-idea.reviewed` section.

### Output to the operator

Print one line per completed task:

```
Reviewed: <seed-slug> → <verdict> (synchestra task <task-id>)
```

After all queued tasks are processed, print a summary:

```
Consilium drained: <N> reviews complete, <M> failed, <K> awaiting human review.
```
````

- [ ] **Step 11: References + Red Flags**

Append:

````markdown
## Red Flags

These patterns indicate misuse and should be refused or refactored:

- Per-slug invocation (`/consilium <slug>`) — not supported in Phase 1; reject with the error in REQ `drain-all-queued`.
- `--limit` or any other invocation-shape variant — same.
- Skipping the transcript capture — every stage MUST record its transcript entry; missing entries are a contract violation.
- Continuing the pipeline past a malformed vote — zero tolerance per REQ `vote-schema`.
- Editing the verdict based on the scribe's output — by contract the scribe cannot change the verdict.
- Reviewing a seed whose `content_hash` doesn't match the task's stored hash — transition to `failed` with reason `seed-mutated` per REQ `seed-mutation-detection`.

## References

- [`shared/sidekick-capture.md`](../shared/sidekick-capture.md) — the directive Phase 0 ships; explains *what* sidekick seeds are.
- [`shared/synchestra-events.md`](../shared/synchestra-events.md) — event envelope, including `sidekick-idea.reviewed`.
- [`roles/`](roles/) — the 11 default role files (researcher + scribe + 9 panel roles).
- [`references/role-file-contract.md`](references/role-file-contract.md) — the markdown contract every role file follows.
- [`references/consilium-config-example.md`](references/consilium-config-example.md) — the `specscore.yaml` `consilium:` block schema.
- [Feature: `sidekick-consilium`](../../spec/features/sidekick-consilium/README.md) — the spec this skill implements.
- [`spec/plans/sidekick-consilium-arbiter-companion.md`](../../spec/plans/sidekick-consilium-arbiter-companion.md) — cross-repo arbiter dependency.
- [`spec/plans/sidekick-consilium-task-companion.md`](../../spec/plans/sidekick-consilium-task-companion.md) — cross-repo task type dependency.
````

- [ ] **Step 12: Lint and commit**

```bash
specscore spec lint --severity warning
git add skills/consilium/SKILL.md
git commit -m "$(cat <<'EOF'
feat(skills/consilium): add the specstudio:consilium skill

Phase 1 orchestrator. Drains queued consilium-review tasks one at a
time, running the 5-stage pipeline: CLI gather → researcher → 9-role
parallel panel → CLI arbiter → scribe. Per-stage transcript capture
threaded through every section (non-retrofittable per REQ pipeline-
transcript-capture). Pre-flight checks the cross-repo dependencies
(specscore consilium verdict subcommand, synchestra consilium-review
task type) before claiming any task.

Implements 17 REQs from Feature sidekick-consilium:
- invocation-triggers, drain-all-queued, pipeline-five-stages
- per-seed-budget-25k, seed-mutation-detection
- researcher-fact-only-briefing, briefing-floor-not-ceiling
- scribe-summary-paragraph, scribe-cannot-change-verdict
- parallel-fan-out, vote-schema, abstain-with-confidence
- pipeline-transcript-capture, verdict-source-of-truth-in-task
- verdict-summary-in-seed, consilium-review-task-lifecycle
- event-reviewed-emitted

Cross-repo dependencies still required for end-to-end function:
- specscore-cli arbiter (companion plan; tracked upstream)
- synchestra consilium-review task type (companion plan; tracked upstream)

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Scaffold Rehearse stubs

**Files:**
- Create: 25 stub files + 1 `_skipped.md` under `spec/features/sidekick-consilium/_tests/`

**Why:** stubs scaffold the testable observable surface for each AC. The Phase 0 precedent (T7 of `sidekick-capture.md`) is the structural template; ACs covered are listed in the Feature spec's `## Rehearse Integration` section.

- [ ] **Step 1: Create the directory**

```bash
mkdir -p spec/features/sidekick-consilium/_tests
```

- [ ] **Step 2: Stub template**

Each stub file at `spec/features/sidekick-consilium/_tests/<ac-slug>.md` follows this format:

```markdown
---
type: rehearse-stub
status: pending
ac: <ac-slug>
feature: sidekick-consilium
---

# Rehearse: <ac-slug>

## Scenario (from AC)

**Given** <verbatim Given from the AC in spec/features/sidekick-consilium/README.md>
**When** <verbatim When>
**Then** <verbatim Then>

## Verification approach

<2–4 sentences describing how to actually execute this scenario: fixture setup, command run, assertion target.>

---
*This document follows the https://specscore.md/scenario-specification*
```

- [ ] **Step 3: Write all 25 stubs**

Read `spec/features/sidekick-consilium/README.md`'s Acceptance Criteria section. For each of the 25 ACs (all except `calibration-set-passes-95-percent`), copy the Given/When/Then verbatim into a stub and add a 2–4-sentence verification approach.

The 25 stub filenames:

```
invocation-drains-all-queued-tasks.md
invocation-rejects-per-slug-argument.md
pipeline-runs-five-stages-in-order.md
token-usage-recorded-on-task.md
seed-mutation-blocks-review.md
researcher-briefing-contains-no-judgment.md
every-expert-receives-briefing-and-may-research-deeper.md
scribe-summary-respects-flavor-and-length.md
scribe-verdict-field-ignored.md
default-roster-is-9-roles-in-three-groups.md
panel-fans-out-in-parallel.md
malformed-vote-fails-pipeline.md
abstain-high-confidence-excluded-from-denominator.md
abstain-low-confidence-caps-verdict.md
custom-role-loads-and-votes.md
roster-with-malformed-custom-role-rejected.md
roster-violating-group-floor-rejected.md
roster-snapshot-stored-on-task.md
pipeline-transcript-payload-shape.md
verdict-task-payload-completeness.md
seed-gets-consilium-verdict-section.md
reviewed-event-emitted-on-success.md
arbiter-reproducibility-snapshot.md
idempotent-task-creation-on-duplicate-hash.md
concurrent-claim-loses-cleanly.md
```

For verification approaches, draw from the Feature's Rehearse Integration list — it describes each stub's observable in a short phrase.

- [ ] **Step 4: Write the skip-record for the calibration AC**

Create `spec/features/sidekick-consilium/_tests/_skipped.md`:

```markdown
---
type: rehearse-skip-record
feature: sidekick-consilium
---

# Rehearse: Skipped ACs

## AC: `calibration-set-passes-95-percent`

**Skip reason:** This AC is a *quality gate on the Phase 1 ship decision*, not a runtime test of skill behavior. Rehearse stubs assert against deterministic observables; calibration assertions require human post-hoc judgment ("would you have made the same call?") on each of 20 verdicts. Not automatable in Phase 1.

**Coverage approach:** manual gate per Task 10 of `spec/plans/sidekick-consilium.md`. A future Rehearse pattern for "AC verified by human review of a Synchestra task field" could pick this up automatically. Tracked as a Rehearse roadmap item; not blocking Phase 1.

---
*This document follows the https://specscore.md/scenario-specification*
```

- [ ] **Step 5: Add `_tests/README.md`**

If lint demands a README in `_tests/` (per the Phase 0 precedent), create `spec/features/sidekick-consilium/_tests/README.md` modeled on the existing `spec/features/skills/ideate/_tests/README.md`.

- [ ] **Step 6: Lint and commit**

```bash
specscore spec lint --severity warning
# Expected: same 3 pre-existing dogfood-seed violations; nothing new

ls spec/features/sidekick-consilium/_tests/*.md | wc -l
# Expected: 27 (25 stubs + _skipped.md + README.md)

git add spec/features/sidekick-consilium/_tests/
git commit -m "$(cat <<'EOF'
test(features/sidekick-consilium): scaffold 25 Rehearse stubs + 1 skip

One stub per testable AC, status: pending. Each carries the AC's
Given/When/Then verbatim from the Feature spec plus a 2–4-sentence
verification approach. One skip-reason record for
calibration-set-passes-95-percent (manual quality gate per Task 10
of the plan).

Per Rehearse Integration section (Feature sidekick-consilium).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: Construct the 20-seed calibration set

**Files:**
- Create: `spec/features/sidekick-consilium/calibration/README.md`
- Create: 5 strong seeds at `spec/features/sidekick-consilium/calibration/strong/<slug>.md`
- Create: 5 weak seeds at `spec/features/sidekick-consilium/calibration/weak/<slug>.md`
- Create: 5 out-of-domain seeds at `spec/features/sidekick-consilium/calibration/out-of-domain/<slug>.md`
- Create: 5 ambiguous seeds at `spec/features/sidekick-consilium/calibration/ambiguous/<slug>.md`

**Why:** REQ `calibration-set-20-verdicts` requires this set before Phase 1 can be declared "Done" (and before Phase 2 is specified). The set is data; constructing it is its own work.

- [ ] **Step 1: Create the directory tree**

```bash
mkdir -p spec/features/sidekick-consilium/calibration/{strong,weak,out-of-domain,ambiguous}
```

- [ ] **Step 2: Write `spec/features/sidekick-consilium/calibration/README.md`**

```markdown
# Sidekick Consilium Calibration Set

20 seed fixtures used by REQ `calibration-set-20-verdicts` to validate that the consilium produces verdicts matching post-hoc human judgment ≥95% of the time, and that adversaries correctly flag known-weak seeds ≥80% of the time.

## Categories

| Category | Count | Expected verdict pattern |
|---|---|---|
| `strong/` | 5 | High confidence `should-implement` from most non-abstain roles |
| `weak/` | 5 | Adversary `should-not-implement` with high confidence in ≥4 of 5; overall verdict `should-not-implement` or `needs-human-review` |
| `out-of-domain/` | 5 | Multiple customer roles high-confidence abstain; verdict still produced (likely `should-implement` if builders converge) |
| `ambiguous/` | 5 | Verdict `needs-human-review`; split votes; rule_trace shows mixed signals |

## Running calibration

After the cross-repo arbiter and task type ship (see Task 11), run:

```bash
# Capture each calibration seed as a sidekick (against this calibration directory as captured_during)
for seed in spec/features/sidekick-consilium/calibration/*/*.md; do
  /sidekick "$(grep -A1 '^# ' "$seed" | tail -1)"
done

# Drain the queue
/consilium

# Inspect the 20 verdicts
synchestra task list --type consilium-review --status complete
```

For each verdict, the human reviewer notes whether they would have made the same call. Calibration passes if ≥ 95% match.

---
*This document follows the https://specscore.md/calibration-specification*
```

(Note: a `calibration-specification` adherence-footer rule may not exist yet in specscore; if lint complains, use the closest available adherence footer or remove the footer with a note.)

- [ ] **Step 3: Write 5 strong seeds**

Each is a real, well-bounded improvement with clear value. Examples:

- `strong/add-test-for-event-emission-failure-path.md` — concrete, tested-elsewhere-pattern, low cost.
- `strong/fix-incorrect-relative-link-in-readme-after-restructure.md` — bug fix, observable.
- `strong/add-gitignore-entry-for-tmp-test-fixtures.md` — hygiene, no risk.
- `strong/document-existing-undocumented-yaml-field-in-events-md.md` — docs gap, easy.
- `strong/extract-shared-fixture-path-constant-used-in-three-tests.md` — DRY, small.

Each file follows the seed format (8-key frontmatter + H1 one-liner). Example for the first:

```markdown
---
type: sidekick-seed
slug: add-test-for-event-emission-failure-path
captured_at: 2026-05-18T00:00:00Z
captured_by: calibration-fixture
captured_during: spec/features/sidekick-consilium/calibration/strong
trigger: explicit
status: queued
synchestra_task: null
---

# Add a test asserting event emission is skipped when seed write fails (failure path is currently uncovered)
```

- [ ] **Step 4: Write 5 weak seeds**

Each is a speculative or unjustified change. Examples:

- `weak/add-a-cli-flag-to-enable-multi-tenant-support-just-in-case.md` — premature abstraction.
- `weak/refactor-init-skill-to-use-a-state-machine-because-it-feels-right.md` — vibes-based refactor.
- `weak/add-emoji-to-all-output-lines-for-visual-pop.md` — taste-bait without signal.
- `weak/precache-all-seed-files-on-startup-in-case-they-are-needed.md` — speculative perf.
- `weak/add-a-config-option-for-the-config-format-version.md` — meta-config without need.

Each follows the same seed format.

- [ ] **Step 5: Write 5 out-of-domain seeds**

Each is a genuinely technical/operational change with no user-visible surface:

- `out-of-domain/upgrade-dev-dependency-version-pinning-policy.md`
- `out-of-domain/migrate-internal-bash-helper-from-bashisms-to-posix.md`
- `out-of-domain/add-internal-debug-logging-flag-disabled-by-default.md`
- `out-of-domain/refactor-test-fixture-directory-naming-convention.md`
- `out-of-domain/extract-internal-helper-function-used-in-one-place-still.md`

- [ ] **Step 6: Write 5 ambiguous seeds**

Each has reasonable arguments both ways:

- `ambiguous/add-versioning-strategy-to-event-payloads.md`
- `ambiguous/introduce-a-second-default-adversary-role-for-balance.md`
- `ambiguous/expose-token-cost-per-role-in-the-task-payload.md`
- `ambiguous/make-the-briefing-pack-cap-configurable-per-project.md`
- `ambiguous/support-utf-8-emojis-in-seed-one-liners.md`

- [ ] **Step 7: Lint and commit**

```bash
specscore spec lint --severity warning
# Will likely report violations for the calibration seeds (same as the dogfood seed —
# spec/ideas/seeds/ rule treats them as misplaced Ideas). Calibration seeds live UNDER
# spec/features/sidekick-consilium/calibration/, NOT spec/ideas/seeds/, so the rule may
# not apply. Verify; if it does apply, add an explicit lint exclusion in specscore.yaml
# scoped to spec/features/sidekick-consilium/calibration/.

git add spec/features/sidekick-consilium/calibration/
git commit -m "$(cat <<'EOF'
test(features/sidekick-consilium): construct 20-seed calibration set

5 strong + 5 weak + 5 out-of-domain + 5 ambiguous fixtures per REQ
calibration-set-20-verdicts. Used to validate the verdict gate's
post-hoc agreement target (≥ 95%) before Phase 1 ships and before
Phase 2 is specified.

Calibration runs against the default 9-role roster + default gate
knobs (per the same REQ); projects with overridden roster/gate own
their own calibration.

Per REQ calibration-set-20-verdicts (Feature sidekick-consilium).

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

---

## Task 11: External gates + final ship checklist

**Files:** none created or modified. This task documents what must be true before Phase 1 transitions from `Implementing` to `Done`.

- [ ] **Step 1: Verify in-repo work is complete**

```bash
# All 8 prior tasks committed and pushed
git log --oneline -15
# Expected: Tasks 1–10 each have a commit.

# Lint is clean against the dogfood seed (i.e., specscore-cli#6 has shipped)
specscore --version
# Expected: a version with the sidekick-seed lint rule registered.

specscore spec lint --severity warning
# Expected: 0 violations.
```

- [ ] **Step 2: Verify cross-repo arbiter has shipped**

```bash
specscore consilium verdict --help
# Expected: subcommand documented; does not error with "unknown command".
```

If this fails, Phase 1 is not yet ready to ship. The arbiter must land in `specscore-cli` first (see `spec/plans/sidekick-consilium-arbiter-companion.md`).

- [ ] **Step 3: Verify cross-repo task type has shipped**

```bash
synchestra task types | grep consilium-review
# Expected: consilium-review listed as a registered task type.
```

If this fails, Phase 1 is not yet ready. Task type must land in `synchestra` first (see `spec/plans/sidekick-consilium-task-companion.md`).

- [ ] **Step 4: Commit the dogfood seed (now unblocked)**

Once Step 1 confirms `specscore-cli#6` has shipped, the seed at `spec/ideas/seeds/rebrand-view-in-specstudio-blockquote-to-view-in-specscore.md` lints cleanly:

```bash
specscore spec lint --severity warning
# Expected: 0 violations.

git add spec/ideas/seeds/rebrand-view-in-specstudio-blockquote-to-view-in-specscore.md
git commit -m "$(cat <<'EOF'
spec(ideas/seeds): land the dogfood seed (specscore-cli#6 shipped)

The first seed in spec/ideas/seeds/ — captured live during the
sidekick-consilium Feature specify session. Lints cleanly now that
the cross-repo sidekick-seed lint rule has shipped.

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"
```

- [ ] **Step 5: Run the calibration set**

```bash
# Capture each calibration seed
for seed in spec/features/sidekick-consilium/calibration/*/*.md; do
  one_liner=$(grep -A1 '^# ' "$seed" | tail -1)
  /sidekick "$one_liner"
done

# Drain the queue
/consilium
```

- [ ] **Step 6: Human review of the 20 verdicts**

For each verdict, record whether the human reviewer agrees with the verdict the consilium produced. Compute:

```
agreement_rate = matches / 20

For known-weak seeds (5 in calibration/weak/):
  weak_flagged_rate = (verdicts of `should-not-implement` or `needs-human-review`) / 5
```

Phase 1 ships only if:
- `agreement_rate >= 0.95` (≥ 19 out of 20)
- `weak_flagged_rate >= 0.80` (≥ 4 out of 5 known-weak)
- Median per-seed token cost ≤ 25K

If any of these fails, iterate on the role prompts, gate knobs, or briefing template; re-run calibration. Phase 1 transitions to `Done` only on a clean pass.

- [ ] **Step 7: Status transition + Idea status update**

```bash
# Flip the Feature status from Implementing to Done
sed -i.bak 's/^\*\*Status:\*\* Implementing$/**Status:** Done/' spec/features/sidekick-consilium/README.md
rm spec/features/sidekick-consilium/README.md.bak

# Lint --fix will autosync the Idea status from Implementing → Done as well
specscore spec lint --fix
specscore spec lint --severity warning
# Expected: 0 violations.

git add spec/features/sidekick-consilium/README.md spec/ideas/sidekick-consilium.md spec/ideas/README.md spec/features/README.md
git commit -m "$(cat <<'EOF'
spec(features): transition sidekick-consilium to Done

Phase 1 calibration passed:
- Agreement rate: <N>/20 (target ≥ 19)
- Weak-flagged rate: <N>/5 (target ≥ 4)
- Median token cost: <N>K (target ≤ 25K)

Cross-repo dependencies verified shipped:
- specscore consilium verdict subcommand (specscore-cli)
- consilium-review task type (synchestra)

Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
EOF
)"

git push
```

---

## Self-Review Checklist

After implementing all 11 tasks, run through this list:

**1. Spec coverage** — every REQ should trace to one or more tasks:

| REQ | Task |
|---|---|
| `invocation-triggers` | T8 step 1 (frontmatter) |
| `drain-all-queued` | T8 step 3 (pipeline overview) |
| `pipeline-five-stages` | T8 steps 5–9 |
| `per-seed-budget-25k` | T8 step 4 (transcript captures tokens_total) |
| `seed-mutation-detection` | T8 step 2 (pre-flight) |
| `researcher-fact-only-briefing` | T4 (researcher.md), T8 step 6 (validation) |
| `briefing-floor-not-ceiling` | T8 step 7 (role agents receive briefing + retain tool access) |
| `scribe-summary-paragraph` | T4 (scribe.md), T8 step 9 (validation) |
| `scribe-cannot-change-verdict` | T8 step 9 (ignore scribe verdict field) |
| `default-roster-9-roles` | T5–T7 (9 role files) |
| `parallel-fan-out` | T8 step 7 (single message, N parallel Agents) |
| `vote-schema` | T1 (role-file-contract.md), T8 step 7 (validation) |
| `abstain-with-confidence` | T6 (PM/Marketing example abstains), T8 step 8 (arbiter applies semantics) |
| `custom-role-markdown-contract` | T1 (role-file-contract.md) |
| `roster-exclude-and-custom` | T3 (consilium-config-example.md), T8 step 7 (load active roster) |
| `gate-knob-set` | T3 (consilium-config-example.md), T8 step 8 (arbiter receives gate) |
| `roster-validation` | T8 step 8 (arbiter validates) — cross-repo enforcement |
| `roster-snapshot-into-task` | T8 step 7 (snapshot loaded), T8 step 10 (written to task) |
| `verdict-source-of-truth-in-task` | T8 step 10 (task update) |
| `pipeline-transcript-capture` | T8 step 4 (every stage), T8 step 10 (task field) |
| `verdict-summary-in-seed` | T8 step 10 (seed mirror) |
| `consilium-review-task-lifecycle` | T8 step 10 (status transitions); cross-repo enforcement |
| `event-reviewed-emitted` | T2 (event spec), T8 step 10 (emission) |
| `specscore-consilium-verdict-subcommand` | T1 (companion stub) — cross-repo |
| `arbiter-gate-rules` | T1 (companion stub) — cross-repo |
| `arbiter-reproducibility` | T1 (companion stub) — cross-repo |
| `idempotent-task-creation` | T1 (companion stub) — cross-repo |
| `single-writer-claim-semantics` | T1 (companion stub) — cross-repo |
| `calibration-set-20-verdicts` | T10 (set construction), T11 (run + verify) |

**2. Placeholder scan** — search for forbidden patterns:

```bash
grep -nE 'TBD|TODO|fill in|placeholder|implement later|appropriate error handling' \
  spec/plans/sidekick-consilium.md
```

Expected: matches only for self-referential mentions in the self-review section itself.

**3. Type consistency** — verify naming across tasks:

- Role-file frontmatter keys match between `role-file-contract.md` (T1), researcher/scribe (T4), Builders (T5), Customers (T6), Adversaries (T7), and the validation logic in T8.
- Vote-schema fields match between T1's contract, T5–T7 example votes, and T8 step 7's validation.
- Transcript stage names match between T8 step 4 (definition) and T8 steps 5–9 (per-stage records).
- Event payload fields match between T2 (definition) and T8 step 10 (emission).

**4. File-path consistency** — every path is correct:

- `skills/consilium/SKILL.md` (singular, lowercase, matches `skills/sidekick/SKILL.md` precedent)
- `skills/consilium/roles/<role>.md` × 11
- `skills/consilium/references/{role-file-contract.md, consilium-config-example.md}`
- `spec/features/sidekick-consilium/_tests/<slug>.md` × 27 (25 stubs + skip + README)
- `spec/features/sidekick-consilium/calibration/{strong,weak,out-of-domain,ambiguous}/<slug>.md` × 20
- `spec/plans/sidekick-consilium-{arbiter,task}-companion.md`

If any task references a path that doesn't match the plan-wide convention, fix it.

---

## Execution Handoff

Two execution options:

**1. Subagent-Driven (recommended)** — A fresh subagent per task with two-stage review (spec compliance + code quality). Best for: maximum review density, catching cross-task drift, keeping the main session's context light. Sub-skill: `superpowers:subagent-driven-development`.

**2. Inline Execution** — Tasks run in this session with checkpoints between each. Best for: low-overhead linear walks, preserving conversation context. Sub-skill: `superpowers:executing-plans`.

Suggested execution order (matches task numbering): **T1 → T2 → T3 → T4 → T5 → T6 → T7 → T8 → T9 → T10 → T11.**

Notes on ordering flexibility:
- T1 must come first (every later task references concepts it establishes).
- T2, T3 can swap with each other (both are reference-doc work).
- T4 (researcher + scribe) should come before T8 (the SKILL.md references the role files by path).
- T5, T6, T7 can run in any order or in parallel (independent role-file authoring).
- T8 must come after all role files exist (T4–T7).
- T9 (stubs) can run in parallel with T10 (calibration set) — both are test/fixture authoring with no in-flight dependency.
- T11 cannot run until all cross-repo dependencies have shipped (verified in its own pre-flight steps).
