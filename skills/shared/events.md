# Events Emitted by SDD Skills

**Status:** Contract

Events are how SDD skills hand work off to downstream consumers and vice-versa. Both `specstudio:ideate` and `specstudio:specify` emit events after successful artifact writes. Downstream automations may also trigger skills in response to events.

## Event Envelope (common fields)

Every event carries this envelope:

```yaml
event: <event-name>
version: 1
timestamp: <ISO-8601>
actor:
  kind: skill | user | external
  id: <e.g., "skill:specstudio:ideate", "user:<username>", "agent:<agent-id>">
artifact:
  type: idea | feature | plan
  id: <stable SpecScore ID>
  path: <path relative to repo root>
  revision: <git SHA at time of emission>
payload:
  # event-specific; see below
publication_result:
  resolved_actions: [stage, commit, push] | []
  executed_actions: [stage, commit, push] | []
  skipped_actions:
    - action: <stage | commit | push>
      reason: <string>
  policy_sources: [<source descriptor>, ...] | []
  commit_sha: <git SHA> | null
  push_target: <remote>/<branch> | null
```

`publication_result` is present on events emitted after a publication-policy checkpoint. It records what the policy resolved, what actually executed, what was skipped and why, and any commit or push target produced by the checkpoint. Additive fields MAY be included as the companion CLI result shape evolves.

## Emission Transport

- **Default:** Append-only JSONL to `.specscore/events.jsonl` at repo root (git-ignored).
- **Hook:** Skills invoke `specscore event emit <event.yaml>` (CLI) when available; fall back to direct file append otherwise.
- **Idempotency:** Each event includes a `uuid` (assigned by emitter). Consumers dedupe by uuid.

## Unaggregated by Design

The event stream is **deliberately unaggregated**. Skills emit one event per successful lint pass, even during heavy iteration sessions where the same artifact changes many times in close succession. There is no skill-side debouncing, throttling, or coalescing.

This is a single-source-of-truth choice: the skill is a faithful **event source**, not an event aggregator. Aggregation is a consumer concern. Different consumers want different debounce policies — a Hub live view may want ~1-minute coalescing for a fluid feel; a CI system that triggers builds on every change wants no debouncing; a Slack notification bot may want 5-minute coalescing to avoid spam. A skill-side debounce would serve none of them well.

**For consumers that want to debounce:** use the envelope `timestamp` to suppress events fired within your chosen window. The envelope's `artifact.id` is stable across emissions so coalescing per artifact is straightforward. The same approach works for any `*.updated` event in this vocabulary.

## Events Emitted by `specstudio:ideate`

### `idea.drafted`
Fired after every successful `specscore spec lint` pass following a write or edit, while the Idea's `**Status:**` is `Draft`. The first emission carries the same event name as subsequent ones — consumers dedupe by event uuid.

```yaml
payload:
  slug: <slug>
  hmw: <How Might We statement>
  target_user: <string>
  approved: false
  changed_sections: [<H2 section name>, ...] | null   # null on first emission (no baseline)
  previous_revision: <git SHA> | null                  # null on first emission
  change_summary: <string ≤2 sentences> | null         # null on first emission
```

**Change-context fields** (`changed_sections`, `previous_revision`, `change_summary`) carry the diff between this emission and the previous one. They are `null` on the first emission for an Idea (no prior revision to diff against). On every subsequent emission they are present and non-null.

`changed_sections` lists the H2 section names whose content differs from `previous_revision`. `change_summary` is a ≤2-sentence **factual** description of the change (no speculation about the user's intent or motivation; no editorializing).

### `idea.approved`
Fired exactly once, after the user explicitly approves the Recommended Direction and the Idea's `status` transitions to `Approved`.

```yaml
payload:
  slug: <slug>
  recommended_direction_summary: <first paragraph>
```

**Consumer:** A downstream orchestrator may react by scheduling `specstudio:specify` (after user confirmation) or by notifying watchers.

### `idea.updated`
Fired after every successful `specscore spec lint` pass following a write or edit, while the Idea's `**Status:**` is `Approved`. Distinguishes post-approval iteration from pre-approval drafting; consumers that watch only for material changes to approved Ideas subscribe here rather than to `idea.drafted`.

```yaml
payload:
  slug: <slug>
  hmw: <How Might We statement>
  target_user: <string>
  approved: true
  changed_sections: [<H2 section name>, ...]   # always non-null (an updated event by definition has a baseline)
  previous_revision: <git SHA>                  # always non-null
  change_summary: <string ≤2 sentences>         # always non-null; same discipline as idea.drafted
```

The change-context fields follow the same semantics and discipline as on `idea.drafted` (see above), but are never `null` here — by definition, an `idea.updated` emission has a prior revision to diff against (the revision at which `idea.approved` last fired, or the previous `idea.updated`).

**Consumer:** A downstream orchestrator may notify Features that declare this Idea as a `Source Ideas` entry, so downstream specs can be re-reconciled. Consumers can filter on `changed_sections` to react only when load-bearing sections (e.g., `Recommended Direction`) change.

## Events Emitted by `specstudio:specify`

### `feature.specified`
Fired after the Feature artifact is written, lint-clean, and the reviewer subagent returns `Approved` — that is, the Feature is structurally and qualitatively ready for user review.

```yaml
payload:
  slug: <slug>
  source_idea_id: <idea id | null>
  requirement_count: <int>
  ac_count: <int>
  rehearse_stubs_generated: <bool>
  rehearse_skip_reason: <string | null>
  changed_sections: [<H2 section name>, ...] | null   # null on first emission (no baseline)
  previous_revision: <git SHA> | null                  # null on first emission
  change_summary: <string ≤2 sentences> | null         # null on first emission
```

The change-context fields (`changed_sections`, `previous_revision`, `change_summary`) follow the same semantics and discipline as on `idea.drafted` / `idea.updated` (see above). They are `null` on the first `feature.specified` emission for a Feature (no prior revision to diff against). On every subsequent emission during reviewer iteration, they are present and non-null.

### `feature.approved`
Fired exactly once, after the user explicitly approves the written Feature and the status transition Under Review → Approved completes successfully.

```yaml
payload:
  slug: <slug>
  grade: <A|B|C|D|F>                            # the released gate grade (per reviewer-gates#req:grade-recording)
  changed_sections: [<H2 section name>, ...]   # always non-null (the approval action implies a baseline)
  previous_revision: <git SHA>                  # always non-null
  change_summary: <string ≤2 sentences>         # always non-null
```

The change-context fields are never `null` here — the prior revision is the last `feature.specified` emission. `grade` is the gate's released grade (since a gate releases only when `grade ≥ threshold`, it is always at or above the configured Approve threshold — `B` by default); it is the "Feature approved with grade `B`" record, mirrored onto the artifact as a `**Grade:**` body-metadata line per [`reviewer-gates#req:grade-recording`](../../spec/features/reviewer-gates/README.md#req-grade-recording).

### `feature.updated`
Fired after every successful `specscore spec lint` pass following a write or edit, while the Feature's `**Status:**` is `Implementing` or `Stable`. Distinguishes post-approval iteration from pre-approval drafting; consumers that watch only for material changes to approved Features subscribe here rather than to `feature.specified`.

```yaml
payload:
  slug: <slug>
  changed_sections: [<H2 section name>, ...]   # always non-null
  previous_revision: <git SHA>                  # always non-null
  change_summary: <string ≤2 sentences>         # always non-null
```

**Consumer:** A downstream orchestrator typically triggers `writing-plans` after `feature.approved`, and notifies downstream consumers (Plans, dependent Features, Hub) on `feature.updated`.

## Events Emitted by `specstudio:sidekick`

### `sidekick-idea.captured`
Fired exactly once per successful seed write at `spec/ideas/seeds/<slug>.md`. The skill writes the seed file before emitting; if emission fails after a successful write, the seed remains on disk and is recoverable by re-emission (see the skill's failure semantics). The event is not fired on validation or write failure.

```yaml
event: sidekick-idea.captured
version: 1
uuid: <generated>
timestamp: <ISO-8601 of capture moment; mirrors seed frontmatter `captured_at`>
actor:
  kind: skill | user
  id: <invoker — "<plugin>:<skill>" form for skills, "user:<username>" for direct user invocation>
artifact:
  type: idea-seed
  id: <slug>                         # matches the seed's frontmatter `slug` and filename
  path: <seed_path>                  # e.g., spec/ideas/seeds/persist-debug-logs.md
  revision: <git SHA at emission, or "uncommitted">
payload:
  slug: <slug>                       # duplicated for direct consumer access
  captured_during: <string or null>  # mirrors seed frontmatter; spec path or null
  trigger: heuristic | explicit       # mirrors seed frontmatter
  content_hash: <SHA-256 lowercase hex of normalized one-liner>
```

**Normalization for `content_hash`:** the one-liner is trimmed (leading/trailing whitespace removed) and lowercased via Unicode default casefolding before hashing. Different emitters compute the same hash for the same idea, which lets the Phase 1 consilium dedupe panel runs across sessions.

**Consumer:** the Phase 1 consilium subscribes here. It dedupes by `content_hash` over a rolling window, then enqueues a `consilium-review` task for the seed. Consumers that want a flat 8-field view (per Feature `sidekick-capture` REQ `event-payload-schema`) project: `event`, `seed_path` (from `artifact.path`), `slug`, `captured_at` (from `timestamp`), `captured_by` (from `actor.id`), `captured_during`, `trigger`, `content_hash`.

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

**Consumer:** A downstream orchestrator's task-prioritization reader surfaces `consilium-review` tasks with `needs-human-review` verdicts at the top of its prioritization report. Phase 2 auto-promote (future Feature) will subscribe to `verdict: should-implement` events.

## Events Emitted by `specstudio:ship`

### `ship.completed`
Fired **exactly once** on a successful ship — after the delegate returns explicit success and ship transitions the Feature `Implementing → Stable`. It is NOT emitted on any refusal (pre-flight or reviewer-gate) or on the no-delegate hand-back. Emitted after publication policy is applied for its checkpoint, carrying `publication_result`.

```yaml
event: ship.completed
version: 1
artifact:
  type: feature
  id: <feature-slug>
  path: spec/features/<feature-slug>/README.md
  revision: <sha>
payload:
  feature_slug: <feature-slug>
  revision: <sha>
  delegate: <delegate-skill-name>     # the deploy skill ship dispatched
  from_status: Implementing
  to_status: Stable
  publication_result:
    resolved_actions: [<stage | commit | push>, ...]
    executed_actions: [<stage | commit | push>, ...]
    skipped_actions: [{action: <...>, reason: <string>}]
    commit_sha: <git SHA> | null
    push_target: <remote>/<branch> | null
```

**Invariants:** emitted at most once per successful ship run; `from_status` is always `Implementing` and `to_status` always `Stable`; absent on every refusal and on the no-delegate hand-back.

## Events Emitted by `specstudio:pull-request`

### `pull_request.created`
Fired **exactly once** on a successful pull-request creation — after the built-in `git push` + `gh pr create` path completes, or a configured `pull_request.delegate` returns explicit success. It is NOT emitted on any refusal (pre-flight or reviewer-gate) or when creation fails. Emitted after publication policy is applied for its checkpoint, carrying `publication_result`.

```yaml
event: pull_request.created
version: 1
payload:
  pr_url: <url>                       # the created pull request's URL
  pr_number: <int>                    # the created pull request's number
  base_branch: <branch>              # the PR's base (repository default branch)
  head_branch: <branch>              # the branch the PR was opened from
  delegate: <delegate-skill-name> | null   # the delegate dispatched, or null for the built-in path
  publication_result:
    resolved_actions: [<stage | commit | push>, ...]
    executed_actions: [<stage | commit | push>, ...]
    skipped_actions: [{action: <...>, reason: <string>}]
    commit_sha: <git SHA> | null
    push_target: <remote>/<branch> | null
```

**Invariants:** emitted at most once per successful run; absent on every refusal and on any creation failure.

## Gate-Point Events (pre-action checkpoints)

Gate-point events fire at **execution checkpoints** rather than at artifact-lifecycle transitions. Unlike the once-per-artifact lifecycle events above, a gate-point event MAY fire **multiple times within a single run**. They exist so that a [reviewer gate](../../spec/features/reviewer-gates/README.md) can be keyed on a pre-action checkpoint (`gates.implementation.pre_commit`, `gates.implementation.pre_push`) and evaluated independently at each occurrence per [reviewer-gates#req:gate-point-events-and-multi-fire](../../spec/features/reviewer-gates/README.md#req-gate-point-events-and-multi-fire).

The `implementation.*` events are the reviewer-gates MVP gate-points. Follow-on consumer Features register their own gate-point events in this catalog when they wire a new pre-action checkpoint — `specstudio:ship` registers `ship.pre_dispatch` below. Inventing arbitrary gate-point identifiers that no consumer fires remains out of scope; each registered gate-point names the Feature that fires it.

### `implementation.pre_commit`
A pre-action gate-point evaluated **before each commit** a producer makes during an `implement` run. It MAY fire multiple times in one run — once per commit/batch — and each occurrence is an independent gate evaluation (no single-shot-per-run caching). Used as a reviewer-gate key (`gates.implementation.pre_commit`).

```yaml
event: implementation.pre_commit
version: 1
payload:
  occurrence: <int>          # 1-based index of this occurrence within the run
  run_id: <string>           # stable identifier for the enclosing implement run
```

### `implementation.pre_push`
A pre-action gate-point evaluated **before a publish/promote** (push) during an `implement` run. Used as a reviewer-gate key (`gates.implementation.pre_push`).

```yaml
event: implementation.pre_push
version: 1
payload:
  occurrence: <int>          # 1-based index of this occurrence within the run
  run_id: <string>           # stable identifier for the enclosing implement run
```

**Consumer:** a [reviewer gate](../../spec/features/reviewer-gates/README.md) keyed on the event evaluates the gate's reviewers at each occurrence. Wiring an actual `implement` run to *fire* these events is a downstream Feature (the implement-autonomy layer), not the reviewer-gates contract itself.

### `ship.pre_dispatch`
A pre-action gate-point evaluated **once per `ship` run** — after `specstudio:ship`'s pre-flight machine gates pass and **before** ship dispatches the deploy to its configured delegate. Unlike the multi-fire `implementation.*` gate-points, it fires at most once per run, because ship performs a single dispatch. Used as a reviewer-gate key (`gates.ship.pre_dispatch`).

```yaml
event: ship.pre_dispatch
version: 1
payload:
  feature_slug: <slug>       # the Feature being shipped
  run_id: <string>           # stable identifier for the enclosing ship run
```

**Consumer:** a [reviewer gate](../../spec/features/reviewer-gates/README.md) keyed on `ship.pre_dispatch` evaluates the gate's reviewers; the consumer that *fires* it is `specstudio:ship` (see the [Ship Skill Feature](../../spec/features/skills/ship/README.md)).

### `pull_request.pre_dispatch`
A pre-action gate-point evaluated **once per `pull-request` run** — after `specstudio:pull-request`'s pre-flight machine gates pass and **before** it creates the pull request (built-in path or delegate). Like `ship.pre_dispatch`, it fires at most once per run, because the skill creates a single PR. Used as a reviewer-gate key (`gates.pull_request.pre_dispatch`), this is where the project's `type: deterministic` verify reviewer (tests, coverage) runs before any PR is opened.

```yaml
event: pull_request.pre_dispatch
version: 1
payload:
  head_branch: <branch>      # the branch a PR would be opened from
  run_id: <string>           # stable identifier for the enclosing pull-request run
```

**Consumer:** a [reviewer gate](../../spec/features/reviewer-gates/README.md) keyed on `pull_request.pre_dispatch` evaluates the gate's reviewers; the consumer that *fires* it is `specstudio:pull-request` (see the [Pull Request Skill Feature](../../spec/features/skills/pull-request/README.md)).

## Events Emitted by Transition Menus

### `lifecycle.phase-skipped`
Fired when a user chooses a non-default downstream option at a transition menu — that is, when one or more SDD phases are skipped. The event is emitted by the skill that presents the menu (`specstudio:ideate` or `specstudio:specify`), **before** the downstream skill is invoked.

The event is **not** emitted when the user chooses the default option (e.g., "Specify" from ideate, "Plan" from specify). Default-path transitions follow the existing event flow (`idea.approved` → `feature.specified` → etc.) with no additional event.

```yaml
event: lifecycle.phase-skipped
version: 1
payload:
  from_phase: ideate | specify
  skipped_phases: [specify] | [plan] | [specify, plan]
  to_phase: plan | implement
  reason: user-requested | scope-suggested
  source_artifact:
    type: idea | feature
    path: <path relative to repo root>
    slug: <slug>
```

**Field semantics:**

| Field | Type | Description |
|---|---|---|
| `from_phase` | `ideate` \| `specify` | The phase whose transition menu presented the choice. |
| `skipped_phases` | `string[]` | Ordered list of phases that were skipped. E.g., choosing "Implement" from ideate yields `[specify, plan]`; choosing "Plan" from ideate yields `[specify]`; choosing "Implement" from specify yields `[plan]`. |
| `to_phase` | `plan` \| `implement` | The phase the user chose to jump to. |
| `reason` | `user-requested` \| `scope-suggested` | `user-requested` when the user chose the option unprompted; `scope-suggested` when the skill's heuristic suggested the lighter path and the user confirmed. |
| `source_artifact.type` | `idea` \| `feature` | The artifact type that was the input to the menu-presenting skill. |
| `source_artifact.path` | `string` | Path to the source artifact relative to repo root (e.g., `spec/ideas/fix-typo.md`). |
| `source_artifact.slug` | `string` | The SpecScore slug of the source artifact. |

**Emission rules:**

1. The event is emitted by the **menu-presenting skill** (ideate or specify), not by the receiving downstream skill.
2. The event is emitted **before** invoking the downstream skill — if the downstream invocation fails, the skip is still recorded.
3. The event is emitted **only on non-default choices**. Default-path transitions produce no `lifecycle.phase-skipped` event.

**Valid combinations:**

| `from_phase` | User choice | `skipped_phases` | `to_phase` |
|---|---|---|---|
| `ideate` | Plan | `[specify]` | `plan` |
| `ideate` | Implement | `[specify, plan]` | `implement` |
| `specify` | Implement | `[plan]` | `implement` |

## Events Emitted by External Lifecycle Tooling (consumed by skills)

### `idea.implementing`
Fired by automated lifecycle tooling (not by a skill) when the **first** Feature is created (or transitioned out of `Stable`) whose `**Source Ideas:**` field references an Idea's slug. The tooling:

1. Transitions the Idea's `**Status:** Approved → Implementing`.
2. Populates (or updates) the Idea's `**Promotes To:**` with the list of Feature slugs.
3. Commits the updated Idea artifact.
4. Emits `idea.implementing`.

```yaml
payload:
  idea_slug: <slug>
  feature_slugs: [<feat-1>, <feat-2>, …]
```

### `idea.specified`
Fired by automated lifecycle tooling (not by a skill) when **every** Feature listed in an Idea's `**Promotes To:**` reaches `Status: Stable`. The tooling:

1. Transitions the Idea's `**Status:** Implementing → Specified`.
2. Commits the updated Idea artifact.
3. Emits `idea.specified`.

```yaml
payload:
  idea_slug: <slug>
  feature_slugs: [<feat-1>, <feat-2>, …]
```

This is a stricter event than the previous `idea.specified` (which fired on first Feature linking). The new semantics ("all Features have stabilized") match the canonical SpecScore Idea spec's `Specified` definition.

**No skill emits these events directly.** Skill authors must not manually edit `promotes_to` or set `Status: Implementing` / `Status: Specified` by hand.

## Event Schema Versioning

- Envelope `version` starts at 1.
- Additive changes (new payload fields) do not bump version.
- Breaking changes (renamed/removed fields) bump version; consumers must handle both until the old version is removed.

## Publication Policy Milestone Catalog

Use artifact lifecycle events as publication-policy targets whenever a canonical event exists. Use milestones only for checkpoints with no event. Stable milestone names:

| Milestone | Command | When |
|---|---|---|
| `implement.batch-approved` | `implement` | After conflict detection, lint, inline self-review, and explicit user approval of a consolidated implementation batch; before any commit or push action for that batch. |
| `implement.single-pass-approved` | `implement` | After explicit user approval of a Feature-sourced or Idea-sourced single-pass implementation diff; before any commit or push action for that change. |

Skills MUST NOT introduce additional configurable milestone names without adding them to this catalog.

## Reference: Full Event Names

| Event | Emitter | Trigger |
|---|---|---|
| `idea.drafted` | `specstudio:ideate` | Every successful lint pass while `**Status:** Draft` |
| `idea.approved` | `specstudio:ideate` | User approves Recommended Direction (exactly once) |
| `idea.updated` | `specstudio:ideate` | Every successful lint pass while `**Status:** Approved` |
| `idea.implementing` | lifecycle tooling | First Feature created with this Idea in `**Source Ideas:**` (Approved → Implementing) |
| `idea.specified` | lifecycle tooling | Every Feature referencing this Idea reaches `Status: Stable` (Implementing → Specified) |
| `feature.specified` | `specstudio:specify` | Reviewer-approved, lint-clean Feature write |
| `feature.approved` | `specstudio:specify` | User approves the written Feature (exactly once) |
| `feature.updated` | `specstudio:specify` | Every successful lint pass while `status` ∈ {Approved, Implementing, Stable} after approval |
| `plan.drafted` | `specstudio:plan` | Reviewer-approved, lint-clean Plan write |
| `plan.approved` | `specstudio:plan` | User approves the written Plan (exactly once) |
| `plan.updated` | `specstudio:plan`, `specstudio:implement` | Every successful lint pass after an approved Plan is edited, including implementation status updates |
| `implementation.pre_commit` | gate-point (implement-autonomy layer) | Pre-action checkpoint before each commit during an `implement` run (multi-fire; per-occurrence gate evaluation) |
| `implementation.pre_push` | gate-point (implement-autonomy layer) | Pre-action checkpoint before a publish/promote during an `implement` run |
| `ship.pre_dispatch` | gate-point (`specstudio:ship`) | Pre-action checkpoint after ship's pre-flight gates and before the deploy dispatch (single-fire per run) |
| `pull_request.pre_dispatch` | gate-point (`specstudio:pull-request`) | Pre-action checkpoint after pull-request's pre-flight gates and before PR creation (single-fire per run) |
| `implement.batch-started` | `specstudio:implement` | A Plan-sourced implementation batch dispatches |
| `implement.batch-completed` | `specstudio:implement` | A Plan-sourced implementation batch reaches its publication checkpoint after user approval |
| `verify.completed` | `specstudio:verify` | Verify report and index are written successfully |
| `recap.completed` | `specstudio:recap` | Recap report and index are written successfully |
| `ship.completed` | `specstudio:ship` | Delegate returned explicit success and the Feature transitioned Implementing → Stable (exactly once per successful ship) |
| `pull_request.created` | `specstudio:pull-request` | Built-in path or delegate created the PR successfully (exactly once per successful run) |
| `lifecycle.phase-skipped` | `specstudio:ideate`, `specstudio:specify` | User chose a non-default downstream option at a transition menu (before downstream invocation) |
| `sidekick-idea.captured` | `specstudio:sidekick` | Successful seed write at `spec/ideas/seeds/<slug>.md` (exactly once per seed) |
| `sidekick-idea.reviewed` | `specstudio:consilium` | Successful `consilium-review` task completion (exactly once per task) |
