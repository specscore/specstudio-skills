# Feature: Init Skill

> [View in SpecScore Studio](https://specscore.studio/project/features?id=specstudio-skills@specscore@github.com&path=spec%2Ffeatures%2Fskills%2Finit) — graph, discussions, approvals

**Status:** Approved
**Source Ideas:** specstudio-init-skill

## Summary

The `specstudio:init` skill bootstraps a SpecScore-managed project in one wizard-driven step. Detects current project state by direct repo inspection, asks 3–5 batched wizard questions with defaults prefilled, then idempotently scaffolds: (a) the `spec/` tree with lint-clean indexes via `specscore init`, (b) `specscore.yaml` with the canonical schema-pointer header and a default `project:` block, (c) the canonical Producer-shape instruction snippet (defined by [`third-party-integration`](../../third-party-integration/README.md)) appended to the right platform agent-instructions file per the platform-detection rule, and (d) any orchestration-side artifacts via `synchestra init`. Prefers the deterministic CLIs (`specscore init`, `synchestra init`) with AI-agent fallback when a CLI is not on PATH; delegates CLI installation symmetrically to [`specscore:install`](https://github.com/synchestra-io/ai-plugin-specscore/blob/main/skills/install/SKILL.md) and [`synchestra:install`](https://github.com/synchestra-io/ai-plugin-synchestra/blob/main/skills/install/SKILL.md). Two modes: default (full wizard) and `--update` (drift-only reconciliation, no wizard). Implementation lives at [`skills/init/`](../../../../skills/init/).

## Problem

SpecStudio's bootstrap is currently scattered: `specstudio:ideate` lazily creates `spec/ideas/` on first invocation, `specstudio:specify` lazily creates `spec/features/`, and the canonical Producer-shape instruction snippet (now defined by the [`third-party-integration`](../../third-party-integration/README.md) Feature) has no install path. A new adopter who wants to bootstrap a SpecScore-managed project has to compose the right CLI calls, hand-write `specscore.yaml`, and manually paste the snippet into the right agent-instructions file — and the platform-detection logic (`CLAUDE.md` vs. `AGENTS.md` vs. both) has no canonical home.

`specstudio:init` exists to make bootstrap a single conversational step, with sane defaults, idempotent reruns, and explicit consent at every action that touches a file outside `spec/`. The Feature defines exactly what that skill must do — including the per-CLI delegation contract, the platform-detection rule, the snippet drift-reconciliation flow, and the event vocabulary — so its behavior is enforceable and the experience consistent across adopters.

### Departures from the source Idea

This Feature preserves the core of the [`specstudio-init-skill`](../../../ideas/specstudio-init-skill.md) Idea's Recommended Direction (wizard pattern, CLI/skill split, CLAUDE.md/AGENTS.md detection rule, `project.initialized` event, idempotent rerun, no state file, delegation to install skills). Two deliberate spec-time refinements depart from the Idea's text and are surfaced here so they can be reviewed explicitly:

1. **Explicit `--update` mode flag** (REQ `skill-modes`). The Idea described a single-mode skill that is "Idempotent on rerun via direct repo inspection" — implying overloaded default-mode behaviour for both fresh init and drift reconciliation. The Feature splits drift-only reconciliation into a distinct invocation `specstudio:init --update`. Rationale: a drift-only flow has no wizard, no scaffolding, and asks per-artifact diff/replace/keep questions; a fresh init has a 3–5-question wizard and creates new artifacts. Overloading default-mode would force the skill to ask both kinds of question and skip whichever doesn't apply, which is error-prone and harder to test. Splitting clarifies the user model — `init` creates, `init --update` reconciles — without changing the underlying repo-state-driven logic.
2. **Split event vocabulary: `project.initialized` and `project.updated`** (REQ `event-project-initialized`, `event-project-updated`). The Idea named one event: "After successful initialization the skill emits a `project.initialized` event for Synchestra Hub to consume." The Feature splits this so `project.initialized` fires exactly once per project (greenfield-only) and `project.updated` fires on subsequent state-changing reruns. Rationale: this matches the discipline already established for `idea.drafted` / `idea.updated` and `feature.specified` / `feature.updated` (per `synchestra-events.md`) — first emission marks the lifecycle transition, subsequent emissions carry change-context for consumers tracking ongoing state. Treating every state-changing rerun as another `project.initialized` would conflate creation with update for downstream consumers.

If either departure is rejected on review, restore the Idea's design — the change is contained: `--update` collapses back into default-mode rerun, and `project.updated` collapses into a second `project.initialized` (or no event on update). Both reversions are mechanical edits to the relevant REQs and ACs.

## Behavior

### Invocation, modes, and project-state detection

The skill detects the project's current init state by direct repo inspection before asking any wizard question, then chooses between greenfield, brownfield, and update flows.

#### REQ: invocation-triggers

The skill MUST respond to the triggers `specstudio:init`, `/specstudio:init`, "set up specstudio", "init synchestra project", and "bootstrap a spec repo". It MAY respond to additional natural-language phrasings of the same intent.

#### REQ: skill-modes

The skill MUST support exactly two operational modes:

1. **Default (full wizard).** Invocation: `specstudio:init` with no flag. Behavior: state detection → CLI prerequisite check + install delegation → wizard (3–5 batched questions) → bootstrap actions → event emission. Greenfield and brownfield repos both flow through this mode; brownfield reuses defaults derived from detected state.
2. **Update (drift reconciliation).** Invocation: `specstudio:init --update`. Behavior: state detection → no wizard → check each managed artifact (snippet pasted in agent-instructions file, `specscore.yaml` schema header, `spec/` indexes) for drift against the canonical version → on drift, present diff and ask the user to replace / keep / abort per artifact → no defaults derived; user's explicit choice is required for each diff. Update mode MUST NOT scaffold new artifacts (e.g., create `spec/features/` if absent); if a managed artifact is missing entirely, update mode reports "not initialized" and refuses to proceed, redirecting to default mode.

The skill MUST NOT introduce additional modes (e.g., `--repair`, `--migrate`, `--ci`) without an additive Feature revision.

#### REQ: project-state-detection

Before any user-facing prompt, the skill MUST inspect the repo to determine: (a) whether `git` is initialized, (b) whether `spec/`, `spec/ideas/`, `spec/features/` exist with lint-clean indexes, (c) whether `specscore.yaml` exists at repo root and is schema-conforming, (d) which agent-instructions files exist (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, files under `.cursor/rules/`), (e) for each existing agent-instructions file, whether the canonical snippet is already pasted and at what version, and (f) whether `specscore` and `synchestra` are on PATH. Detection MUST NOT use any state file or hidden cache; the repo's filesystem and `command -v` results are the only source of truth.

#### REQ: skip-condition-fully-initialized

In default mode, when state detection determines the project is fully initialized AND no drift is detected (snippet at canonical version, `specscore.yaml` schema-conforming, all expected `spec/` indexes lint-clean), the skill MUST NOT run the wizard. It MUST report "already initialized; nothing to do" with the detected artifact paths and exit cleanly without emitting an event.

### Wizard interaction

The skill asks a small, bounded set of questions in a single batch. It does NOT chain questions one at a time except in the explicit consent gates around install delegation and snippet write.

#### REQ: wizard-question-count

The wizard MUST ask between 3 and 5 questions in a single batched `AskUserQuestion` call. The skill MUST NOT exceed five wizard questions in default mode. If a sixth concern emerges that seems to require a question, redesign the wizard to derive that concern from a default or move it out of MVP scope.

#### REQ: wizard-question-defaults

Every wizard question MUST have a sane default derived from project-state-detection results. Examples: when `CLAUDE.md` exists at repo root and `AGENTS.md` does not, the platform-target default is `CLAUDE.md`; when `git remote get-url origin` succeeds, the `project.host`/`project.org`/`project.repo` defaults are inferred from the URL; when `git config user.name` is set, the default `project.title` is the basename of the repo working directory. The user may override any default; the wizard MUST NOT silently apply a default the user did not see.

#### REQ: wizard-questions-fixed-set

The wizard's question set is fixed in MVP. The five questions (some MAY be skipped when their answer is unambiguous from detected state):

1. **Platform agent-instructions target.** Which file to install the canonical snippet into. Skipped when only one platform file exists.
2. **Optional spec subdirectories.** A single batched question presenting both subdirs (`spec/research/` and `spec/decisions/`) with their visible defaults — `decisions/` defaulted to `yes` (selected) and `research/` defaulted to `no` (unselected). The user toggles either or both before submitting; no sub-prompt or chained question. The mandatory subdirs (`spec/ideas/`, `spec/features/`) are always created and not surfaced as options.
3. **Custom viewer.** Whether to register a non-default `viewer:` block in `specscore.yaml`. Default: no (Repo Config defaults apply).
4. **Synchestra orchestration features.** Whether to enable optional `synchestra init` features (state repo pointer, runtime caches). Default: ask.
5. **Greenfield confirmation.** When state detection finds non-trivial pre-existing structure (e.g., a `docs/` tree with what look like spec artifacts), confirm before scaffolding. Skipped when greenfield is unambiguous.

Future questions added by this skill require an additive Feature revision and a versioning increment to this Feature's documented behavior.

### CLI prerequisites and install delegation

The skill checks for `specscore` and `synchestra` on PATH and delegates installation when missing. It does NOT run install commands itself.

#### REQ: cli-detection

The skill MUST run `command -v specscore` and `command -v synchestra` as part of project-state-detection (see `project-state-detection`). Results are recorded as either "present" or "missing"; presence does NOT imply version compatibility — version pinning is out of MVP scope.

#### REQ: specscore-install-delegation

When `specscore` is missing, the skill MUST invoke the [`specscore:install`](https://github.com/synchestra-io/ai-plugin-specscore/blob/main/skills/install/SKILL.md) skill via the platform's skill-invocation mechanism (e.g., the Claude Code `Skill` tool, the equivalent on other platforms). Before invoking, the skill MUST ask the user for explicit consent: "specscore is not on PATH. Invoke specscore:install to install it now? (yes / no)". On `yes`, invoke; on `no`, abort with "specscore is required for full bootstrap; please install manually and re-run init."

#### REQ: synchestra-install-delegation

When `synchestra` is missing, the skill MUST invoke the [`synchestra:install`](https://github.com/synchestra-io/ai-plugin-synchestra/blob/main/skills/install/SKILL.md) skill via the same mechanism, with the same consent gate. Decline aborts with the same shape of message ("synchestra is required for orchestration-side bootstrap; please install manually and re-run init"). The skill MUST handle each install delegation independently — declining one does not skip the consent gate for the other.

#### REQ: post-install-resume

After an install skill returns, the skill MUST re-run `command -v <cli>` to detect the newly installed binary. On detection, resume the bootstrap step that prompted the install. On still-missing (the install skill returned but the binary is not on PATH), abort with a message identifying the unexpected state and recommending the user open a new shell or check `PATH`. The skill MUST NOT loop install-delegation more than once per CLI.

### Spec tree and config bootstrap

The skill prefers the `specscore init` CLI subcommand and falls back to direct AI-agent file writes when the CLI is missing or declined.

#### REQ: prefer-specscore-init-cli

When `specscore` is on PATH, the skill MUST invoke `specscore init` (as a subprocess) to scaffold the `spec/` tree and create `specscore.yaml`. The skill MUST pass through the wizard's resolved answers as CLI flags where the CLI accepts them; remaining values are filled by `Edit` after the CLI returns. The skill MUST NOT invent CLI flags `specscore init` does not document.

#### REQ: ai-agent-fallback-spec-tree

When `specscore` is missing AND the user declined to install it (per `specscore-install-delegation`), the skill MUST fall back to writing the `spec/` tree and `specscore.yaml` directly via the AI agent. The fallback path MUST produce schema-equivalent artifacts to the CLI path: identical mandatory schema-pointer header on `specscore.yaml`, identical lint-clean structure on `spec/ideas/README.md` and `spec/features/README.md`, and identical title-prefix dispatch on each index per the canonical schemas.

#### REQ: schema-equivalence-cli-fallback

CLI and fallback paths MUST produce schema-equivalent artifacts. Schema equivalence means: identical mandatory content, identical section structure, identical metadata fields, and a clean `specscore spec lint` exit on both paths. Cosmetic differences (whitespace, blank-line counts, comment style, default ordering of optional fields) are permitted. The verification rule: run init in a fresh repo with `specscore` on PATH, run init in a fresh repo with `specscore` absent (and the user accepts the AI-agent fallback), then `diff -r` the resulting `spec/` trees and `specscore.yaml`. Any non-cosmetic difference is a contract violation to fix in the fallback path.

#### REQ: specscore-yaml-conformance

The created `specscore.yaml` MUST conform to the canonical [SpecScore Repo Config Feature](https://github.com/synchestra-io/specscore/blob/main/spec/features/repo-config/README.md): line 1 is exactly the schema-pointer comment (`# SpecScore Repo Config Schema: https://specscore.md/repo-config`), the optional `project:` block is populated from git-remote-inferred and wizard-confirmed values, and any orchestrator extension fields (e.g., `reviewers:` per the [`third-party-integration`](../../third-party-integration/README.md) Feature) are NOT pre-populated by init — those are added by their respective consumer Features when registration happens.

### Instruction snippet installation

The canonical Producer-shape instruction snippet defined by the [`third-party-integration`](../../third-party-integration/README.md) Feature is installed into the platform agent-instructions file selected by the platform-detection rule.

#### REQ: snippet-source

The skill MUST read the canonical snippet from `spec/features/third-party-integration/snippet.md` at the time of install. The skill MUST NOT cache or vendor the snippet; the source of truth is the file in the project's checkout (which itself is a copy of the canonical published snippet from the SpecStudio repo). When the snippet does not yet exist (e.g., the SpecStudio version in use has not yet shipped the snippet artifact), the skill MUST report "snippet not yet shipped at <path>" and skip the snippet-install step without aborting other bootstrap steps.

#### REQ: platform-detection-rule

The skill MUST apply this exact rule to determine the snippet target file:

1. **No agent-instructions file present** → ask the user which to create from the canonical paste-target list (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, or a Cursor rules file — default name `.cursor/rules/specstudio.md`, see Outstanding Questions on naming collision).
2. **Exactly one agent-instructions file present** → target it.
3. **Multiple agent-instructions files present.** Inspect the contents:
   - If `CLAUDE.md` exists AND its content references `AGENTS.md` (e.g., contains a literal `AGENTS.md` mention, an `@AGENTS.md` import, or a "see AGENTS.md" pointer) → target only `AGENTS.md` (CLAUDE.md is acting as a redirect to the canonical AGENTS.md).
   - Otherwise → ask the user which to update (with multi-select option to update all).

This rule is the canonical home for platform-instructions-file detection across SpecStudio; the [`third-party-integration`](../../third-party-integration/README.md) Feature defers to it.

#### REQ: snippet-paste-confirmation

Before writing the snippet to the target file, the skill MUST display: (a) the snippet content (or a length-bounded summary if longer than 50 lines), (b) the target file path, (c) the action ("append at end of file" for new install; "replace existing snippet block" for drift reconciliation), and (d) an explicit consent prompt. The skill MUST NOT write the snippet without affirmative consent. On decline, abort the snippet-install step with "snippet not installed; you can re-run `specstudio:init --update` later."

#### REQ: snippet-version-comment-preserved

The pasted snippet MUST preserve the embedded version comment (`<!-- specstudio-snippet-version: <semver> -->`) per the [`third-party-integration`](../../third-party-integration/README.md) `producer-snippet-versioning` REQ. The version comment is what `--update` mode uses to detect drift; stripping or modifying it breaks the drift-detection contract.

#### REQ: snippet-conflict-resolution

When `--update` mode (or default mode rerun) detects that the target file already contains a snippet block at a version different from the canonical:

1. Display a unified diff between the pasted version and the canonical version.
2. Display the canonical version's changelog if available (currently: just the version increment; future revisions may carry a changelog).
3. Ask the user to choose: **replace** (overwrite the pasted block with the canonical), **keep** (leave the pasted block as-is; report drift but do not act), or **abort** (exit init without writing). The skill MUST NOT auto-merge; merging is a user decision in their own editor when neither replace nor keep fits.
4. On replace, write atomically (single-file write, no partial state).

### Orchestration-side bootstrap (`synchestra init`)

The skill delegates orchestration-side artifacts to `synchestra init`, mirroring the specscore-side delegation.

#### REQ: prefer-synchestra-init-cli

When `synchestra` is on PATH, the skill MUST invoke `synchestra init` (as a subprocess) for orchestration-side artifacts: state-repo pointer setup, runtime cache directory creation, any `.gitignore` rules Synchestra requires for its caches. The skill MUST pass through wizard answers relevant to Synchestra orchestration features (per `wizard-questions-fixed-set` question 4).

#### REQ: ai-agent-fallback-synchestra

When `synchestra` is missing AND the user declined to install it (per `synchestra-install-delegation`), the skill MUST fall back to a limited AI-agent path: it MAY write a placeholder `.synchestra/` directory with a README explaining "synchestra runtime not yet installed; run `specstudio:init --update` after installing synchestra". The fallback MUST NOT attempt to replicate `synchestra init`'s full behavior — orchestration-side semantics are owned upstream. Fallback's only goal is to leave the repo in a state where re-running init after installing synchestra completes the orchestration setup cleanly.

#### REQ: synchestra-cli-contract-dependency

The init Feature depends on the upstream `synchestra init` CLI subcommand contract (flag set, exit codes, idempotence guarantees) defined by [`synchestra@spec/features/cli/init/`](https://github.com/synchestra-io/synchestra/blob/main/spec/features/cli/init/README.md). When the upstream contract changes, this Feature revises in place to match. The contract guarantees: `synchestra init` writes a dedicated `synchestra.yaml` at the repo root (per the [synchestra repo-config Feature](https://github.com/synchestra-io/synchestra/blob/main/spec/features/repo-config/README.md)) — synchestra-only orchestration metadata lives in its own file, NOT as extension keys inside `specscore.yaml`. The two files compose without duplication; the skill never writes synchestra fields into specscore.yaml.

The upstream `synchestra init` v1 implements only `--state-mode embedded`; `--state-mode separate-repo` and `--state-mode hub-managed` are recognized but exit 2 with a "not yet implemented" message. When the user requests an unimplemented mode, the skill MUST treat that exit as a soft failure and either fall back to embedded mode (with the user's explicit consent) or report degradation and continue. When `synchestra` is absent (declined install), the synchestra-side bootstrap degrades per `ai-agent-fallback-synchestra`.

### Idempotence and event emission

Reruns are safe and reconcile drift; events fire only when state changed.

#### REQ: idempotent-rerun

Re-running `specstudio:init` (default mode) on a fully-initialized project with no drift MUST be a no-op user-facing — no wizard, no writes, no commits. Re-running on a project with partial state (some artifacts present, others missing) MUST resume the bootstrap from where it left off without overwriting the present artifacts. Re-running on a project with detected drift MUST present the drift via `snippet-conflict-resolution` (for snippet drift) or analogous prompts (for `specscore.yaml` schema drift, etc.) — never silently update.

#### REQ: no-state-file

Idempotence MUST be achieved via direct repo inspection only. The skill MUST NOT create a `.specstudio/init-state.yaml`, `.synchestra/init.lock`, or any other file whose purpose is to track init state. The repo's actual filesystem state is the single source of truth; reconciliation never lies about what's there.

#### REQ: never-clobber-user-edits

When the skill encounters a managed artifact (e.g., `specscore.yaml`, the snippet block in `CLAUDE.md`) that has been hand-edited by the user beyond the snippet/config schema's required content, it MUST NOT silently overwrite. Drift detection treats user edits identically to version drift: present the diff, ask the user. The skill MUST NOT distinguish "user-edit drift" from "version drift" — both are "the file differs from canonical, ask the user."

#### REQ: event-project-initialized

On first successful greenfield init (state detection found no `spec/` tree, no `specscore.yaml`, no snippet pasted; bootstrap created all of them), the skill MUST emit `project.initialized` exactly once. The event payload MUST include: `project_id` (slug derived from `project.repo` in `specscore.yaml`), `revision` (current git SHA after staging), `cli_versions` (object with `specscore` and `synchestra` versions or `null` for AI-agent-fallback paths), `snippet_target_file` (path to the agent-instructions file written, or `null` if snippet-install was skipped), `artifacts_created` (list of paths written by this invocation).

#### REQ: event-project-updated

On subsequent runs that change repo state — `--update` mode resolving drift, default mode resuming a partial bootstrap, snippet replacement on a version-drift detection — the skill MUST emit `project.updated`. Payload mirrors `project.initialized` plus `change_summary` (≤2 factual sentences describing what changed; same discipline as `idea.updated` and `feature.updated` per `synchestra-events.md`). The skill MUST NOT emit `project.initialized` more than once per project; subsequent state-changing runs always emit `project.updated`.

#### REQ: no-event-on-noop

When state detection determines no action is needed (per `skip-condition-fully-initialized`), the skill MUST NOT emit any event. Empty rerun is a non-event.

### Tone and gates

#### REQ: explicit-consent-on-writes

The skill MUST require affirmative user consent before writing any file outside `spec/`. Specifically: snippet install (target file is at repo root or under `.cursor/rules/`), `synchestra init` orchestration artifacts that touch non-`spec/` paths, and `.gitignore` modifications via `synchestra init`. Consent gates MUST display the exact target path and the change summary before requesting yes/no. The skill MUST NOT bundle multiple unrelated writes behind a single consent prompt.

#### REQ: honest-pushback

The skill MUST NOT yes-machine adopters into a degraded init. When state detection reveals an existing non-canonical spec layout (e.g., `docs/`-based, ad-hoc YAML front-matter, legacy `specscore-spec-repo.yaml`), the skill MUST surface the layout, recommend manual migration before init (and note that "a future `specstudio:migrate` skill will automate this — until then, migration is by hand"), and only proceed if the user explicitly waives. Silent overlay of canonical artifacts on top of a non-canonical layout is a contract violation. The user-facing message MUST NOT direct the adopter at the `specstudio:migrate` skill as if it currently exists; the message phrasing acknowledges that the skill is a forward-reference until it ships.

## Architecture and Components

The init skill orchestrates six functional components. Each is small enough to reason about in isolation and is invoked sequentially by the main flow.

### 1. State detector

- **What:** Reads the repo to determine current init state. Outputs a structured record: `{git_initialized, spec_tree_status, specscore_yaml_status, agent_instruction_files, snippet_versions_per_file, cli_versions}`.
- **How used:** Runs once at skill invocation, before any user-facing prompt. The output drives wizard defaults and skip-condition logic.
- **Depends on:** Filesystem access; `command -v specscore` and `command -v synchestra` shell calls; `specscore` binary (when present) for parsing existing `specscore.yaml` schema-conformance.

### 2. Wizard

- **What:** Single batched `AskUserQuestion` call asking 3–5 questions per `wizard-questions-fixed-set`. Defaults pre-filled from state detector output.
- **How used:** Runs once per default-mode invocation. Skipped entirely in `--update` mode and in the fully-initialized skip path.
- **Depends on:** State detector (for defaults); the platform's `AskUserQuestion` (or equivalent) tool.

### 3. CLI delegators

- **What:** Wrappers around `specscore init` and `synchestra init` subprocess invocations. Pass through wizard answers, capture stdout/stderr/exit-code, surface failures cleanly.
- **How used:** Invoked after the wizard, conditionally based on CLI presence.
- **Depends on:** `specscore` CLI binary (preferred path); `synchestra` CLI binary (preferred path); the AI-agent fallback paths (when CLI is absent and install was declined).

### 4. Install delegators

- **What:** Hand off to `specscore:install` / `synchestra:install` skills via the platform's skill-invocation mechanism. Capture the return state; re-run `command -v` to confirm the install succeeded.
- **How used:** Invoked from `cli-detection` results when a CLI is missing. Each delegator runs at most once per init invocation.
- **Depends on:** The existing [`specscore:install`](https://github.com/synchestra-io/ai-plugin-specscore/blob/main/skills/install/SKILL.md) and [`synchestra:install`](https://github.com/synchestra-io/ai-plugin-synchestra/blob/main/skills/install/SKILL.md) skills (hard prerequisites — both skills MUST exist by the time this Feature ships).

### 5. Snippet installer

- **What:** Reads `spec/features/third-party-integration/snippet.md`, applies the platform-detection rule to choose target, presents diff/consent, writes atomically.
- **How used:** Invoked after the spec-tree-and-config bootstrap step. Skipped when snippet does not yet exist at the canonical path (per `snippet-source`).
- **Depends on:** The [`third-party-integration`](../../third-party-integration/README.md) Feature (provides the canonical snippet); the platform-detection rule logic (lives in this Feature, not in `third-party-integration`).

### 6. Event emitter

- **What:** Publishes `project.initialized` or `project.updated` to Synchestra Hub per the canonical event vocabulary in `shared/synchestra-events.md`. Payload assembled from state detector output plus the artifacts-created log.
- **How used:** Invoked exactly once at the end of a state-changing init run. Skipped on no-op runs.
- **Depends on:** The Synchestra event bus (when running under Synchestra orchestration); local-only emission as a JSON line to stderr when not running under Synchestra (mirrors how `idea.drafted` etc. surface in this repo today).

### Component interaction diagram

```
Invocation
  ↓
[State detector]
  ↓
   ├─→ skip-condition-fully-initialized? → exit no-op
   ↓
   default mode                          --update mode
   ↓                                     ↓
[Wizard]                                 (no wizard)
   ↓                                     ↓
[CLI delegators] ←──→ [Install delegators] ─→ specscore:install / synchestra:install
   ↓                                     ↓
[Snippet installer] ──────────────── reads third-party-integration/snippet.md
   ↓                                     ↓
[Event emitter] ────────────────── project.initialized / project.updated
```

## Interaction with Other Features

| Feature | Interaction |
|---|---|
| [Ideate Skill](../ideate/README.md) | Currently bootstraps `spec/ideas/` lazily via direct file writes. Out of MVP scope: future revision rewrites the lazy-bootstrap path to delegate to `specstudio:init` (or directly to `specscore init`). The behavior change is transparent to users — lazy-bootstrap on first use still happens — but the implementation moves to one canonical place. Tracked in this Feature's Outstanding Questions. |
| [Specify Skill](../specify/README.md) | Same as ideate — currently lazy-bootstraps `spec/features/`; same future-delegation refactor applies. Out of MVP scope. |
| [Third-Party Skill Integration](../../third-party-integration/README.md) | Hard upstream dependency. The init Feature consumes the canonical Producer-shape instruction snippet at `spec/features/third-party-integration/snippet.md`; without that snippet, the snippet-install step is skipped (`snippet-source`). The platform-detection rule is canonical here, not there. |
| [SpecScore Repo Config](https://github.com/synchestra-io/specscore/blob/main/spec/features/repo-config/README.md) | Hard upstream dependency. The created `specscore.yaml` conforms to its schema. The init Feature does not own the schema; it conforms. |
| [`specscore:install`](https://github.com/synchestra-io/ai-plugin-specscore/blob/main/skills/install/SKILL.md) | Hard prerequisite. Invoked when `specscore` is missing and the user consents to install. The install skill's contract (input/output, exit semantics) is load-bearing for `post-install-resume`. |
| [`synchestra:install`](https://github.com/synchestra-io/ai-plugin-synchestra/blob/main/skills/install/SKILL.md) | Hard prerequisite. Symmetric to `specscore:install`; same handoff contract. |
| `synchestra init` CLI subcommand | Soft upstream dependency. When present, this Feature invokes it for orchestration-side bootstrap. When absent, the synchestra-side bootstrap is gracefully degraded (`synchestra-cli-contract-dependency`). |
| Synchestra Events | Emits `project.initialized` and `project.updated`, both with the canonical change-context payload shape used by `idea.*` and `feature.*` events. Consumers — Hub, watchers — observe these to advance their own state. |
| Future `specstudio:migrate` (forward reference) | When state detection finds a non-canonical pre-existing layout, the skill recommends `specstudio:migrate` (which does not yet exist as a skill or a Feature). Forward-referencing the migrate skill is acceptable because this Feature does not invoke or import it — it only mentions it in user-facing recommendations. When `specstudio:migrate` ships, this Feature revises in place to add an explicit invocation hook. |

## Acceptance Criteria

### AC: invocation-and-mode-selection

**Requirements:** skills/init#req:invocation-triggers, skills/init#req:skill-modes

**Given** a project repo and a user invocation
**When** the skill receives one of `specstudio:init`, `/specstudio:init`, "set up specstudio", "init synchestra project", or "bootstrap a spec repo" (default mode), OR `specstudio:init --update` (update mode)
**Then** the skill enters the corresponding mode (default: full wizard flow; update: drift-only flow with no wizard); a third mode invocation (e.g., `specstudio:init --repair`) MUST be rejected with "unknown flag" rather than silently treated as default mode.

### AC: state-detection-precedes-prompts

**Requirements:** skills/init#req:project-state-detection, skills/init#req:skip-condition-fully-initialized

**Given** any invocation
**When** the skill starts
**Then** before any user-facing prompt, state detection has run and recorded: git-initialized status, spec-tree status, specscore.yaml status, agent-instructions files present and their snippet versions, and CLI presence for both specscore and synchestra. When state detection determines no action is needed (project fully initialized, no drift), the skill exits cleanly with "already initialized" and no event is emitted (no `project.initialized`, no `project.updated`).

### AC: wizard-bounded-and-defaulted

**Requirements:** skills/init#req:wizard-question-count, skills/init#req:wizard-question-defaults, skills/init#req:wizard-questions-fixed-set

**Given** default-mode invocation in a greenfield or partial-state repo
**When** the wizard runs
**Then** it asks at most 5 questions in a single batched `AskUserQuestion` call; every question carries a default derived from state detection (default visible to the user before they answer); the question set is drawn from the fixed five (`platform agent-instructions target`, `optional spec subdirectories`, `custom viewer`, `synchestra orchestration features`, `greenfield confirmation`); questions whose answer is unambiguous from detected state are skipped, not asked with a forced default.

### AC: cli-prerequisites-and-install-delegation

**Requirements:** skills/init#req:cli-detection, skills/init#req:specscore-install-delegation, skills/init#req:synchestra-install-delegation, skills/init#req:post-install-resume

**Given** an invocation in a repo where one or both CLIs are missing
**When** state detection records the missing CLI
**Then** the skill asks the user for explicit consent before invoking the corresponding install skill; on consent, invokes `specscore:install` and/or `synchestra:install` via the platform's skill-invocation mechanism; on each install skill's return, re-runs `command -v` to confirm; on still-missing, aborts with a clear next-step message; on decline, aborts the bootstrap step that prompted the install with a clear "manual install required, re-run init" message; the skill MUST NOT loop install delegation more than once per CLI within a single invocation.

### AC: cli-vs-fallback-equivalence

**Requirements:** skills/init#req:prefer-specscore-init-cli, skills/init#req:ai-agent-fallback-spec-tree, skills/init#req:schema-equivalence-cli-fallback, skills/init#req:specscore-yaml-conformance

**Given** a fresh repo and a fully-completed init wizard
**When** init runs once with `specscore` on PATH (CLI path) and once with `specscore` absent and the user accepting the AI-agent fallback (fallback path)
**Then** `diff -r <cli-path-result>/spec/ <fallback-path-result>/spec/` shows only cosmetic differences (whitespace, blank-line counts, comment style, optional-field ordering); both `specscore.yaml` files have identical line 1 (the mandatory schema-pointer comment), identical mandatory fields, and pass `specscore spec lint` cleanly; any non-cosmetic difference is a bug to fix in the fallback path.

### AC: cli-happy-path-and-flag-passthrough

**Requirements:** skills/init#req:prefer-specscore-init-cli, skills/init#req:prefer-synchestra-init-cli

**Given** a fresh repo with both `specscore` and `synchestra` on PATH and both `init` subcommands shipped, and a default-mode init wizard with non-default answers (e.g., `spec/research/` opted in via question 2; Synchestra orchestration features enabled via question 4)
**When** the bootstrap step runs after the wizard completes
**Then** the skill invokes `specscore init` and `synchestra init` as subprocesses (observable in the trace as exec calls); the wizard's resolved answers appear either as documented CLI flags passed to the corresponding subprocess invocation OR as `Edit` operations applied immediately after the subprocess returns; the resulting `spec/research/` directory exists with a lint-clean index when question 2 was opted in; orchestration artifacts written by `synchestra init` exist when question 4 enabled them. Wizard answers that match a CLI's documented flag MUST go through the flag (not post-CLI `Edit`); inventing flags that the CLI does not document is forbidden per `prefer-specscore-init-cli`'s flag discipline.

### AC: snippet-install-with-consent-and-version

**Requirements:** skills/init#req:snippet-source, skills/init#req:platform-detection-rule, skills/init#req:snippet-paste-confirmation, skills/init#req:snippet-version-comment-preserved

**Given** the third-party-integration Feature's canonical snippet exists at `spec/features/third-party-integration/snippet.md` and the wizard has selected a target file
**When** the snippet-install step runs
**Then** the skill reads the canonical snippet at install time (no caching); applies the platform-detection rule (only-one→target-it, both-with-CLAUDE-referencing-AGENTS→target-AGENTS-only, otherwise→ask); displays the snippet, target path, and action to the user; writes only on affirmative consent; the written snippet block contains the embedded `<!-- specstudio-snippet-version: <semver> -->` comment unmodified; on decline, the snippet-install step is skipped without aborting other bootstrap steps; when the canonical snippet does not yet exist at its path, the snippet-install step is skipped with "snippet not yet shipped" and other steps continue.

### AC: snippet-drift-reconciliation

**Requirements:** skills/init#req:snippet-conflict-resolution, skills/init#req:never-clobber-user-edits

**Given** a repo with a previously-pasted snippet at version X and a canonical snippet at version Y where X ≠ Y
**When** the user runs `specstudio:init --update` (or a default-mode rerun)
**Then** the skill displays a unified diff between the pasted version and the canonical version; offers the choice replace / keep / abort; on replace, writes the canonical version atomically (single-file write); on keep, leaves the pasted block unchanged and reports drift; on abort, exits without writing; the skill MUST NOT auto-merge and MUST NOT distinguish between "version drift" and "user-edit drift" — both prompt the same diff-and-confirm flow.

### AC: idempotence-and-event-discipline

**Requirements:** skills/init#req:idempotent-rerun, skills/init#req:no-state-file, skills/init#req:event-project-initialized, skills/init#req:event-project-updated, skills/init#req:no-event-on-noop

**Given** a project that has already been successfully initialized (`project.initialized` previously fired)
**When** the user re-runs `specstudio:init`
**Then** if no drift is detected, the skill exits no-op without writing or emitting any event; if drift is detected and resolved (snippet replaced, config schema-pointer fixed, etc.), the skill emits exactly one `project.updated` with `change_summary` describing the change factually in ≤2 sentences; the skill never emits a second `project.initialized` for the same project; idempotence is achieved via direct repo inspection only — no state file is created or read for this purpose.

### AC: greenfield-project-initialized-payload

**Requirements:** skills/init#req:event-project-initialized

**Given** a fresh repo with no `spec/` tree, no `specscore.yaml`, and no snippet pasted in any agent-instructions file
**When** `specstudio:init` runs to completion (default mode, all wizard questions answered, all bootstrap steps executed)
**Then** exactly one `project.initialized` event is emitted whose payload is a structured object containing all five required fields: `project_id` is a non-empty string slug derived from `project.repo` in the just-written `specscore.yaml`; `revision` is a non-empty git SHA reflecting the working-tree state after staging; `cli_versions` is an object with keys `specscore` and `synchestra`, each value a semver string when the CLI was used or `null` when the AI-agent fallback wrote that side; `snippet_target_file` is the path to the agent-instructions file the snippet was written to (e.g., `CLAUDE.md`), or `null` if snippet-install was skipped; `artifacts_created` is a non-empty list of paths the invocation wrote (at minimum: `spec/ideas/README.md`, `spec/features/README.md`, `specscore.yaml`). A payload missing any of these fields, or carrying them with the wrong type, is a contract violation.

### AC: synchestra-cli-degraded-bootstrap

**Requirements:** skills/init#req:prefer-synchestra-init-cli, skills/init#req:ai-agent-fallback-synchestra, skills/init#req:synchestra-cli-contract-dependency

**Given** a repo where `synchestra` CLI is on PATH but `synchestra init` (the subcommand) does not yet exist (upstream has not shipped it)
**When** the orchestration-side bootstrap step runs
**Then** the skill reports "synchestra orchestration setup deferred — upstream `synchestra init` not yet shipped"; continues with the specscore-side bootstrap and the snippet-install step without aborting; emits `project.initialized` (or `project.updated`) with `cli_versions.synchestra` populated but the orchestration artifacts absent from `artifacts_created`. The user is informed; no error.

### AC: explicit-consent-on-out-of-spec-writes

**Requirements:** skills/init#req:explicit-consent-on-writes, skills/init#req:honest-pushback

**Given** an invocation that would write outside `spec/` (snippet install to `CLAUDE.md`/`AGENTS.md`/etc., `.gitignore` modification via `synchestra init`, orchestration artifacts in `.synchestra/`)
**When** the skill reaches each such write
**Then** it displays the exact target path and the change summary; requests yes/no consent; the consent prompt is per-write, not bundled across unrelated writes; on decline, that specific write is skipped without aborting other bootstrap steps; when state detection finds an existing non-canonical spec layout, the skill surfaces the layout and recommends manual migration (noting that "a future `specstudio:migrate` skill will automate this — until then, migration is by hand"), only continuing if the user explicitly waives. The recommendation MUST NOT direct the adopter at the `specstudio:migrate` skill as if it currently exists.

## Rehearse Integration

**Selective Rehearse stub scaffolding.** This Feature's ACs span both runtime behavior (testable: state detection logic, CLI delegation flow, snippet write atomicity, event emission) and process discipline (less testable: consent UX feel, wizard question wording). The runtime ACs are testable via filesystem fixtures (greenfield repo, partial-state repo, fully-initialized-no-drift repo, drift repo) and event-payload inspection. The UX ACs are not directly Rehearse-testable.

Stubs scaffolded under `spec/features/skills/init/_tests/` for these ACs:

- `state-detection-precedes-prompts` — fixture-based: set up greenfield/partial/initialized repos, assert detection output and skip-condition behavior
- `cli-vs-fallback-equivalence` — runs init twice in two fresh repos and `diff -r`s the results
- `snippet-install-with-consent-and-version` — fixture: stub canonical snippet, drive wizard, assert pasted content and target file
- `snippet-drift-reconciliation` — fixture: pre-paste snippet at version X, run --update against canonical version Y, assert diff display and write semantics
- `idempotence-and-event-discipline` — runs init twice on an initialized repo, asserts second run emits no event

Skipped (UX/discipline-shaped, not directly testable):

- `wizard-bounded-and-defaulted` — UX shape and wording belongs to manual review, not Rehearse
- `explicit-consent-on-out-of-spec-writes` — consent prompt UX belongs to manual review

Rehearse stubs are scaffolded with `**Status:** pending` per the rehearse-heuristic; authoring the actual scenario steps follows the implementation plan.

## Outstanding Questions

- **Wizard question wording.** This Feature pins question count (3–5), the fixed set, and that defaults are visible — but not the exact prompt strings or option labels. Wording is implementation-time work; the next skill (`writing-plans`) decides whether to lock copy in the plan or leave it to the implement phase. Track signal post-MVP if adopters report specific prompts as confusing.
- **Delegation-refactor sequencing for ideate/specify.** This Feature notes the future delegation refactor (`ideate` and `specify` calling into `init` for their lazy-bootstrap paths) as out of MVP scope. The sequencing question — does the refactor become a sub-Feature of this one, or its own Feature `skills/lazy-bootstrap-delegation/`? — is deferred. Decide when at least one consumer Feature actually adopts the refactor.
- **`project.updated` change_summary discipline reuse.** This Feature borrows the `change_summary` pattern from `idea.updated` and `feature.updated` (factual, ≤2 sentences, no editorializing). The exact discipline is documented in `synchestra-events.md` for those events; whether `project.updated` payloads need additional discipline (e.g., enumerating which managed artifacts changed) is open. Revise additively if a consumer needs more structure.
- **Multi-platform paste targets — update-all semantics.** The platform-detection rule's "ask the user (with multi-select option to update all)" branch implicitly supports updating multiple agent-instructions files in one invocation. The semantic of "update all" in `--update` mode (when versions differ across files) is unspecified — does the skill ask per-file or apply one decision to all? Resolve when a real adopter has multi-platform divergence.
- **`.cursor/rules/<filename>.md` naming.** The platform-detection rule lists `.cursor/rules/specstudio.md` as the canonical Cursor target file. If the user already has files under `.cursor/rules/` for other purposes, the snippet might collide or be misfiled. Open: should the file name be `specstudio.md` always, or should the skill ask? Default to `specstudio.md`; revisit if collision becomes common.
- **Authoring of `synchestra init` CLI contract.** This Feature has a soft dependency on `synchestra init` existing upstream. If it has not shipped by the time this Feature reaches Implementing status, the synchestra-side bootstrap is degraded per `synchestra-cli-contract-dependency`. Track upstream readiness; the synchestra repo's roadmap owns the timeline.
- **Reviewer registration UX home.** REQ `specscore-yaml-conformance` explicitly excludes init from pre-populating the `reviewers:` extension key (defined by [`third-party-integration`](../../third-party-integration/README.md)). That leaves "where do reviewers actually get added to `specscore.yaml`?" unanswered. Candidates: (a) a future `specstudio:reviewers:add` skill, (b) adopters edit the file by hand, (c) a sub-flow in `specstudio:specify` when a registered-but-uninstalled reviewer is referenced. Defer to the downstream Feature that owns reviewer registration UX; init only conforms to the schema, it does not own this decision.

---
*This document follows the https://specscore.md/feature-specification*
