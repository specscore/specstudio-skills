# SpecStudio Autonomous Mode — Design

**Date:** 2026-07-03
**Status:** Approved — dogfooding through the SpecStudio pipeline (see `spec/ideas/`)
**Topic:** An "autonomous" run that drives an idea — from a cold raw prompt or any partial artifact — to MVP implementation without stopping for questions or approvals.

## Problem

Today the SpecStudio pipeline is deliberately interactive. Going from raw idea to shipped code means walking four producer skills — `specstudio:ideate` → `specstudio:specify` → `specstudio:plan` → `specstudio:implement` — each of which:

- asks **clarifying questions** (ideate's divergent/convergent questions; specify's scope/approach/section questions; plan's existing-plan check), and
- stops at **approval gates** (`gates.specify.reviewers` and `gates.plan` carry `type: human` entries; ideate and plan also carry prose-level user-review gates on the Idea and Plan).

The user wants: *"if I say do it autonomously it goes … to MVP implementation without asking questions"* — **and to be able to request the autonomous run while ideating, even on the very first prompt.**

## What "autonomous" means here

Confirmed with the user:

- **Ambiguity handling:** when the pipeline hits a decision it would normally ask about (which of 2–3 approaches, an ambiguous requirement, a scope cut), it **picks the most sensible default, records the choice, and keeps going**. Zero stops until the MVP builds. The user reviews the decision trail *after*, not during.
- **Entry point:** the run can start **anywhere in the pipeline, including a cold raw prompt with no artifact yet**. The trigger phrase *is* the human consent; autonomy does not require a pre-approved Idea. When it starts before or at the Idea stage, it runs `ideate` too — shaping the Idea for you with no per-question stops.
- **One checkpoint, at the crystallized Idea (`confirm_idea`, default on):** when the run auto-creates an Idea, it pauses **exactly once** to let the user approve that Idea before the pipeline barrels downstream. This is the single deliberate human confirmation — chosen because a raw one-line prompt is the most ambiguous possible input, so bounding drift at the cheapest, highest-leverage point (a 30-second Idea read) is worth one stop. After that approval the run is unbroken through specify → plan → implement → PR. When the run *starts* from an already-approved Idea (or later), there is no checkpoint — the human already approved it.

**Consequence, stated up front:** past the Idea checkpoint the run makes and records its own decisions. The **Autonomous Decisions log + the up-front disclosure are the safety mechanism** — you read afterward exactly what the run assumed. This is the deliberate trade the "decide + record" choice buys.

Confirmed with the user (were open questions; now resolved):

- **Shape:** a **new thin orchestrator skill**, `specstudio:autopilot`, not a bare config flag. It reuses the existing `autonomy:` namespace and scope ladder and the reviewer-gates masking mechanism; the existing producer skills gain a single small "if autonomous" branch each.
- **Publish ceiling: local commits + one PR.** An autonomous run stages and commits per the existing publication policy, then extends the chain to `specstudio:pull-request` to push the feature branch and open **exactly one** PR. It **never merges, deploys, or touches `ship`.** The PR is the hand-back surface.
- **`confirm_idea` default true** (see the Idea checkpoint above).
- **Dogfood:** this feature is itself built through the SpecStudio pipeline (`spec/ideas/…` → Feature → Plan → implementation).

## Design

Two cooperating pieces: an **orchestrator skill** and a **run-scoped autonomy context** that the existing skills already have most of the hooks for.

### 1. `specstudio:autopilot` — the orchestrator

A new skill under `skills/autopilot/`. It does not re-implement any producer logic; it is a driver.

**Trigger:** "do it autonomously" / "autonomously build `<raw prompt>`" on the very first message; "/autopilot", "/autopilot `<idea-slug>`"; or the phrase "autonomously" appended to any pipeline request (ideate/specify/plan/implement).

**Entry-point detection (no fixed precondition — the run starts wherever the work currently is):** the orchestrator inspects the input and repo to find the *furthest-along* existing artifact for this work, then runs every remaining stage autonomously:

| Input state | Autonomous run covers |
|---|---|
| Raw prompt, no artifact | ideate → specify → plan → implement |
| Draft / In-Review Idea | ideate (finish + approve) → specify → plan → implement |
| Approved Idea | specify → plan → implement |
| Approved Feature (no Plan) | plan → implement *(or implement directly, per `implement`'s Feature-sourced mode)* |
| Approved Plan | implement |

The only refusal is a genuinely empty ask (no prompt and no artifact). There is **no** "must be an approved Idea first" precondition — the trigger phrase is the consent to auto-approve every artifact the run creates, including the Idea.

**Run flow:**

1. Establish a **run-scoped autonomy context** (see §2) and disclose it up front in one message: the resolved entry point and the stages that will run, that human gates are masked for this run **except the single Idea checkpoint**, that decisions will be auto-made and logged, and that it will commit locally and open one PR (never merge/deploy).
2. **(If entry ≤ Idea stage)** Invoke `specstudio:ideate` on the raw prompt / Draft Idea → Idea. Divergent/convergent questions resolve to recorded defaults; the Recommended Direction is chosen; auto-shaped assumptions land in the Idea's `## Key Assumptions`. **Then honor `confirm_idea`:** when `true` (default), this is the one stop — present the crystallized Idea and wait for approval (the `idea.approved` human gate is *not* masked here). When `false`, mask it and proceed. On approval (or when masked), continue.
3. **(If entry ≤ Feature stage)** Invoke `specstudio:specify` on the Idea → Feature. Any clarifying question resolves to its recorded default; the `feature.approved` gate's `type: human` entry is masked.
4. **(If entry ≤ Plan stage)** Invoke `specstudio:plan` → Plan. Existing-plan check defaults to revise-in-place; plan's human review gate masked.
5. Invoke `specstudio:implement` → code. `implementation.pre_commit` is already `auto-approve`; commits land locally. Its `implementation.pre_push` human gate is masked (the push happens at the PR stage, not here).
6. Invoke `specstudio:pull-request` → **exactly one PR**. Its `pull_request.pre_dispatch` `type: human` entry is masked; its `deterministic` verify gate (if configured) is **not** masked and can still block. This pushes the feature branch and runs `gh pr create` (or the project's configured delegate). It never merges.
7. Hand back to the user with: the resolved entry point, the Idea/Feature/Plan paths, the commit SHAs, **the PR URL**, and the aggregated **Autonomous Decisions log** (the single most important review surface — read this first). Never auto-invokes `specstudio:verify`/`ship` beyond what the sub-skills already hand to; surfaces the recommendation.

**On any hard failure** (a reviewer gate returns `Issues Found` on its *AI/deterministic* reviewers, a lint failure that `--fix` can't clear, an `implement` subagent returns `BLOCKED`): the run **stops and hands back** with the exact blocker. Autonomy masks *human approval*, never *quality gates*. A `Blocker`-grade AI review finding is a real stop — the user is told and the run ends there.

### 2. The run-scoped autonomy context

The context is a single run-scoped signal the orchestrator sets and the producer skills read. It has three effects, each mapped onto machinery that already exists:

**(a) Human-gate masking — reuse reviewer-gates masking.** The runner already defines the exact semantics we want for a masked `type: human` entry: *"neither dispatched nor counted toward the verdict… the gate can release on its remaining (matching) entries alone"* (`shared/reviewer-gates/runner.md`, Step 2's `when:` handling — explicitly called "the home for per-branch autonomy masks"). We generalize that one mechanism from *branch-scoped* (`when:`) to also honor a *run-scoped* autonomy signal: when the autonomy context is active, `type: human` entries are masked exactly as a non-matching `when:` masks them. No new verdict type, no new gate semantics — the gate simply releases on its AI/deterministic entries, and if those pass, the run proceeds. If an AI/deterministic reviewer returns `Issues Found`, the gate blocks as normal and the run stops (per §1).

**(b) Decide-and-record instead of ask.** Each producer skill's question steps gain one branch: *if the autonomy context is active, do not call `AskUserQuestion`; instead choose the documented default and append the choice to the artifact's decision log.* The defaults are the ones already implied by each skill's own "default" annotations:
  - `ideate`: treat the raw prompt as the seed; run divergent expansion internally; in convergent mode pick the highest-conviction Recommended Direction; auto-fill MVP Scope / Not Doing / Key Assumptions; resolve every `## Open Questions` item to a stated assumption rather than surfacing the wizard; auto-approve the Idea. Every convergent choice is logged.
  - `specify`: proceed (don't decompose) when the Idea is single-scope; pick the **recommended** approach from its own 2–3 proposal; accept each spec section as drafted.
  - `plan`: revise-in-place on an existing plan; accept deferred-AC set as computed.
  - `implement`: `commit_cadence` from the resolved `autonomy.implement` ladder (default `batch`); conflict-detection still stops the run (it is a correctness gate, not a preference).

**(c) The Autonomous Decisions log.** Every auto-made choice is recorded in the artifact it belongs to, so the trail is co-located with the work and survives the session:
  - Idea-level assumptions already have a home: `## Key Assumptions`.
  - Feature and Plan get a new optional `## Autonomous Decisions` section (H2, appended near the end, lint-allowed) with one bullet per decision: *what was decided, the alternatives, why the default was chosen.*
  - The orchestrator aggregates all sections into the hand-back summary.

### 3. Configuration surface

Under the existing `autonomy:` namespace in `specscore.yaml` (sibling of `autonomy.implement`):

```yaml
autonomy:
  autopilot:
    publish_ceiling: pr            # stage | commit | pr   (default: pr — commit locally + open one PR)
    confirm_idea: true             # default: the ONE early checkpoint — approve the crystallized
                                   # Idea, then unbroken autonomy after. Only fires when the run
                                   # auto-creates the Idea (entry ≤ Idea stage). Set false for a
                                   # fully unbroken cold-start run.
    stop_on:                       # decision classes that still halt even in autonomous mode
      - conflict                   # implement line-overlap conflict (always on, not removable)
```

The verbal trigger sets the autonomy context at **run** scope (top of the scope ladder), so it overrides nothing durable and evaporates when the run ends. A user who wants standing autonomy can set `autonomy.autopilot` at project/user scope, but the MVP ships the verbal-trigger path only.

## Scope

**In scope (MVP):**
- The `specstudio:autopilot` orchestrator skill covering **ideate → specify → plan → implement → pull-request**, entering at the furthest-along existing artifact (raw prompt included).
- Auto-shaping the Idea when the run starts at or before the Idea stage, with the single `confirm_idea` checkpoint (default on) — this is the change the "request autonomy while ideating, on the very first prompt" requirement adds.
- Run-scoped autonomy masking of `type: human` reviewer entries, via generalizing the existing runner mask. The `confirm_idea` Idea gate is exempt from masking when enabled.
- Decide-and-record branches in `ideate`, `specify`, and `plan` question steps; `## Autonomous Decisions` section for Feature and Plan; Idea assumptions in `## Key Assumptions`.
- Publish ceiling = local commits + **one PR** via `specstudio:pull-request` (push feature branch + `gh pr create`, or project-configured delegate).
- Up-front disclosure and end-of-run hand-back with the decision log and PR URL.

**Explicitly not in scope (YAGNI):**
- Merge / deploy / `ship` (autopilot stops at an open PR).
- More than one PR per run; PR stacking; retry of a failed PR delegate.
- Cross-repo master-plan autonomy.
- Standing/persistent autonomous mode as the primary path (config exists but verbal trigger is the MVP surface).
- Any weakening of AI/deterministic quality gates, lint, or conflict detection.

## Isolation & boundaries

- **`autopilot`** depends on the five producer skills (ideate, specify, plan, implement, pull-request) and the autonomy context; it owns orchestration and hand-back only, no producer logic.
- **The autonomy context** is a read-only run-scoped signal; producers read it, only the orchestrator sets it.
- **Reviewer-gates runner** gains one generalization (run-scoped mask alongside branch `when:`); its verdict contract is unchanged.
- Each producer skill's change is a single localized branch — testable by running that skill with the context on vs. off.

## Testing approach

- **Gate masking:** run `specify` with autonomy context on against a fixture Idea; assert the `type: human` entry is not dispatched and the gate releases on the AI reviewer alone; assert it still blocks when the AI reviewer returns `Issues Found`.
- **Decide-and-record:** assert no `AskUserQuestion` call fires and a `## Autonomous Decisions` bullet (or `## Key Assumptions` bullet for the Idea) is written for each choice point.
- **Cold-start end-to-end:** a **raw one-line prompt** → autopilot → assert the run stops once at the `confirm_idea` checkpoint; on approval, assert an Approved Idea, Feature, Plan, at least one local commit, and one open PR all exist; assert the hand-back lists every recorded decision from all stages and the PR URL.
- **`confirm_idea: false`:** assert the cold-start run makes zero stops (the Idea gate is masked) and still produces the full artifact chain + PR.
- **Entry-point resolution:** for each input state in the entry-point table (Draft Idea, Approved Idea, Approved Feature, Approved Plan), assert autopilot runs exactly the remaining stages and no earlier one; assert `confirm_idea` fires only when the run auto-creates the Idea.
- **Publish ceiling:** assert exactly one PR is created and no merge/deploy occurs.
- **Stop conditions:** force an AI `Issues Found`, a `pull_request` deterministic-gate failure, and an `implement` conflict; assert the run halts and hands back the blocker in each case.

## Resolved Decisions

1. **Shape** → new orchestrator skill `specstudio:autopilot`.
2. **Publish ceiling** → local commits + one PR (never merge/deploy).
3. **Cold-start guard** → `confirm_idea` default **on**: one Idea checkpoint, then unbroken.
4. **Dogfood** → yes; this design becomes a real `spec/ideas/…` Idea → Feature → Plan → implementation, built through the SpecStudio pipeline itself.

## Open Questions

None — all resolved above. Remaining detail (exact default-selection heuristics per skill, `## Autonomous Decisions` lint allowance) is deferred to the Feature/Plan.
