# Idea: Multi-Repo Destination Resolution for Idea-Creation Skills

**Status:** Specified
**Date:** 2026-05-20
**Owner:** alexandertrakhimenok
**Promotes To:** sidekick-capture/destination-resolution
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we prevent `specstudio:sidekick` from silently misfiling captured seeds in multi-repo SpecScore workspaces — by forcing the AI-agent host to deliberate about destination before writing, surfacing its pick inline for one-key human confirmation, and providing a cheap recovery path when the pick is wrong?

## Context

Sidekick capture is structurally different from deliberate Idea work. When a user invokes `specstudio:ideate` or `specstudio:specify`, they're in deliberation mode and can be expected to be conscious of which repo their work belongs in. Sidekick is fire-and-forget — the user is mid-flow on a different task, the host AI agent detects a sideline idea, and the skill writes the seed wherever cwd happens to be. Neither the user nor the agent stops to ask "wait, which repo?"

On 2026-05-19, the `artifact-frontmatter-convention` Idea was misfiled into `specscore/specstudio-skills` during an `ideate` session (commit `c4114cb`) and relocated by hand 2026-05-20 (~30 minutes of work: copy file, rewrite stale `specscore/*` org references to `specscore/*`, disambiguate "this repo" wording, commit on both sides with cross-linking messages; commits `7e32851` in `specscore/specscore`, `160ae03` in `specstudio-skills`). That specific misfile is treated here as an ideate-side user-awareness gap rather than a skill responsibility — but the relocate mechanics it surfaced are the same ones a sidekick misfile would need. This Idea uses that case as the model for the recovery skill, while scoping the *preventive* part (destination resolution) to sidekick where the structural cognitive gap lives.

`specstudio:sidekick` lives in `specscore/specstudio-skills`. Each SpecScore-managed sibling repo already carries a `specscore.yaml` declaring its identity (`project.title`, `org`, `repo`) — an unused signal that's a natural input for routing.

## Recommended Direction

`specstudio:sidekick` gains a pre-write **destination resolution** step that forces the host AI agent to deliberate, then surfaces the pick inline for human confirmation:

1. The skill detects multi-repo workspace by scanning `../` for sibling directories containing `specscore.yaml`. Each such sibling — plus cwd itself — becomes a candidate destination repo.
2. When ≥2 candidates exist, the skill blocks the write and presents the host agent with a **deliberation prompt**: the seed one-liner + body, the candidate list with each repo's `project.title` and top-level Feature directory names, and an explicit instruction to commit to a destination + one-line reason. The host agent already has full context about what it was just doing; the prompt forces it to use that context rather than defaulting to cwd. **Output is hard-constrained to a single line ≤120 characters** in the form `<repo>; <reason>` so it fits the inline confirmation UX.
3. The skill displays the agent's pick + reasoning as a single-line inline confirmation: `Routing to <repo> because <reason> — press enter to accept, type other to override.` The human is always the gate; the agent precomputes the default.
4. On enter, the skill routes to the agent's pick. On override (the user types another repo), the skill routes there instead. The destination is always visible to the human *before* write, so misroutes are structurally impossible to ship silently.
5. The skill writes the seed to the resolved repo's `spec/ideas/seeds/` with the standard sidekick success line (`Captured: <slug> at <path>`).

A companion **`specstudio:relocate-idea` skill** automates the manual relocate ritual. The skill in this repo is a **thin shell** around a new **`specscore idea relocate` CLI verb** in `specscore/specscore-cli` — the actual work lives in the CLI library where it's testable, reusable, and version-pinned. The CLI verb is responsible for:

- **Pre-flight clean-tree check.** Refuses to start if any affected repo (source, target, or any sibling with link updates queued) has uncommitted changes in the paths the verb would touch. Aborts early with a clear `Repo <X> has uncommitted changes in <path> — clean up and retry` message.
- Copying the artifact file from source to target (handling both `spec/ideas/<slug>.md` Ideas and `spec/ideas/seeds/<slug>.md` seeds).
- Rewriting cross-repo references inside the artifact itself (`specscore/*` → `specscore/*`, "this repo" disambiguation).
- **Finding and updating every reference to the old path/slug across all sibling SpecScore-managed repos.** References live in: `spec/ideas/README.md` indexes; other Ideas via `**Related Ideas:**` / `**Supersedes:**`; Features via `**Source Ideas:**`; seeds via back-link sections.
- **Auto-committing per repo by default**, with cross-linking commit messages so the relocate is traceable in git history. `--no-commit` flag stages changes everywhere without committing, for users who want to review before pulling the trigger. On first-failure during commit phase, the verb stops and reports exactly which repos were committed and which weren't, leaving the user to manually roll back already-committed repos if desired.

When the user invokes the relocate skill (i.e., the host agent's pick was accepted but turned out wrong, or the user overrode and the override was still wrong), the skill appends a record to an opt-in, workspace-local learning log at `.synchestra/destination-resolution-log.jsonl`. The log captures enough context to tune the deliberation prompt over time. No telemetry leaves the user's machine.

## Alternatives Considered

- **Always-ask (no agent deliberation).** The skill prompts the human user with an unranked list before every sidekick capture whenever ≥2 candidate repos exist. Solves the silent-misfile bug fully with no new failure modes; ~1-day implementation. Rejected because (a) it doesn't leverage the host agent's existing context, and (b) the inline-confirmation UX from the recommended direction collapses the prompt-frequency cost to one keystroke anyway — so we get the same flow-preservation property without giving up the agent's pre-computed default.

- **Silent route when agent is confident, prompt when uncertain.** Earlier draft of this Idea. Rejected once we realized the inline-confirmation UX makes "silent" unnecessary — the human gate is one keystroke, not a derail. Removing the silent path also removes the entire "express uncertainty" mechanism question (sentinel token? confidence rating? semantic check?), which was the most brittle piece of the prior design.

- **Deterministic keyword-scoring function.** The skill scores each candidate by substring/token matching the Idea body against each repo's `project.title` + Feature directory names, then routes by threshold. Rejected: the host AI agent already has rich context (current task, the seed content it just produced, the candidate repos) and is a strictly better inference engine than a substring matcher. Keyword scoring would require replay-testing to tune a threshold and would still miss semantic routing the agent gets for free.

- **Apply destination resolution to `specstudio:ideate` too.** Rejected: when a user invokes deliberation-mode Idea skills, they're consciously starting Idea work and can reasonably be expected to know which repo their work belongs in. The 2026-05-19 `c4114cb` misfile is treated as a user-awareness case, not a skill responsibility. If demand emerges later, widening to ideate is a sibling Idea once the pattern is proven on sidekick.

- **Topic-glob registry in `specscore.yaml`.** Each repo declares explicit topic globs (e.g., `topics: [lint, cli, spec-text]`). Rejected for v1: requires every repo to be edited up-front and locks topic vocabulary; the deliberation-prompt approach doesn't need it because the agent infers from `project.title` + Feature names on the fly. Forward-compatible.

- **Workspace as a first-class SpecScore concept.** Introduce `specscore workspace init`, a workspace-level config, route everything (Idea routing, cross-repo search, lint-all, status reports) through it. Rejected as premature — only current cross-repo capability needed is "create artifact in the right place" and "relocate when wrong." Workspace concept is justified only when a second cross-repo use case appears.

## MVP Scope

A scoped delivery covering sidekick destination resolution plus the relocate safety net. **Two-repo coordinated change**: the CLI verb in `specscore-cli` lands first, then the skill changes in `specstudio-skills` consume it.

In `specscore/specscore-cli`:

1. New `specscore idea relocate <artifact-path> <target-repo>` CLI verb that does all of:
   - Pre-flight clean-tree check across source, target, and every sibling repo whose docs reference the original path. Aborts with a clear error if any has uncommitted changes in the touched paths.
   - Copies the artifact file to the target repo's matching directory.
   - Rewrites cross-repo references inside the artifact (`specscore/*` → `specscore/*`, "this repo" disambiguation, footer paths).
   - Finds every reference to the original slug/path across all sibling SpecScore-managed repos (markdown docs in `spec/**/*.md` only — code annotations deferred to v1.5).
   - Updates each found reference to point at the new path.
   - **Auto-commits per repo by default** (source: file removal + index update; target: file addition + index update; siblings: link updates). Cross-links commit SHAs in messages. Stop-and-report on first commit failure.
   - `--no-commit` flag: stage changes everywhere without committing. For paranoid review-first workflows.

In `specscore/specstudio-skills`:

2. New shared helper `skills/shared/destination-resolution.md` containing the **deliberation prompt template** that the sidekick skill presents to the host agent before writing. The prompt:
   - Includes the seed content + candidate repos with `project.title` and top-level Feature directory names.
   - Instructs the agent to commit to a destination with one-line reasoning.
   - **Enforces output format**: single line, ≤120 characters, exact shape `<repo>; <reason>`.
3. `specstudio:sidekick` invokes the helper before any write when ≥2 candidate repos exist. The agent's response is displayed as the inline confirmation: `Routing to <repo> because <reason> — press enter to accept, type other to override.` The user presses enter (route to agent's pick) or types an alternative repo name (route there).
4. Standard sidekick success line on completion (`Captured: <slug> at <path>`); the user already saw the destination at confirmation time.
5. `specstudio:relocate-idea` skill: thin shell that invokes `specscore idea relocate` and surfaces its output. Triggerable as `/relocate-idea`.
6. Opt-in mismatch logging: when relocate runs, append a JSON line to `.synchestra/destination-resolution-log.jsonl` in cwd. Schema deferred to Feature-spec time; minimum context to retrospect on agent decisions.

Out of scope for MVP:

- `specstudio:ideate`, `specstudio:specify`, `specstudio:plan`, task-creation skills (same multi-repo problem applies but separate Idea once the pattern is proven on sidekick).
- Code-annotation cleanup (`specscore:` comments and bare URLs in source code) — relocate v1.5 extension.
- Topic-glob registry in `specscore.yaml`.
- Workspace-level config artifacts.
- Atomic-rollback across repos on partial commit failure.
- Automated analysis of the mismatch log.

## Not Doing (and Why)

- Applying destination resolution to `specstudio:ideate` — deliberation-mode invocation; the user can and should be conscious of destination.
- A silent-when-confident routing path — collapsed to inline confirmation per the inline-UX choice; no escape-hatch mechanism needed.
- A deterministic keyword-scoring function — the host AI agent has richer context than a substring matcher.
- A workspace-level config file (`~/.specscore-workspace.yaml`) — siblings-of-cwd auto-detection is free and sufficient for v1.
- Putting the relocate logic in the skill markdown (instead of the CLI) — the skill is for discoverability; the CLI is where the work belongs.
- Atomic-rollback across repos — git has no distributed transactions; stop-and-report is the pragmatic v1.
- Telemetry that leaves the user's machine — the mismatch log is local-only.
- Code-annotation reference cleanup in the relocate verb's v1 — deferred to v1.5.

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | Sibling-dir scan reliably enumerates the candidate repos in a user's workspace | Confirm against the actual layout in `~/projects/specscore/` (10+ siblings, all with `specscore.yaml`); confirm POSIX `find ../ -maxdepth 2 -name specscore.yaml` works on macOS and Linux; ship a smoke test |
| Must-be-true | The host AI agent, when forced to deliberate with the prompt template, picks the right repo materially more often than the cwd-default does today | Replay currently-queued seeds + the `artifact-frontmatter-convention` case against a prototype prompt; measure agent's destination choice vs. ground truth; iterate prompt wording until accuracy is acceptable |
| Must-be-true | A 120-char single-line reason is enough for the agent to give a useful justification | Replay-test as above; if reasoning is consistently truncated or unhelpful, raise the limit incrementally and re-test |
| Should-be-true | The inline confirmation UX (`press enter to accept, type other to override`) is fast enough that users don't perceive it as a derail | Dogfood the implementation against a real session with high capture frequency (5+ seeds per session); measure perceived friction |
| Should-be-true | The `specscore idea relocate` CLI verb's link-cleanup across sibling repos is implementable in a reasonable scope | Prototype against today's `7e32851` + `160ae03` as reference output; verify it would have updated any cross-doc references (in that case, none beyond source/target indexes); measure complexity |
| Should-be-true | Pre-flight clean-tree check + stop-on-first-failure gives clean enough failure semantics in practice | Test with intentional pre-commit-hook failures in one of N affected repos; verify the user can recover with stated rollback commands |

## SpecScore Integration

- **New Features this would create:**
  - `cli/idea/relocate` (in `specscore/specscore-cli`) — the new CLI verb implementing relocation mechanics including pre-flight check, cross-sibling link cleanup, and auto-commit semantics
  - `skills/sidekick/destination-resolution` (in `specscore/specstudio-skills`) — the shared deliberation-prompt helper consumed by `specstudio:sidekick`
  - `specstudio:relocate-idea` skill (in `specscore/specstudio-skills`) — thin shell around the CLI verb for discoverability and slash-trigger
- **Existing Features affected:**
  - `specstudio:sidekick` (this repo) — gains pre-write deliberation step + inline confirmation UX
- **Dependencies:**
  - Cross-repo coordination with `specscore/specscore-cli` to ship the `idea relocate` verb. **CLI verb lands first** (likely behind an "experimental" flag); skill changes in this repo land after, depending on a pinned CLI version.

## Open Questions

- **Deliberation-prompt wording.** The exact prompt text (within the constraints already locked: includes candidates' `project.title` + Feature dir names, instructs single-line output `<repo>; <reason>` ≤120 chars) is a Feature-spec deliverable. Iterate via replay-testing against captured seeds.
- **Mismatch-log record fields.** Schema deferred to Feature-spec time; not load-bearing for MVP.
- **Cross-repo change sequencing.** The CLI verb in `specscore-cli` must land before the skill in this repo can rely on it. Recommended: ship CLI verb behind an experimental flag, ship skill changes here, then promote CLI verb to GA. Detail at implementation time.

---
*This document follows the https://specscore.md/idea-specification*
