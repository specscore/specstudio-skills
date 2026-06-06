---
name: plan
description: |
  Turns an approved SpecScore Feature or Idea into a lint-clean Plan artifact
  at spec/plans/<slug>.md — an ordered list of tasks where each task
  references one or more AC IDs from its source Feature. When sourced from
  an Idea, AC coverage (P-001) does not apply. Hard-gates on lint,
  reviewer-subagent approval, and user approval. Optionally scaffolds
  follow-on artifacts via downstream skills.
  Trigger: "plan", "/plan", "plan this feature", "specstudio:plan", or
  event `feature.approved`.
aliases: [plan]
---

# Plan

Turn an approved SpecScore Feature (or Idea) into a lint-clean, ordered task list.

## Hard Gate

<HARD-GATE>
Do NOT invoke `specstudio:implement`, `writing-plans`, `frontend-design`, `mcp-builder`, or ANY implementation skill until ALL of the following are true:
  1. The Plan artifact exists at `spec/plans/<slug>.md` and contains at least one `### Task N: <name>` task inside the `## Tasks` section.
  2. Every acceptance criterion in the source Feature is either covered by ≥1 task (via the task's `**Verifies:**` line) OR explicitly listed under `## Deferred AC Coverage` with a concrete one-sentence reason. *(Skip in Idea-sourced mode — there are no ACs.)*
  3. `specscore spec lint` passes.
  4. The plan-document reviewer subagent returned `Approved`.
  5. The user has explicitly approved the written Plan.

The only skill invoked after `specstudio:plan` is `specstudio:implement` (or — while `implement` is unshipped — a hand-back to the user with that recommendation).
</HARD-GATE>

## When to Use

- A SpecScore Feature is `**Status:** Approved` and the user is ready to decompose it into work.
- A SpecScore Idea is `**Status:** Approved` and the user wants to plan directly from it without a Feature.
- The event `feature.approved` has fired and the user has confirmed they want to plan.
- The user wants to revise an existing approved Plan (revise-in-place by default).

**Refuse and redirect when:**

- The source Feature is `Draft` or `Under Review` → tell the user to run `specstudio:specify` to approve the Feature first. Do **not** plan against unapproved specs.
- The source Idea is `Draft` or `Under Review` → tell the user to approve the Idea first. Do **not** plan against unapproved Ideas.
- The request spans two or more Features → say so and offer to write two separate Plans.
- The user asks for a DAG / parallel-branch plan → linear-only is the MVP; offer a linearized best-effort order or defer.

## Pre-Flight

1. **Source artifact check.** Resolve the input to one of:
   - **Feature-sourced (default):** a path `spec/features/<feature-slug>/README.md`. Confirm `**Status:**` is one of `{Approved, Implementing, Stable}`. If not, stop and recommend `specstudio:specify`.
   - **Idea-sourced:** a path `spec/ideas/<slug>.md`. Confirm `**Status:** Approved`. If not, stop and tell the user to approve the Idea first. When an Idea is the source, enter **Idea-sourced mode** — the remaining steps note where behavior differs.
2. **Parse AC list.** *(Skip in Idea-sourced mode — Ideas have no formal ACs; proceed to step 3.)* Read the source Feature's `## Acceptance Criteria` section. Build the full list of AC slugs — each `### AC: <slug>` heading or each inline `Scenario:` AC block with an explicit `verifies REQ:<slug>` reference. This list is the AC-coverage contract; every entry must be accounted for in the final Plan.
3. **Slug derivation.** Default Plan slug = source Feature slug. If the user explicitly asks for an alternative breakdown ("a spike plan", "a stable plan"), use `<feature-slug>-<variant>` (single segment, lowercase, hyphen-separated, URL-safe). Refuse chained-variant suffixes.
4. **Existing-plan check.** If `spec/plans/<slug>.md` already exists, ask the user: revise in place (default), supersede with a new plan that invalidates the prior task list, or pick a different variant suffix.
5. **Single-source scope.** If the user's intent references work in more than one Feature or Idea, stop and offer to write two Plans.

## Checklist

Create a task for each and complete in order:

1. **Pre-flight** (steps above).
2. **Convergent task-breakdown dialogue.** In Feature-sourced mode: walk the AC list with the user. For each AC, decide: dedicated task / bundled into a multi-AC task / deferred to `## Deferred AC Coverage` with a reason. In Idea-sourced mode: walk the Idea's goals and scope with the user, decomposing into concrete tasks without AC mapping. Keep the conversation tight — batched questions, multiple-choice where possible.
3. **Order tasks linearly 1..N.** Respect inferable dependencies (an AC whose precondition is established by another AC's task). No gaps, no DAG, no parallel branches in the MVP.
4. **Create the Plan artifact** — run `specscore plan new <slug>` with its source flag (the required-CLI creation path; see `## Artifact Creation`), then FILL the scaffolded sections via `Edit`.
5. **Directory + index** — `specscore plan new` creates `spec/plans/` and its `README.md` index if absent. Tell the user the directory is being bootstrapped.
6. **Checkpoint manifest** — add all created or edited Plan paths and CLI-reported touched paths to the publication manifest.
7. **Lint** — `specscore spec lint`. On failure, run `specscore spec lint --fix` exactly once and re-lint; on persistent failure, surface violations with rule IDs and stop.
8. **Inline self-review** — placeholders, empty `**Verifies:**` lines, AC IDs that look misspelled vs the source Feature, task-name vs `**Verifies:**` contradictions.
9. **Status: Draft → Under Review.**
10. **Dispatch the baseline reviewer subagent** — see [reviewer-prompt.md](references/reviewer-prompt.md). Must return `Approved` before user review.
11. **Dispatch additional registered reviewers** (if any) from `specscore.yaml`'s `reviewers:` extension key. AND composition — every reviewer must return `Approved`.
12. **Publish + emit `plan.drafted`** after all reviewers Approved + lint passes: apply [publication-policy.md](../shared/publication-policy.md) for `plan.drafted`, then emit the event with `publication_result`.
13. **User review gate** — present the Plan with the summary of deferred ACs (if any) and explicitly ask for approval.
14. **On user approval** — status Under Review → Approved, re-run lint, apply publication policy for `plan.approved`, then emit `plan.approved` with `publication_result`.
15. **Transition to `specstudio:implement`** — or, while `implement` is unshipped, hand back with the recommendation that the user implement task-by-task using a general-purpose skill.
16. **Throughout** — watch for sidekick ideas. When an out-of-scope improvement surfaces (e.g., a Feature change), invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step immediately. Do not derail.

## Artifact Layout

Plans are **single files**, not directories. Unlike Features, a Plan has no `_tests/` or `assets/` siblings in the MVP.

```
spec/plans/
├── README.md                # Plans index
└── <slug>.md                # The Plan artifact (single file per plan)
```

The skill carries **no embedded schema** — `specscore plan new` is the single source of truth for the Plan's structure (title prefix, body metadata, the `## Summary` / `## Approach` / `## Tasks` / `## Deferred AC Coverage` / `## Open Questions` sections, and the adherence footer). For a read-only spec reference, see [`https://specscore.md/plan-specification`](https://specscore.md/plan-specification) — never fetch it to write the file; the CLI scaffold produces the canonical structure.

## Artifact Creation

**Creation is required-CLI.** `specscore plan new <slug>` is the ONLY way to create the Plan artifact — it produces a lint-clean skeleton by construction and bootstraps `spec/plans/` and its `README.md` index for you. There is **no direct-write fallback**: the CLI scaffold is the single source of truth for the Plan's structure. This follows the Required-CLI Artifact Creation policy — see the **Creation-class row** in [`../shared/cli-detection.md`](../shared/cli-detection.md).

### Step 1 — Run the creation call and branch on exit status

Do **not** run a standalone `command -v` probe. Invoke `specscore plan new <slug>` with **exactly one** source flag — `--feature <feature-slug>` (default) or `--idea <idea-slug>` — and branch on its exit status per the **Creation-class row** in [`../shared/cli-detection.md`](../shared/cli-detection.md) (which carries the per-outcome rationale): **`0`** → scaffold written, continue to Step 2; **`127`** → install message (`/specscore:install`), then **install-then-retry**; **exit `8`** → **upgrade-then-retry**, naming the missing `plan new` subcommand; **any other non-zero** → surface verbatim, **never** a direct-write fallback.

### Step 2 — Fill the scaffolded sections

With the lint-clean skeleton in place, FILL its sections via `Edit`: the `## Summary`, the `## Approach`, each `### Task N` entry with its `**Verifies:** <feature-slug>#ac:<ac-slug>` line (Feature-sourced) or `**Source:** idea:<slug>` line (Idea-sourced), the `## Deferred AC Coverage` section (Feature-sourced only; omit when no ACs are deferred), and `## Open Questions`. Preserve the scaffold's lint-clean structure — never replace it with a hand-authored layout.

**Idea-sourced differences:** `**Source:** idea:<slug>` replaces `**Source Feature:**` in body metadata. Tasks use `**Source:** idea:<slug>` instead of `**Verifies:**`. The `## Deferred AC Coverage` section is omitted (there are no ACs to defer).

**Schema notes:**

- The canonical id is the file slug; there is no separate `id` field.
- **Feature-sourced plans:** `**Source Feature:**` is **required** and MUST resolve to an existing Feature at `spec/features/<feature-slug>/README.md` with `**Status:**` in `{Approved, Implementing, Stable}`.
- **Idea-sourced plans:** `**Source:** idea:<slug>` is **required** and MUST resolve to an existing Idea at `spec/ideas/<slug>.md` with `**Status:** Approved`.
- `**Supersedes:**` MUST be present with value `—` when empty.
- Tasks are numbered linearly starting from 1 with no gaps.
- **Feature-sourced:** every task MUST have a `**Verifies:**` line listing ≥1 AC ID from the source Feature. AC IDs use the form `<feature-slug>#ac:<ac-slug>` so they are unambiguous when planning across Features in the future.
- **Idea-sourced:** every task MUST have a `**Source:** idea:<slug>` line referencing the source Idea. No AC IDs are required.

## AC Coverage Contract

The defining discipline of `plan`: **every AC in the source Feature is accounted for, exactly once.** Either:

1. Listed in at least one task's `**Verifies:**` line, OR
2. Listed under `## Deferred AC Coverage` with a one-sentence reason that names *why* and *when* (e.g., "Deferred to v2; waits on the third-party SDK release"; "Out of MVP scope; covered by the follow-on `<slug>` plan").

**Lint enforces this as `P-001`** (hard rule, not advisory). The reviewer subagent enforces it semantically (catches stale AC references, vague defer-reasons, AC-wrapper tasks).

Silent omission of an AC is a contract violation.

**Idea-sourced exemption:** When a Plan is sourced from an Idea (`**Source:** idea:<slug>`), the AC coverage contract does not apply — there are no ACs to enforce coverage against. Lint rule `P-001` does not fire for Idea-sourced plans. All other lint rules and gates still apply.

## Task Format

Each task MUST have:

1. A `### Task N: <task-name>` heading (numbered linearly).
2. **Feature-sourced:** a `**Verifies:** <feature-slug>#ac:<ac-slug>, …` line listing one or more AC IDs from the source Feature. **Idea-sourced:** a `**Source:** idea:<slug>` line referencing the source Idea.
3. A 1–3 sentence prose description.

Tasks MAY include optional fields (e.g., `**Notes:**`). The three above are required.

### Task granularity

**Lower bound (lint-enforced):** every task references ≥1 AC ID. *(In Idea-sourced mode, no AC reference is required; the `**Source:**` line is sufficient.)*

**Upper bound (reviewer-advisory):** each task SHOULD be sized to be implementable in one focused work session and SHOULD reference no more than ~3 AC IDs. The reviewer flags:

- **AC wrappers** — tasks that restate a single AC's `Then` clause with no implementation work beyond it.
- **Bundled-too-fat tasks** — tasks that reference many ACs and look like they should be split.

These are advisory findings only; the user can override.

## Order Discipline

In the MVP, task order is **strictly linear** — numbered 1..N with no gaps, no DAG, no parallel branches. If the user requests DAG-style ordering, decline and offer one of:

- Linearize the DAG into a best-effort order (the reviewer will check the order respects inferable dependencies).
- Defer until a future plan-schema revision supports `**Depends-On:**` per task.

If the reviewer flags an ordering that violates an inferable dependency, the finding **must cite** the specific REQ slug or prose passage that establishes the dependency. Uncited dependency claims are not actionable.

## Lint and Self-Review

Run `specscore spec lint` after every write. On failure:

1. Run `specscore spec lint --fix` exactly **once**.
2. Re-run `specscore spec lint` to verify.
3. If passing, continue and tell the user what was auto-fixed.
4. If still failing, surface remaining violations with rule IDs and stop.

The skill MUST NOT loop `--fix`, and MUST NOT carry its own knowledge of which rules are auto-fixable.

**Inline self-review** (before dispatching the reviewer subagent) checks:

1. **Placeholders** — `TBD`, `TODO`, `???`, `FIXME`.
2. **Empty `**Verifies:**` lines** — every task has at least one AC ID. *(In Idea-sourced mode: verify every task has a `**Source:** idea:<slug>` line instead.)*
3. **Misspelled AC IDs** — every `<feature-slug>#ac:<ac-slug>` reference resolves in the source Feature. *(Skip in Idea-sourced mode.)*
4. **Name/Verifies contradictions** — a task whose name describes work outside its claimed AC scope. *(Skip in Idea-sourced mode.)*

Fix inline. Don't re-review; move on.

## Reviewer Subagent

Dispatch the **baseline reviewer** using [reviewer-prompt.md](references/reviewer-prompt.md). It enforces the six baseline blocker categories:

1. **AC coverage gap.** An AC in the source Feature appears neither in a task's `**Verifies:**` nor in `## Deferred AC Coverage`. *(Does not apply in Idea-sourced mode.)*
2. **Stale AC reference.** A task references an AC ID that does not exist in the current source Feature. *(Does not apply in Idea-sourced mode.)*
3. **Order violates dependency.** A task is ordered before another task whose `**Verifies:**` ACs it depends on. Finding MUST cite the specific REQ slug or prose passage that establishes the dependency.
4. **Task is an AC wrapper.** A task restates a single AC's `Then` clause with no implementation work beyond it. *(Does not apply in Idea-sourced mode.)*
5. **Hidden multi-source scope.** A task description references work in a Feature or Idea other than the declared source.
6. **Defer-reason vague.** A `## Deferred AC Coverage` entry has a reason that does not specify why or when (e.g., "later", "TBD", "out of scope" with no follow-up reference).

Findings outside these six categories MAY be returned as `Advisory` severity.

**Additional registered reviewers (third-party).** Read `specscore.yaml` at the repo root. If it contains a top-level `reviewers:` extension key (per the [`third-party-integration`](../../spec/features/third-party-integration/README.md) Feature), dispatch each registered reviewer in addition to the baseline.

For each entry: read the `prompt` file (which MUST be a path inside the repo working tree), confirm it contains an explicit blocker/advisory taxonomy section, and dispatch the reviewer using the same Agent-tool pattern as the baseline. A reviewer prompt missing a documented taxonomy fails the reviewer registration AC — surface as a registry error; do not silently skip.

**Composition is AND.** The User Review Gate releases only when **every** registered reviewer plus the baseline returns `Approved`. Any single `Issues Found` from any reviewer blocks the gate. On `Issues Found`:

1. Address every blocker-severity finding from every failing reviewer.
2. Re-dispatch every reviewer that previously returned `Issues Found`.
3. Re-dispatch reviewers that previously returned `Approved` when the fix changes the Plan's structural sections (Tasks, Deferred AC Coverage) — MUST for structural changes, SHOULD for task-list edits covered by previously-Approved reviewers.

Advisory findings MAY be ignored.

The skill MUST NOT silently downgrade a blocker to advisory, MUST NOT skip a registered reviewer, and MUST NOT proceed past the User Review Gate while any reviewer's last verdict is `Issues Found`.

When `specscore.yaml` does not contain a `reviewers:` key, only the baseline runs — the absence of the key is not an error.

## User Review Gate

After all reviewers return `Approved`:

> "Plan written and lint-clean at `spec/plans/<slug>.md`. Reviewer subagent approved. [N tasks covering M ACs; K ACs deferred — list them here if any.] Please review and let me know if you want changes before we move to `specstudio:implement`."

In Idea-sourced mode, replace the AC coverage summary with: "[N tasks sourced from idea:\<slug\>.]"

Wait. If the user requests changes, fix, re-lint, re-review, re-gate. Only proceed once the user approves.

**Approval phrases** (English + semantic equivalents in any language the user is communicating in): `approve`, `approved`, `accept`, `accepted`, `lgtm`. On detection of any qualifying phrase, transition status directly.

**Vague positive signals** (`looks good`, `yeah`, `ship it`, `+1`, `🚀`, `yes`, `ok`): treat as soft signal and ask one explicit confirmation question. Do not silently transition status.

On confirmed approval:

- Update `**Status:** Under Review → Approved`.
- Re-run lint.
- Apply publication policy for `plan.approved` using a manifest that includes the status-transition edit and any CLI-reported touched paths. This runs only after explicit user approval; policy MUST NOT bypass the gate.
- Emit `plan.approved` event with `publication_result`.

## Event Emission

The skill participates in the event vocabulary with **three events**:

| Event | When |
|---|---|
| `plan.drafted` | After reviewer subagent returns `Approved` and lint passes. First emission carries `previous_revision: null`, `changed_sections: null`, `change_summary: null`. Subsequent emissions during reviewer iteration carry the previous revision. |
| `plan.approved` | Exactly once, after user approval and the Draft → Approved transition completes. |
| `plan.updated` | On every successful lint pass after a subsequent edit while `status ∈ {Approved, Implementing}`. |

All three carry `changed_sections`, `previous_revision`, and a factual `change_summary` (≤2 sentences, no speculation/editorializing) — null on the very first `plan.drafted`, non-null thereafter.

The skill MUST NOT emit `plan.drafted` for an already-approved Plan, and MUST NOT re-emit `plan.approved` for further iteration.

## Publication in Git

Every file the skill creates or edits — bootstrapped `spec/plans/`, `spec/plans/README.md`, the Plan file, and any CLI-reported touched path — MUST be added to the checkpoint manifest. Apply [publication-policy.md](../shared/publication-policy.md) at `plan.drafted`, `plan.approved`, and `plan.updated` checkpoints. If the policy includes `stage`, stage only manifest paths. If it includes `commit` or `push`, run the shared manifest, unrelated-index, approval-gate, and branch-safety checks first. Report the resolved policy, executed actions, skipped actions, and manifest paths in the same response as the artifact write. If publication fails (no git repository, lock contention, branch refusal), surface the failure and continue without aborting the artifact write.

## Transition to Build

After `plan.approved`, present the user with exactly **three** next-step options and wait for an explicit choice (per [Feature: Detached Background Plan Implementation](../../spec/features/detached-background-implement/README.md), AC `#ac:three-options-presented`):

1. **Approve** — the Plan is approved; stop here. No implementation is started.
2. **Approve + implement in this session** — transition to `specstudio:implement` in the current session.
3. **Approve + implement in background** — launch a detached background `claude` session that runs `specstudio:implement` autonomously in its own git worktree, so the user keeps working and can attach to steer it later.

For options 2 and 3, do **not** invoke any downstream skill other than `specstudio:implement` — no `frontend-design`, no `mcp-builder`, no other implementation skill. While `specstudio:implement` is unshipped, hand back to the user instead with: "Plan approved. The `specstudio:implement` skill is not yet shipped — implement task-by-task using a general-purpose implementation skill. Each commit should reference the AC IDs from the satisfied task's `**Verifies:**` line."

### Option 3 — detached background launch

When the user chooses option 3, the host session runs this sequence. It MUST NOT change the host's current branch (`#ac:worktree-precreated-host-unchanged`):

1. **Resolve the target branch.** Default to `feat/<plan-slug>`; let the user override with a branch name of their choice (`#ac:branch-defaults-and-overridable`).
2. **Validate the branch is safe.** Refuse to launch — and report the conflict — if the target branch equals the host session's current branch (`git rev-parse --abbrev-ref HEAD`) or is already checked out in another worktree (`git worktree list`) (`#ac:refuse-same-or-checked-out-branch`).
3. **Precreate the worktree.** `git worktree add <dir> <branch>` (existing branch) or `git worktree add <dir> -b <branch>` (new). Creating a worktree never changes the host's current branch, index, or files (`#ac:worktree-precreated-host-unchanged`).
4. **Launch with `claude --bg`** (NOT `claude -p`, which is non-attachable) from inside the worktree, with `--permission-mode acceptEdits` and a scoped `--allowedTools`. An action outside the allowlist pauses the session for approval rather than failing the run (`#ac:launches-bg-session`, `#ac:permissions-pause-not-fail`). The launched prompt instructs `specstudio:implement` against the approved Plan, operating only inside the worktree and never checking out another branch (`#ac:launches-bg-session`, `#ac:background-stays-in-worktree`), and carries the autonomous progress contract.

   > **Flag-order gotcha:** `--allowedTools` is variadic and will swallow the positional prompt if the prompt immediately follows it. Put `--allowedTools` before a non-variadic flag (e.g., `--permission-mode`), never directly before the prompt.

   ```bash
   BRANCH="feat/<plan-slug>"            # or user override; must differ from current branch
   WT="$(git rev-parse --show-toplevel)/../wt-$BRANCH"
   git show-ref --verify --quiet "refs/heads/$BRANCH" \
     && git worktree add "$WT" "$BRANCH" \
     || git worktree add "$WT" -b "$BRANCH"
   ( cd "$WT" && claude --bg \
       --allowedTools "Read Edit Write Bash(git *) Task Skill" \
       --permission-mode acceptEdits \
       "Run /implement on spec/plans/<slug>.md. You are an UNATTENDED background
        session, already on branch '$BRANCH' in a dedicated worktree. Operate ONLY
        inside this worktree; never 'git checkout' another branch or touch other
        branches. Progress contract: complete every task you can; if a task is
        blocked, skip it and continue with other unblocked tasks (only its
        dependents are blocked) — do not abort the run; schedule approval-requiring
        actions last; when only blocked tasks remain, PAUSE and wait for input —
        do not improvise a human decision." )
   ```

5. **Tell the user how to steer** (`#ac:steer-controls-available`): `claude agents` (list), `claude attach <id>` (steer), `claude logs <id>` (watch), `claude stop <id>` (stop). Multiple background runs may run concurrently; the user can switch between them. Note: `--bg` runs are daemon-hosted — they survive the launching session but not a full quit of all Claude clients.

## Revise vs Supersede

When the user wants to change an existing Plan:

- **Default: revise in place.** Git history is the record of evolution.
- **Supersede only when** the change invalidates the existing task list wholesale (e.g., the source Feature's ACs changed enough that the old tasks are stale). Set `**Supersedes:** <predecessor-slug>` in the new Plan's body metadata.

The skill MUST NOT silently delete or rewrite an existing Plan without the user's explicit choice between revise-in-place and supersede.

## Tone

The skill MUST NOT yes-machine weak Plans. When a task is vague, an AC is uncovered without justification, the task order violates a clear dependency, or the user asks to span multiple Features, say so with specificity and propose the alternative. Honest disagreement, not performative agreement.

## Verification

- [ ] `spec/plans/<slug>.md` exists
- [ ] **Feature-sourced:** `**Source Feature:**` resolves to an Approved/Implementing/Stable Feature — OR — **Idea-sourced:** `**Source:** idea:<slug>` resolves to an Approved Idea
- [ ] **Feature-sourced:** every task has a `**Verifies:**` line with ≥1 valid AC ID — OR — **Idea-sourced:** every task has a `**Source:** idea:<slug>` line
- [ ] **Feature-sourced:** every source-Feature AC is covered (in a task) or deferred (with a concrete reason) — *(skip in Idea-sourced mode)*
- [ ] Tasks numbered 1..N with no gaps
- [ ] `specscore spec lint` passes
- [ ] Baseline reviewer subagent returned `Approved`
- [ ] All registered third-party reviewers returned `Approved`
- [ ] User explicitly approved the Plan
- [ ] `**Status:** Approved` in body metadata
- [ ] Created/edited paths added to the checkpoint manifest and publication policy applied with disclosure
- [ ] `plan.drafted` + `plan.approved` events emitted with `publication_result`

## Red Flags

- Proceeding to `specstudio:implement` without user approval
- Tasks without `**Verifies:**` lines (Feature-sourced) or without `**Source:**` lines (Idea-sourced)
- ACs neither covered nor deferred (Feature-sourced only)
- Defer-reasons like "later" or "TBD"
- DAG-shaped task graphs (linear-only in MVP)
- Multi-Feature scope in a single Plan
- Planning against a Draft or Under Review Feature or Idea
- Writing to `docs/plans/` instead of `spec/plans/`
- Skipping the reviewer subagent or a registered third-party reviewer
- Invoking any skill other than `specstudio:implement` on transition
- Hard-coding stage-only handoff behavior instead of resolving publication policy for Plan events
- Letting publication policy bypass the reviewer or user approval gates

## References

- [reviewer-prompt.md](references/reviewer-prompt.md) — plan-document reviewer subagent template.
- [philosophy.md](../shared/philosophy.md) — shared tenets.
- [path-conventions.md](../shared/path-conventions.md) — `spec/` vs `docs/` rules.
- [publication-policy.md](../shared/publication-policy.md) — checkpoint resolution, manifest safety, first-run preference prompt, and publication disclosure.
- [specscore-lint-rules.md](../shared/specscore-lint-rules.md) — lint contract this skill assumes.
- [events.md](../shared/events.md) — event payloads emitted by this skill.
- [question-cadence.md](../shared/question-cadence.md) — when to batch vs single-question.
- [sidekick-capture.md](../shared/sidekick-capture.md) — sidekick-idea handling during the skill's flow.
- [Plan Skill Feature](../../spec/features/skills/plan/README.md) — the SpecScore Feature this skill implements.
