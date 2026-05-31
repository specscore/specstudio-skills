# Feature: Survey Skill

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/survey?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/survey?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/survey?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/skills/survey?op=request-change) |

**Status:** Approved
**Date:** 2026-05-31
**Owner:** alex
**Source Ideas:** survey-skill
**Supersedes:** —

## Summary

The `specstudio:survey` skill produces a fast architecture survey for an existing codebase without reading source files. It inspects the file tree and an allowlist of structural manifests, writes a schema-shaped JSON artifact at `spec/research/<slug>-survey.json`, and renders a deterministic Markdown companion at `spec/research/<slug>-survey.md`. The survey is useful on its own and is the required upstream input for future `specstudio:retrofit` runs.

## Problem

SpecStudio's forward lifecycle assumes a team starts with an Idea and works toward code. Existing projects need the reverse entry point: a low-cost way to understand repository shape before deciding whether to retrofit Feature specs. A deep retrofit pass is too expensive and too sensitive to be the first touch. Users need a bounded survey that can be reviewed, committed, and passed downstream without exposing source content.

## Behavior

### Invocation and scope

#### REQ: invocation-triggers

The skill MUST respond to `specstudio:survey`, `/survey`, `/specstudio:survey`, "survey this repo", "architecture survey", and "map this repo". The "map this repo" phrasing is accepted as a compatibility alias; all output names and user-facing labels MUST use `survey`.

#### REQ: no-source-reads

The skill MUST NOT read source implementation files. Source implementation files include, but are not limited to, `*.js`, `*.jsx`, `*.ts`, `*.tsx`, `*.py`, `*.go`, `*.rs`, `*.java`, `*.cs`, `*.rb`, `*.php`, `*.swift`, and `*.kt`, except when a file path is explicitly listed in the structural manifest allowlist. The skill MAY list source file paths and count them. It MUST NOT open their contents.

#### REQ: allowed-inputs

The skill MAY read only:

- The repository file tree from `git ls-files` or a non-git fallback.
- Structural manifests named in `REQ:manifest-allowlist`.
- Root-level documentation files named in `REQ:manifest-allowlist`, subject to size caps.

Any attempted read outside this set is a contract violation unless the user explicitly switches to a future deeper skill such as `retrofit`.

#### REQ: manifest-allowlist

The skill MUST restrict content reads to the v1 allowlist in [`skills/shared/survey-manifest-allowlist.md`](../../../../skills/shared/survey-manifest-allowlist.md). That shared file is the operational source of truth used by the skill; this Feature owns the contract that the list is intentionally narrow and structural.

| Category | Allowed files |
|---|---|
| JavaScript / TypeScript | `package.json`, `pnpm-workspace.yaml`, `lerna.json`, `nx.json`, `turbo.json`, `tsconfig*.json`, `next.config.*`, `vite.config.*`, `nuxt.config.*`, `astro.config.*`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock` |
| Python | `pyproject.toml`, `setup.py`, `setup.cfg`, `requirements*.txt`, `poetry.lock`, `Pipfile`, `Pipfile.lock`, `tox.ini` |
| Go | `go.mod`, `go.sum`, `go.work` |
| Rust | `Cargo.toml`, `Cargo.lock` |
| Other languages | `Gemfile`, `composer.json`, `*.csproj`, `*.sln`, `pom.xml`, `build.gradle*` |
| Infra and ops | `docker-compose*.yml`, `Dockerfile`, `terraform/*.tf`, `serverless.yml`, `helm/Chart.yaml`, `kustomization.yaml` |
| CI and tooling | `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`, `Makefile`, `justfile`, `Taskfile.yml` |
| Release and packaging | `.goreleaser.yml`, `.goreleaser.yaml`, `release-please-config.json`, `.release-please-manifest.json`, `.releaserc`, `.releaserc.json`, `.changeset/config.json` |
| Runtime pinning | `.tool-versions`, `.nvmrc`, `.python-version`, `.ruby-version` |
| SpecScore | `specscore.yaml` |
| Docs | repo-root `README*`, `ARCHITECTURE*`, `CONTRIBUTING*`, and top-level `docs/**` filenames; docs content reads are size-capped |

The skill MUST summarize any skipped allowlisted file whose content exceeds the size cap as "skipped due to size" rather than reading it partially without disclosure.

### Repository scan

#### REQ: file-tree-scan

The skill MUST prefer `git ls-files` for the file inventory. When the repository is not a git repository, the skill MAY fall back to `find`, excluding `.git`, `node_modules`, virtualenv directories, dependency caches, build outputs, and other obvious generated directories. The scan output MUST record which method was used.

#### REQ: repo-state-record

The JSON artifact MUST include the current `HEAD` SHA when available, a dirty-tree boolean, and a digest or literal capture of `git status --short`. When git metadata is unavailable, the JSON MUST explicitly record `git_available: false`.

#### REQ: sensitive-path-inventory

The skill MUST include a filename-pattern-only sensitive-path inventory. It MUST flag paths matching `.env*`, `secrets/**`, `*.pem`, `*.key`, fixture directories, `git-crypt` markers, submodule entries, and LFS pointer-looking files by filename or manifest signal. The survey MUST state that this is not a content secret scan and that downstream `retrofit` owns content scanning.

#### REQ: monorepo-v1-refusal

When manifest signals indicate a monorepo or multi-workspace repository, the skill MUST refuse a whole-repo survey and recommend rerunning with `--scope <subdir>`. The refusal MUST identify the signals that triggered it. In v1, the skill MUST NOT synthesize a single multi-architecture survey.

### Output artifacts

#### REQ: output-paths

The skill MUST write both artifacts under `spec/research/` by default:

- `spec/research/<slug>-survey.json`
- `spec/research/<slug>-survey.md`

The slug MUST be derived from the root manifest name when available, falling back to the repository directory name. The skill MAY accept `--slug <slug>` to override ambiguous derivation. The output directory MAY be overridden by `--output-dir <path>`, but the default remains `spec/research/`.

#### REQ: json-first

The JSON artifact is the source of truth. The Markdown artifact MUST be rendered from the JSON content, not independently invented. If the Markdown and JSON disagree, the JSON wins and the Markdown renderer is wrong.

#### REQ: survey-json-fields

The JSON artifact MUST include, at minimum:

- `schema`: literal `survey-output-schema-v1`.
- `slug`.
- `repo_state`.
- `scan_method`.
- `scope`.
- `file_inventory_summary`.
- `manifest_inventory`.
- `detected_frameworks`.
- `architecture_summary`.
- `directory_clusters`.
- `research_zones`.
- `sensitive_path_inventory`.
- `warnings`.

Future fields MAY be added without breaking v1 consumers.

#### REQ: markdown-shape

The Markdown artifact MUST contain, in order:

1. H1 `# Survey: <title>`.
2. Body metadata lines for `Status`, `Date`, `Repo SHA`, `Scope`, and `JSON`.
3. `## Summary`.
4. `## Architecture`.
5. `## Directory Clusters`.
6. `## Research Zones`.
7. `## Detected Frameworks`.
8. `## Sensitive Path Inventory`.
9. `## Warnings`.
10. `## Open Questions`.
11. SpecScore research-artifact footer.

The Markdown MAY include Mermaid diagrams, but it MUST remain useful when Mermaid is not rendered.

#### REQ: research-index

When `spec/research/README.md` does not exist, the skill MUST create a lint-clean index for the research directory. When it already exists, the skill MUST add or update a row for the generated survey. The index update MUST be staged with the generated artifacts.

### Cost and safety boundaries

#### REQ: cost-ceiling

Before any LLM synthesis, the skill MUST estimate whether the collected structural input fits the configured cost ceiling. The default ceiling is the equivalent of a small single synthesis pass. If the estimate exceeds the ceiling, the skill MUST either narrow scope with user confirmation or refuse with a recommendation to rerun with `--scope <subdir>`.

#### REQ: no-midflow-approval

The skill MUST NOT require user approval between the scan and synthesis phases. Survey output is read-only research, not code or spec canon. The user reviews the generated artifacts after the run.

#### REQ: staging

After writing and linting, the skill MUST stage the generated survey artifacts and the research index via `git add`. It MUST NOT commit.

#### REQ: lint

The skill MUST run `specscore spec lint` after writing the artifacts. If lint fails, the skill MAY attempt one focused fix pass and rerun lint. Remaining violations MUST be surfaced to the user with the affected paths.

## Architecture

- **Scanner:** Builds the file inventory, reads only allowlisted manifests, records repo state, and flags sensitive paths by filename pattern.
- **Synthesizer:** Converts the structured scanner output into architecture summary, directory clusters, detected frameworks, and research zones.
- **Renderer:** Writes JSON first and renders Markdown from that JSON.
- **Indexer:** Creates or updates `spec/research/README.md`.

## Interaction with Other Features

| Feature | Relationship |
|---|---|
| [Retrofit Skill](../../../ideas/retrofit-skill.md) | `retrofit` consumes `survey-output-schema-v1` and uses the survey's research zones as its starting point. |
| [Init Skill](../init/README.md) | `survey` is useful for brownfield repositories after `init` bootstraps the `spec/` tree. |
| [Skills](../README.md) | `survey` is a shipped adoption skill outside the forward lifecycle spine. |

## Acceptance Criteria

### AC: no-source-file-content-read

**Given** a repository containing `src/app.ts`, `src/app.test.ts`, `package.json`, and `README.md`,
**When** the user runs `specstudio:survey`,
**Then** the skill MAY list and count `src/app.ts` and `src/app.test.ts`, MUST read `package.json`, MAY read size-capped `README.md`, and MUST NOT read the contents of either TypeScript source file.

### AC: writes-json-and-markdown

**Given** a single-package repository with a clear package name,
**When** the user runs `specstudio:survey`,
**Then** `spec/research/<slug>-survey.json` MUST exist with `schema: survey-output-schema-v1`, `spec/research/<slug>-survey.md` MUST exist, and the Markdown MUST reference the JSON artifact path.

### AC: records-repo-state

**Given** a git repository with a dirty working tree,
**When** the user runs `specstudio:survey`,
**Then** the JSON MUST record the current `HEAD` SHA, `dirty_tree: true`, and the status entries used to determine dirtiness.

### AC: monorepo-refuses-whole-repo

**Given** a repository with `pnpm-workspace.yaml` or `nx.json` indicating multiple workspace members,
**When** the user runs `specstudio:survey` without `--scope`,
**Then** the skill MUST refuse to synthesize a whole-repo survey, identify the monorepo signal, and recommend rerunning with `--scope <subdir>`.

### AC: sensitive-paths-are-hints

**Given** a repository containing `.env.example`, `secrets/demo.key`, and `tests/fixtures/customer.json`,
**When** the user runs `specstudio:survey`,
**Then** the survey MUST list those paths under `sensitive_path_inventory` and MUST label the inventory as filename-pattern hints, not a content secret scan.

### AC: research-index-updated

**Given** no `spec/research/README.md` exists,
**When** the user runs `specstudio:survey`,
**Then** the skill MUST create a lint-clean research index and add a row for the generated survey artifacts.

**Given** `spec/research/README.md` already exists,
**When** the user runs `specstudio:survey` again for the same slug,
**Then** the skill MUST update or replace the existing row for that slug rather than duplicate it.

### AC: lint-and-stage

**Given** a successful survey run,
**When** the run completes,
**Then** `specscore spec lint` MUST pass, the generated JSON, Markdown, and research index MUST be staged with `git add`, and the skill MUST NOT create a commit.

## Not Doing

- Source-code behavior inference. That belongs to `retrofit`.
- Content secret scanning. Survey only produces filename-pattern hints.
- Whole-monorepo synthesis. v1 requires `--scope <subdir>` for monorepos.
- New `specscore survey` CLI command. The skill proves the output shape first.
- Incremental re-surveying or cache reuse. MVP rescans each time.

## Open Questions

- Should `spec/research/` become a first-class linted artifact type with its own schema footer?
- Should `survey-output-schema-v1` move into shared schema documentation before retrofit ships?
- What exact token and cost estimate should define the default cost ceiling?

---
*This document follows the https://specscore.md/feature-specification*
