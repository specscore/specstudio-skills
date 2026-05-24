# `idea-refine` vs. `brainstorming`: Deep Analysis and a SpecScore/Synchestra-Native Merge

**Status:** Partially superseded — see status note below
**Date:** 2026-04-14 (original); status note added 2026-05-08
**Author:** Alexander Trakhimenok (with Claude)
**Audience:** Internal / self
**Scope reviewed:**
- [`addyosmani/agent-skills/skills/idea-refine`](https://github.com/addyosmani/agent-skills/tree/main/skills/idea-refine) — `SKILL.md`, `frameworks.md`, `refinement-criteria.md`, `examples.md`, `scripts/idea-refine.sh`
- [`obra/superpowers/skills/brainstorming`](https://github.com/obra/superpowers/tree/main/skills/brainstorming) — `SKILL.md`, `spec-document-reviewer-prompt.md`, `visual-companion.md`, `scripts/` (server.cjs, start/stop-server.sh, frame-template.html, helper.js)

## Status Note (added 2026-05-08)

The strategic argument (two skills not one, divergent vs. gating roles, lifecycle alignment) **stands**. The concrete artifact schemas in §7.1 and §8 (YAML front-matter with `type:` / `id:` / `promotes_to:` fields) **do not** — they pre-date the discovery that canonical SpecScore explicitly rejects YAML front-matter and uses bold-prefixed body metadata (`**Status:**`, `**Date:**`, `**Owner:**`, `**Promotes To:**`, …) with title-prefix dispatch (`# Idea: …`, `# Feature: …`).

Treat any YAML schema example below as historical record of the original proposal. The authoritative current schema is the canonical SpecScore spec at [`specscore/specscore`](https://github.com/specscore/specscore), reflected in `skills/ideate/SKILL.md`, `skills/specify/SKILL.md`, and `skills/shared/specscore-lint-rules.md`.

---

## 1. TL;DR

Both skills solve the "don't just start coding" problem, but they target **different points on the idea-to-spec pipeline**:

- **`idea-refine`** is a **creative convergence engine**. It assumes you have a *vague idea* and turns it into a *ranked one-pager* with a Not-Doing list. Its output is a pre-spec artifact (`docs/ideas/<name>.md`).
- **`brainstorming`** is a **pre-implementation gatekeeper**. It assumes you have an *approved intent to build* and turns it into a *reviewed design document* (`docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md`) that feeds directly into `writing-plans`.

They are **not competitors — they are sequential stages**. `idea-refine` ends where `brainstorming` begins. A clean pipeline would be:

```
raw idea ──► idea-refine ──► approved one-pager ──► brainstorming ──► design spec ──► writing-plans
```

For SpecScore/Synchestra, the right move is **neither fork nor merge-into-one**. Build a **two-skill pair** with a shared SpecScore spine:

1. `specstudio:ideate` — divergent/convergent ideation → produces a **SpecScore Idea** artifact (pre-feature).
2. `specstudio:specify` — intent-to-design with hard gates and self-review → produces a **SpecScore Feature** artifact (feature with requirements + acceptance criteria).

Both emit Synchestra-addressable artifacts (YAML front-matter, `specscore:` annotations, machine-readable status).

---

## 2. Head-to-Head Comparison

### 2.1 Purpose & Stage in the Workflow

|   | `idea-refine` | `brainstorming` |
|---|---|---|
| **Question it answers** | "Is this worth building, and what should we actually build?" | "What exactly are we building and how?" |
| **Position in pipeline** | Pre-spec ideation | Pre-implementation design |
| **Entry state** | Raw, possibly vague concept | Intent to build something (possibly post-`idea-refine`) |
| **Exit state** | Ranked direction + Not-Doing list | Approved design doc ready for planning |
| **Terminal handoff** | User decides whether to save the one-pager | **Hard gate:** must invoke `writing-plans`, nothing else |

### 2.2 Process Shape

|   | `idea-refine` | `brainstorming` |
|---|---|---|
| **Phases** | 3 explicit (Understand→Expand, Evaluate→Converge, Sharpen→Ship) | Single flow with 9-item checklist (explore → visual? → ask → propose → present → write → self-review → user-review → transition) |
| **Divergence step** | Required — generate 5–8 variations via lenses (inversion, constraint removal, audience shift, combination, simplification, 10x, expert) | Optional / implicit — "propose 2–3 approaches with trade-offs" |
| **Convergence step** | Cluster into 2–3 directions, stress-test (user value / feasibility / differentiation), surface assumptions | User approves design section-by-section |
| **Interaction style** | Can front-load many sharpening questions; "3–5 sharpening questions" in a batch | **Strict one-question-at-a-time**, multiple-choice preferred |
| **Framework library** | Yes — separate `frameworks.md` (SCAMPER, HMW, First Principles, JTBD, Constraint-Based, Pre-mortem, Analogous) | No explicit library; relies on YAGNI + isolation/clarity principles |
| **Evaluation rubric** | Yes — separate `refinement-criteria.md` (painkiller vs vitamin, differentiation tiers, assumption audit, decision matrix, MVP scoping) | No formal rubric; uses design-quality principles (boundaries, interfaces, testability) |
| **Scope-decomposition logic** | Not addressed | Explicit — if the request spans multiple subsystems, stop and decompose before questioning |

### 2.3 Output Artifacts

**`idea-refine` one-pager schema:**

```
# [Idea Name]
## Problem Statement      (single "How Might We…" sentence)
## Recommended Direction  (2–3 paragraphs)
## Key Assumptions to Validate  (checkbox list w/ test method)
## MVP Scope              (in / out)
## Not Doing (and Why)    (explicit trade-offs)
## Open Questions
```

Saved to `docs/ideas/<name>.md` *after user confirmation* (optional, not forced).

**`brainstorming` design doc:**

- Saved to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` (mandatory, committed to git).
- No fixed template — sections scale to complexity.
- Expected to cover: architecture, components, data flow, error handling, testing.
- Undergoes a **spec self-review** (placeholder scan, internal consistency, scope check, ambiguity check) — see `spec-document-reviewer-prompt.md` for the reviewer subagent template.

### 2.4 Guardrails and Enforcement

|   | `idea-refine` | `brainstorming` |
|---|---|---|
| **Hard gates** | Soft — "do not proceed until you understand *who* and *success*" | **Explicit** — `<HARD-GATE>` blocks implementation until design approved |
| **"This is too simple" defense** | Implicit (keep it to 5–8 variations) | **Explicit section** — "every project goes through this, including a single-function utility" |
| **Anti-patterns documented** | Yes, 7 listed (yes-machining, 20+ ideas, skipping user, etc.) | Yes, scattered across Key Principles + HARD-GATE |
| **Verification checklist** | 7-item post-session checklist | Implicit (self-review + user-review gates) |
| **Subagent delegation** | None | Yes — ships with `spec-document-reviewer-prompt.md` for dispatching a reviewer subagent |

### 2.5 Tooling Footprint

|   | `idea-refine` | `brainstorming` |
|---|---|---|
| **Scripts** | 1-liner bash (creates `docs/ideas/`) | Full Node.js **visual companion server** (`server.cjs`, start/stop scripts, HTML frame template, helper.js) |
| **Persistence** | Just a markdown file | `.superpowers/brainstorm/<session-id>/` session directories with content + event state |
| **Browser integration** | None | Opt-in browser-based visual tool for mockups/diagrams/side-by-sides |
| **External deps** | None | Node.js runtime for the companion |

### 2.6 Tone & Voice

- **`idea-refine`:** "Direct, thoughtful, slightly provocative… sharp thinking partner, not a facilitator reading from a script." Explicitly anti-sycophant. Quotes Jobs-flavored philosophy ("simplicity is the ultimate sophistication", "say no to 1,000 things", "show people the future — don't just give them better horses").
- **`brainstorming`:** "Collaborative dialogue" — more facilitative. Tone leans structured / Socratic. Lets the user drive pace via one-question-at-a-time.

---

## 3. Where They Overlap

- Both insist on **understanding before building**.
- Both advocate **multiple alternatives** before committing (idea-refine: 5–8; brainstorming: 2–3).
- Both call out **YAGNI / simplicity / scope discipline**.
- Both want a **written artifact** at the end (though only brainstorming mandates it).
- Both are hostile to **yes-machining** (`idea-refine` names it directly; `brainstorming` implicitly by requiring challenges to "too-simple" framings).
- Both surface **trade-offs** explicitly rather than hiding them.

## 4. Where They Genuinely Diverge

1. **Creativity vs. discipline center of gravity.** `idea-refine` is weighted ~70% creative expansion, ~30% convergence. `brainstorming` is ~20% exploration, ~80% gate-keeping and spec authoring.
2. **Question cadence.** `idea-refine` will batch 3–5 sharpening questions. `brainstorming` hard-enforces one at a time. These reflect different user experiences — batch is faster for experienced users, one-at-a-time is safer for unclear intent.
3. **Artifact type and binding.** `idea-refine` produces an optional "pre-spec" that doesn't commit you to anything. `brainstorming` produces a *committed design spec* that gates implementation and plugs into a larger Superpowers pipeline (writing-plans → executing-plans).
4. **Attitude toward implementation.** `idea-refine` is silent on implementation handoff — you can do whatever you want with the one-pager. `brainstorming` terminates in a specific skill invocation (`writing-plans`), refusing others.
5. **Visual affordances.** Only `brainstorming` ships a browser companion — for UX-heavy design work this is a real advantage; for pure idea refinement it's overkill.
6. **Framework surface area.** `idea-refine` separates creative frameworks (`frameworks.md`) and evaluation rubric (`refinement-criteria.md`) from the SKILL for clean composition. `brainstorming` embeds everything inline.

---

## 5. Which One Is "Better"?

**Neither is better in isolation — they excel at different jobs.** Judging them head-to-head is a category error. But for specific axes:

| Axis | Winner | Why |
|---|---|---|
| Opening a closed thought space | **`idea-refine`** | Lens-based variation generation; explicit divergence phase; named frameworks |
| Preventing premature implementation | **`brainstorming`** | Literal `<HARD-GATE>`; terminal-state enforcement; "too simple" defense |
| Producing a durable artifact | **`brainstorming`** | Mandatory commit; self-review; user-review; subagent-reviewable |
| Evaluating ideas rigorously | **`idea-refine`** | Painkiller-vs-vitamin, differentiation tiers, decision matrix in `refinement-criteria.md` |
| Driving a clear implementation handoff | **`brainstorming`** | Explicit `writing-plans` terminal invocation |
| Visual/UX-heavy design work | **`brainstorming`** | Ships the visual companion |
| Ideating *before* committing to build | **`idea-refine`** | Low-friction, not tied to implementation pipeline |
| Scope decomposition of too-large requests | **`brainstorming`** | Explicit: stop and decompose before refining |

### The Honest Take

- If I could pick only one, I'd pick **`brainstorming`** — because the cost of skipping it (building the wrong thing) is higher than the cost of skipping `idea-refine` (building the right thing a bit less creatively).
- But running them **sequentially** is the highest-EV approach. `idea-refine` expands the possibility space and kills weak bets cheaply; `brainstorming` locks in the winner and enforces the design rigor needed to hand off to implementation.
- For SpecScore's purposes, I want **both** — and I want their outputs to be **typed SpecScore artifacts**, not freeform markdown.

---

## 6. Can They Be Merged? Should They Be?

### Two Plausible Paths

**Path A — Single fat skill** (`specstudio:ideate-and-specify`).
- Pros: One invocation; one artifact; less context-switching.
- Cons: Context pollution (creative divergence in the same conversation as spec self-review); confusing mental model; violates single-responsibility; gates become ambiguous (when does the `<HARD-GATE>` apply?).

**Path B — Two composable skills with a shared backbone** ← *recommended*.
- `specstudio:ideate` produces a SpecScore **Idea** artifact.
- `specstudio:specify` consumes an Idea (or a raw intent) and produces a SpecScore **Feature** artifact.
- Each skill has one job, matches one stage of the SpecScore hierarchy, and is independently invocable.
- Synchestra orchestrates the handoff: when an Idea is promoted, it triggers `specstudio:specify`.

I recommend **Path B**. The two skills solve genuinely different problems — conflating them loses the disciplined gating of `brainstorming` and the creative pressure of `idea-refine`. Keep them separate, but give them a shared spec vocabulary and shared philosophy section.

### What to Use as the Base

| Skill | Base | Grafts From the Other |
|---|---|---|
| `specstudio:ideate` | **`idea-refine`** (3-phase structure, frameworks/criteria as sidecar files, tone) | From `brainstorming`: scope-decomposition gate, mandatory artifact write (not optional), spec self-review pass on the Idea doc, `<HARD-GATE>` against jumping to design without an approved Idea |
| `specstudio:specify` | **`brainstorming`** (checklist flow, HARD-GATE, self-review, reviewer subagent prompt, visual companion) | From `idea-refine`: named divergence lenses for the 2–3 approaches step, painkiller-vs-vitamin test in the design review, explicit assumption audit section, "Not Doing" as a required design section |

---

## 7. Tailoring to SpecScore + Synchestra

### 7.1 SpecScore Alignment

The fundamental shift vs. both upstream skills: **outputs are not freeform markdown; they are SpecScore artifacts with schemas**, and they live in `spec/` — not `docs/`.

#### Path Conventions: `spec/` vs. `docs/`

This split is a core SpecScore convention and one of the deliberate overrides of the upstream skills:

| Tree | Purpose | Audience | Format | Tooling |
|---|---|---|---|---|
| `spec/` | Machine-addressable SpecScore artifacts (ideas, features, requirements, acceptance criteria, plans) | Tooling (`specscore lint`, Rehearse, Synchestra agents) and contributors editing specs | Typed: YAML front-matter + structured body sections | Lintable, schema-validated, stable IDs, bidirectional links |
| `docs/` | User-facing prose | Human readers of the project (users, adopters, reviewers) | Freeform markdown | Rendered as-is; no schema constraints |

**Consequences for the skills:**

- Ideas go in **`spec/ideas/<slug>.md`** (not `docs/ideas/` as upstream `idea-refine` does). Lintable. Promotable to Features inside the same tree.
- Features go in **`spec/features/<slug>/README.md`** with `spec/features/<slug>/requirements/*.md` (not `docs/superpowers/specs/YYYY-MM-DD-*.md` as upstream `brainstorming` does). Date goes in front-matter, not the filename.
- Plans go in **`spec/plans/<slug>/`** — same pattern.
- `docs/` stays clean for guides, tutorials, READMEs, ADRs written for humans.
- Promotion links (`promotes_to`, `supersedes`, `depends_on`) resolve as relative paths within `spec/`, so a single tree move or rename doesn't cross a boundary.

**Overrides from upstream skills:**

| Upstream convention | SpecScore override | Why |
|---|---|---|
| `idea-refine` → `docs/ideas/<name>.md` (optional save) | `spec/ideas/<slug>.md` (mandatory, lintable) | Ideas are typed artifacts, not prose; unsaved ideation is waste |
| `brainstorming` → `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` | `spec/features/<slug>/README.md` | Dates belong in front-matter; slugs are stable IDs; Features have sub-artifacts (requirements) |
| Either skill producing freeform markdown | YAML front-matter + schema-validated body | Required for `specscore lint` + Synchestra addressability |

This convention should be codified in a shared spine file (`skills/shared/path-conventions.md`) that both `specstudio:ideate` and `specstudio:specify` reference, so future SDD skills don't drift.

| Artifact | Location | Schema |
|---|---|---|
| Idea | `spec/ideas/<slug>.md` | YAML front-matter (`type: idea`, `status`, `owner`, `date`, `supersedes`, `promotes_to`) + structured body sections |
| Feature (design) | `spec/features/<slug>/README.md` + `spec/features/<slug>/requirements/*.md` | SpecScore feature schema — requirements, acceptance criteria (Given/When/Then scenarios), architecture notes |

Both artifacts are **lintable by `specscore lint`**, which means:
- Placeholders (`TBD`, `TODO`) fail lint (enforcing `brainstorming`'s placeholder-scan via tooling, not vibes).
- Missing "Not Doing" list fails lint for Ideas.
- Missing acceptance criteria fails lint for Features.
- Broken promotion links (Idea says `promotes_to: X` but X doesn't exist) fail lint.

### 7.2 Synchestra Alignment

- **Addressable artifacts:** Every Idea and Feature has a stable ID. Synchestra agents reference them by ID in plans, tasks, and PRs.
- **Status lifecycle:** `Draft → Under Review → Approved → Promoted → Archived` (Ideas); `Draft → Approved → In Progress → Shipped` (Features). Synchestra owns status transitions.
- **Orchestration hooks:** `specstudio:ideate` publishes an `idea.drafted` event; `specstudio:specify` can be triggered by `idea.approved`. Both emit `artifact.written` events for Rehearse to pick up.
- **Traceability:** `specscore:` source annotations link code back to requirements back to features back to the originating Idea. A PR touching a feature that traces to an Archived Idea fails CI.
- **Parallelism:** Because skills are independent, Synchestra can dispatch multiple `specstudio:ideate` sessions in parallel on different raw ideas (natural fit with Superpowers' `dispatching-parallel-agents`).

### 7.3 Things to Deliberately Drop from the Upstream Skills

- **Visual companion** (from `brainstorming`): keep as optional reference, don't ship by default. Most SDD work is textual; the Node server is a heavy dep. Opt-in via a separate `specstudio:specify-visual` companion skill.
- **`docs/superpowers/specs/YYYY-MM-DD-topic-design.md` naming** (from `brainstorming`): replace with SpecScore's `spec/features/<slug>/` layout. Dates go in front-matter, not filenames.
- **"Save optional" behavior** (from `idea-refine`): make the Idea artifact mandatory. Unsaved ideation is waste.

---

## 8. Proposed SKILL.md Outline for the Merged Pair

### 8.1 `skills/ideate/SKILL.md` (Draft Outline — skill name `specstudio:ideate`)

```markdown
---
name: ideate
description: |
  Refines raw ideas into SpecScore Idea artifacts through divergent and
  convergent thinking. Produces a lintable pre-spec one-pager that can be
  promoted to a SpecScore Feature. Use when the user has a vague concept
  and isn't ready for design yet. Trigger: "ideate", "refine this idea",
  "stress-test this".
---

# SDD Ideate

Turn raw ideas into sharp, SpecScore-compatible Idea artifacts through
structured divergent and convergent thinking.

## Hard Gate
<HARD-GATE>
Do NOT invoke specstudio:specify, writing-plans, or any implementation skill until
an Idea artifact has been written to spec/ideas/<slug>.md, passes
`specscore lint`, and the user has approved its Recommended Direction.
Ideas that can't be lint-clean aren't ready to be designed.
</HARD-GATE>

## When to Use
- Raw, vague, or unvalidated concept.
- User unsure whether the idea is worth building.
- Multiple possible directions, none yet chosen.
- Skip if user already has an approved Idea or a clear feature to design.

## Pre-Flight: Scope Decomposition
If the idea describes multiple independent subsystems, stop and help the
user decompose first. Each subsystem gets its own Idea. (Borrowed from
brainstorming.)

## Checklist
You MUST create a task for each of these and complete them in order:

1. Explore project context (existing features, architecture, prior ideas)
2. Confirm scope is single-idea (else decompose)
3. Phase 1 — Understand & Expand
4. Phase 2 — Evaluate & Converge
5. Phase 3 — Crystallize as a SpecScore Idea
6. Lint the artifact (`specscore lint spec/ideas/<slug>.md`)
7. Inline self-review (placeholders, contradictions, ambiguity)
8. Ask user to review and approve
9. Emit `idea.drafted` / `idea.approved` event for Synchestra

## Phase 1 — Understand & Expand (Divergent)
- Restate as a "How Might We…" statement.
- Ask 3–5 sharpening questions (AskUserQuestion). Focus: who, success,
  constraints, prior art, why now.
- Generate 5–8 variations using lenses (see references/frameworks.md):
  Inversion, Constraint Removal, Audience Shift, Combination,
  Simplification, 10x, Expert Lens.
- Ground variations in the codebase when one exists (Glob, Grep, Read).

## Phase 2 — Evaluate & Converge
- Cluster into 2–3 meaningfully different directions.
- Stress-test on User Value, Feasibility, Differentiation
  (see references/refinement-criteria.md).
- Surface hidden assumptions (Must / Should / Might be true).
- Be honest, not supportive. Weak ideas die here.

## Phase 3 — Crystallize as a SpecScore Idea
Write to `spec/ideas/<slug>.md` with this exact schema:

    ---
    type: idea
    id: idea-<slug>
    status: Draft
    date: YYYY-MM-DD
    owner: <author>
    promotes_to: []          # filled when promoted to Feature(s)
    supersedes: []
    ---

    # <Idea Name>

    ## Problem Statement
    <One HMW sentence>

    ## Context
    <Triggering observation, related specs, prior art>

    ## Recommended Direction
    <2–3 paragraphs — what and why over alternatives>

    ## Alternatives Considered
    <2–3 directions that lost, and why>

    ## MVP Scope
    <Single job the MVP nails, timeboxed>

    ## Not Doing (and Why)          # REQUIRED — lint fails without this
    - …

    ## Key Assumptions to Validate
    | Tier | Assumption | Validation |
    |------|------------|-----------|
    | Must-be-true | … | … |

    ## SpecScore Integration
    - New Features this would create: …
    - Existing Features affected: …
    - Dependencies on other Ideas or in-flight work: …

    ## Open Questions
    - …

## Self-Review (inline, no subagent)
Placeholder scan / internal consistency / scope / ambiguity. Fix, don't
re-review.

## Verification
- [ ] Artifact exists at spec/ideas/<slug>.md
- [ ] `specscore lint` passes
- [ ] HMW, target user, success criteria all explicit
- [ ] ≥2 alternatives stress-tested
- [ ] Assumptions audited (Must/Should/Might)
- [ ] Not-Doing list non-empty
- [ ] User approved the Recommended Direction

## Red Flags
- 20+ shallow variations vs 5–8 considered ones
- No named target user
- No assumptions surfaced
- "Not Doing" section skipped
- Saving without user approval
- Attempting to skip ahead to specstudio:specify without lint-clean artifact

## References
- references/frameworks.md       (SCAMPER, HMW, First Principles, JTBD,
                                   Constraint-Based, Pre-mortem, Analogous)
- references/refinement-criteria.md
- references/examples.md
- references/events.md  (which events this skill emits)
```

### 8.2 `skills/specify/SKILL.md` (Draft Outline — skill name `specstudio:specify`)

```markdown
---
name: specify
description: |
  Turns an approved SpecScore Idea (or a clear intent) into a SpecScore
  Feature with requirements and acceptance criteria. Gates implementation
  — no code, no plans, no scaffolding until the Feature is lint-clean and
  user-approved. Trigger: "design this", "spec this out", or Synchestra
  event `idea.approved`.
---

# SDD Design

Turn approved intent into a lintable, testable SpecScore Feature.

## Hard Gate
<HARD-GATE>
Do NOT invoke writing-plans, frontend-design, mcp-builder, or ANY
implementation skill until:
  1. The Feature artifact exists at spec/features/<slug>/README.md.
  2. Each requirement has ≥1 acceptance criterion with Given/When/Then
     scenarios.
  3. `specscore lint` passes.
  4. The user has explicitly approved the written Feature.

This applies to EVERY project, regardless of perceived simplicity. The
only skill invoked after specstudio:specify is writing-plans.
</HARD-GATE>

## When to Use
- A SpecScore Idea is approved and ready to become a Feature.
- User has a clear buildable intent (may skip specstudio:ideate if truly clear).
- Changing behavior of an existing Feature (creates an updated revision).

## Anti-Pattern: "This Is Too Simple To Need A Design"
Every Feature goes through this. A toggle, a one-line config, a single
utility — all of them. Simple features are where unexamined assumptions
cost the most. Design can be short; it cannot be skipped.

## Pre-Flight: Inputs
- If triggered from an approved Idea, load it and list Idea assumptions
  that the Feature must validate.
- If no Idea exists, ask: "Is this ready to design or should we ideate
  first?" (Don't force specstudio:ideate if the user has high conviction.)

## Checklist
1. Explore project context (files, recent commits, related Features)
2. Scope decomposition check (multiple subsystems → multiple Features)
3. Offer visual companion if visual questions are ahead (own message)
4. Ask clarifying questions — one at a time, multiple-choice preferred
5. Propose 2–3 approaches using specstudio:ideate lenses where useful
6. Present design sections, get approval after each
7. Author the Feature artifact (README.md + requirements/)
8. Run lint + inline self-review
9. Dispatch spec-document-reviewer subagent
10. User reviews the written Feature — wait for approval
11. Emit `feature.specified` event for Synchestra
12. Transition to writing-plans

## Design Sections (scale to complexity)
- Purpose & user job (from Idea's HMW if applicable)
- Requirements (numbered, each with ≥1 acceptance criterion)
- Architecture & components (isolation, interfaces, dependencies)
- Data flow
- Error handling & failure modes
- Testing strategy (ties to Rehearse)
- Not Doing / Out of Scope  (inherited from Idea + design-level cuts)
- Assumption carryover (which Idea assumptions survive to this Feature)

## Acceptance Criterion Format
Each AC is a scenario:

    Scenario: <name>
    Given <precondition>
    When <action>
    Then <observable outcome>

## Self-Review Pass (inline)
Placeholders / consistency / scope / ambiguity — fix in place.

## Subagent Review (required)
Dispatch the spec-document-reviewer subagent using
references/reviewer-prompt.md. Status must be "Approved" before user
review. If "Issues Found", fix and re-dispatch.

## Visual Companion (optional)
Same logic as Superpowers brainstorming — per-question decision. Offered
as its own message. See references/visual-companion.md.

## Verification
- [ ] spec/features/<slug>/README.md exists
- [ ] Every requirement has ≥1 AC
- [ ] Every AC is Given/When/Then
- [ ] `specscore lint` passes
- [ ] Reviewer subagent returned "Approved"
- [ ] User explicitly approved
- [ ] Source Idea (if any) is linked bidirectionally

## Red Flags
- Proceeding to writing-plans without approval
- Requirements without ACs
- ACs that aren't Given/When/Then
- "Too simple to spec" rationalization
- Scope covering multiple subsystems
- Assumptions from the Idea silently dropped

## References
- references/reviewer-prompt.md    (subagent template)
- references/visual-companion.md   (optional mockup server)
- references/events.md
- references/specscore-feature-schema.md
```

### 8.3 Shared Spine (Reused Across Both Skills)

Co-located so both skills stay DRY:

```
skills/
├── ideate/          ← dir name uses `-`; skill name is `specstudio:ideate`
│   ├── SKILL.md
│   └── references/
│       ├── frameworks.md              ← from idea-refine
│       ├── refinement-criteria.md     ← from idea-refine
│       ├── examples.md                ← adapted from idea-refine
│       └── events.md       ← NEW
├── specify/          ← dir name uses `-`; skill name is `specstudio:specify`
│   ├── SKILL.md
│   └── references/
│       ├── reviewer-prompt.md         ← from brainstorming
│       ├── visual-companion.md        ← from brainstorming (optional)
│       ├── events.md       ← NEW
│       └── specscore-feature-schema.md← NEW (canonical schema)
└── shared/
    ├── philosophy.md                  ← Jobs-flavored + simplicity tenets
    ├── question-cadence.md            ← when to batch vs one-at-a-time
    ├── path-conventions.md            ← spec/ vs docs/ rules (see §7.1)
    └── specscore-lint-rules.md        ← lint rules both skills assume
```

---

## 9. Key Design Decisions (and Why)

| Decision | Choice | Rationale |
|---|---|---|
| Single skill vs. pair | **Pair** | Different jobs, different gates, different artifacts. Conflating loses discipline. |
| Mandatory vs. optional artifact | **Mandatory** (both skills) | Unsaved ideation is waste; Synchestra needs addressable artifacts. |
| Freeform markdown vs. schema'd | **Schema'd with YAML front-matter** | SpecScore is a spec format, not a style guide. Lint > vibes. |
| Visual companion | **Optional, in specstudio:specify only** | Heavy dep; not useful for pure ideation; opt-in when UX matters. |
| Question cadence | **Contextual** | Batch in Phase 1 (user has context); one-at-a-time in Phase 2+ design. Document in `shared/question-cadence.md`. |
| "Too simple" defense | **In specstudio:specify only** | specstudio:ideate is explicitly for low-commitment exploration; gating it would kill low-stakes ideation. |
| Reviewer subagent | **specstudio:specify only** | The cost of a bad Feature spec is high; the cost of a bad Idea is low. |
| Handoff terminal | **specstudio:ideate → (optional) specstudio:specify → writing-plans** | Mirrors Superpowers' terminal-invocation model. |

---

## 10. Resolved Questions and Decisions

1. **Idea promotion mechanics — RESOLVED.** An Idea may have empty `promotes_to` in `Draft` and `Approved` states. When one or more Features are created from an Idea, Lifecycle tooling transitions its status to `Specified` and **auto-populates `promotes_to`** with the list of Feature IDs. The Idea author never edits `promotes_to` manually. Lifecycle: `Draft → Under Review → Approved → Specified → Archived`.

2. **Revision semantics — RESOLVED.** Features **revise in place**; git history is the record of design changes. A revision does not create a new slug or file. `supersedes:` is reserved for the rare case where a Feature is replaced wholesale by a differently-scoped successor (e.g., the original Feature is archived). Lint should tolerate evolution of a Feature's body across revisions as long as the schema remains valid.

3. **Lint granularity — RESOLVED.** `specscore lint` supports **single-file linting** (e.g., `specscore lint spec/ideas/<slug>.md`) in addition to whole-tree. The skills invoke single-file lint on the artifact they just wrote. No separate Idea-only lint subset is needed — the lint rule engine dispatches by artifact `type:` in the front-matter.

4. **Event schema — RESOLVED.** Central file: `shared/events.md`. Emitted by the skills: `idea.drafted`, `idea.approved`, `idea.specified`, `feature.specified`, `feature.approved`. Payload shapes defined in that file.

5. **Parallel / competing Ideas — OUT OF SCOPE.** For now, users manage competing-hypothesis scenarios manually (archive the loser, keep the winner). Revisit if/when it becomes a pattern.

6. **Visual companion — STILL OPEN.** Deferred to a dedicated session. See Open Question section below for clarifications on what "shipped node server" actually means in upstream `brainstorming` and what the SpecScore alternatives could be.

7. **Rehearse integration — DECIDED (partial).** Rehearse test generation is **optional**. The skill (`specify`) should attempt a lightweight heuristic decision: does this Feature have clear observable Given/When/Then scenarios that map to automatable tests (API, CLI, pure function, UI with stable selectors)? If yes, scaffold stub test files (one per AC) and mark them `pending`. If the Feature is abstract, documentation-only, or has untestable outcomes, skip scaffolding and note "No Rehearse stubs generated — reason: …" in the Feature artifact. User can always override the heuristic.

8. **Naming — DECIDED.** Primary namespaced form: **`specstudio:ideate`** and **`specstudio:specify`**. Short slash-command aliases: `/ideate` and `/specify`. (Not `/brainstorm` — we're explicitly distancing from the Superpowers name since our `specstudio:specify` plays both ideation-convergence *and* design-gating roles differently from Superpowers `brainstorming`.)

---

## 11. Still-Open Questions

### 11.1 Visual Companion — full context

**Clarification on "shipped node server":** Upstream `obra/superpowers/skills/brainstorming` does **not** bundle a Node.js runtime binary. It ships:

- `scripts/server.cjs` — a CommonJS Node.js script (an HTTP server that watches a content directory and streams the newest HTML fragment to the user's browser, plus records click/selection events to a state dir).
- `scripts/start-server.sh` / `stop-server.sh` — bash wrappers that launch/terminate the `.cjs` script via the user's locally installed `node` binary (`#!/usr/bin/env bash` + `node server.cjs …`).
- `scripts/frame-template.html` — the page shell (header, CSS, selection indicator, helper script injection).
- `scripts/helper.js` — client-side helper loaded into the user's browser.

In other words: **the user must have Node.js installed on their machine**. The skill doesn't ship a runtime — it assumes one. Session files land in `<project>/.superpowers/brainstorm/<session-id>/{content,state}/` when `--project-dir` is passed.

**Open sub-questions for a separate session:**

1. Is a local HTTP server + browser the right UX for SpecScore design sessions, or would simpler alternatives cover the same ground? Candidates:
   - **Mermaid / PlantUML embedded in the Feature markdown** — rendered by the user's IDE or a viewer; zero runtime.
   - **SVG fragments committed alongside the Feature** — static artifacts, reviewable in PRs.
   - **Claude Code / Cursor inline image rendering** — platform-native visual return.
   - **Optional companion (keep upstream as-is, opt-in)** — accept the Node dependency only when the user enables it.
   - **Reuse upstream `obra/superpowers` visual companion directly** — if the user has `superpowers` installed alongside our skills, `specstudio:specify` can simply invoke the upstream `brainstorming` skill's visual-companion workflow (or delegate to it) rather than forking/vendoring. Requires: (a) confirming the upstream companion can be triggered standalone without the full `brainstorming` flow, (b) agreeing a contract for where session files land (their `.superpowers/brainstorm/` vs our preferred location), (c) deciding how `specstudio:specify` detects upstream availability (file existence check, skill-registry query). Biggest win: zero maintenance burden on our side. Biggest risk: coupling our skills to an external skill pack's lifecycle.
2. If we do adopt something server-based, does it need to integrate with Synchestra (so remote agents can render visuals for a watching user)?
3. How do visual mockups relate to SpecScore lintable artifacts? (Probably not lintable, live alongside as `spec/features/<slug>/assets/*`.)
4. Licensing — can we vendor `server.cjs` / `frame-template.html` from `obra/superpowers` under their license, or do we need a clean-room rewrite?

### 11.2 Other Open

- **Rehearse heuristic detail.** What exactly triggers the "testable AC" heuristic? (Candidate signals: AC mentions a CLI command, an HTTP endpoint, a pure-function input/output, or a UI element with a stable ID.) Needs a written rubric in `shared/rehearse-heuristic.md` before `specify` can implement it reliably.
- **`supersedes` vs in-place revision.** Draw the line in lint: when does a change force a new slug (supersedes) vs. stay in place? Probably a rule like "scope change that invalidates existing ACs → new slug; everything else → in place."

---

## 12. Action Items

| P | Action | Status |
|---|---|---|
| P0 | Write `shared/specscore-lint-rules.md` with rules both skills assume | ☐ |
| P0 | Write `shared/path-conventions.md` codifying `spec/` vs `docs/` split | ☐ |
| P0 | Write `shared/events.md` payload specs | ☐ |
| P0 | Write `shared/philosophy.md` and `shared/question-cadence.md` | ☐ |
| P1 | Draft `skills/ideate/SKILL.md` per outline in §8.1 | ☐ |
| P1 | Draft `skills/specify/SKILL.md` per outline in §8.2 | ☐ |
| P1 | Port `frameworks.md` and `refinement-criteria.md` from upstream `idea-refine` (MIT/permissive license permitting) | ☐ |
| P2 | Port `reviewer-prompt.md` from upstream `brainstorming` and adapt to SpecScore Feature schema | ☐ |
| P2 | Write `shared/rehearse-heuristic.md` (Q7 implementation rubric) | ☐ |
| P3 | Visual-companion decision session (Q6) | ☐ |
| P3 | Draw `supersedes` vs in-place-revision line for lint | ☐ |
