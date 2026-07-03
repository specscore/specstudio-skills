# Autopilot Autonomy Contract

**Status:** Canonical
**Applies to:** `specstudio:autopilot`, and the producer skills it drives (`ideate`, `specify`, `plan`, `implement`, `pull-request`) when a run-scoped autonomy signal is active.

This doc owns two things `specstudio:autopilot` and its collaborators read: the `autonomy.autopilot` **config namespace** (the durable knobs) and the **run-scoped autonomy signal** (the ephemeral per-run arming). It is the shared foundation; the orchestration that consumes it lives in `skills/autopilot`, the gate-masking mechanics live in [`reviewer-gates/runner.md`](reviewer-gates/runner.md), and the scope-ladder resolution lives in [`publication-policy.md`](publication-policy.md).

`autonomy.autopilot` is a sibling of `autonomy.implement` (see the "Commit Cadence and the `autonomy:` Namespace" section of `skills/implement`). As with that namespace, `autonomy:` declares **execution knobs**, distinct from `gates:` which declares **who approves**. Workflow-step names MUST NOT appear as top-level config keys; the knobs are reached only via `autonomy.autopilot.*`.

## Config namespace: `autonomy.autopilot`

```yaml
autonomy:
  autopilot:
    publish_ceiling: pr        # stage | commit | pr   — how far an autonomous run publishes
    confirm_idea: true         # pause once to approve a run-created Idea before going downstream
    stop_on:                   # decision classes that halt the run even under autonomy
      - conflict               # implement line-overlap conflict (always present, not removable)
```

### Fields and defaults-when-absent

Every field resolves to a built-in default when unset at all scopes — a `specscore.yaml` with **no `autonomy.autopilot` block at all** resolves exactly as the defaults below.

| Field | Values | Default when absent | Meaning |
|---|---|---|---|
| `publish_ceiling` | `stage` \| `commit` \| `pr` | `pr` | How far an autonomous run publishes without stopping. `stage`: leave changes staged. `commit`: local commits, no PR. `pr`: local commits **plus exactly one** pull request (never merge/deploy). |
| `confirm_idea` | `true` \| `false` | `true` | When the run auto-creates an Idea, `true` pauses exactly once for the user to approve that crystallized Idea before `specify`. `false` makes even a cold-start run fully unbroken. Only ever fires when the run enters at or before the Idea stage. |
| `stop_on` | list of decision classes | `[conflict]` | Decision classes that halt the run even under autonomy. `conflict` (the `implement` line-overlap conflict) is **always present and MUST NOT be removable** — a config that omits it still stops on conflict. Additional entries may be added; `conflict` cannot be subtracted. |

### Resolution across the scope ladder

`autonomy.autopilot` values resolve across the publication-policy **scope ladder** (run → session → project → user), narrower overriding broader, exactly as `autonomy.implement.commit_cadence` does. When a value is unset at every scope, the built-in default above applies. Durable scopes (project / user) are set through the SpecScore config; the run scope is set by the verbal trigger (below) and is not durable.

## The run-scoped autonomy signal

Separate from the durable knobs, an autonomous run carries a **run-scoped autonomy signal** — the per-run arming that tells the producer skills and the reviewer-gates runner "this is an autonomous run."

- **Set by the verbal trigger.** "do it autonomously" / `/autopilot` / "autonomously …" appended to any pipeline request arms the signal at **run scope** — the top of the publication-policy scope ladder.
- **Overrides nothing durable.** Because it sits at run scope, it shadows no project/user config; it only marks the current run.
- **Evaporates at run end.** The signal does not persist past the run. A subsequent run starts unarmed and re-reads the durable `autonomy.autopilot` / `gates:` config from scratch — never a prior run's armed state. (This mirrors `implement`'s re-arm rule: autonomy is per-run, never carried forward.)
- **Armed once per run.** The trigger arms the signal a single time for the whole run; no per-stage re-arming is required to keep the run moving. (The `implement` stage still inherits `approval-autonomy`'s post-anomaly explicit-re-arm, which is a genuine anomaly stop, not a per-stage gate.)

### What reads the signal

| Reader | Effect when the signal is active |
|---|---|
| `reviewer-gates` runner | Masks `type: human` reviewer entries (run-scoped generalization of the branch `when:` mask) — see [`reviewer-gates/runner.md`](reviewer-gates/runner.md). |
| `ideate` / `specify` / `plan` | Take the decide-and-record branch of their clarifying-question steps instead of calling `AskUserQuestion`. |
| `autopilot` orchestrator | Applies `confirm_idea`, the `publish_ceiling`, and the `stop_on` halts. |

The signal never masks `type: ai` / `type: deterministic` reviewers, `specscore spec lint`, or `implement` conflict detection — only `type: human` approval entries. Quality gates always run.

## Decide-and-record

Masking the `type: human` reviewer entries releases the **approval** gates. It does nothing about the **clarifying questions** the producer skills ask (`ideate`'s divergent/convergent questions, `specify`'s scope/approach/section questions, `plan`'s existing-plan check). Those are a separate concern, and this is where they are settled.

**The rule.** When the run-scoped autonomy signal is active, a producer skill's clarifying-question step MUST NOT call `AskUserQuestion`. Instead it selects the skill's own **documented default** and records the choice (below). The defaults:

| Skill | Documented default under autonomy |
|---|---|
| `ideate` | Choose the highest-conviction Recommended Direction; auto-fill MVP Scope / Not Doing / Key Assumptions; resolve every `## Open Questions` item to a stated assumption rather than surfacing the wizard. |
| `specify` | Proceed-not-decompose when the intent is single-scope; pick the **recommended** approach from the 2–3 proposal; accept each spec section as drafted. |
| `plan` | Revise-in-place on an existing plan; for a fresh plan, accept the generated task breakdown and deferred-AC set as computed. |

The `confirm_idea` checkpoint is the **one** exception: even under autonomy, a run-created Idea still stops once for approval unless `confirm_idea: false` (see the config table). Decide-and-record governs everything *except* that single stop.

## The `## Autonomous Decisions` section

Every auto-made clarifying decision is recorded in the artifact it belongs to, so the trail is co-located with the work and survives the session:

- **Idea-stage** decisions land in the Idea's existing `## Key Assumptions` (as stated assumptions).
- **Feature- and Plan-stage** decisions land in a `## Autonomous Decisions` section — an H2 near the end of the artifact, one bullet per decision in the form *what was decided — the alternatives — why the default was chosen*.

Rules for the section:

- **Optional and omit-when-empty.** The section MUST be absent when the stage auto-made no clarifying decision (an all-human-run artifact carries no such section). Never write an empty `## Autonomous Decisions` heading.
- **Lint-clean.** The section is plain Markdown (an H2 followed by a bullet list) and MUST keep the artifact lint-clean; it introduces no new required field and no schema change.
- **Aggregated at handback.** `specstudio:autopilot` collects every stage's recorded decisions (Idea `## Key Assumptions` additions + Feature/Plan `## Autonomous Decisions`) into its end-of-run handback so the user reviews the whole trail in one place.
