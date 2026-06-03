---
name: consilium
description: |
  Drains captured sidekick (sideline-idea) seeds and produces deterministic
  verdicts via a 5-stage pipeline: CLI gather → researcher agent → 9-role
  parallel expert panel → CLI arbiter → scribe agent. The orchestrator task
  is the structured source of truth; the seed gets only the scribe's prose
  summary mirrored into a ## Consilium Verdict section. Per-project roster
  and gate configurable via specscore.yaml → consilium: block. No auto-
  promotion at this layer (Phase 2).
  Triggers: "specstudio:consilium", "/consilium", "run the consilium",
  "drain the sidekick queue", "review sidekick ideas".
aliases: [consilium]
---

# Consilium

The consilium drains queued `consilium-review` orchestrator tasks one at a time, running the full 5-stage pipeline per task. On every successful task, the verdict + scribe summary mirror onto the seed and the `sidekick-idea.reviewed` event fires. On any stage failure, the task transitions to `failed` and the next queued task continues.

For *what* a sidekick seed is and how it gets captured, read [Phase 0's `sidekick-capture` Feature](../../spec/features/sidekick-capture/README.md). For the verdict gate's full algorithm, read [REQ `arbiter-gate-rules`](../../spec/features/sidekick-consilium/README.md#req-arbiter-gate-rules) in this skill's source Feature.

## When to Use

- The user typed `/consilium` or "run the consilium" or any other trigger phrase.
- The user wants to drain queued sidekick captures into reviewed verdicts.

## Pre-flight

Before claiming any task — and **before the Stage 3 nine-role panel** — verify the cross-repo dependencies are present:

1. **specscore present + capable (capability-gated detection).** Per [`../shared/cli-detection.md`](../shared/cli-detection.md), `consilium` is a **capability-gated** skill: it needs the `specscore consilium verdict` subcommand, not merely the binary. Do **not** run a standalone `command -v` probe; detect by invoking `specscore consilium verdict` (e.g. with `--help`) and branch on the exit status:
   - **exit `127`** (binary not installed) → emit the install message (point the user at `/specscore:install`) and stop.
   - **exit `8`** (`UnsupportedCommand` — unsupported subcommand / too old) → the binary is present but predates `consilium verdict`; emit an **upgrade** message: "Phase 1 requires `specscore` with the `consilium verdict` subcommand (cross-repo dependency, tracked in `spec/plans/sidekick-consilium-arbiter-companion.md`). Upgrade and re-run."
   - **any other non-zero** → surface the CLI's error.
   This capability check MUST run here, before the Stage 3 panel — never after.
2. The orchestrator CLI is on PATH — the task lifecycle lives there (`orchestrator task claim`, `orchestrator task update`).
3. `orchestrator task types` — must include `consilium-review`. If absent, exit cleanly with the analogous message referencing the task-type companion plan.

If either cross-repo dependency is missing, do NOT claim any task and do NOT modify any file. Exit with the actionable error.

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

**The transcript is written to the orchestrator task as `pipeline_transcript` before the task transitions to `complete`.** This is what makes the transcript-shape ACs (`pipeline-runs-five-stages-in-order`, `every-expert-receives-briefing-and-may-research-deeper`, `panel-fans-out-in-parallel`, `pipeline-transcript-payload-shape`) observable. If the transcript is missing or malformed, the task MUST transition to `failed` with reason `malformed-transcript` — even if the verdict itself is otherwise valid.

## Stage 1: CLI gather (deterministic)

**Transcript:** start a record with `stage: gather, started_at: <now>`.

Run these commands in order (capture stdout + stderr; truncate the summary to 4KB):

```bash
# 1. Related features
specscore feature list --related "$SEED_SLUG" 2>&1 | head -50

# 2. Code-to-spec refs for captured_during
specscore code refs "$CAPTURED_DURING" 2>&1 | head -50

# 3. Recent git activity in relevant paths
git log --oneline --since="30 days ago" -- "$CAPTURED_DURING" 2>&1 | head -20

# 4. Prior captures within the dedupe window (30 days)
ls -t spec/ideas/seeds/ 2>/dev/null | head -10
```

Assemble the outputs into a single context bundle file at `.specscore/consilium/<task-id>/raw-context.md`. This file is the researcher's primary input.

**Transcript:** finalize with `ended_at: <now>, outcome: ok, commands: […], output_summary: <truncated combined output>`. On any command failure, set `outcome: failed, error: <stderr first line>` and transition the task to `failed` with reason `gather-failed`.

## Stage 2: Researcher (one LLM call)

**Transcript:** start a record with `stage: researcher, started_at: <now>`.

Dispatch one Agent tool call with:

- `subagent_type: general-purpose`
- The seed file's full content + the raw context bundle from Stage 1
- The researcher prompt body from `skills/consilium/roles/researcher.md` (read it and include verbatim as the agent's instruction prompt)

Capture the agent's response. Write it to `.specscore/consilium/<task-id>/briefing.md`.

**Validate the briefing**:
- Must contain the four `##` sections per the researcher's contract (Related artifacts, Code references, Recent git activity, Prior captures within dedupe window).
- Total length ≤ 1500 tokens (use `wc -c` and divide by ~4 for a quick approximation).

If validation fails, retry the Agent call ONCE with the same prompt. If it fails again, transition the task to `failed` with reason `researcher-malformed`.

**Transcript:** finalize with `ended_at: <now>, outcome: ok, input: {seed_path, raw_context_ref}, output: {briefing_pack: <file ref or inline ≤2KB>}`.

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

## Stage 4: CLI arbiter (deterministic)

**Transcript:** start a record with `stage: arbiter, started_at: <now>`.

Write the votes, roster snapshot, and active gate config to temporary YAML files:

```bash
mkdir -p .specscore/consilium/<task-id>
echo "$VOTES_YAML" > .specscore/consilium/<task-id>/votes.yaml
echo "$ROSTER_YAML" > .specscore/consilium/<task-id>/roster.yaml
specscore consilium config --print-gate > .specscore/consilium/<task-id>/gate.yaml
```

Invoke the arbiter:

```bash
specscore consilium verdict \
  --votes .specscore/consilium/<task-id>/votes.yaml \
  --roster .specscore/consilium/<task-id>/roster.yaml \
  --gate .specscore/consilium/<task-id>/gate.yaml \
  --seed "$SEED_PATH" \
  > .specscore/consilium/<task-id>/verdict.yaml
```

The arbiter's stdout YAML contains: `verdict`, `rule_trace`, `excluded_votes`, `denominators`.

On non-zero exit (validation failure: malformed vote, invalid roster, file not found), transition the task to `failed` with reason from the arbiter's stderr.

**Transcript:** finalize with `ended_at: <now>, outcome: ok, input: {<paths>}, output: <parsed verdict YAML>`.

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

## Write-back (post-pipeline, before task transitions to complete)

### Update the task payload

Per REQ `verdict-source-of-truth-in-task`, the task carries the full structured payload:

```bash
orchestrator task update <task-id> \
  --field roster_snapshot=@.specscore/consilium/<task-id>/roster.yaml \
  --field votes=@.specscore/consilium/<task-id>/votes.yaml \
  --field briefing_pack=@.specscore/consilium/<task-id>/briefing.md \
  --field arbiter_output=@.specscore/consilium/<task-id>/verdict.yaml \
  --field pipeline_transcript=@.specscore/consilium/<task-id>/transcript.yaml \
  --field tokens_total=<computed_total> \
  --field scribe_summary=<scribe paragraph>
```

### Mirror the scribe summary to the seed

Per REQ `verdict-summary-in-seed`, append a `## Consilium Verdict` section to the seed file. Placement: immediately before the SpecScore footer line if present, else end-of-file. If a `## Consilium Verdict` section already exists (re-enqueue case), replace in place (do NOT duplicate).

Section format:

```markdown
## Consilium Verdict

**Verdict:** <verdict-enum> (<YYYY-MM-DD>)

Full payload: orchestrator task <task-id>.

<scribe prose paragraph>
```

### Transition the task to complete

```bash
orchestrator task update <task-id> --status complete
```

### Emit `sidekick-idea.reviewed`

Per REQ `event-reviewed-emitted`, emit the event with the envelope+payload from REQ:

```bash
# If specscore event emit is available, prefer it
if specscore event emit --help &>/dev/null 2>&1; then
  specscore event emit sidekick-idea.reviewed <event-yaml>
else
  # Fallback: append JSONL directly
  jq -c -n \
    --arg event "sidekick-idea.reviewed" \
    --arg uuid "$(uuidgen)" \
    --arg ts "$(date -u +%FT%TZ)" \
    --arg slug "$SEED_SLUG" \
    --arg verdict "$VERDICT" \
    '{event: $event, version: 1, uuid: $uuid, timestamp: $ts, …, payload: {slug: $slug, verdict: $verdict, …}}' \
    >> .specscore/events.jsonl
fi
```

The full envelope+payload structure is in `skills/shared/events.md` under the `sidekick-idea.reviewed` section.

### Output to the operator

Print one line per completed task:

```
Reviewed: <seed-slug> → <verdict> (orchestrator task <task-id>)
```

After all queued tasks are processed, print a summary:

```
Consilium drained: <N> reviews complete, <M> failed, <K> awaiting human review.
```

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
- [`shared/events.md`](../shared/events.md) — event envelope, including `sidekick-idea.reviewed`.
- [`roles/`](roles/) — the 11 default role files (researcher + scribe + 9 panel roles).
- [`references/role-file-contract.md`](references/role-file-contract.md) — the markdown contract every role file follows.
- [`references/consilium-config-example.md`](references/consilium-config-example.md) — the `specscore.yaml` `consilium:` block schema.
- [Feature: `sidekick-consilium`](../../spec/features/sidekick-consilium/README.md) — the spec this skill implements.
- [`spec/plans/sidekick-consilium-arbiter-companion.md`](../../spec/plans/sidekick-consilium-arbiter-companion.md) — cross-repo arbiter dependency.
- [`spec/plans/sidekick-consilium-task-companion.md`](../../spec/plans/sidekick-consilium-task-companion.md) — cross-repo task type dependency.
