# Idea: Map Skill — Fast Architecture Summary & Research Zone Proposal

**Status:** Draft
**Date:** 2026-05-22
**Owner:** alex
**Promotes To:** —
**Supersedes:** —
**Related Ideas:** —

## Problem Statement

How might we give engineers a fast, low-cost architectural overview of any codebase — derived purely from filesystem topology and structural manifests, never source — that is valuable as a standalone artifact AND consumable as input to deeper specstudio skills like retrofit?

## Context

Born from the pre-approval review of `retrofit-skill` (see `spec/ideas/retrofit-skill.md` Review Notes). The Developer Advocate flagged that retrofit's activation moment landed too late — users invested through three phases before any value, and abandonment risk was high. The accepted alternative was to carve off retrofit's upstream skeleton-scan phase as a standalone skill. `map-skill` is that carve-out. Family context: `map-skill` (this) → `retrofit-skill` (opt-in deep path) → `retrofit-evaluation-skill` (internal quality loop). Map is the activation hook; the other two depend on it.

## Recommended Direction

A new specstudio skill — `specstudio:map` — that runs entirely on file tree plus an allowlist of structural manifests, never source files. Two phases.

**Phase 1 — Topology scan** (deterministic, no LLM). `git ls-files` for the file tree (depth-capped, `.gitignore`-respecting), with non-git fallback to `find`. Reads only the allowlist (see below). Output is a **structured intermediate** (in-memory JSON): file inventory, manifest contents parsed, framework signals detected, sensitive-path matches enumerated, repo-state (HEAD SHA + `git status --porcelain` digest, dirty-tree flag). This phase is unit-testable without an LLM in the loop — that's the whole point of separating it from Phase 2.

**Phase 2 — Synthesis** (LLM, consumes the Phase 1 intermediate). Produces a **JSON artifact first** (schema-validated, `<slug>-map.json`) containing every field downstream consumers need; then **deterministically renders the markdown narrative from that JSON** (`<slug>-map.md`). This direction prevents drift: there is no scenario where the markdown says one thing and the JSON another, because the markdown is a view over the JSON. Both files land at `spec/research/` (default; configurable). The JSON contract is the named cross-Idea contract `map-output-schema-v1`, owned jointly with `retrofit-skill`.

**Allowlist of structural manifests** (versioned at v1; expansion via documented PR):
- **JavaScript/TypeScript:** `package.json`, `pnpm-workspace.yaml`, `lerna.json`, `nx.json`, `turbo.json`, `tsconfig*.json`, `next.config.{js,ts,mjs}`, `vite.config.*`, `nuxt.config.*`, `astro.config.*`, lockfiles (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`).
- **Python:** `pyproject.toml`, `setup.py`, `setup.cfg`, `requirements*.txt`, `poetry.lock`, `Pipfile`, `Pipfile.lock`, `tox.ini`.
- **Go:** `go.mod`, `go.sum`, `go.work`.
- **Rust:** `Cargo.toml`, `Cargo.lock`, workspace member manifests.
- **Other languages (signal only):** `Gemfile`, `composer.json`, `*.csproj`, `*.sln`, `pom.xml`, `build.gradle*`.
- **Infra & ops:** `docker-compose*.yml`, `Dockerfile`, `terraform/*.tf`, `serverless.yml`, `helm/Chart.yaml`, `kustomization.yaml`.
- **CI & tooling:** `.github/workflows/*.yml`, `.gitlab-ci.yml`, `.circleci/config.yml`, `Makefile`, `justfile`, `Taskfile.yml`.
- **Runtime pinning:** `.tool-versions`, `.nvmrc`, `.python-version`, `.ruby-version`.
- **Docs:** repo-root `README*`, `ARCHITECTURE*`, `CONTRIBUTING*`, top-level `docs/**` (file names and tree only at v1; content reads are size-capped).

**Output artifacts:**
1. **JSON bundle** (`<slug>-map.json`) — schema `map-output-schema-v1`. Source of truth. Carries structured graph data (nodes, edges, hierarchies) suitable for any rendering target.
2. **Markdown narrative** (`<slug>-map.md`) — deterministic render of (1). What humans read. Diagrams embedded as **Mermaid** code blocks (no user prompt — Mermaid is the default and renders inline in GitHub/GitLab/VSCode/mkdocs where SpecScore artifacts already live, is diffable in git, and is LLM-generatable with high reliability). Specifically: directory tree as Mermaid `graph TD` or markdown nested list (whichever renders better for the depth), architecture overview as Mermaid `flowchart LR`, framework-dependency relationships as Mermaid `graph`. ASCII fallback is unnecessary at the current state of tooling; SVG is overkill for a 2-minute synthesis tool and harms diffability.

The JSON contains the architecture summary, detected frameworks, directory-cluster proposal with one-line descriptions and file-count budgets, **sensitive-path inventory (filename-pattern-based only — explicitly NOT content-scanned; downstream `retrofit-skill` runs the actual content scanner)**, and a repo-state record (HEAD SHA, dirty-tree flag, `git status --porcelain` digest). Two render modes: full markdown narrative (default) or JSON-only via `--json` (skips the markdown render for tool consumers).

**Directory clusters are emitted as a hierarchical proposal**, not a flat list — because SpecScore Features are hierarchical (`spec/features/<parent>/<child>/`, used for composition and inheritance) and the natural codebase hierarchy is the strongest signal for proposing parent/child Feature structure downstream. A cluster like `src/auth/` with subdirectories `login/`, `logout/`, `reset/` is emitted as parent "Auth" with proposed children "Login", "Logout", "Password Reset" — and retrofit-skill consumes this hierarchy as its `proposed_parent` / `proposed_children` hint for researchers. Map is not authoritative on the final hierarchy (researchers may reorganize based on what they observe inside the source), but its proposal is the starting tree. **Cluster shape is bounded** to prevent noise: max depth 3 levels; max 12 children per parent; overflow buckets into a synthetic "other" child labeled with file-count so the user sees what was dropped. These bounds are configurable but conservative defaults.

**Renderer purity contract.** The markdown renderer that converts `<slug>-map.json` to `<slug>-map.md` is a **pure function**: no clock reads, no locale-dependent formatting, no nondeterministic map iteration (sorted keys), no embedded timestamps beyond what's in the JSON. Same JSON in → byte-identical markdown out, on any machine, on any day. This is what makes the "JSON is source of truth, markdown is a view" discipline enforceable; CI checks the property by re-rendering and diffing.

**Cost-ceiling enforcement under hierarchical summarization.** When the structured intermediate exceeds the token budget and Phase 2 falls back to hierarchical summarization (N cluster-summaries + 1 meta-summary = N+1 LLM calls), the cost ceiling applies to the **sum of all projected calls**, not just the first. Pre-flight estimator computes `sum(estimated_input_tokens(call_i))` and refuses if the total exceeds the ceiling. Cost ceiling is not "ceiling per call" — that would silently 10× the spend on hierarchical paths.

**Target runtime ~2 minutes for repos up to 10k files; not a hard guarantee at larger scales.** Scaling strategy: Phase 1 is sub-second up to 100k files (`git ls-files` is fast). Phase 2's LLM call is the bottleneck. If the structured intermediate exceeds a token budget (default 50k input tokens; configurable), the synthesis falls back to **hierarchical summarization**: cluster directories by top-level, summarize each cluster independently, then synthesize a meta-summary. Monorepos beyond 50k files trigger a **degrade-to-partial-map mode** that returns a top-level-only summary plus a "scope this to a subdirectory for full map" recommendation. Per-run **cost ceiling** is a hard limit (default $0.50 input; configurable via `--max-cost`); the skill refuses to proceed past the ceiling rather than burning a budget silently.

**Monorepo handling** is **out of scope for v1**. v1 maps assume a single top-level architecture. Detected monorepos (turbo/nx/lerna/pnpm-workspaces/go-workspaces) trigger a **refuse-and-recommend** path: "this is a monorepo; re-run with `--scope <subdir>` to map a workspace member." Multi-architecture maps are a follow-on Idea.

No user-approval gates mid-flow — the artifact is read-only synthesis; users review afterwards. The skill warns rather than refuses on dirty working trees (a dirty-tree map is still useful, just stamped as such).

## Alternatives Considered

- **Bake map functionality into retrofit (the original monolithic shape).** Rejected — the Developer Advocate review on `retrofit-skill` showed the activation moment landed too late and standalone value was high. Carving out `map` addresses both.
- **Build a `specscore tree` or `specscore scan` CLI command instead of a skill.** Rejected for now — we do not yet know the stable output shape (markdown vs JSON, what consumers actually want). Building the CLI first is speculative; ship the skill, observe usage, extract a CLI later if a stable pattern emerges. The skill is a thin orchestrator over `git ls-files` + `cat` + LLM synthesis for MVP.
- **Read source files lightly to enrich the architecture summary.** Rejected — the moment you read source, you become retrofit; speed and safety properties collapse. The hard line between "topology + manifests only" (map) and "reads source" (retrofit) is the whole product distinction.
- **Output multiple competing maps (different abstraction levels).** Rejected — produces one canonical map per run; users can re-invoke with different parameters if they need a different view. One canonical artifact, one canonical format.
- **Generate the map purely deterministically from heuristics, no LLM synthesis.** Considered. Pure-deterministic output would be reproducible but produces inventory, not insight. LLM synthesis on the topology + manifests is the value-add — it turns "here are 47 directories" into "this is a Next.js app with a Prisma DB layer, an Express webhook handler, and a separate worker pool." Kept LLM synthesis with a tight prompt.

## MVP Scope

A two-week spike. Single-repo only. JavaScript/TypeScript and Python detection in the framework-glossary inference; other languages get a 'detected but not deeply classified' label. Output to `spec/research/<slug>-map.md` with JSON sidecar. Success criterion (falsifiable): produce a map for `specstudio-skills` plus one external repo of comparable size, and have a human reviewer rate ≥80% of the architecture summary and zone proposal as accurate. No multi-repo, no caching, no Synchestra events in MVP — those are explicit follow-ons.

## Not Doing (and Why)

- Reading source files — the entire speed and safety property hinges on this; reading source files turns map into retrofit.
- LLM-based deep classification of file contents beyond structural-manifest parsing — keeps the runtime budget and avoids hallucination.
- Multi-repo aggregation — defer to follow-on once single-repo proves out; retrofit-skill's multi-repo wizard will invoke map once per repo for now.
- Cost estimation for downstream retrofit token spend — move that concern entirely to retrofit; map produces the inputs (file/zone counts, framework markers), retrofit computes the dollars.
- Persistent caching across runs or incremental re-mapping — defer; MVP re-scans every invocation.
- Output to multiple destinations or formats beyond markdown + JSON sidecar — one canonical location, one canonical format; add formats only when a consumer needs one.
- Multi-architecture monorepo maps — v1 refuses and recommends scoping to a workspace member; multi-architecture mapping is a follow-on Idea.
- Content-scanning the sensitive-path inventory — map produces filename-based heuristic hints only; the actual content scan (gitleaks/trufflehog) belongs to `retrofit-skill`'s Phase 1 security gate. Map's inventory is *hints*, not a security guarantee.
- Operating past the per-run cost ceiling — the skill refuses rather than burning budget silently. Users can override via `--max-cost` but the default is a hard refuse, not a warn.

## Key Assumptions to Validate

| Tier | Assumption | How to validate |
|------|------------|-----------------|
| Must-be-true | File tree + structural manifests alone produce an architecture summary a human reviewer accepts as accurate at ≥80% | Run on 3 representative repos; human grades the summary against ground truth |
| Must-be-true | The standalone map artifact is valuable enough that users would invoke `specstudio:map` *without* intending to run retrofit afterward | Ask 5 candidate users to run map standalone; measure follow-on retrofit invocation rate AND ask them whether they'd run map again for a different repo |
| Must-be-true | JSON-first / markdown-rendered-from-JSON eliminates the drift risk between the two artifacts. The data flow MUST be (1) Phase 2 produces JSON conforming to `map-output-schema-v1`, (2) markdown is deterministically rendered from the JSON. There is no scenario where the LLM generates both independently | Spike both flows; verify the renderer is pure (same JSON → same markdown byte-for-byte); add a CI check that markdown matches the rendered JSON |
| Must-be-true | ~2-minute runtime is achievable for typical repos (≤10k files, ≤500MB) without exceeding LLM context budgets | Measure runtime on 3 representative repos; if any exceed 5 minutes, the hierarchical-summarization fallback or stricter depth caps must engage |
| Must-be-true | Hierarchical summarization fallback produces a meta-summary that humans still rate accurate at ≥75% (lower bar than direct synthesis because some fidelity loss is acceptable when the alternative is failure) | Force the fallback path on a 50k-file repo; human-grade the resulting summary |
| Must-be-true | Monorepo refuse-and-recommend correctly detects monorepos via manifest signals (turbo/nx/lerna/pnpm-workspaces/go-workspaces/Cargo workspaces) without false positives on single-architecture repos that happen to have workspace markers for other reasons | Audit detection on 5 monorepos and 5 non-monorepos; measure precision/recall |
| Must-be-true | The per-run cost ceiling (default $0.50) is enforceable before the LLM call begins — i.e. input tokens are estimated, compared to ceiling, and the run refuses *before* spending. Post-call enforcement is too late | Wire the estimator; verify refusal triggers on a synthetic oversized input |
| Should-be-true | The allowlist of structural manifests is comprehensive enough for the common stacks (JS/TS, Python, Go, Rust) without expanding per release | Audit 10 real repos; count how often inferred architecture is wrong because a non-allowlisted manifest existed |
| Should-be-true | The Phase 2 sensitive-path inventory (filename-pattern hints only) is useful as a seed for retrofit's Phase 1 content-scanner gate — i.e. flagging files is enough; map does not need to read them | Verify retrofit's Phase 1 consumes the inventory without re-scanning the file tree from scratch |
| Might-be-true | Monorepos with multiple top-level package roots produce sensible single maps without needing a separate workflow | Spike on one real monorepo (e.g. a turborepo project); see whether one map suffices or multiple are needed |
| Might-be-true | The output location `spec/research/<slug>-map.md` is the right home — vs `spec/maps/`, `spec/scans/`, or a non-tracked location | Defer; decide at spec time based on whether maps should be committed at all |


## SpecScore Integration

- **New Features this would create:** `map-skill` (one parent Feature, likely subdivided at spec time into `topology-scan` and `architecture-synthesis` matching the two phases).
- **Existing Features affected:** None directly. Unblocks `retrofit-skill` (hard dependency). May later be consumed by `init` (existing-project detection) and `verify` (drift detection); both are out of scope here.
- **Dependencies:**
  - `git ls-files` (preferred) with `find` fallback for non-git working trees.
  - Existing `specscore` CLI for spec-tree validation (`specscore spec lint` applies to the output index, not the map artifact itself in MVP — see Open Questions on whether map artifacts get a lint contract).
  - No new CLI surface in MVP.

## Open Questions

**Assumed choices, recorded for review** (best-judgment defaults made because the user is away; flag any to override):
- *Diagram format.* Default Mermaid (no user prompt). Rationale: renders inline where SpecScore artifacts live (GitHub/GitLab/VSCode/mkdocs), diffable, LLM-generatable. ASCII fallback only on explicit user request. SVG rejected: harms diffability, LLMs produce SVG poorly, overkill for this use case. *(Already folded into Recommended Direction.)*
- *Output location — DECIDED.* `spec/research/<slug>-map.md` (committed). Pitch is "shareable architecture summary" — that property only works if it's a real, reviewable artifact in the spec tree. Staleness is detectable via the repo-state record (HEAD SHA + dirty-tree flag) embedded in the JSON; a stale map produces a visible warning on next consumer read, and the user re-runs.
- *Slug derivation.* Assumed the slug is derived from the repo's `package.json#name` / `pyproject.toml` `[project].name` / `go.mod` module name, with fallback to the directory name. Sanity-stripped to a SpecScore slug.
- *JSON sidecar location.* Assumed colocated as `<slug>-map.json` next to the markdown.
- *Map artifact lint contract.* Assumed map artifacts are **not** subject to `specscore spec lint` (they're research artifacts, not first-class spec artifacts like Feature/Plan/Idea). If retrofit needs the JSON to conform to a schema, that schema is validated by retrofit on ingestion, not by `specscore` lint.
- *Frontmatter / metadata.* Assumed map artifacts use the same body-metadata pattern as Ideas (no YAML frontmatter), with `**Status:**`, `**Date:**`, `**Repo SHA:**` lines. Note that the existing Idea schema uses `Owner` etc. — maps need a smaller set since they're auto-generated.

**Genuine opens (no assumed answer):**
- Slug derivation fallback chain — when no manifest names exist, when manifests disagree, when the repo is docs-only or shell-only. Proposal: `package.json#name` > `pyproject.toml [project].name` > `go.mod` module > `Cargo.toml [package].name` > `basename $(git rev-parse --show-toplevel)`, slugified. On manifest disagreement, prefer the root-level manifest; if multiple at root, refuse with a "pick one via `--slug <name>`" message.
- Two-phase interface contract — Phase 1's structured intermediate is in-memory JSON; should it be exposed as a `--dump-intermediate` flag for debugging, or kept strictly internal? Argues for: testability, transparency. Argues against: yet another artifact to version.
- Monorepo-detection false-negative path — when detection fails and a polyglot monorepo is treated as single-architecture, the resulting map will be garbled. Proposed mitigation: emit a confidence signal on monorepo detection; if multiple workspace-root manifests surface mid-scan despite no top-level workspace marker, escalate to refuse-and-recommend. Confidence-threshold and detection rules deferred.
- `--scope <subdir>` × repo-state-capture interaction — when scoped to a subdir, does the HEAD SHA reflect whole-repo or scoped subtree? Proposal: whole-repo SHA + the scoped path is captured as a separate field so downstream tools can distinguish "this map covers a subset of repo at SHA X." Confirm at spec time.
- ~~How does map handle monorepos~~ *(answered in design: v1 refuses-and-recommends; multi-architecture is a follow-on Idea)*
- File-staleness signal — when is a map "out of date"? Date alone is weak (the repo might not have changed). Tree-hash check, commit SHA at generation, or both?
- Should map emit a Synchestra event (`map.produced`)? For what consumer? Retrofit reads the file directly today; an event only matters if something else listens.
- Map output is not currently a "managed artifact" in the SpecScore lifecycle sense — does it need a lifecycle (Draft → Approved → Stale)? Or is it always just "freshly generated"?
- Privacy: maps disclose architectural details and may expose internal service names, framework choices, infra topology. For a public open-source repo, fine. For internal codebases shared via this artifact, the user needs to know what gets disclosed. UX implication: a default `.gitignore` rule on map output? Or explicit "this map will be committed unless you `gitignore` it" warning?
- Cost-budget enforcement — what happens when a repo is too large for the 2-minute target? Refuse, downgrade to a partial map, or warn and proceed? MVP probably warns and proceeds with a partial-map indicator.
- LLM model selection — does map pin to a specific model tier (e.g. Haiku for speed) or use the same model as the calling agent? Cheaper/faster model is tempting given map is the activation-hook and frequent-use surface.
- Should map ever be re-run incrementally (delta against the previous map) once caching exists post-MVP, or always full-scan?

---
## Review Notes

This Idea was drafted in an autonomous-mode pass after the user stepped away. It has not yet been through a pre-approval reviewer panel; that pass happens after the sibling Ideas (`retrofit-skill`, `retrofit-evaluation-skill`) are also drafted, so a cross-cutting family review can run alongside per-Idea reviews. Findings will be folded back in or recorded as additional Open Questions before user approval.

---
*This document follows the https://specscore.md/idea-specification*
