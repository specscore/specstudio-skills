# Feature: Sidekick Consilium

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/sidekick-consilium?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/sidekick-consilium?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/sidekick-consilium?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/sidekick-consilium?op=request-change) |

**Status:** Approved
**Date:** 2026-05-18
**Owner:** alexandertrakhimenok
**Source Ideas:** sidekick-consilium
**Supersedes:** —

## Summary

Phase 1 of the [`sidekick-ideas`](../../ideas/sidekick-ideas.md) Idea: a `specstudio:consilium` skill that drains captured *sidekick* (sideline-idea) seeds — the durable one-line captures produced by Phase 0's `specstudio:sidekick` — and produces deterministic, auditable verdicts via a 5-stage pipeline (CLI gather → researcher → 9-role parallel expert panel → CLI arbiter → scribe). The Synchestra `consilium-review` task is the structured source of truth for the verdict; the seed gets only the scribe's prose summary mirrored into a `## Consilium Verdict` section so a human browsing seeds sees the outcome alongside the captured one-liner. The verdict gate and panel roster are per-project configurable via `specscore.yaml`. No auto-promotion at this layer — that is Phase 2.

## Problem

Phase 0 (`sidekick-capture`, Implementing) ships an idea-capture pipeline that durably records sideline ideas during host-skill work. Seeds queue up in `spec/ideas/seeds/` but Phase 0 stops at write-and-continue — there is no triage. Without Phase 1, the seeds pile up as an unreviewed inbox that defeats the write-and-continue discipline (you derail at *review* time instead of capture time). Phase 1 turns the inbox into a triaged queue whose verdicts a human, the `synchestra:whats-next` prioritizer, and the future Phase 2 auto-promote can all consume.

The non-negotiable requirement: the verdict gate must remain *deterministic*. LLMs synthesize at the ends of the pipeline (researcher fact-gathering, scribe prose) but the gate that decides `should-implement` vs `needs-human-review` is a deterministic rule engine over typed votes. Letting LLM judgment leak into the gate makes verdicts non-reproducible, non-auditable, and unable to support unit tests — none of which are acceptable for a system whose output will gate auto-promotion in Phase 2.

### Departures from the source Idea

The sub-Idea ([`sidekick-consilium`](../../ideas/sidekick-consilium.md)) listed three new Features in its SpecScore Integration section: the skill, the `consilium-review` task type (cross-repo: synchestra), and the `specscore consilium verdict` subcommand (cross-repo: specscore-cli). This Feature consolidates all three into one in-repo Feature: the cross-repo work is *contracted* here as REQs (so the contract has a single canonical home), with the *implementation* tracked via companion plan stubs at plan time. This matches the Phase 0 precedent (`sidekick-capture` Feature contracted the `seed-lint-rule`; specscore-cli#6 tracks the implementation).

## Behavior

The Feature ships eight topic groups: skill orchestration, researcher and scribe agents, the expert panel, per-project configuration, verdict storage, the deterministic arbiter, concurrency and idempotency, and the calibration quality gate.

### The `specstudio:consilium` skill (orchestration)

#### REQ: invocation-triggers

The skill MUST respond to the triggers `specstudio:consilium`, `/consilium`, "run the consilium", "drain the sidekick queue", and "review sidekick ideas". It MAY respond to additional natural-language phrasings of the same intent.

#### REQ: drain-all-queued

The skill MUST support exactly one operational mode: *drain-all-queued*. On invocation it claims every `consilium-review` Synchestra task in status `queued`, runs the full 5-stage pipeline once per claimed task, and exits. It MUST NOT accept a per-slug argument (`/consilium <slug>`), a `--limit` argument, or any other invocation-shape variant in Phase 1. Per-invocation overrides are an Outstanding Question for a later phase.

#### REQ: pipeline-five-stages

Per claimed task, the skill MUST execute the pipeline in this exact order, where each stage's output is the next stage's input:

1. **CLI gather** (deterministic) — assemble a raw context bundle by running graph queries: `synchestra:feature` for related features, `synchestra:code` for code-to-spec refs, `git log` over relevant paths, and a dedupe-window lookup against prior seeds.
2. **Researcher agent** (one LLM call) — read the raw bundle and produce a *fact-only* briefing pack per REQ `researcher-fact-only-briefing`.
3. **Expert panel** (parallel LLM calls) — fan out one agent per active roster role per REQ `parallel-fan-out`; each returns a structured YAML vote per REQ `vote-schema`.
4. **CLI arbiter** (deterministic) — invoke `specscore consilium verdict` with the vote bundle and the active roster snapshot; the subcommand applies the gate rules per REQ `arbiter-gate-rules` and returns the verdict.
5. **Scribe agent** (one LLM call, advisory only) — synthesize the *Panel summary* paragraph per REQ `scribe-summary-paragraph`. The scribe cannot change the verdict per REQ `scribe-cannot-change-verdict`.

The skill MUST NOT skip stages, reorder them, or short-circuit on intermediate results. A failure at any stage transitions the task to `failed` per REQ `consilium-review-task-lifecycle` and the next claimed task continues.

#### REQ: per-seed-budget-25k

Each per-seed pipeline run SHOULD complete within ~25K total tokens consumed (envelope target: 5K researcher + 9 × ~2K experts + 1K scribe). The skill MUST instrument actual token usage and record it on the `consilium-review` task as `tokens_total`. Phase 1 does NOT enforce a hard cap — the budget is a target for calibration, not a runtime kill-switch. If the median of the first 20 calibration runs exceeds 25K, the researcher prompt or panel size MUST be tightened before Phase 2 ships.

#### REQ: seed-mutation-detection

Before claiming a task, the skill MUST recompute the seed's `content_hash` (SHA-256 of the trimmed lowercase one-liner, per Phase 0's REQ `event-payload-schema`) and compare against the `content_hash` stored on the task at capture time. If they differ, the seed has been mutated since capture; the skill MUST NOT review it, MUST transition the task to `failed` with reason `seed-mutated`, and MUST surface the mutation to the operator as a single short line. The seed can be re-enqueued by a separate operation (out of scope for Phase 1).

### Researcher and scribe agents

#### REQ: researcher-fact-only-briefing

The researcher's output (the briefing pack) MUST contain only structured facts: file references with line ranges, related artifacts with paths, prior captures with slugs, and recent git activity. The briefing MUST NOT contain scoring, judgment, opinion, or recommendations. The researcher prompt enforces this with explicit instructions; a research output that contains judgment-laden language (e.g., "this looks important", "this is concerning") is a contract violation by the prompt, not by the skill, and is fixed by tightening the prompt at iteration time.

#### REQ: briefing-floor-not-ceiling

The briefing pack is the shared context floor. Every expert in the panel receives it plus the seed. Every expert MAY also do role-specific deeper research using its tool access (Read, Grep, Glob, etc.) when its role demands it. The briefing is a floor, not a ceiling. This is the safety mechanism that prevents panel-wide pre-convergence on a researcher blind spot, especially for adversaries whose job is to find weaknesses the briefing missed.

#### REQ: scribe-summary-paragraph

The scribe MUST produce exactly one *Panel summary* paragraph in one of three flavors based on the deterministic verdict:

- **converged** — when verdict is `should-implement`: synthesize the shared reasoning across roles ("All three builders agreed the change is local to module X. PM and Marketing anchored on the same user pain. Adversaries stood down because of Y.")
- **rejected** — when verdict is `should-not-implement`: synthesize the shared reasoning against ("YAGNI Cop's argument that 3 higher-value items sit in queue was uncontested. Engineer flagged a coupling concern Architect confirmed.")
- **split** — when verdict is `needs-human-review`: quote the dissenting arguments side-by-side ("Builders all approved. Marketing dissented with high confidence: 'unclear who asks for this.' YAGNI Cop vetoed.")

Length cap: 500 characters (paragraph form, no headers, no lists, no code blocks).

#### REQ: scribe-cannot-change-verdict

The scribe runs *after* the CLI arbiter has set the verdict. The scribe agent receives the verdict as input. The CLI MUST ignore any verdict field the scribe emits; only the prose paragraph is consumed. This is the by-contract enforcement of the "LLMs are bad at gating" principle.

### Expert panel (default and custom roles)

#### REQ: default-roster-9-roles

The default roster MUST contain exactly these 9 roles, organized in three groups:

- **Builders** (3): `engineer`, `architect`, `qa`
- **Customers** (3): `pm`, `ux`, `marketing`
- **Adversaries** (3): `yagni-cop`, `skeptic`, `security-ops`

Each role's prompt lives at `skills/consilium/roles/<role>.md`, ships with the skill, and follows the role-file contract per REQ `custom-role-markdown-contract`. The default roster is what the calibration set (REQ `calibration-set-20-verdicts`) validates against.

#### REQ: parallel-fan-out

The active roster's role agents MUST be dispatched concurrently (a single message containing N parallel `Agent` tool uses, where N is the active roster size). The skill MUST wait for all agents to return before invoking the arbiter. Sequential or batched-by-group dispatch is explicitly NOT allowed in Phase 1 — parallelism is what produces the verdict in O(slowest agent) time rather than O(sum of agents).

#### REQ: vote-schema

Every role agent (default and custom) MUST return a structured YAML vote with exactly these five fields:

```yaml
verdict: should-implement | should-not-implement | no-opinion | abstain
confidence: low | medium | high
cost: 🟢 | 🟡 | 🔴
complexity: 🟢 | 🟡 | 🔴
argument: <one-sentence strongest argument, ≤ 280 characters>
```

A vote that fails YAML parse, fails enum validation, or exceeds the argument length cap is malformed; the CLI arbiter MUST reject the entire pipeline run for that seed (transition task to `failed` with reason `malformed-vote`). Tolerance for malformed votes is zero — the prompts are responsible for producing parseable output.

#### REQ: abstain-with-confidence

The fourth verdict option `abstain` MUST be available to every role. Its semantics are:

- **High-confidence abstain** ("clearly not my domain") — the expert is excluded from the consensus denominator for its group. A role that high-confidence-abstains does not count toward either approval or rejection.
- **Low-confidence abstain** ("I can't tell if this matters to me") — caps the overall verdict at `needs-human-review` regardless of other votes. This is the safety signal for panel confusion.
- **Medium-confidence abstain** — treated as low-confidence (cautious default) in Phase 1.

These semantics inherit directly from the parent Idea's Recommended Direction.

#### REQ: custom-role-markdown-contract

A custom role's markdown file MUST contain:

- Body metadata header lines:
  - `**Name:** <kebab-case-slug>` (must match the filename without `.md`)
  - `**Group:** builders | customers | adversaries`
  - `**Output Schema Version:** 1` (literal, for forward-compat)
- A `## Role Prompt` H2 section containing the prompt body the agent receives
- A `## Example Vote` H2 section with one fully-formed YAML vote per REQ `vote-schema`

The skill MUST validate these requirements at roster-load time and reject a custom-role file that fails parse. A reject is a clear error to the operator naming the missing field; the run aborts before any seed is reviewed.

### Per-project configuration (`specscore.yaml` → `consilium:`)

#### REQ: roster-exclude-and-custom

`specscore.yaml` MUST accept a `consilium.roster` block with two optional sub-keys:

```yaml
consilium:
  roster:
    exclude: [marketing, security-ops]           # list of default role slugs to exclude
    custom:                                       # list of custom roles to add
      - name: accessibility
        group: customers
        path: .specscore/roles/accessibility.md
```

Each `custom` entry specifies the role's slug, its group, and the path to its markdown definition. Path resolution is relative to the repo root. Both sub-keys are optional; the absence of either means "use defaults".

#### REQ: gate-knob-set

`specscore.yaml` MUST accept a `consilium.gate` block with these knobs:

```yaml
consilium:
  gate:
    adversary_veto_confidence: high       # high | medium | low (which confidence floor triggers veto)
    cost_ceiling: medium                  # low (🟢-only) | medium (🟢+🟡)
    complexity_ceiling: medium            # low | medium
    min_median_confidence: medium         # low | medium | high
    require_all_builders: true            # true (unanimous) | false (supermajority ⅔)
    require_all_customers: true           # true | false
```

The defaults shown above match the parent Idea's strict baseline. Each knob's allowed values are a discrete enum; continuous numeric thresholds are explicitly NOT supported in Phase 1.

#### REQ: roster-validation

The arbiter CLI (REQ `specscore-consilium-verdict-subcommand`) MUST reject an invalid roster before any review proceeds. Validation:

- After exclude and add: each of the three groups (builders / customers / adversaries) MUST contain ≥ 1 role.
- Total active roster size MUST be ≤ 12 roles (cap to bound token cost; revisit if real adopter rosters demand more).
- No custom role name MUST collide with any default role name (case-insensitive).
- Every custom-role path MUST resolve to an existing markdown file that parses per REQ `custom-role-markdown-contract`.

Validation failure produces a single clear error message naming the violation; the skill exits non-zero and reviews nothing.

#### REQ: roster-snapshot-into-task

When a `consilium-review` task is reviewed, the skill MUST write the *active roster snapshot* (the resolved list of role slugs and their groups, after exclude/add) onto the task before invoking the arbiter. The arbiter receives the snapshot as input and the snapshot is stored alongside the verdict in the task payload. If the `specscore.yaml` roster configuration changes later (between two reviews of related seeds), prior verdicts retain their snapshot and are NOT invalidated. Re-review only happens on explicit re-enqueue by a separate operation (out of scope for Phase 1).

### Verdict and storage (V3)

#### REQ: verdict-source-of-truth-in-task

The `consilium-review` Synchestra task MUST carry the full structured verdict payload as the source of truth, including:

- The active roster snapshot
- All N votes (one per active role) as the YAML structures of REQ `vote-schema`
- The briefing pack (so audit can reconstruct what the panel reasoned over)
- The arbiter's rule-trace (which gate rules fired, which votes were excluded for high-confidence abstain, etc.)
- The deterministic `verdict` enum (`should-implement | should-not-implement | needs-human-review`)
- The `content_hash` of the seed at review time
- The `tokens_total` actually consumed
- The scribe summary paragraph (for direct read by `synchestra:whats-next` and future Phase 2)
- The structured pipeline transcript per REQ `pipeline-transcript-capture` (so transcript-shape ACs are observable and post-hoc audit of stage ordering is possible)

Machine consumers query the task; humans see the seed's mirror.

#### REQ: pipeline-transcript-capture

The skill MUST emit a structured transcript of each pipeline run and write it to the `consilium-review` task as `pipeline_transcript`. The transcript MUST record, in execution order:

- **Per stage** (gather, researcher, panel, arbiter, scribe): a record with `stage` (literal stage name), `started_at` (ISO-8601), `ended_at` (ISO-8601), `outcome` (`ok | failed`), and `error` (string, only present when `outcome: failed`).
- **For CLI gather**: the bash commands run and a stdout/stderr summary (truncated to a documented cap, default 4KB).
- **For the researcher**: the agent's input (seed + raw context bundle reference) and the output (briefing pack identifier or full text, depending on size).
- **For the panel**: a list of per-role records. Each contains `role` (the role slug from the active roster snapshot), `input_includes_briefing` (boolean, always `true` per REQ `briefing-floor-not-ceiling`), `tool_calls` (list of tool-name strings the role agent invoked beyond the briefing — e.g., `["Read", "Grep"]`, empty list if none), and `vote` (the parsed YAML vote per REQ `vote-schema`).
- **For the CLI arbiter**: the inputs (paths to votes/roster/gate YAML files) and the output (the verdict + rule_trace + excluded_votes + denominators per REQ `specscore-consilium-verdict-subcommand`).
- **For the scribe**: the input (verdict + votes + briefing) and the output (the prose paragraph).

The transcript MUST be stored on the task as a structured field (not as free-form prose) so consumers can query specific stages or roles. Storage format is YAML or JSON at the task type's discretion (Synchestra concern). The skill is responsible for *producing* the transcript at runtime; threading the capture through every stage is non-retrofittable, hence this REQ exists at spec time rather than implementation time.

#### REQ: verdict-summary-in-seed

After the arbiter sets the verdict and the scribe produces its paragraph, the skill MUST append a `## Consilium Verdict` section to the seed file at `spec/ideas/seeds/<slug>.md`. The section contains:

- A first line with the verdict and date: `**Verdict:** should-implement | should-not-implement | needs-human-review (YYYY-MM-DD)`
- A second line linking to the synchestra task for the structured payload
- The scribe's prose paragraph

Placement of the new section follows the same rules as Phase 0's `## Sidekick Seeds Generated` back-link (REQ `back-link-section-format` in `sidekick-capture`): immediately before the SpecScore footer line if present, else end-of-file. If a `## Consilium Verdict` section already exists from a prior run (rare — would only happen if a seed was re-enqueued), it MUST be replaced in place, NOT duplicated.

#### REQ: consilium-review-task-lifecycle

The `consilium-review` Synchestra task type MUST support exactly these states and transitions:

```
queued → claimed → in_review → complete
                            → failed
                            → aborted
```

- `queued` — task exists, awaiting a `/consilium` run
- `claimed` — a `/consilium` invocation has claimed it (atomic transition; second claimer sees `claimed` and skips)
- `in_review` — pipeline is mid-execution
- `complete` — verdict written, scribe summary mirrored, event emitted
- `failed` — pipeline aborted due to a deterministic failure (seed mutated, malformed vote, arbiter validation error)
- `aborted` — operator-cancelled or upstream `sidekick-idea.captured` event invalidated

#### REQ: event-reviewed-emitted

On successful task transition to `complete`, the skill MUST emit a `sidekick-idea.reviewed` event per the convention in [`shared/synchestra-events.md`](../../../skills/shared/synchestra-events.md). Envelope and payload:

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
  path: <seed_path>                    # path to the seed file
  revision: <git SHA at emission>
payload:
  slug: <seed slug>
  content_hash: <seed content_hash>
  verdict: should-implement | should-not-implement | needs-human-review
  roster_snapshot: [<role slugs in active roster>]
  tokens_total: <int>
```

The event is emitted exactly once per successful task completion. On `failed` or `aborted` transitions, NO event is emitted.

### The deterministic arbiter (cross-repo contract)

These REQs contract the behavior of the `specscore consilium verdict` subcommand. The subcommand's *implementation* lives in [`specscore/specscore-cli`](https://github.com/specscore/specscore-cli) and is tracked via a companion plan stub authored at plan time.

#### REQ: specscore-consilium-verdict-subcommand

The CLI subcommand MUST accept exactly these inputs:

- `--votes <file>` — path to a YAML file containing the N votes from the panel
- `--roster <file>` — path to a YAML file containing the active roster snapshot
- `--gate <file>` (optional) — path to a YAML file containing gate-knob overrides; default = strict baseline
- `--seed <file>` — path to the seed file (for content_hash extraction)

And produce exactly these outputs to stdout (machine-readable YAML):

```yaml
verdict: should-implement | should-not-implement | needs-human-review
rule_trace: [<list of rule-name strings that fired during gate evaluation>]
excluded_votes: [<role slugs whose votes were excluded for high-confidence abstain>]
denominators: {builders: <int>, customers: <int>, adversaries: <int>}
```

Exit code: 0 on successful verdict (including `should-not-implement` and `needs-human-review` — these are not errors); non-zero on validation failure (malformed vote, invalid roster, file not found, etc.).

#### REQ: arbiter-gate-rules

The arbiter MUST apply the gate rules in this exact order, using configurable gate-knob values (REQ `gate-knob-set`):

```
1. Validate inputs (parse, schema, roster validation).
2. Exclude high-confidence abstain votes from the consensus denominator.
3. If any vote is a low-confidence abstain:
     → verdict = needs-human-review (rule: low-abstain-veto)
     → STOP
4. If any adversary returns should-not-implement
     AND confidence ≥ adversary_veto_confidence:
       → verdict = needs-human-review (rule: adversary-veto)
       → STOP
5. Count non-abstain approvals:
   approve_builders  = count(builders where verdict=should-implement)
   approve_customers = count(customers where verdict=should-implement)
6. If require_all_builders=true:
     gate_builders = (approve_builders == denominators.builders)
   else:
     gate_builders = (approve_builders >= ceil(2/3 * denominators.builders))
7. Same logic for require_all_customers.
8. median_conf = median confidence across all non-abstain votes
   gate_confidence = (median_conf >= min_median_confidence)
9. median_cost = median cost across all non-abstain votes
   gate_cost = (median_cost ≤ cost_ceiling)
10. Same for median_complexity ≤ complexity_ceiling.
11. If gate_builders AND gate_customers AND gate_confidence
       AND gate_cost AND gate_complexity:
       → verdict = should-implement
12. If builders+customers both vote majority should-not-implement:
       → verdict = should-not-implement
13. Otherwise:
       → verdict = needs-human-review
```

Each rule name in the `rule_trace` output reflects exactly which step fired.

#### REQ: arbiter-reproducibility

Same `--votes` + same `--roster` + same `--gate` MUST always produce the same `--verdict`, the same `rule_trace`, and the same `excluded_votes`. The subcommand MUST be snapshot-testable: a fixture set of vote+roster+gate inputs in the `specscore-cli` repo produces stable outputs that CI gates on.

### Concurrency and idempotency

#### REQ: idempotent-task-creation

A `consilium-review` task is created lazily by the consilium skill at drain time, keyed by the seed's `content_hash`. If a task with the same `content_hash` already exists (regardless of its current state), the skill MUST NOT create a duplicate; instead, the skill processes the existing task per its current state (skip if `complete`, re-claim if `failed`, etc.).

#### REQ: single-writer-claim-semantics

When the skill claims a task, the transition `queued → claimed` MUST be atomic at the Synchestra task layer. If two `/consilium` invocations race on the same task, the loser observes `claimed` and skips that task without re-claiming. This is the cross-repo contract for the `consilium-review` task type implementation in `specscore/synchestra`.

### Calibration and quality gate

#### REQ: calibration-set-20-verdicts

Phase 1 MUST NOT be declared ready to ship (and Phase 2 MUST NOT be specified) until a 20-verdict calibration set has been run against the default roster + default gate knobs with the following outcome:

- ≥ 95% of verdicts match the user's post-hoc judgment (the user reviewing the verdict reports "I would have made the same call") for `should-implement` and `should-not-implement` outcomes.
- ≥ 80% of known-weak seeds (intentionally seeded into the set) produce `should-not-implement` or `needs-human-review`.
- Median token cost per review ≤ 25K tokens (the budget target).

The calibration set is constructed by the implementer: 5 known-strong seeds, 5 known-weak seeds, 5 known-out-of-domain seeds (to exercise abstain), 5 known-ambiguous seeds. Failure to meet the calibration gate blocks Phase 1's transition from Implementing to Done.

## Acceptance Criteria

### AC: invocation-drains-all-queued-tasks (verifies REQ:drain-all-queued, REQ:invocation-triggers)

**Given** a project with N ≥ 2 `consilium-review` tasks in status `queued`
**When** the user invokes `/consilium`
**Then** all N tasks transition through `claimed → in_review → complete` (or `failed`); the skill returns one verdict per task; no `queued` tasks remain at the end of the run.

### AC: invocation-rejects-per-slug-argument (verifies REQ:drain-all-queued)

**Given** a project with multiple queued tasks
**When** the user invokes `/consilium <some-slug>`
**Then** the skill exits non-zero with the error message `"per-slug invocation not supported in Phase 1; /consilium drains all queued tasks"`; no task is claimed.

### AC: pipeline-runs-five-stages-in-order (verifies REQ:pipeline-five-stages)

**Given** one queued `consilium-review` task with a valid seed
**When** the skill processes the task
**Then** the task's transcript shows exactly: (1) CLI gather invoked, (2) one researcher Agent call, (3) N parallel role-agent calls (N = active roster size), (4) one `specscore consilium verdict` invocation, (5) one scribe Agent call — in that order; out-of-order or skipped stages are a regression.

### AC: token-usage-recorded-on-task (verifies REQ:per-seed-budget-25k)

**Given** a completed `consilium-review` task
**When** the task payload is inspected
**Then** the payload contains `tokens_total: <int>` reflecting the actual sum of tokens consumed across the researcher, panel, and scribe calls; the value is non-zero.

### AC: seed-mutation-blocks-review (verifies REQ:seed-mutation-detection)

**Given** a queued `consilium-review` task whose stored `content_hash` does not match the current seed file's normalized one-liner hash
**When** the skill attempts to claim the task
**Then** the task transitions `queued → failed` with reason `seed-mutated`; no review proceeds; no verdict is produced; the operator sees a single short error line referencing the seed path.

### AC: researcher-briefing-contains-no-judgment (verifies REQ:researcher-fact-only-briefing)

**Given** a fixture seed and a completed pipeline run
**When** the briefing pack stored on the task is inspected
**Then** the briefing contains only structured facts (file paths, line ranges, related-artifact slugs, git activity); a grep for judgment-laden tokens (e.g., "looks important", "concerning", "should consider", "recommend") finds none. (Test asserts the negative.)

### AC: every-expert-receives-briefing-and-may-research-deeper (verifies REQ:briefing-floor-not-ceiling)

**Given** a completed pipeline run
**When** the panel agents' invocation transcripts are inspected
**Then** every role agent's input includes the briefing pack; at least one role agent's transcript also shows tool calls (Read/Grep/Glob) beyond the briefing — confirming the briefing is a floor, not a ceiling.

### AC: scribe-summary-respects-flavor-and-length (verifies REQ:scribe-summary-paragraph)

**Given** three pipeline runs with verdicts `should-implement`, `should-not-implement`, `needs-human-review`
**When** the scribe paragraphs on each completed task are inspected
**Then** each is ≤ 500 characters, contains no markdown headers/lists/code blocks, and uses language matching its verdict's flavor (e.g., the `needs-human-review` summary cites the dissenting argument).

### AC: scribe-verdict-field-ignored (verifies REQ:scribe-cannot-change-verdict)

**Given** a fixture scribe agent that emits a `verdict:` field in its response in addition to the prose paragraph
**When** the pipeline completes
**Then** the task's stored verdict is the arbiter's value, not the scribe's; the scribe's `verdict:` field is silently ignored at parse time.

### AC: default-roster-is-9-roles-in-three-groups (verifies REQ:default-roster-9-roles)

**Given** a fresh project with no `consilium.roster` configuration in `specscore.yaml`
**When** the skill loads the active roster
**Then** the roster contains exactly 9 roles: `engineer, architect, qa` (builders); `pm, ux, marketing` (customers); `yagni-cop, skeptic, security-ops` (adversaries); each role's markdown file is at `skills/consilium/roles/<role>.md`.

### AC: panel-fans-out-in-parallel (verifies REQ:parallel-fan-out)

**Given** a 9-role active roster
**When** the skill processes one queued task
**Then** the panel stage dispatches 9 Agent tool uses in a single message (parallel invocation); the wall-clock time for the panel stage is approximately max(role-agent-time), not sum(role-agent-time).

### AC: malformed-vote-fails-pipeline (verifies REQ:vote-schema)

**Given** a fixture role agent that returns a vote with an invalid `verdict` value (e.g., `verdict: maybe`)
**When** the pipeline reaches the arbiter
**Then** the arbiter returns non-zero exit; the task transitions to `failed` with reason `malformed-vote`; no verdict is written.

### AC: abstain-high-confidence-excluded-from-denominator (verifies REQ:abstain-with-confidence)

**Given** a panel where one customer role returns `verdict: abstain, confidence: high` and the other two customers return `verdict: should-implement, confidence: medium`
**When** the arbiter computes the verdict with `require_all_customers: true`
**Then** the abstaining customer is excluded from `denominators.customers` (which is now 2); both remaining customers approve, so the customer gate passes; `excluded_votes` in the arbiter output contains the abstaining role's slug.

### AC: abstain-low-confidence-caps-verdict (verifies REQ:abstain-with-confidence)

**Given** a panel where one role returns `verdict: abstain, confidence: low` and all other votes are `should-implement` with high confidence
**When** the arbiter computes the verdict
**Then** the verdict is `needs-human-review` (rule `low-abstain-veto` fires); the rule_trace records this; the strict-gate path is not evaluated.

### AC: custom-role-loads-and-votes (verifies REQ:custom-role-markdown-contract)

**Given** a `specscore.yaml` with `consilium.roster.custom` listing one valid custom role at `.specscore/roles/accessibility.md` (markdown file with `**Name:** accessibility`, `**Group:** customers`, `**Output Schema Version:** 1`, a `## Role Prompt` section, and a `## Example Vote` section)
**When** the skill loads the roster and processes a queued task
**Then** the active roster has 10 roles (9 defaults + 1 custom); the custom role's agent is dispatched alongside the others; its vote is included in the arbiter's denominator for the customers group.

### AC: roster-with-malformed-custom-role-rejected (verifies REQ:custom-role-markdown-contract, REQ:roster-validation)

**Given** a `specscore.yaml` listing a custom role whose markdown file is missing the `**Group:**` metadata line
**When** the skill is invoked
**Then** the arbiter's roster-load validation fails with a clear error naming the missing field and the file path; no task is claimed; the skill exits non-zero.

### AC: roster-violating-group-floor-rejected (verifies REQ:roster-validation)

**Given** a `specscore.yaml` with `consilium.roster.exclude: [yagni-cop, skeptic, security-ops]` (excluding all 3 default adversaries) and no custom adversary added
**When** the skill is invoked
**Then** roster validation fails with error `"adversaries group has 0 members; ≥1 required"`; no task is claimed; the skill exits non-zero.

### AC: roster-snapshot-stored-on-task (verifies REQ:roster-snapshot-into-task)

**Given** a completed `consilium-review` task and a subsequent `specscore.yaml` change that excludes one of the originally-active roles
**When** the task payload is inspected after the config change
**Then** the task's `roster_snapshot` field still reflects the roster active at review time (the excluded role is still listed); the task is NOT invalidated by the config change.

### AC: pipeline-transcript-payload-shape (verifies REQ:pipeline-transcript-capture)

**Given** one completed `consilium-review` task
**When** the task's `pipeline_transcript` field is inspected
**Then** it contains exactly five stage records in the order `gather, researcher, panel, arbiter, scribe`; each stage record has `started_at`, `ended_at`, and `outcome: ok`; the `panel` stage's record contains one per-role entry per active roster role; each per-role entry has `role`, `input_includes_briefing: true`, `tool_calls` (a list, possibly empty), and a `vote` parsed as YAML matching REQ `vote-schema`.

### AC: verdict-task-payload-completeness (verifies REQ:verdict-source-of-truth-in-task)

**Given** one completed `consilium-review` task
**When** the task payload is inspected
**Then** it contains: `roster_snapshot` (list of role slugs + groups), `votes` (one YAML vote per role in the snapshot), `briefing_pack` (the researcher's output), `rule_trace` (the arbiter's trace), `verdict` (one of three enum values), `content_hash` (matches the seed at review time), `tokens_total` (non-zero int), and `scribe_summary` (the prose paragraph, ≤500 chars).

### AC: seed-gets-consilium-verdict-section (verifies REQ:verdict-summary-in-seed)

**Given** a seed at `spec/ideas/seeds/<slug>.md` and a successful pipeline completion against it
**When** the seed file is read after the pipeline
**Then** the seed contains a `## Consilium Verdict` section positioned immediately before the SpecScore footer line; the section's first line is `**Verdict:** <verdict-enum> (<YYYY-MM-DD>)`; the second line links to the synchestra task; the rest is the scribe's prose paragraph.

### AC: reviewed-event-emitted-on-success (verifies REQ:event-reviewed-emitted)

**Given** a successful pipeline completion against one queued task
**When** `.specscore/events.jsonl` is inspected after the run
**Then** exactly one new line has been appended: a JSON event with `event: sidekick-idea.reviewed`, the envelope fields from REQ `event-reviewed-emitted`, and a payload containing `verdict`, `roster_snapshot`, and `tokens_total`.

### AC: arbiter-reproducibility-snapshot (verifies REQ:arbiter-reproducibility, REQ:specscore-consilium-verdict-subcommand, REQ:arbiter-gate-rules)

**Given** a fixture vote bundle + roster + gate config that the gate rules deterministically produce `should-implement` for
**When** `specscore consilium verdict --votes votes.yaml --roster roster.yaml --gate gate.yaml --seed seed.md` is invoked twice
**Then** both invocations produce identical stdout YAML (verdict, rule_trace, excluded_votes, denominators all byte-identical); exit code 0 both times.

### AC: idempotent-task-creation-on-duplicate-hash (verifies REQ:idempotent-task-creation)

**Given** a queued `consilium-review` task for seed slug `X` and a second `sidekick-idea.captured` event arriving with the same `content_hash`
**When** the second event is processed
**Then** no second task is created; the existing task remains in its current state; the event is acknowledged without duplication.

### AC: concurrent-claim-loses-cleanly (verifies REQ:single-writer-claim-semantics)

**Given** two concurrent `/consilium` invocations claiming the same task
**When** both reach the claim step
**Then** exactly one invocation transitions the task `queued → claimed` and proceeds; the other observes `claimed` and skips the task without retry or error.

### AC: calibration-set-passes-95-percent (verifies REQ:calibration-set-20-verdicts)

**Given** the 20-seed calibration set (5 strong, 5 weak, 5 out-of-domain, 5 ambiguous) and the default roster + default gate config
**When** `/consilium` is run against all 20 seeds and the user reviews each verdict post-hoc
**Then** ≥ 95% of `should-implement` and `should-not-implement` verdicts match the user's post-hoc judgment; ≥ 80% of the 5 known-weak seeds produce `should-not-implement` or `needs-human-review`; median per-seed token cost ≤ 25K. Phase 1 is not declared ready until this AC passes.

## Architecture and Components

The Feature ships seven components across three repos.

In **this repo** (`specstudio-skills`):
1. **`specstudio:consilium` skill** — orchestrator. Lives at `skills/consilium/SKILL.md` plus the role-file references. Stateless. Single responsibility: claim tasks → run pipeline → write verdicts → emit events.
2. **Default role files** — 9 markdown files at `skills/consilium/roles/<role>.md` (plus `researcher.md` and `scribe.md` for the pipeline ends). Each follows the custom-role contract (REQ `custom-role-markdown-contract`).
3. **`specscore.yaml` schema extension** — adds the `consilium:` top-level block with `roster` and `gate` sub-keys, contracted in REQs `roster-exclude-and-custom` and `gate-knob-set`. Phase 2 will additively add `consilium.auto_promote` to the same block.
4. **`sidekick-idea.reviewed` event addendum** — extends `skills/shared/synchestra-events.md` (per the Phase 0 precedent) with the new event's envelope and payload.

In **`specscore/specscore-cli`** (cross-repo):
5. **`specscore consilium verdict` subcommand** — the deterministic arbiter. Contracted in REQs `specscore-consilium-verdict-subcommand`, `arbiter-gate-rules`, `arbiter-reproducibility`, and `roster-validation`. Implementation tracked via companion plan stub at plan time.

In **`specscore/synchestra`** (cross-repo):
6. **`consilium-review` task type** — the queue resource. Contracted in REQs `consilium-review-task-lifecycle`, `idempotent-task-creation`, and `single-writer-claim-semantics`. Implementation tracked via companion plan stub at plan time.

Conceptually shared (no code artifact):
7. **The 5-stage pipeline contract** — REQ `pipeline-five-stages` is the spine; all other REQs in this Feature compose around it.

The skill and role files are tightly coupled (the skill reads role files); the arbiter and the skill couple only through the YAML I/O contract (REQ `specscore-consilium-verdict-subcommand`); the task type and the skill couple through the Synchestra task lifecycle.

## Interaction with Other Features

- **`sidekick-capture`** ([Feature, Implementing](../sidekick-capture/README.md)) — no change. This Feature consumes the `sidekick-idea.captured` event emitted by `sidekick-capture` and produces `sidekick-idea.reviewed` in turn.
- **`synchestra:whats-next`** (in `synchestra` repo) — extends to surface `consilium-review` tasks and to prioritize seeds with `needs-human-review` verdicts. This Feature's REQ `verdict-source-of-truth-in-task` is the data contract whats-next reads.
- **`skills/shared/synchestra-events.md`** (in this repo) — extends with the `sidekick-idea.reviewed` event per REQ `event-reviewed-emitted`.
- **Phase 2 auto-promotion (future Feature)** — consumes the `sidekick-idea.reviewed` event and the task payload. Reads `verdict == should-implement` to decide auto-promote actions. Phase 1's REQ `verdict-source-of-truth-in-task` is what Phase 2 will query.

## Not Doing / Out of Scope

- **Auto-promotion to Feature spec or plan.** Phase 2 Feature. This Feature only produces verdicts; no consumer action.
- **Hook ergonomics (`Stop` hook, scheduled drain).** Phase 3 Feature. Phase 1 invocation is manual `/consilium` only.
- **Per-invocation roster overrides** (`/consilium --without security`). Parent-Idea OQ; revisit after calibration if friction is real.
- **Per-role model selection.** All roles share one model in Phase 1; revisit only if calibration shows adversary correlation.
- **Live multi-agent debate UI.** Agents return YAML in parallel; no chat theatre.
- **Cross-project consilium memory.** Each project has its own queue, roster, and verdicts.
- **`consilium.auto_promote` config block.** Phase 2 will additively add this to `specscore.yaml`; Phase 1 reserves the namespace but does not define knobs.
- **Notification surface** (Slack/email/webhook on verdict). Phase 3 territory.
- **Lint-sync drift reconciliation** between seeds and verdict-summary mirrors. Cross-repo, future Feature.
- **Continuous numeric gate thresholds.** Phase 1 gate knobs are discrete enums; numeric thresholds are explicitly out of scope.

## Rehearse Integration

Most ACs are testable via filesystem, task-payload, and event observation; per the rehearse heuristic, those scaffold to stubs at `spec/features/sidekick-consilium/_tests/<ac-slug>.md` with `**Status:** pending`. Stubs scaffold for:

- `invocation-drains-all-queued-tasks` — fixture project with seeded tasks; observe state transitions
- `invocation-rejects-per-slug-argument` — exit code + error message
- `pipeline-runs-five-stages-in-order` — transcript inspection
- `token-usage-recorded-on-task` — task-payload assertion
- `seed-mutation-blocks-review` — fixture seed mutation; observe `failed` transition
- `researcher-briefing-contains-no-judgment` — grep on briefing-pack content
- `every-expert-receives-briefing-and-may-research-deeper` — agent transcript inspection
- `scribe-summary-respects-flavor-and-length` — three-fixture pipeline run; string-shape assertions
- `scribe-verdict-field-ignored` — fixture scribe agent; assert ignored field
- `default-roster-is-9-roles-in-three-groups` — load-roster + assert structure
- `panel-fans-out-in-parallel` — single-message Agent-tool-use count
- `malformed-vote-fails-pipeline` — fixture vote; observe `failed` transition
- `abstain-high-confidence-excluded-from-denominator` — fixture votes + arbiter call; assert denominators
- `abstain-low-confidence-caps-verdict` — fixture votes; assert `needs-human-review`
- `custom-role-loads-and-votes` — fixture custom role; observe load + dispatch
- `roster-with-malformed-custom-role-rejected` — fixture missing field; assert exit code + error
- `roster-violating-group-floor-rejected` — fixture exclude-all-adversaries config; assert error
- `roster-snapshot-stored-on-task` — config change after review; assert snapshot stability
- `pipeline-transcript-payload-shape` — task-field assertion over the structured transcript
- `verdict-task-payload-completeness` — payload-field assertion
- `seed-gets-consilium-verdict-section` — file-content assertion
- `reviewed-event-emitted-on-success` — JSONL line inspection
- `arbiter-reproducibility-snapshot` — twice-invoked CLI; byte-identical output
- `idempotent-task-creation-on-duplicate-hash` — second-event observation
- `concurrent-claim-loses-cleanly` — two-invocation race; assert one wins

That covers all 25 runtime ACs as testable stubs (26 total ACs: 25 scaffolded + 1 calibration-gate AC tracked separately as a manual quality gate).

Skipped (process gate, not a runtime AC):

- `calibration-set-passes-95-percent` — this is a quality gate on the Phase 1 ship decision, not a runtime test of skill behavior. Track as a manual gate operated by the project owner before Phase 2 is specified. A future automated calibration harness could pick this up.

Rehearse stubs are scaffolded with `**Status:** pending`; authoring the actual scenario steps follows the implementation plan.

## Open Questions

- **`consilium.auto_promote` schema reservation.** Phase 2 will add this block to `specscore.yaml`. Should Phase 1 explicitly *reserve* the key (lint warns on unknown sub-keys today) or wait for Phase 2 to add the schema entry? Tentative: wait; the additive schema-update pattern doesn't need preemption.
- **Custom-role security in multi-tenant scenarios.** Phase 1 reads custom-role files as prompt text (never executes anything), but a malicious or adversarial custom prompt could (a) leak data, (b) systematically vote `should-implement` to game Phase 2 auto-promote, (c) impersonate a default role. Phase 1 mitigations: roster validation + name-collision detection. Phase 2+: signing/attestation. Resolve only if multi-tenant scenarios materialize.
- **Briefing-pack size cap.** REQ `researcher-fact-only-briefing` constrains content but not length. Reasonable starting cap: ≤ 1500 tokens of structured facts. Validate after the first 20 calibration runs; tighten the researcher prompt if the median exceeds.
- **Roster-snapshot diff visibility.** When the active roster changes between two reviews, a human reading both verdicts may want to see the roster diff. Should the seed's `## Consilium Verdict` section also mention the active roster? Tentative: no — keeps the seed-side mirror small; the task carries the full snapshot.
- **`tokens_total` granularity.** The task payload records the total; should it also break down per-stage (researcher / panel / scribe)? Useful for calibration triage. Tentative: yes, but additively, in Phase 1.5 or Phase 2 once we know what's worth measuring.
- **Re-enqueue operation.** REQ `seed-mutation-detection` says re-enqueue is "out of scope for Phase 1." How does a user actually re-enqueue a seed they want re-reviewed? Tentative: edit the task directly via `synchestra:task`; resolve at plan time.
- **`needs-human-review` fan-out.** When a verdict is `needs-human-review`, where does the human see it? Phase 1: it sits in the seed and the task; `synchestra:whats-next` is expected to surface it. Phase 3 could add notifications. Confirm during plan.
- **Concurrent `/consilium` semantics.** REQ `single-writer-claim-semantics` handles per-task races. Open: is the skill itself safe under two concurrent `/consilium` invocations against the same project (e.g., two terminals)? Tentative answer: yes, by atomic task claim. Validate during implementation.

---

## Sidekick Seeds Generated

- [rebrand-view-in-specstudio-blockquote-to-view-in-specscore](../../ideas/seeds/rebrand-view-in-specstudio-blockquote-to-view-in-specscore.md) — captured 2026-05-18 by specstudio:specify

- [extend-consilium-to-review-regular-specscore-ideas-not-just](../../ideas/seeds/extend-consilium-to-review-regular-specscore-ideas-not-just.md) — captured 2026-05-18 by superpowers:writing-plans
---
*This document follows the https://specscore.md/feature-specification*
