---
name: init
description: |
  Bootstraps a SpecScore-managed project in one wizard-driven step. Detects
  current project state by direct repo inspection, asks 3-5 batched wizard
  questions with defaults pre-filled from detection, then idempotently
  scaffolds: specscore.yaml + spec/{,ideas,features}/README.md (via
  `specscore init`, with AI-agent fallback when the CLI is absent), pastes
  the canonical Producer-shape instruction snippet into the right platform
  agent-instructions file, and runs `synchestra init` for orchestration-side
  setup. Two modes: default (full wizard) and `--update` (drift-only
  reconciliation, no wizard). Delegates CLI installation symmetrically to
  `specscore:install` and `synchestra:install`.
  Trigger: "specstudio:init", "/specstudio:init", "set up specstudio", "init
  synchestra project", "bootstrap a spec repo".
aliases: [init]
---

# Init

Bootstrap a SpecScore-managed project — `specscore.yaml`, `spec/` tree, snippet pasted, orchestration set up — in one wizard.

## Hard Gate

<HARD-GATE>
This skill writes files outside `spec/` (the canonical instruction-file paste targets `CLAUDE.md` / `AGENTS.md` / `GEMINI.md`, the Synchestra runtime cache, optionally `.gitignore`). Every such write MUST go through an explicit per-write user-consent prompt. The skill MUST NOT silently overwrite user content. The skill MUST NOT install CLIs itself — install delegation goes to `specscore:install` and `synchestra:install`, and only after the user consents.

Non-default mode: `--update`. In update mode the skill MUST NOT scaffold anything — only diff and reconcile drift on already-managed artifacts. If the project is not yet initialized, update mode reports "not initialized" and refuses, redirecting to default mode.
</HARD-GATE>

## When to Use

- A user wants to bootstrap a SpecScore-managed project (greenfield or brownfield).
- An existing SpecScore project's snippet has drifted from the canonical version (e.g., the SpecStudio repo shipped a snippet update; adopters need to reconcile). Use `--update`.
- An existing project is partially initialized (e.g., `spec/ideas/` exists from a previous `ideate` run, but no `specscore.yaml`). The skill resumes the missing pieces without erroring.

**Skip** when: the project is fully initialized AND no drift is detected. The skill detects this state and exits cleanly with "already initialized; nothing to do" — no wizard, no event.

## Two operational modes

The mode is determined by invocation argument, not by detected state.

### Default mode — `specstudio:init`

Full wizard flow: state detection → CLI prerequisite check + install delegation → 3–5-question wizard → bootstrap actions (`specscore init` or AI-agent fallback) → snippet install with consent → `synchestra init` (or graceful-degradation) → event emission.

Greenfield AND brownfield repos both flow through this mode; brownfield reuses defaults derived from detected state.

### Update mode — `specstudio:init --update`

Drift-only flow: state detection → no wizard → check each managed artifact (snippet pasted in agent-instructions file, `specscore.yaml` schema header, `spec/` indexes) for drift against the canonical version → on drift, present diff and ask the user to **replace** / **keep** / **abort** per artifact.

Update mode MUST NOT scaffold new artifacts. If a managed artifact is missing entirely, update mode reports "not initialized" and refuses, redirecting to default mode.

## Step 1 — State detection (always runs first)

Before any user-facing prompt, inspect the repo to record:

- `git_initialized` — does `git rev-parse --git-dir` succeed?
- `spec_tree_status` — does `spec/`, `spec/ideas/`, `spec/features/` exist? Are their `README.md` files lint-clean per `specscore spec lint`?
- `specscore_yaml_status` — does `<root>/specscore.yaml` exist? If yes, is line 1 the canonical schema-pointer comment?
- `agent_instruction_files` — for each of `CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, files under `.cursor/rules/`, record whether the file exists at repo root (or under `.cursor/rules/`).
- `snippet_versions_per_file` — for each existing agent-instructions file, scan for the snippet block (delimited by the version comment). If found, record the version. If absent, record `none`.
- `cli_versions` — `command -v specscore` and `command -v synchestra`. For each, record `present` (with version from `<cli> --version`) or `missing`.

**No state file.** Detection is via filesystem inspection only — never via a hidden `.specstudio/init-state.yaml` or similar. The repo's actual state IS the state.

Use the result to drive the rest of the flow:

- All-clean → exit no-op (skip condition).
- Partial state (missing some artifacts) → resume.
- Drift (existing snippet at older version) → reconcile in update mode; in default mode, surface during wizard.
- Greenfield (nothing initialized) → full bootstrap.

## Step 2 — CLI prerequisite check & install delegation

For each missing CLI, ask explicit consent before delegating to its install skill. Each install delegation runs at most once per invocation.

**`specscore` missing**:
1. Surface to user: "specscore is not on PATH. Invoke `specscore:install` to install it now? (yes / no)"
2. On `yes` → invoke the `specscore:install` skill via the platform's skill-invocation mechanism (Claude Code `Skill` tool, or equivalent). Wait for return.
3. After return → re-run `command -v specscore`. If present, continue. If still missing, abort with "specscore install completed but binary still not on PATH; check `PATH` or open a new shell, then re-run init."
4. On `no` → continue with AI-agent fallback for the specscore-side bootstrap (Step 4). Surface to user: "Continuing without specscore CLI; using AI-agent fallback. Some operations will be slower and produce schema-equivalent (not byte-identical) artifacts."

**`synchestra` missing**: same handoff pattern with `synchestra:install`. Symmetric. Each install delegation is independent — declining one does not skip the consent gate for the other.

## Step 3 — Wizard (default mode only)

Skip in `--update` mode and in the all-clean skip condition.

Use a single batched `AskUserQuestion` call. **Maximum five questions.** Each question carries a default derived from state detection; the default is visible to the user before they answer. Questions whose answer is unambiguous from detected state (e.g., only one platform agent-instructions file exists → no need to ask which to target) are skipped, not asked with a forced default.

The fixed five:

1. **Platform agent-instructions target.** Which file to install the canonical snippet into. Skipped when only one platform file exists. Apply the platform-detection rule below.
2. **Optional spec subdirectories.** Single batched question presenting both `spec/research/` and `spec/decisions/` with their visible defaults (decisions=yes, research=no). Implementation note: today the `specscore init` CLI does NOT scaffold these (out of MVP scope upstream); the skill MAY create empty placeholders if requested, OR defer until upstream Indexes ship. Default behavior: defer.
3. **Custom viewer.** Whether to register a non-default `viewer:` block in `specscore.yaml`. Default: no (Repo Config defaults apply).
4. **Synchestra orchestration features.** Whether to enable optional `synchestra init` features (state-repo pointer, runtime caches). Default: ask.
5. **Greenfield confirmation.** When state detection finds non-trivial pre-existing structure (e.g., a `docs/` tree with what look like spec artifacts), confirm before scaffolding. Skipped when greenfield is unambiguous.

### Platform-detection rule for the snippet install step

This rule is canonical for SpecStudio (the [`third-party-integration`](../../spec/features/third-party-integration/README.md) Feature explicitly defers to it):

1. **No agent-instructions file present** → ask the user which to create from the canonical paste-target list (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, or `.cursor/rules/specstudio.md`).
2. **Exactly one agent-instructions file present** → target it.
3. **Multiple agent-instructions files present.** Inspect contents:
   - If `CLAUDE.md` exists AND its content references `AGENTS.md` (literal `AGENTS.md` mention, `@AGENTS.md` import, or "see AGENTS.md" pointer) → target only `AGENTS.md` (CLAUDE.md is a redirect).
   - Otherwise → ask the user which to update (with multi-select option to update all).

## Step 4 — Bootstrap via `specscore init` (preferred) or AI-agent fallback

### Preferred path: invoke `specscore init`

When `specscore` is on PATH (after Step 2), invoke `specscore init` as a subprocess from the project root. Pass the wizard's resolved answers as documented CLI flags:

```bash
specscore init --project <root> [--title <title>] [--host <host>] [--org <org>] [--repo <repo>] [--force]
```

Use `--force` only when state detection found an existing `specscore.yaml` AND the user explicitly opted into overwriting it. Never invent flags `specscore init` does not document; if a wizard answer has no matching flag, write that field via `Edit` after the CLI returns.

Capture stdout/stderr/exit-code. Surface failures cleanly to the user with the underlying message.

### Fallback path: AI-agent direct write

When `specscore` is missing AND the user declined to install (Step 2), write the artifacts directly:

1. **`specscore.yaml`**: line 1 is exactly `# SpecScore Repo Config Schema: https://specscore.md/repo-config`. Lines 2+ are an optional `project:` block with whichever fields the wizard supplied (and any inferred from `git remote get-url origin` for `host`/`org`/`repo`, basename for `title`). Omit empty fields rather than emit them as empty strings.
2. **`spec/README.md`**, **`spec/ideas/README.md`**, **`spec/features/README.md`**: lint-clean indexes per the canonical [Index Feature](https://github.com/specscore/specscore/blob/main/spec/features/index/README.md), [Ideas Index](https://github.com/specscore/specscore/blob/main/spec/features/ideas-index/README.md), [Features Index](https://github.com/specscore/specscore/blob/main/spec/features/features-index/README.md). Use the same content shape `specscore init` produces.

The fallback path MUST produce schema-equivalent artifacts to the CLI path: identical mandatory content, identical section structure, identical metadata fields. Cosmetic differences (whitespace, blank-line counts) are permitted.

After writing, run `specscore spec lint` (if `specscore` is somehow on PATH despite the earlier detection — re-check) and surface any violations.

## Step 5 — Snippet installation (canonical Producer-shape snippet)

If state detection found the snippet already pasted at the canonical version, skip Step 5 entirely.

Otherwise:

1. Read the canonical snippet from `spec/features/third-party-integration/snippet.md` in the project. If the file does not exist (the SpecStudio version in use has not yet shipped the snippet artifact), report "snippet not yet shipped at <path>; skipping snippet install" and continue with subsequent steps. This is graceful degradation — not an error.
2. Apply the platform-detection rule from Step 3 to determine the target file.
3. **Display to the user**: (a) the snippet content (or a length-bounded summary if longer than 50 lines), (b) the target file path, (c) the action ("append at end of file" for new install, "replace existing snippet block" for drift reconciliation), and (d) an explicit consent prompt: "Install / replace the snippet at this path? (yes / no)".
4. On `yes` → write atomically (single-file write). The pasted snippet MUST preserve the embedded `<!-- specstudio-snippet-version: <semver> -->` comment unmodified — that comment is what `--update` mode uses to detect drift on subsequent runs.
5. On `no` → skip snippet install with "snippet not installed; you can re-run `specstudio:init --update` later". Continue with subsequent steps (snippet install is independent of orchestration setup).

### Drift reconciliation (`--update` mode or default-mode rerun)

When state detection finds an existing snippet at a version different from canonical:

1. Display a unified diff between pasted and canonical.
2. Display the canonical version's changelog if available (currently: just the version increment).
3. Ask the user to choose:
   - **replace** — overwrite the pasted block with the canonical
   - **keep** — leave the pasted block as-is; report drift but do not act
   - **abort** — exit init without writing
4. On `replace` → write atomically. The skill MUST NOT auto-merge.

The skill MUST NOT distinguish between "version drift" and "user-edit drift" — both prompt the same diff-and-confirm flow.

## Step 6 — Orchestration via `synchestra init`

When `synchestra` is on PATH (after Step 2), invoke `synchestra init` as a subprocess from the project root. Pass through wizard answers relevant to Synchestra orchestration features (Step 3 question 4) as flags: `--state-mode embedded` (default), `--branch <name>`, `--no-push` if appropriate.

`synchestra init` writes a dedicated `synchestra.yaml` at the repo root (per the [synchestra repo-config Feature](https://github.com/specscore/synchestra/blob/main/spec/features/repo-config/README.md)). It does **not** write extension keys inside `specscore.yaml` — Synchestra-only orchestration metadata lives in its own file, parallel to specscore.yaml. Project identity (title, host, org, repo, repositories) stays in specscore.yaml; synchestra reads it from there. The two files compose without duplication.

In v1, `synchestra init` implements only `--state-mode embedded` (orphan branch + worktree at `.synchestra/`). The `separate-repo` and `hub-managed` modes are recognized but exit 2 with a "not yet implemented" message; treat that exit as a soft failure and continue with embedded mode if the user chose otherwise.

When `synchestra` is missing AND the user declined to install: report "synchestra orchestration setup deferred — `synchestra` CLI is not installed" and continue. The skill does NOT attempt to replicate the orphan-branch / worktree provisioning via AI-agent fallback — orchestration-side semantics are owned upstream and require git operations that belong inside the CLI.

## Step 7 — Event emission

After all bootstrap steps complete:

- **First successful greenfield init** (state detection found nothing initialized; bootstrap created `specscore.yaml` AND at least one index AND optionally the snippet): emit `project.initialized` exactly once. Payload includes `project_id` (slug derived from `project.repo`), `revision` (current git SHA after staging), `cli_versions` (object with `specscore` and `synchestra` versions or `null` for fallback), `snippet_target_file` (path or `null` if skipped), `artifacts_created` (list of paths written this invocation).
- **Subsequent state-changing run** (`--update` resolving drift, default-mode rerun resuming partial bootstrap, snippet replacement): emit `project.updated`. Payload mirrors `project.initialized` plus `change_summary` (≤2 factual sentences).
- **No-op rerun** (skip condition triggered, nothing changed): emit no event.

The skill MUST NOT emit `project.initialized` more than once per project; subsequent state-changing runs always emit `project.updated`.

## Idempotence

Re-running `specstudio:init` (default mode) MUST be safe:

- Fully-initialized + no drift → no-op (skip condition; no wizard, no writes, no event).
- Partial state → resume; create missing pieces; preserve existing pieces byte-identical.
- Drift detected → present via the diff-and-confirm flow; never silently update.

The skill MUST NOT distinguish "user-edit drift" from "version drift" — both use the same prompt: "the file differs from canonical; what do you want to do?"

## Auto-stage in git

When the skill creates files (via `specscore init`, AI-agent fallback writes, snippet install), stage the affected paths with `git add` and report the staged paths to the user in the same response. Never commit on the user's behalf — commits are the user's call.

If the project root is not a git repository, surface "not a git repository — skipping auto-stage; you can `git init` if you want versioned spec history" and continue.

## Tone

Direct, helpful, honest about partial states and degraded paths. The skill is a wizard; the user is the operator. Show what's about to happen before it happens; ask permission for everything outside `spec/`.

## Red Flags

- Running install commands directly (`brew install`, `curl | sh`) instead of delegating to `specscore:install` / `synchestra:install`.
- Writing to agent-instructions files (`CLAUDE.md`, etc.) without explicit per-write consent.
- Silent overwrite of user content on rerun.
- Emitting `project.initialized` more than once for the same project.
- Inventing `specscore init` flags that don't exist.
- Bundling multiple unrelated writes behind a single consent prompt.
- Distinguishing "user-edit drift" from "version drift" in any user-facing way (both are the same prompt).
- Falling back to AI-agent path for `synchestra init` (orchestration-side semantics live upstream — degrade gracefully, don't replicate).
- Creating a state file (`.specstudio/init-state.yaml`, `.synchestra/init.lock`) for idempotence; the filesystem IS the state.
- Treating "not a git repository" as an error instead of a graceful skip-auto-stage condition.

## Verification

- [ ] State detection ran before any user-facing prompt
- [ ] Skip condition (fully initialized + no drift) exits no-op without event
- [ ] CLI install delegations went to `specscore:install` / `synchestra:install`, not direct install commands
- [ ] Wizard asked at most 5 batched questions with defaults visible
- [ ] `specscore init` invoked when CLI present; AI-agent fallback otherwise
- [ ] CLI and fallback paths produce schema-equivalent artifacts
- [ ] Snippet install displayed snippet + target + action, then waited for consent
- [ ] Pasted snippet preserved the version comment unmodified
- [ ] Drift detection used diff-and-confirm; no auto-merge
- [ ] `project.initialized` emitted at most once per project
- [ ] `project.updated` emitted on subsequent state-changing runs
- [ ] No event on no-op rerun
- [ ] No state file created for idempotence
- [ ] Files created were `git add`'d (when project is a git repo) and never committed

## References

- [`spec/features/skills/init/`](../../spec/features/skills/init/README.md) — the Feature spec this skill implements
- [`spec/features/third-party-integration/`](../../spec/features/third-party-integration/README.md) — the Feature defining the canonical snippet and the platform-detection rule
- [`spec/features/third-party-integration/snippet.md`](../../spec/features/third-party-integration/snippet.md) — the canonical Producer-shape instruction snippet this skill installs
- [SpecScore Repo Config Feature](https://github.com/specscore/specscore/blob/main/spec/features/repo-config/README.md) — the schema `specscore.yaml` conforms to
- [SpecScore CLI init Feature](https://github.com/specscore/specscore-cli/blob/main/spec/features/cli/init/README.md) — the `specscore init` subcommand contract this skill delegates to
- [`specscore:install`](https://github.com/specscore/ai-plugin-specscore/blob/main/skills/install/SKILL.md) — install delegate for the `specscore` CLI
- [`synchestra:install`](https://github.com/specscore/ai-plugin-synchestra/blob/main/skills/install/SKILL.md) — install delegate for the `synchestra` CLI
- [`shared/events.md`](../shared/events.md) — event vocabulary `project.initialized` / `project.updated` participate in
