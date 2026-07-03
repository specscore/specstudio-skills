---
name: autopilot
description: |
  Drives an idea from any pipeline entry point — a cold raw prompt included —
  to one open MVP pull request in a single autonomous run, pausing only at a
  single Idea checkpoint and halting on genuine anomalies. Thin orchestrator:
  it detects the furthest-along artifact, then chains the existing producer
  skills ideate → specify → plan → implement → pull-request, arming a
  run-scoped autonomy signal that releases the human-approval gates and switches
  the producers into decide-and-record. It re-implements no producer logic and
  reuses the approval-autonomy implement/plan gate layer unchanged. Publish
  ceiling is local commits + one PR — never merge, deploy, or ship.
  Trigger: "do it autonomously", "/autopilot", "/autopilot <slug>", or
  "autonomously" appended to any ideate/specify/plan/implement request.
aliases: [autopilot]
---

# Autopilot

Turn a single trigger into an end-to-end autonomous run: from a raw prompt (or any partial artifact) to one open MVP pull request, with no per-question stops and exactly one human checkpoint at the crystallized Idea. Autopilot is a **thin orchestrator** — it drives the existing `ideate → specify → plan → implement → pull-request` skills, arms the run-scoped autonomy signal defined in [autonomy-autopilot.md](../shared/autonomy-autopilot.md), and hands back the trail. It owns orchestration, the Idea checkpoint, the publish ceiling, and the handback — nothing else.

## Hard Gate

<HARD-GATE>
Autopilot MUST NOT:
  - Re-implement any producer skill's logic, gates, or artifact writes — it only invokes them.
  - Mask any gate other than a `type: human` approval entry. `type: ai` / `type: deterministic` reviewers, `specscore spec lint`, and `implement`'s conflict detection ALWAYS run and can halt the run.
  - Skip the `confirm_idea` checkpoint when the run auto-creates an Idea and `autonomy.autopilot.confirm_idea` is not `false`.
  - Publish past its ceiling: local commits + exactly ONE pull request. It MUST NOT merge, deploy, invoke `ship`, open a second PR, or retry a failed PR delegate.
  - Auto-invoke `specstudio:verify` or `specstudio:ship`.

If a stage's quality gate returns `Issues Found`, or an anomaly halts `implement`, autopilot STOPS and hands back the blocker — it never releases a quality gate to keep moving.
</HARD-GATE>

## When to Use

- The user says "do it autonomously", `/autopilot`, `/autopilot <slug>`, or appends "autonomously" to any pipeline request — **even on the very first prompt**, before any artifact exists.
- The user wants to reach an MVP without answering per-stage clarifying questions or approving each gate, accepting one Idea checkpoint and a decision log they review afterward.

**Refuse only** when the input is a genuinely empty ask — no prompt and no resolvable artifact to act on.

## Entry-Point Detection

Autopilot has **no fixed precondition** — it starts wherever the work currently is. Resolve the furthest-along existing artifact for the work, then run every remaining stage:

| Input state | Autopilot runs |
|---|---|
| Raw prompt, no artifact | ideate → specify → plan → implement → pull-request |
| `Draft` / `In Review` Idea | ideate (finish) → specify → plan → implement → pull-request |
| `Approved` Idea | specify → plan → implement → pull-request |
| `Approved` Feature, no Plan | plan → implement → pull-request *(or implement directly, per `implement`'s Feature-sourced mode)* |
| `Approved` Plan | implement → pull-request |

Resolution steps:

1. If the trigger names a slug (`/autopilot <slug>`), resolve that artifact; otherwise infer the work from the prompt and scan `spec/ideas/`, `spec/features/`, `spec/plans/` for a matching in-flight artifact.
2. Pick the furthest-along artifact in the pipeline order `idea < feature < plan`. Enter at the stage that produces the **next** artifact.
3. The trigger phrase **is** the consent to auto-create and auto-approve every artifact the run produces (subject only to the `confirm_idea` checkpoint). There is no separate "must be an approved Idea first" gate.
4. If nothing resolves and there is no prompt to ideate from, refuse and say what's missing.

## The Run

Autopilot performs the run as an ordered drive over the producer skills. It does **not** reimplement any of them; each stage is the existing skill invoked in the current session.

1. **Arm.** Establish the run-scoped autonomy signal ([autonomy-autopilot.md](../shared/autonomy-autopilot.md)) at run scope — **once** for the whole run (see Single Arming). Resolve `autonomy.autopilot` (`publish_ceiling`, `confirm_idea`, `stop_on`) across the scope ladder.
2. **Disclose.** Emit the single up-front disclosure message (see the Disclosure section) before any stage runs.
3. **Drive the stages** from the resolved entry point, in order, advancing to the next stage only after the current stage completes successfully:
   - **ideate** (if entry ≤ Idea) → then honor the `confirm_idea` checkpoint (see the Idea Checkpoint section).
   - **specify** (if entry ≤ Feature) — its `gates.feature.approved` `type: human` entry is masked by the run-scoped signal; the `type: ai` reviewer still runs.
   - **plan** (if entry ≤ Plan) — its human review gate is masked; the baseline reviewer and P-001 coverage still hold.
   - **implement** — `implementation.pre_commit` is autonomous per `approval-autonomy`; `implementation.pre_push` is masked here (the push happens at the PR stage). Anomaly-halts still stop the run.
   - **pull-request** — open exactly one PR (see the Publish Ceiling section).
4. **Hand back** the trail (see the Handback section).

At every stage, a masked `type: human` entry is released by the reviewer-gates runner's Step 1.6; the producers take their decide-and-record branch instead of asking clarifying questions. Autopilot itself asks nothing except the one `confirm_idea` checkpoint.

## Single Arming

The trigger arms the run-scoped autonomy signal **once**, for the remainder of the run. Autopilot does NOT re-arm at each stage boundary — crossing from `specify` into `plan` into `implement` requires no fresh signal.

The one exception is inherited, not owned: if `implement` hits an **anomaly-halt** (sibling conflict, BLOCKED subagent, unresolved lint, source-Feature drift), that is a genuine stop under `approval-autonomy`'s rules, and `implement` requires the explicit `continue` re-arm before resuming. That anomaly re-arm is `implement`'s, scoped to its own stage — it is not a per-stage gate autopilot introduces. A subsequent autopilot run starts unarmed and re-reads config from scratch.

## Anomaly Halts

Autonomy masks *human approval*, never *quality*. Autopilot stops and hands back the blocker whenever:

- a stage's `type: ai` / `type: deterministic` reviewer returns `Issues Found`;
- `specscore spec lint` fails and `--fix` does not resolve it in one pass;
- `implement` reports any anomaly in its halt set (per `approval-autonomy`);
- a `stop_on` decision class fires (`conflict` is always in the set and not removable).

On any halt, autopilot names the specific cause, performs no auto-resolution, and does not advance. The user fixes it and re-triggers (or issues `implement`'s `continue` re-arm for an implement-stage anomaly).

## Idea Checkpoint (`confirm_idea`)

When the run **auto-creates an Idea** (entry at or before the Idea stage), autopilot pauses **exactly once** — the single human-approval stop of an otherwise unbroken run. This is the cheapest, highest-leverage place to bound drift: a raw one-line prompt is the most ambiguous possible input, and a 30-second Idea read catches divergence before it propagates downstream.

- **Default on.** With `autonomy.autopilot.confirm_idea` unset or `true`, after `ideate` crystallizes the Idea autopilot presents it and waits for explicit approval before invoking `specify`. The `idea.approved` human review gate is **not** masked at this checkpoint — it is deliberately preserved. Recognize approval with the standard explicit-approval phrase set; a vague positive gets one confirmation question.
- **Opt-out.** With `confirm_idea: false`, the checkpoint is masked like every other human gate and the cold-start run is fully unbroken.
- **Not applicable when the Idea pre-exists.** When the run enters at or after an **already-`Approved` Idea**, no checkpoint fires — the human already approved that Idea. The checkpoint is specifically for Ideas the run itself shaped.

On explicit approval (or when `confirm_idea: false`), continue to `specify`. On a change request, fold it into the Idea and re-present — the run does not proceed past an unapproved run-created Idea.

## Publish Ceiling

Autopilot's ceiling is **local commits + exactly one pull request**, resolved from `autonomy.autopilot.publish_ceiling` (default `pr`):

- Commits land during `implement` via the existing `implementation.pre_commit` gate (autonomous per `approval-autonomy`). `implementation.pre_push` is masked in the implement stage — the push happens once, at the PR stage.
- After `implement` completes, autopilot invokes `specstudio:pull-request` **exactly once** to push the feature branch and open a single PR (built-in `git push` + `gh pr create`, or the project's configured `pull_request.delegate`). Its `pull_request.pre_dispatch` `type: human` entry is masked; a `type: deterministic` verify gate there is **not** masked and can still block.
- `publish_ceiling: commit` stops after local commits (no PR); `publish_ceiling: stage` stops after staging.

Autopilot MUST NOT merge, deploy, invoke `ship`, open more than one PR, or retry a failed PR delegate. The push safety floor (`main`/`master`/`release/*` denied by default) still applies — autonomy never weakens it.

## Disclosure

Before any stage runs, autopilot emits **one** disclosure message so the user knows exactly what the run will do unattended. It states:

- the **resolved entry point** and the **stages** that will run;
- that human-approval gates are **masked for the run**, *except* the single `confirm_idea` checkpoint (named explicitly);
- that clarifying decisions will be **auto-made and recorded** to each artifact's decision log;
- that the run will **commit locally and open one PR**, and will **not** merge or deploy.

The disclosure is informational — it is not a per-stage approval prompt. The run proceeds immediately after it.

## Handback

On successful completion, autopilot hands back a single summary — the primary review surface, meant to be read first:

- the **resolved entry point** and stages run;
- the created **artifact paths** (Idea / Feature / Plan);
- the local **commit SHAs**;
- the opened **PR URL**;
- the **aggregated Autonomous Decisions log** — every decide-and-record entry from every stage (Idea `## Key Assumptions` additions + Feature/Plan `## Autonomous Decisions`), collected in one place.

Autopilot does **not** auto-invoke `specstudio:verify` or `specstudio:ship`; it MAY recommend the user run them next.

## References

- [autonomy-autopilot.md](../shared/autonomy-autopilot.md) — the `autonomy.autopilot` config namespace, the run-scoped autonomy signal, and the decide-and-record + `## Autonomous Decisions` contract.
- [reviewer-gates/runner.md](../shared/reviewer-gates/runner.md) — Step 1.6, the run-scoped human-gate mask this skill arms.
- [approval-autonomy Feature](../../spec/features/approval-autonomy/README.md) — the reused implement/plan gate layer, anomaly-halts, and `continue` re-arm.
- [publication-policy.md](../shared/publication-policy.md) — the scope ladder the run-scoped signal sits on, and the push safety floor.
- Producer skills: [ideate](../ideate/SKILL.md), [specify](../specify/SKILL.md), [plan](../plan/SKILL.md), [implement](../implement/SKILL.md), [pull-request](../pull-request/SKILL.md).
