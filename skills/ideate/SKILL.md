---
name: ideate
description: |
  Refines raw ideas into SpecScore Idea artifacts through structured
  divergent and convergent thinking. Produces a lintable pre-spec
  one-pager at spec/ideas/<slug>.md that can be promoted to one or
  more SpecScore Features. Use when the user has a vague concept and
  isn't ready to specify yet.
  Trigger: "ideate", "/ideate", "refine this idea", "stress-test this".
aliases: [ideate]
---

# Ideate

Turn raw ideas into sharp, SpecScore-compatible Idea artifacts through structured divergent and convergent thinking.

## Hard Gate

<HARD-GATE>
Do NOT invoke `specstudio:specify`, `specstudio:plan`, `specstudio:implement`, `writing-plans`, or any implementation skill until:
  1. An Idea artifact has been written to `spec/ideas/<slug>.md`.
  2. `specscore spec lint` passes.
  3. The user has explicitly approved the Recommended Direction.

Ideas that can't be lint-clean aren't ready to be specified.
</HARD-GATE>

## When to Use

- Raw, vague, or unvalidated concept.
- User unsure whether an idea is worth building.
- Multiple possible directions with no clear winner.
- **Promoting a captured seed** — the user wants to turn an existing sidekick seed (`spec/ideas/seeds/<slug>.md`) into a full Idea. See [Promoting a Sidekick Seed](#promoting-a-sidekick-seed) below.
- **Skip** when: the user already has an approved Idea or a clear, high-conviction feature to specify — go straight to `specstudio:specify`.

## Promoting a Sidekick Seed

Entry mode for turning an existing sidekick seed at `spec/ideas/seeds/<slug>.md` into a full Idea. Triggers: "promote seed `<slug>`", "pick up seed `<slug>`", "ideate from the `<slug>` seed". This path delegates the file transformation to the `specscore idea promote` CLI verb and then continues the normal authoring flow below — it does **not** hand-move the seed or hand-write the Idea.

1. **Resolve the seed** at `spec/ideas/seeds/<slug>.md`. If it doesn't exist, say so and stop (don't silently scaffold a fresh Idea).
2. **Consilium offer (unreviewed, manual pick).** A manually-picked seed is pulling from the queue out of band, so if the seed has **no `## Consilium Verdict` section**, OFFER to run the consilium first (`specstudio:consilium`) before promoting. Default to **yes** on an empty response (offer-and-default-to-yes). The offer is **suppressible**: when `specscore.yaml` has `promote.offer_consilium: false`, skip it and promote directly. The user MAY decline; on decline, promote without a verdict. Never hard-require a verdict, never block on decline.
3. **Delegate to the CLI.** Run `specscore idea promote <slug>` (add `--verdict=<pointer|full|drop>` only to override the project default; `--force` only to overwrite an existing Idea at that slug). The verb owns the `git mv`, the seed→Idea transform, same-repo back-link reconciliation, and the cross-repo archive — do NOT replicate any of that by hand. Branch on exit status per [`../shared/cli-detection.md`](../shared/cli-detection.md):
   - **exit `127`** — the verb isn't installed. Tell the user to update the `specscore` CLI (the `idea promote` verb ships in **v0.6.1+**). Do **not** fall back to hand-moving the seed — there is no fallback for promotion.
   - **any other non-zero** — surface the error verbatim and stop; do not hand-edit.
   - **success** — the verb prints the created Idea path (and the seed's fate: `moved` for same-repo, `archived` for cross-repo).
4. **Fill the skeleton.** The verb leaves a lint-clean Idea at `spec/ideas/<slug>.md` with the seed body folded into `## Context` and HTML-comment prompts for the rest. Fill the remaining sections (Recommended Direction, Alternatives Considered, MVP Scope, Not Doing, Key Assumptions, SpecScore Integration, Open Questions) via the normal **Phase 3** authoring, then continue the **Checklist from step 6 (Lint)** onward — self-review, publication checkpoint, user review gate, and `idea.drafted` / `idea.approved` events all apply unchanged. Promotion yields a `**Status:** Draft` Idea.

## Philosophy

See [philosophy.md](../shared/philosophy.md). Key tenets here: *simplicity is the ultimate sophistication*, *say no to 1,000 things*, *challenge every assumption*, *unsaved ideation is waste*, *prefer stable CLI contracts over ad-hoc file writes when both are possible*.

## Path Conventions

Artifacts land at `spec/ideas/<slug>.md`. See [path-conventions.md](../shared/path-conventions.md). Never use `docs/ideas/`.

If `spec/ideas/` does not exist when the skill is invoked, **bootstrap it** before writing the first artifact: create the directory and a lint-clean `spec/ideas/README.md` index (`type: index`, empty Contents table, `Open Questions: None at this time.`). Tell the user explicitly that you bootstrapped it. Never silent.

## Checklist

Create a task for each and complete in order:

1. **Explore project context** — existing Features, architecture, related Ideas (`Glob`, `Grep`, `Read`).
2. **Scope decomposition check** — if the request describes multiple independent subsystems, stop and help the user split into multiple Ideas before proceeding.
3. **Phase 1 — Understand & Expand** (divergent).
4. **Phase 2 — Evaluate & Converge**.
5. **Phase 3 — Crystallize** as a SpecScore Idea artifact. Bootstrap `spec/ideas/` if missing. Create the artifact with the `specscore idea new` CLI scaffold — the required creation path (see Phase 3 below).
6. **Lint** the artifact: `specscore spec lint`. On failure, run `specscore spec lint --fix` once, re-lint; surface remaining violations to the user.
7. **Publication checkpoint** — add every file you created or edited (`spec/ideas/<slug>.md`, plus bootstrap files if any) to the checkpoint manifest and apply [publication-policy.md](../shared/publication-policy.md) for the lifecycle event being emitted.
8. **Inline self-review** — placeholders, contradictions, ambiguity, scope.
9. **User review** — ask the user to review and approve the Recommended Direction. Recognize explicit approval phrases (`approve`, `approved`, `accept`, `accepted`, `lgtm`, plus their semantic equivalents in the user's language); treat vague positive signals as soft and ask one explicit confirmation question.
10. **Emit events** — `idea.drafted` on every successful lint pass while `**Status:** Draft`; `idea.approved` exactly once on approval; `idea.updated` on every successful lint pass while `**Status:** Approved`. Apply publication policy before emission so each event carries `publication_result`. Both `drafted` and `updated` payloads carry `changed_sections`, `previous_revision`, and a factual `change_summary` (≤2 sentences). See [events.md](../shared/events.md).
11. **Throughout** — watch for sidekick ideas per [sidekick-capture.md](../shared/sidekick-capture.md). When an out-of-scope improvement surfaces, invoke `specstudio:sidekick` with a one-liner, acknowledge in one line, and return to the current checklist step immediately. Do not derail to discuss the sideline idea.

## Phase 1 — Understand & Expand (Divergent)

**Goal:** Open the idea up before narrowing.

1. **Restate as a "How Might We…"** sentence. Forces clarity on what's actually being solved.
2. **Ask 3–5 sharpening questions** (batched — see [question-cadence.md](../shared/question-cadence.md)). Focus:
   - Who is this for, specifically?
   - What does success look like?
   - What are the real constraints (time, tech, resources)?
   - What's been tried before?
   - Why now?

   Use `AskUserQuestion`. Do NOT proceed until you know *who* and *success*.

3. **Generate 5–8 variations** using these lenses:
   - **Inversion** — what if we did the opposite?
   - **Constraint removal** — what if budget/time/tech weren't factors?
   - **Audience shift** — what if this were for a different user?
   - **Combination** — what if we merged this with an adjacent idea?
   - **Simplification** — what's the version that's 10x simpler?
   - **10x** — what would this look like at massive scale?
   - **Expert lens** — what would domain experts find obvious?

   Push beyond what the user asked for. See [frameworks.md](references/frameworks.md) for additional frameworks (SCAMPER, JTBD, First Principles, Pre-mortem, Analogous Inspiration). Pick the lens that fits — don't run every framework mechanically.

**If inside a codebase:** ground variations in what actually exists. Reference specific files and patterns.

## Phase 2 — Evaluate & Converge

After the user reacts to Phase 1, shift to convergent mode. Cadence becomes **single question at a time**.

1. **Cluster** ideas that resonated into 2–3 meaningfully different directions.
2. **Stress-test** each direction on three axes — see [refinement-criteria.md](references/refinement-criteria.md):
   - **User value** (painkiller vs. vitamin)
   - **Feasibility** (technical + resource + time-to-value)
   - **Differentiation** (new capability? 10x? new audience? new context? better UX?)
3. **Surface hidden assumptions** in three tiers:
   - **Must be true** (dealbreakers — validate before building)
   - **Should be true** (significant impact, but recoverable)
   - **Might be true** (secondary; don't validate yet)

**Be honest, not supportive.** If a direction is weak, say so with specificity and kindness.

## Phase 3 — Crystallize as a SpecScore Idea

**Creation is required-CLI.** `specscore idea new <slug>` is the ONLY way to create the artifact — it produces a lint-clean skeleton by construction and updates `spec/ideas/README.md` for you. There is no direct-write fallback: the CLI scaffold is the single source of truth for the Idea's structure. Subsequent edits must preserve that lint-clean state. This follows the Required-CLI Artifact Creation policy — see the Creation-class row in [`../shared/cli-detection.md`](../shared/cli-detection.md).

### Step 3.0 — Bootstrap `spec/ideas/` if missing

Before invoking the CLI, check that `spec/ideas/` exists. If not:

1. Create the directory.
2. Create `spec/ideas/README.md` as a lint-clean Index artifact (title `# Ideas Index`, `**Status:** Stable`, empty Contents table, `Open Questions: None at this time.`, adherence footer). The CLI will append to this index when it scaffolds the artifact.
3. Tell the user, e.g., *"Bootstrapped `spec/ideas/` and `spec/ideas/README.md` (this project didn't have an ideas tree yet)."*

This step MUST NOT happen silently.

### Step 3a — Detect the CLI (per the shared convention)

`ideate` is a **creation-class** skill: the CLI is required and there is no direct-write fallback. Do **not** run a standalone `command -v` probe. Invoke the scaffold call (Step 3b's `specscore idea new`) and branch on its exit status per the **Creation-class row** in [`../shared/cli-detection.md`](../shared/cli-detection.md) (which carries the per-outcome rationale): **`0`** → continue on the CLI path (Step 3b, step 2); **`127`** → install message (`/specscore:install`), then **install-then-retry**; **exit `8`** → **upgrade-then-retry**, naming the missing `idea new` subcommand; **any other non-zero** → surface verbatim, **never** a hand-written fallback.

### Step 3b (CLI path) — Scaffold, then fill

1. Invoke `specscore idea new <slug>` with every field you already have. Only these flags exist — do not invent others:

   - `--title`
   - `--owner`
   - `--hmw` (Problem Statement)
   - `--context`
   - `--recommended-direction`
   - `--mvp`
   - `--not-doing` (repeatable; format `<thing> — <reason>`)
   - `--force` (only if overwriting an existing draft at the same slug)
   - `--project` (only if cwd isn't inside the target repo)

   Example:

   ```bash
   specscore idea new my-slug \
     --title "My Idea" \
     --owner "alex" \
     --hmw "How might we …?" \
     --context "Triggered by …" \
     --recommended-direction "We should …" \
     --mvp "A two-week spike that …" \
     --not-doing "Multi-tenant auth — out of scope for MVP" \
     --not-doing "iOS client — web-first"
   ```

   The command writes `spec/ideas/<slug>.md`, runs lint-fix, and exits non-zero if the result isn't lint-clean.

2. **Fill the scaffold's sections with `Edit`.** The scaffold lands sections with no matching flag — `Alternatives Considered`, `Key Assumptions to Validate` (the Must/Should/Might table), `SpecScore Integration`, `Open Questions` — as HTML-comment prompts. Replace each prompt with real content via `Edit`. Sections you *did* pass via flags are already filled. For the meaning of each section, treat the specification page `https://specscore.md/idea-specification` as a **read-only reference** — never fetch that URL to write or generate the file; the scaffold the CLI produced is the authoritative structure.

   A few section semantics to keep in mind while filling the prompts:

   - `**Promotes To:**` is **managed state** — Lifecycle tooling populates it when a Feature references this Idea. Authors MUST NOT edit it manually.
   - The **"Not Doing"** section is mandatory — lint rule `I-002` fails without it. Fill its entries; never leave it empty.

### Step 3c — Lint (with bounded auto-recovery)

After the artifact is complete, run `specscore spec lint`. The CLI scaffold is lint-clean on generation; your subsequent `Edit`s must not break that.

**On lint failure:**

1. Run `specscore spec lint --fix` exactly once.
2. Re-run `specscore spec lint`.
3. If lint now passes: continue. Tell the user what was auto-fixed.
4. If lint still fails: surface the remaining violations to the user with rule IDs and affected sections. Do NOT loop `--fix`.

**Trust the CLI's fix policy.** The skill does NOT carry its own list of which lint rules are safely auto-fixable — that policy lives in the `specscore` CLI. If `--fix` silently repairs a violation that should require human input, file the issue against `specscore`, not this skill.

### Step 3d — Publication checkpoint

After every successful artifact write or edit, build a checkpoint manifest containing the affected paths:

```bash
spec/ideas/<slug>.md
# plus any bootstrap files you created in Step 3.0
spec/ideas/README.md
```

Resolve and apply [publication-policy.md](../shared/publication-policy.md) for `idea.drafted` before emitting that event. The disclosure must name the policy source when known, resolved actions, any branch-safety effect, and the manifest paths. If the policy includes `stage`, stage only manifest paths. If it includes `commit` or `push`, run the manifest and branch safety checks from the shared protocol first. If publication fails (no git repo, detached worktree, lock contention, branch refusal), surface the failure and continue without aborting the artifact write or bypassing the user review gate.

## Inline Self-Review

Look at the written artifact with fresh eyes. Check for:

- **Placeholders** — `TBD`, `TODO`, `???`. Fix in place.
- **Internal consistency** — does the Recommended Direction match the MVP Scope? Do the assumptions line up?
- **Scope** — is this one Idea or three smuggled into one?
- **Ambiguity** — could a reader interpret a requirement two ways?

Fix inline. Don't re-review; move on.

## User Review Gate

After lint + self-review pass:

> "Idea drafted and lint-clean at `spec/ideas/<slug>.md`. Please review the Recommended Direction and MVP Scope. Approve to choose your next step (Specify, Plan, or Implement), or request changes."

Wait. If the user requests changes, make them, re-lint (with the auto-recovery flow above), apply publication policy for the next `idea.drafted` checkpoint, and emit a fresh `idea.drafted` event. Only proceed once the user approves.

### Recognizing approval

**Explicit approval — transition immediately, no confirmation prompt:**

The following phrases (case-insensitive, optional surrounding punctuation) count as unambiguous approval:

- English: `approve`, `approved`, `accept`, `accepted`, `lgtm`
- Direct semantic equivalents in any language the user is communicating in: `aprobar` / `aprobado` (Spanish), `approuver` / `approuvé` (French), `承認` / `承認する` (Japanese), `одобрить` / `одобрено` / `принято` (Russian), `批准` / `同意` (Chinese), `genehmigen` / `genehmigt` (German), and so on.

The criterion is semantic, not lexical: the phrase must function as a verb form meaning "I give explicit approval" in the source language. Recognize it when the phrase is the standalone response or the dominant content of a short response. If the phrase appears only incidentally in a longer message ("we should approve this approach but I have one concern…"), do NOT treat it as approval — address the concern first.

**Vague positive signals — ask one explicit confirmation question:**

Phrases like `looks good`, `yeah`, `nice`, `ship it`, `+1`, `🚀`, `confirm`, `yes`, `ok`, `sí`, `oui`, `да`, `はい`, `好` are NOT explicit approval. They signal positive sentiment but are ambiguous in conversational context. On detecting one, respond with a single confirmation prompt:

> "Treat that as approval?"

Wait for the user. Proceed only on a follow-up explicit phrase. Never silently transition status on a vague signal.

### On confirmed approval

- Update `**Status:** Draft → Approved` in the body metadata.
- Re-run lint.
- Apply publication policy for `idea.approved` using a manifest that includes the status-transition edit and any CLI-reported touched paths.
- Emit `idea.approved` event (exactly once per Idea) with `publication_result`.

## Post-Approval Iteration

Once `**Status:** Approved`, the Idea is alive but not frozen. The user MAY edit it further (refine the Recommended Direction, add an Outstanding Question, update assumptions). On every subsequent successful lint pass after a write or edit:

- Status remains `Approved` (never roll back to `Draft`).
- Apply publication policy for `idea.updated` using a manifest containing the changed Idea file and any CLI-reported touched paths.
- Emit `idea.updated` (NOT `idea.drafted`) with `publication_result`.
- Do NOT re-emit `idea.approved`.

`idea.updated` is the signal downstream consumers use to notify Features that declare this Idea as a `Source Ideas` entry, so downstream specs can reconcile.

### Computing the change-context payload

Both `idea.drafted` (re-emissions) and `idea.updated` events carry three change-context fields. Compute them before emission:

- **`previous_revision`** — the git SHA at which the previous emission of the same event fired (for `idea.updated`, the SHA at which `idea.approved` last fired, or the previous `idea.updated`; for `idea.drafted` re-emissions, the SHA of the prior `idea.drafted`). On the very first `idea.drafted` for an Idea, this is `null`.
- **`changed_sections`** — list of H2 section names whose content differs between `previous_revision` and the current revision. H3 changes within a section roll up to the parent H2. Computed by parsing both versions and comparing per-section. `null` on the first `idea.drafted`.
- **`change_summary`** — a string of at most two sentences describing the change. **Factual only.** Describe what content was added, removed, or modified, in which section. Do NOT speculate about the user's intent or motivation. Do NOT editorialize about the quality or wisdom of the change. If the only change is whitespace or formatting, say so explicitly. `null` on the first `idea.drafted`.

**Examples of disciplined `change_summary`:**

- ✅ "Replaced two paragraphs in Recommended Direction; the proposed approach now uses event sourcing instead of CRUD."
- ✅ "Added one entry to Not Doing; updated one assumption in the Should-be-true tier."
- ✅ "Whitespace and trailing-newline cleanup; no semantic content changed."

**Examples of forbidden `change_summary`:**

- ❌ "User is rethinking the approach because of performance concerns." *(speculates about motivation)*
- ❌ "This change makes the spec more aligned with industry best practices." *(editorializes)*
- ❌ "Important changes to the recommended direction." *(vague; not factual)*

## Transition

After the user approves the Idea, present a transition menu via `AskUserQuestion` with exactly three options. List them in this order:

1. **Specify** (default) — invoke `specstudio:specify` with the Idea slug.
2. **Plan** — invoke `specstudio:plan` with the Idea slug as the source artifact.
3. **Implement** — invoke `specstudio:implement` with the Idea slug as the source artifact.

### Keyword heuristic

Before presenting the menu, scan the user's description of the change (the Problem Statement and any conversational context). If the language indicates trivial scope — keywords: `trivial`, `quick`, `typo`, `one-liner`, `simple`, `tiny`, `small fix`, `minor` — append "(suggested for small scope)" to the Implement option label.

### Menu prompt

> "The Idea is approved. How would you like to proceed?"
>
> 1. **Specify** — break it into a full Feature spec with requirements and acceptance criteria *(default)*
> 2. **Plan** — skip the Feature spec and go straight to a task plan
> 3. **Implement** — skip both spec and plan and go straight to code

When the heuristic fires, the third option reads:

> 3. **Implement** — skip both spec and plan and go straight to code *(suggested for small scope)*

### On user choice

- **Specify** (default): invoke `specstudio:specify`. No `lifecycle.phase-skipped` event.
- **Plan**: emit `lifecycle.phase-skipped` (see [events.md](../shared/events.md)) with `from_phase: ideate`, `skipped_phases: [specify]`, `to_phase: plan`, then invoke `specstudio:plan`.
- **Implement**: emit `lifecycle.phase-skipped` with `from_phase: ideate`, `skipped_phases: [specify, plan]`, `to_phase: implement`, then invoke `specstudio:implement`.

Set `reason` to `scope-suggested` when the keyword heuristic fired and the user confirmed the suggested option; otherwise `user-requested`.

### Promotion via lifecycle tooling

Promotion bookkeeping (`**Promotes To:**`, `**Status:** Approved → Specified`) remains out of scope for this skill. Lifecycle tooling handles it when a downstream artifact references the Idea's slug. **Do not manually edit `**Promotes To:**`.** It's managed state.

## Verification

- [ ] Artifact exists at `spec/ideas/<slug>.md`
- [ ] `spec/ideas/` and `spec/ideas/README.md` exist (bootstrap if needed; never silent)
- [ ] `specscore spec lint` passes (auto-recovery via `--fix` attempted at most once on initial failure)
- [ ] Created/edited paths added to the checkpoint manifest and publication policy applied with disclosure
- [ ] "How Might We" statement, target user, success criteria all explicit
- [ ] ≥2 alternatives stress-tested
- [ ] Assumptions audited across Must/Should/Might tiers
- [ ] "Not Doing" list non-empty
- [ ] User approved the Recommended Direction (explicit phrase OR vague-signal followed by explicit confirmation)
- [ ] `**Status:**` is `Approved` (if approved) or `Draft` (if not yet)
- [ ] Events emitted per state: `idea.drafted` while Draft; `idea.approved` once on transition; `idea.updated` while Approved
- [ ] Emitted events include `publication_result`
- [ ] `idea.drafted` and `idea.updated` payloads include `changed_sections`, `previous_revision`, and a factual `change_summary` (all `null` on first `idea.drafted`, non-null thereafter)

## Red Flags

- Generating 20+ shallow variations vs 5–8 considered ones
- Skipping the "who is this for" question
- No assumptions surfaced before committing to a direction
- Yes-machining weak ideas instead of pushing back
- Empty "Not Doing" list
- Writing to `docs/ideas/` instead of `spec/ideas/`
- Manually editing `**Promotes To:**` (managed state)
- Hand-moving a seed or hand-writing the Idea when promoting, instead of delegating to `specscore idea promote` (and falling back to a hand-move on exit `127` — promotion has no fallback)
- Skipping the consilium offer when promoting an unreviewed, manually-picked seed (unless suppressed by `promote.offer_consilium: false`), or hard-requiring a verdict / blocking when the user declines
- Jumping to `specstudio:specify`, `specstudio:plan`, or `specstudio:implement` before user approval
- Silently bootstrapping `spec/ideas/` without telling the user
- Looping `specscore spec lint --fix` more than once
- Encoding the skill's own list of "rules `--fix` shouldn't touch" — that policy belongs to the `specscore` CLI; if it gets it wrong, fix it there
- Treating `yes` / `ok` / `sí` / `oui` / `+1` / `🚀` as explicit approval (silent status transition on vague signal)
- Hard-coding stage-only behavior instead of resolving publication policy for `idea.drafted`, `idea.approved`, or `idea.updated`
- Letting publication policy bypass the Recommended Direction approval gate
- Emitting `idea.drafted` for an Approved artifact, or re-emitting `idea.approved`
- Speculating about user intent or editorializing in `change_summary` ("User is rethinking…", "Important changes…", "More aligned with best practices…") instead of describing the observable change in factual terms
- `change_summary` longer than two sentences
- Including `changed_sections` line numbers (consumers compute lines from `git diff <previous_revision>..<revision>` — payload carries section names only)

## Tone

Direct, thoughtful, slightly provocative. A sharp thinking partner, not a facilitator reading from a script.

## References

- [frameworks.md](references/frameworks.md) — SCAMPER, HMW, First Principles, JTBD, Constraint-Based, Pre-mortem, Analogous Inspiration.
- [refinement-criteria.md](references/refinement-criteria.md) — evaluation rubric for Phase 2.
- [examples.md](references/examples.md) — sample ideation sessions.
- [philosophy.md](../shared/philosophy.md) — shared tenets.
- [path-conventions.md](../shared/path-conventions.md) — `spec/` vs `docs/` rules.
- [publication-policy.md](../shared/publication-policy.md) — checkpoint resolution, manifest safety, first-run preference prompt, and publication disclosure.
- [specscore-lint-rules.md](../shared/specscore-lint-rules.md) — lint contract this skill assumes.
- [events.md](../shared/events.md) — event payloads emitted by this skill.
- [question-cadence.md](../shared/question-cadence.md) — when to batch vs single-question.
- `specscore idea new <slug>` — CLI scaffolder used in Phase 3 (see `internal/cli/new.go` in the specscore repo for the authoritative flag list).
