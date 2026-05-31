---
name: survey
description: |
  Produces a fast architecture survey for an existing codebase without reading
  source implementation files. Scans the file tree, reads only allowlisted
  structural manifests, writes JSON-first survey artifacts under spec/research/,
  and stages the results. Use before retrofit or when the user wants a cheap
  architecture overview of an existing repo.
  Trigger: "specstudio:survey", "/survey", "/specstudio:survey",
  "survey this repo", "architecture survey", "map this repo".
aliases: [survey]
---

# Survey

Produce a fast architecture survey of the current repository without reading source implementation files.

## Hard Gate

<HARD-GATE>
This skill MUST NOT read source implementation file contents. It may list source file paths and count them, but it must not open implementation files such as `*.js`, `*.ts`, `*.py`, `*.go`, `*.rs`, `*.java`, `*.cs`, `*.rb`, `*.php`, `*.swift`, or `*.kt`, unless the file is explicitly allowed as a structural manifest below.

The output is research, not canonical Feature intent. Do not write Feature, Plan, or code artifacts from this skill. Do not invoke `specstudio:retrofit` automatically.
</HARD-GATE>

## When To Use

- A user wants a cheap architecture overview of an existing repository.
- A user is deciding whether retrofit is worth running.
- A future `specstudio:retrofit` run needs a `survey-output-schema-v1` input.

Skip when the user wants behavior, requirements, or acceptance criteria derived from code. That is retrofit, not survey.

## Inputs and flags

Supported invocations:

- `specstudio:survey`
- `specstudio:survey --scope <subdir>`
- `specstudio:survey --slug <slug>`
- `specstudio:survey --output-dir <path>`
- `specstudio:survey --json`

Flag behavior:

- `--scope <subdir>` limits the file inventory and manifest reads to that subtree, while repo state still records the whole-repo HEAD.
- `--slug <slug>` overrides slug derivation.
- `--output-dir <path>` overrides the default `spec/research/`.
- `--json` writes only the JSON artifact and skips Markdown rendering.

Unknown flags are refused.

## Step 1 - Pre-flight

1. Determine repo root with `git rev-parse --show-toplevel`. If that fails, use the current directory and record `git_available: false`.
2. Resolve `scope` from `--scope`, defaulting to repo root.
3. Derive `slug` from, in order:
   - `package.json#name`
   - `pyproject.toml [project].name`
   - `go.mod` module basename
   - `Cargo.toml [package].name`
   - repository directory basename
   - `--slug` override, if supplied
4. Sanitize slug to lowercase `a-z0-9-`, collapse repeated dashes, and trim leading/trailing dashes.
5. Set output paths:
   - JSON: `<output-dir>/<slug>-survey.json`
   - Markdown: `<output-dir>/<slug>-survey.md`
   - Default output dir: `spec/research/`

## Step 2 - File inventory

Prefer:

```bash
git ls-files
```

When scoped, filter the inventory to paths under the scope.

If git is unavailable, use `find` and exclude obvious generated or dependency directories:

- `.git`
- `node_modules`
- `.venv`, `venv`, `__pycache__`
- `dist`, `build`, `target`, `.next`, `.turbo`
- `.cache`, `.pytest_cache`

Record the scan method as `git-ls-files` or `find-fallback`.

## Step 3 - Repo state

When git is available, record:

- `head_sha`: `git rev-parse HEAD`
- `dirty_tree`: whether `git status --short` is non-empty
- `status_short`: literal lines from `git status --short`

When git is unavailable, record:

- `git_available: false`
- `head_sha: null`
- `dirty_tree: null`
- `status_short: []`

## Step 4 - Allowed manifest reads only

Read content only from the operational allowlist at [`skills/shared/survey-manifest-allowlist.md`](../shared/survey-manifest-allowlist.md). The current v1 categories are:

- JavaScript / TypeScript: `package.json`, `pnpm-workspace.yaml`, `lerna.json`, `nx.json`, `turbo.json`, `tsconfig*.json`, `next.config.*`, `vite.config.*`, `nuxt.config.*`, `astro.config.*`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`
- Python: `pyproject.toml`, `setup.py`, `setup.cfg`, `requirements*.txt`, `poetry.lock`, `Pipfile`, `Pipfile.lock`, `tox.ini`
- Go: `go.mod`, `go.sum`, `go.work`
- Rust: `Cargo.toml`, `Cargo.lock`
- Other languages: `Gemfile`, `composer.json`, `*.csproj`, `*.sln`, `pom.xml`, `build.gradle*`
- Infra and ops: `docker-compose*.yml`, `Dockerfile`, `terraform/*.tf`, `serverless.yml`, `helm/Chart.yaml`, `kustomization.yaml`
- CI and tooling: `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`, `Makefile`, `justfile`, `Taskfile.yml`
- Release and packaging: `.goreleaser.yml`, `.goreleaser.yaml`, `release-please-config.json`, `.release-please-manifest.json`, `.releaserc`, `.releaserc.json`, `.changeset/config.json`
- Runtime pinning: `.tool-versions`, `.nvmrc`, `.python-version`, `.ruby-version`
- SpecScore: `specscore.yaml`
- Docs: root `README*`, `ARCHITECTURE*`, `CONTRIBUTING*`; top-level `docs/**` filenames only unless a doc file is small enough to read under the size cap

Generated release outputs remain excluded. Do not read `dist/**`, packaged archives, generated checksums, generated changelogs, or binary artifacts as release manifests.

Size cap for any single text read: 80 KB. If an allowlisted file exceeds the cap, do not read it. Record a warning: `skipped due to size`.

## Step 5 - Monorepo detection

Detect monorepo signals before synthesis:

- `pnpm-workspace.yaml`
- `nx.json`
- `lerna.json`
- `turbo.json`
- `go.work`
- Cargo workspace members in root `Cargo.toml`
- multiple root package directories indicated by manifests

If monorepo signals are present and no `--scope` was supplied:

1. Refuse to synthesize a whole-repo survey.
2. Identify the signal paths.
3. Recommend rerunning with `specstudio:survey --scope <subdir>`.
4. Do not write artifacts.

## Step 6 - Build the structured survey

Construct a JSON object with this minimum shape:

```json
{
  "schema": "survey-output-schema-v1",
  "slug": "<slug>",
  "scope": "<scope-or-null>",
  "scan_method": "git-ls-files",
  "repo_state": {
    "git_available": true,
    "head_sha": "<sha>",
    "dirty_tree": false,
    "status_short": []
  },
  "file_inventory_summary": {
    "total_files": 0,
    "by_extension": {},
    "top_level_dirs": []
  },
  "manifest_inventory": [],
  "detected_frameworks": [],
  "architecture_summary": "",
  "directory_clusters": [],
  "research_zones": [],
  "sensitive_path_inventory": [],
  "warnings": []
}
```

Populate:

- `file_inventory_summary`: counts by extension, top-level directory counts, test path counts, docs path counts.
- `manifest_inventory`: allowlisted files read, skipped, or absent.
- `detected_frameworks`: framework/tool signals inferred from manifests and filenames.
- `directory_clusters`: hierarchical directory groups, max depth 3, max 12 children per parent.
- `research_zones`: proposed bounded zones for retrofit researchers, each with path roots, file counts, and one-line purpose.
- `sensitive_path_inventory`: filename-pattern hints only.
- `warnings`: dirty tree, skipped oversized manifests, monorepo refusal signals when scoped, ambiguous slug signals.

Do not invent behavior from source. If a conclusion depends only on file names or manifests, phrase it as inferred.

## Step 7 - Sensitive path inventory

Flag filename-pattern hints for:

- `.env*`
- `secrets/**`
- `*.pem`
- `*.key`
- `**/fixtures/**`
- git-crypt markers
- submodule entries
- LFS pointer-looking files
- lockfile mismatch hints, such as multiple package-manager lockfiles

Label the section explicitly: "Filename-pattern hints only; not a content secret scan."

## Step 8 - Write JSON first

Create the output directory if needed. Write the JSON artifact first. Sort object keys where practical and keep arrays in deterministic path order.

If `--json` is set, skip Markdown rendering and go to indexing/lint/staging.

## Step 9 - Render Markdown from JSON

Render Markdown from the JSON artifact. The Markdown must contain:

1. `# Survey: <title>`
2. `**Status:** Current`
3. `**Date:** <YYYY-MM-DD>`
4. `**Repo SHA:** <sha-or-unavailable>`
5. `**Scope:** <scope-or-repo-root>`
6. `**JSON:** <relative path to json>`
7. `## Summary`
8. `## Architecture`
9. `## Directory Clusters`
10. `## Research Zones`
11. `## Detected Frameworks`
12. `## Sensitive Path Inventory`
13. `## Warnings`
14. `## Open Questions`
15. Footer: `*This document follows the https://specscore.md/research-artifact-specification*`

Use Mermaid diagrams only when they add clarity. Always include text lists so the artifact remains useful without Mermaid rendering.

## Step 10 - Research index

For default output under `spec/research/`, ensure `spec/research/README.md` exists.

If absent, create:

```markdown
# Research

Research artifacts produced by SpecStudio skills.

## Contents

| Artifact | Description |
|---|---|

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/index-specification*
```

Add or update one row for the survey:

```markdown
| [<slug>-survey](<slug>-survey.md) | Architecture survey for `<scope-or-repo>`. |
```

Do not duplicate rows for the same survey slug.

## Step 11 - Lint and fix once

Run:

```bash
specscore spec lint
```

If lint fails, make one focused fix pass for the generated artifacts and rerun lint. If violations remain, surface them with paths and stop.

## Step 12 - Stage

Stage all generated or updated files:

```bash
git add <json> <markdown-if-written> <research-index-if-updated>
```

Never commit. Report the staged paths.

## Output summary

End with:

- JSON path
- Markdown path, unless `--json`
- Research index path, if updated
- Lint result
- Staged paths
- Any warnings, especially dirty tree and sensitive-path hints

## Relationship to retrofit

`specstudio:retrofit` consumes the JSON artifact. Survey does not invoke retrofit automatically. If the user wants to continue, recommend running retrofit with the generated JSON path once retrofit ships.
