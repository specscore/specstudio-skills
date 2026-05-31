# Survey Manifest Allowlist

This file is the operational v1 content-read allowlist for `specstudio:survey`.
Survey may list and count any tracked file path, but it may read file contents only
for paths covered here and only under the skill's size cap.

## JavaScript / TypeScript

- `package.json`
- `pnpm-workspace.yaml`
- `lerna.json`
- `nx.json`
- `turbo.json`
- `tsconfig*.json`
- `next.config.*`
- `vite.config.*`
- `nuxt.config.*`
- `astro.config.*`
- `package-lock.json`
- `pnpm-lock.yaml`
- `yarn.lock`

## Python

- `pyproject.toml`
- `setup.py`
- `setup.cfg`
- `requirements*.txt`
- `poetry.lock`
- `Pipfile`
- `Pipfile.lock`
- `tox.ini`

## Go

- `go.mod`
- `go.sum`
- `go.work`

## Rust

- `Cargo.toml`
- `Cargo.lock`

## Other Languages

- `Gemfile`
- `composer.json`
- `*.csproj`
- `*.sln`
- `pom.xml`
- `build.gradle*`

## Infra And Ops

- `docker-compose*.yml`
- `Dockerfile`
- `terraform/*.tf`
- `serverless.yml`
- `helm/Chart.yaml`
- `kustomization.yaml`

## CI And Tooling

- `.github/workflows/*.yml`
- `.gitlab-ci.yml`
- `.circleci/config.yml`
- `Makefile`
- `justfile`
- `Taskfile.yml`

## Release And Packaging

- `.goreleaser.yml`
- `.goreleaser.yaml`
- `release-please-config.json`
- `.release-please-manifest.json`
- `.releaserc`
- `.releaserc.json`
- `.changeset/config.json`

Generated release outputs remain excluded. In particular, survey must not read
`dist/**`, packaged archives, generated checksums, generated changelogs, or
binary artifacts as release manifests.

## Runtime Pinning

- `.tool-versions`
- `.nvmrc`
- `.python-version`
- `.ruby-version`

## SpecScore

- `specscore.yaml`

## Docs

- Root `README*`
- Root `ARCHITECTURE*`
- Root `CONTRIBUTING*`
- Top-level `docs/**` filenames

Docs content reads remain size-capped. Top-level docs may be read only when
they are small enough under the survey size cap; otherwise record a skipped
warning rather than reading partially.
