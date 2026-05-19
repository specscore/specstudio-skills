# Feature: Third-Party Skill Integration

> [View in SpecScore Studio](https://specscore.studio/project/features?id=specstudio-skills@specscore@github.com&path=spec%2Ffeatures%2Fthird-party-integration) — graph, discussions, approvals

**Status:** Approved
**Source Ideas:** third-party-skill-integration-contracts

## Summary

Defines the contract for integrating third-party agent skills (Superpowers `brainstorming`, addyosmani `idea-refine`, future reviewers and capabilities) with SpecScore artifacts. Classifies every third-party skill into one of three shapes — **Producer**, **Reviewer**, **Capability** — and pins normative MUST / SHOULD / MUST-NOT requirements per shape. Reviewers and Capabilities become first-class integration paths with concrete contracts. Shape-3 third-party Producers (skills that write canonical SpecScore Idea or Feature artifacts directly) are an explicit non-goal: their output is draft input to `specstudio:ideate` or `specstudio:specify`, never a finished artifact. Closes the `reviewer-extension-hook` registration Outstanding Question in [`specify`](../skills/specify/README.md).

## Problem

SpecStudio adopters increasingly run third-party agent skills (Superpowers, addyosmani agent-skills, etc.) alongside SpecStudio in the same repository. Without an explicit contract, three failure modes surface: (1) third-party Producer-class skills write artifacts that look canonical but fail `specscore spec lint` and lack lifecycle integration (events, status transitions, Synchestra reconciliation); (2) potential Reviewer-class subagents have no registration path even though [`skills/specify`](../skills/specify/README.md)'s `reviewer-extension-hook` already specifies an AND-composition contract for them; (3) Capability-class tools (visual companions, diagram renderers, accessibility auditors) have no defined invocation surface, so each integration becomes an ad-hoc adapter.

This Feature replaces ad-hoc handling with a typed contract that adopters and skill authors can target.

## Behavior

### Three-shape taxonomy and Producer non-goal

Every third-party agent skill SpecStudio interoperates with classifies into exactly one shape. Shape determines the integration mechanism, not the other way around.

#### REQ: shape-classification

Every third-party agent skill SpecStudio interoperates with MUST be classified as exactly one of: **Producer** (writes a draft artifact intended to be canonicalized by SpecStudio later), **Reviewer** (returns a structured verdict on a candidate artifact), or **Capability** (emits a static asset or runs a runtime tool consumed by SpecStudio). A skill that does not fit any of the three shapes MUST be rejected as out of scope for integration; the contract does not stretch to accommodate novel shapes via reinterpretation.

#### REQ: producer-non-goal

Third-party agent skills MUST NOT write artifacts directly into the canonical SpecScore tree (`spec/ideas/`, `spec/features/`, `spec/plans/`, `spec/research/`) or modify existing canonical artifacts there. The canonical producers are `specstudio:ideate` (Ideas) and `specstudio:specify` (Features); future canonical producers (e.g., `specstudio:plan`) are added by this skill set, not by third parties. A Producer-shape third-party skill's output is **always draft input** — its destination is the user's working dialogue with `specstudio:ideate` or `specstudio:specify`, which canonicalize the draft into a SpecScore artifact.

### Producer shape — instruction injection

The Producer mechanism is a canonical instruction snippet that adopters paste into their platform agent-instructions file. The snippet shapes Producer skill behavior so its output is closer to canonical SpecScore form before handoff, but does not replace the canonicalization step. The contract is platform-agnostic by virtue of the snippet being platform-neutral text — per-platform skill-runtime semantics (how Claude Code, Codex, Cursor, Gemini CLI each load and dispatch skills) are out of scope, and the platform-detection logic for which agent-instructions file to update when multiple are present is delegated to [`specstudio:init`](../../ideas/specstudio-init-skill.md).

#### REQ: producer-canonical-snippet

SpecStudio MUST maintain a single canonical Producer-shape instructions block at `spec/features/third-party-integration/snippet.md`. The snippet is a Feature asset — versioned with this Feature, lintable as part of the Feature directory, and the source of truth that all paste targets reflect. Drift between the canonical snippet and pasted copies in adopter agent-instructions files is detectable via the embedded version comment per `producer-snippet-versioning`; reconciliation of detected drift (re-paste, three-way merge, or deferred update) is owned by `specstudio:init --update`, not by this Feature.

#### REQ: producer-snippet-versioning

The canonical snippet at `snippet.md` MUST carry an embedded version comment as its first content line: `<!-- specstudio-snippet-version: <semver> -->`. The version increments per the contract-versioning rules below — patch for editorial fixes, minor for additive content, major for breaking semantics. Pasted copies in adopter agent-instructions files preserve this comment; consumers (notably `specstudio:init --update`) compare the pasted version against the canonical to detect drift.

#### REQ: producer-paste-targets

The snippet MUST be platform-neutral text (no `Skill` tool calls, no slash-command syntax, no platform-specific tool invocation). Documented paste targets: `CLAUDE.md` (Claude Code), `AGENTS.md` (Codex and equivalents), `GEMINI.md` (Gemini CLI), files under `.cursor/rules/` (Cursor). Adopters select the target their primary agent reads. When a repo carries both `CLAUDE.md` and `AGENTS.md`, the platform-detection rule for which file to update belongs to [`specstudio:init`](../../ideas/specstudio-init-skill.md), not to this Feature.

#### REQ: producer-output-is-draft

A third-party Producer-shape skill's output MUST be treated as draft input only. The snippet text MUST instruct the third-party skill to (a) use SpecScore-canonical paths and section structure, (b) use canonical body-metadata header fields, (c) use `Given / When / Then` for any acceptance-criterion-shaped content, and (d) explicitly suggest the user run `specstudio:ideate` or `specstudio:specify` to canonicalize when complete. The snippet MUST NOT instruct the third-party skill to bypass the canonical producers, MUST NOT instruct it to write directly to `spec/ideas/` or `spec/features/`, and MUST NOT instruct it to commit on the user's behalf.

#### REQ: producer-handoff

The snippet MUST require the third-party Producer skill to surface an explicit handoff prompt at the end of its output, naming the canonical follow-up skill (`specstudio:ideate` for Idea-shaped drafts, `specstudio:specify` for Feature-shaped drafts). The handoff prompt is what `specstudio:init --update` uses to confirm a snippet pasted in the wild is current and intact; absence of the handoff prompt in the produced output indicates either snippet drift or a third-party skill ignoring the snippet — both are recoverable user-visible errors, not silent contract violations.

### Reviewer shape — registration hook

Reviewers register through a single declarative mechanism that integrates with the canonical SpecScore Repo Config. Closes the `reviewer-extension-hook` Outstanding Question in [`skills/specify`](../skills/specify/README.md).

#### REQ: reviewer-registration-mechanism

Reviewers MUST register through a `reviewers:` extension key inside the project's `specscore.yaml`. This relies on the canonical [SpecScore Repo Config Feature](https://github.com/synchestra-io/specscore/blob/main/spec/features/repo-config/README.md)'s `unknown-fields-preserved` requirement — orchestration tools MAY add fields, and SpecScore tooling preserves them on read/write. The registration mechanism does NOT introduce a new file convention, a new dotfile, or a plugin-manifest dependency. One config file (`specscore.yaml`) — created by `specstudio:init`, edited by adopters — owns the reviewer registry.

#### REQ: reviewer-registry-entry-shape

Each entry under `reviewers:` MUST declare: `name` (unique within the project, lowercase + hyphens), `prompt` (path to the reviewer's prompt file, relative to repo root), and OPTIONAL `description` (one-line human-readable summary). Future entry fields (e.g., `applies_to:` for scoping which Feature shapes a reviewer reviews, `severity_overrides:` for blocker/advisory tuning) are deferred to additive minor revisions of this contract.

#### REQ: reviewer-prompt-location

The `prompt` value in a registry entry MUST be a path resolving to a file inside the project repo's working tree, expressed as a path relative to repo root. Absolute filesystem paths outside the repo and network URLs are forbidden. Convention for project-owned reviewers: `spec/reviewers/<name>/prompt.md`. Skill-pack-shipped reviewers (where the prompt lives inside an installed plugin's directory rather than the project repo) are out of MVP scope; integrating them requires either vendoring the prompt into `spec/reviewers/` or an additive contract revision that defines a plugin-resolution syntax. Track demand; revisit if at least one third-party skill pack requests a shipped-reviewer integration. The contract does not constrain the prompt's content beyond the reviewer-contract REQs below.

#### REQ: reviewer-contract

A reviewer subagent MUST return exactly one of two verdicts: `Approved` or `Issues Found`. On `Issues Found`, the reviewer MUST attach a structured findings list, where each finding declares its severity as either `Blocker` or `Advisory`. The reviewer's prompt MUST document its blocker/advisory taxonomy — the categories of finding it treats as blocker-severity vs. advisory-severity — so consumers (notably `specstudio:specify`) can audit composition before invoking.

#### REQ: reviewer-composition

When `specstudio:specify` (or any future SpecStudio skill that consumes reviewers) runs multiple registered reviewers, composition is **AND**: every reviewer MUST return `Approved` for the consuming skill's User Review Gate to release. Any single `Issues Found` from any reviewer blocks the gate until every blocker-severity finding from every failing reviewer is addressed. Advisory findings MAY be ignored. The consuming skill MUST NOT silently downgrade a blocker finding to advisory severity, MUST NOT skip a registered reviewer, and MUST re-dispatch every reviewer that previously returned `Issues Found` after fixes are applied. Reviewers that previously returned `Approved` MAY be re-dispatched after a fix cycle to detect regressions; consumers SHOULD re-dispatch all reviewers when the fix touches REQs or ACs covered by previously-Approved reviewers, and MUST re-dispatch all when the fix changes the Feature's structural sections (Behavior, Architecture, Acceptance Criteria).

#### REQ: reviewer-no-canonical-writes

Reviewers MUST NOT write or modify any artifact in `spec/`. A reviewer's only output is its verdict + findings payload. A reviewer that attempts to write to `spec/` is a misclassified Producer; reject it from the reviewer registry and either reclassify or refuse integration.

### Capability shape — tool-invocation contract

Capabilities are subprocess-shaped tools (visual companions, diagram renderers, accessibility auditors, screenshot grabbers). They emit static assets consumed by SpecStudio; they do not write Features.

#### REQ: capability-invocation-contract

A Capability MUST expose a CLI invocation that accepts at minimum these inputs: `--out <path>` (output directory injection — SpecStudio passes the destination), and a documented completion signal (zero exit code on success; non-zero on failure with a stderr message). Capabilities MAY accept additional flags specific to their function. Capabilities MUST NOT prompt for input on stdin during invocation — SpecStudio invokes them non-interactively.

#### REQ: capability-asset-location

Capability output MUST land under `spec/features/<feature-slug>/assets/` when invoked from `specstudio:specify`. The asset path is composed by SpecStudio (the consumer) and passed via `--out`; the Capability MUST NOT decide its own destination. Supported asset types: PNG, SVG, HTML fragments, mermaid `.mmd` source files. Other asset types require additive revision of this contract before acceptance.

#### REQ: capability-no-canonical-writes

Capabilities MUST NOT write to `spec/ideas/`, `spec/features/<slug>/README.md`, `spec/features/<slug>/requirements/`, or `spec/plans/`. Capabilities write only to `assets/` (and only the path passed via `--out`). A Capability that attempts to write Features is misclassified — reclassify, or refuse integration.

#### REQ: capability-single-shape

A single Capability contract covers all current Capability classes (visual companions, diagram renderers, accessibility auditors). Sub-shapes (e.g., a separate "interactive companion" sub-shape vs. "batch renderer" sub-shape) are NOT introduced in this contract. If empirical experience with three or more shipped Capability integrations reveals a need for sub-shapes, propose a contract revision per the versioning rules below; do not introduce sub-shapes via interpretation.

### Contract versioning and evolution

This Feature is itself a contract third-party skill authors and SpecStudio adopters target. Its evolution rules need to be explicit so consumers can plan for change.

#### REQ: contract-revise-in-place

Non-breaking revisions to this Feature MUST happen as in-place edits — additive REQs, clarifying language, new paste targets, new asset types — under the same slug `third-party-integration` with `Status` cycling Draft → Approved → Stable → (additive edit, re-Approved) over time. Git history is the record of evolution.

#### REQ: contract-supersedes

Breaking revisions — changes that invalidate existing reviewer entries, change the snippet's required handoff semantics, or restructure the shape taxonomy — MUST create a successor Feature with `**Supersedes:** third-party-integration` declared. The successor MUST publish a deprecation window before the predecessor is archived; the window length is NOT pinned by this Feature (governance decision per breaking change), but MUST be at least 30 days from successor approval to predecessor archival.

#### REQ: contract-snippet-version-discipline

The canonical snippet's embedded version (per `producer-snippet-versioning`) increments synchronously with this Feature's revisions. Patch increments cover editorial-only changes (typo fixes, link updates) that do not change the instructions agents follow. Minor increments cover additive content (new paste targets, additional handoff guidance). Major increments cover breaking semantics (changed handoff prompt format, changed canonicalization target) — major increments accompany a `supersedes:` Feature transition.

## Architecture and Components

This Feature defines a contract, not a runtime. Its "components" are three artifacts and the rules connecting them.

### 1. Canonical Producer-shape instruction snippet

- **What:** A single markdown file at `spec/features/third-party-integration/snippet.md` containing the platform-neutral instructions text + an embedded version comment.
- **How used:** Adopters (or `specstudio:init`) paste the snippet into the platform agent-instructions file (`CLAUDE.md` / `AGENTS.md` / `GEMINI.md` / etc.) at repo root.
- **Depends on:** Nothing in the runtime — it's static text. Conceptually depends on the `specstudio:ideate` and `specstudio:specify` Features (the canonical producers it points handoffs to).

### 2. Reviewer registry — `specscore.yaml` extension

- **What:** A `reviewers:` block inside the project's `specscore.yaml`, conforming to the entry shape defined by `reviewer-registry-entry-shape`. Lives alongside the rest of the canonical Repo Config schema as an orchestrator extension.
- **How used:** `specstudio:specify` reads the registry on each invocation to determine which reviewer subagents to dispatch as part of its reviewer subagent gate. Future SpecStudio skills (`specstudio:plan`, etc.) MAY consume the same registry.
- **Depends on:** The [SpecScore Repo Config Feature](https://github.com/synchestra-io/specscore/blob/main/spec/features/repo-config/README.md)'s `unknown-fields-preserved` requirement — without it, SpecScore lint would reject the extension key.

### 3. Capability invocation contract

- **What:** An interface specification — output path injection, completion signal, supported asset types — that any Capability-shape tool conforms to.
- **How used:** `specstudio:specify` invokes registered Capabilities as subprocesses during Feature design (e.g., to render a mockup, generate a diagram). The Capability writes assets to the path SpecStudio passes via `--out`.
- **Depends on:** Nothing in the local runtime — it's an interface definition. Capabilities themselves are external (the Superpowers visual companion, future tools).

The three artifacts are independent — adopters can use only Producer-shape instruction injection, only Reviewers, only Capabilities, or any combination. The contract does not require all three to be in use simultaneously.

### Write-permission matrix

At-a-glance summary of which shape may write to which path. Authoritative requirements are in the relevant REQs (`producer-non-goal`, `reviewer-no-canonical-writes`, `capability-no-canonical-writes`); this table is a digest, not a redefinition.

| Path | Producer (third-party) | Reviewer (third-party) | Capability (third-party) | Canonical SpecStudio skills |
|---|---|---|---|---|
| `spec/ideas/<slug>.md` | ✗ MUST NOT | ✗ MUST NOT | ✗ MUST NOT | ✓ `specstudio:ideate` only |
| `spec/features/<slug>/README.md` | ✗ MUST NOT | ✗ MUST NOT | ✗ MUST NOT | ✓ `specstudio:specify` only |
| `spec/features/<slug>/requirements/` | ✗ MUST NOT | ✗ MUST NOT | ✗ MUST NOT | ✓ `specstudio:specify` only |
| `spec/features/<slug>/assets/` | ✗ MUST NOT | ✗ MUST NOT | ✓ at injected `--out` only | ✓ |
| `spec/plans/<slug>/` | ✗ MUST NOT | ✗ MUST NOT | ✗ MUST NOT | ✓ future `specstudio:plan` only |
| `spec/reviewers/<name>/` | ✗ MUST NOT | ✗ MUST NOT (own prompt is read-only at runtime) | ✗ MUST NOT | ✓ author-edited; not skill-modified at runtime |
| User's working dialogue / chat output | ✓ this is the only output channel | ✗ — verdict + findings only | ✗ — assets only | ✓ |
| `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` / `.cursor/rules/*` | ✗ MUST NOT (adopter-edited; `specstudio:init` may write here) | ✗ MUST NOT | ✗ MUST NOT | ✓ `specstudio:init` only |

## Interaction with Other Features

| Feature | Interaction |
|---|---|
| [Specify Skill](../skills/specify/README.md) | Closes its `reviewer-extension-hook` Outstanding Question. `specstudio:specify` reads the `reviewers:` block in `specscore.yaml` per `reviewer-registration-mechanism` and dispatches each registered reviewer per `reviewer-composition`. The built-in baseline reviewer in `specify` remains unchanged; this Feature governs additional reviewers. |
| [Ideate Skill](../skills/ideate/README.md) | Producer non-goal codifies that `specstudio:ideate` is the only sanctioned Idea producer. Third-party Producers MAY produce Idea-shaped drafts but MUST hand off to `ideate` for canonicalization. |
| [SpecScore Repo Config](https://github.com/synchestra-io/specscore/blob/main/spec/features/repo-config/README.md) | Hosts the `reviewers:` extension key inside `specscore.yaml`. This Feature relies on Repo Config's `unknown-fields-preserved` requirement — without it, lint would reject the extension. The reverse dependency does not exist: Repo Config is a generic schema; this Feature is one of its consumers. |
| `specstudio-init-skill` (Idea, [`spec/ideas/specstudio-init-skill.md`](../../ideas/specstudio-init-skill.md)) | Consumes the canonical snippet at `spec/features/third-party-integration/snippet.md` and installs it into the right platform agent-instructions file per its detection rule. Init's `--update` flow reads the snippet's embedded version comment to detect drift. The init Feature is downstream — it cannot be specified before this Feature defines the snippet artifact. |
| Synchestra Events | This Feature defines a contract, not a runtime. It emits no events directly. Consuming features (notably `specstudio:specify`) emit their own events; reviewer dispatch and Capability invocation are observable through those existing event streams. |
| `obra/superpowers` `brainstorming` (Producer integration example) | A canonical example of a third-party Producer skill that this contract enables. Integration is the natural pasted-snippet path; no adapter Feature is required by this contract. If adopters want a wrapped `specstudio:brainstorm` adapter skill in the future, that is a separate downstream Feature, NOT a sub-feature of this one. |

## Acceptance Criteria

### AC: shape-taxonomy-complete

**Requirements:** third-party-integration#req:shape-classification, third-party-integration#req:producer-non-goal

**Given** a third-party agent skill is proposed for integration with SpecStudio
**When** the integration request is evaluated against this Feature's shape taxonomy
**Then** the skill is classified as exactly one of Producer, Reviewer, or Capability — or rejected as out of scope. A skill that proposes to write canonical artifacts directly is rejected as a Shape-3 Producer non-goal violation, with the explicit alternative of producing a draft and handing off to `specstudio:ideate` or `specstudio:specify`.

### AC: producer-snippet-canonical-and-versioned

**Requirements:** third-party-integration#req:producer-canonical-snippet, third-party-integration#req:producer-snippet-versioning, third-party-integration#req:producer-paste-targets

**Given** the Producer mechanism is in use
**When** an adopter or `specstudio:init` consults the canonical snippet
**Then** the snippet exists at exactly `spec/features/third-party-integration/snippet.md`, its first content line is the `<!-- specstudio-snippet-version: <semver> -->` comment, its body is platform-neutral (no `Skill` tool calls, no slash-command syntax, no platform-specific tool invocation), and the documented paste targets (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.cursor/rules/*`) are explicitly listed within the snippet or the Feature.

### AC: producer-output-canonicalization

**Requirements:** third-party-integration#req:producer-output-is-draft, third-party-integration#req:producer-handoff

**Given** a third-party Producer skill (e.g., Superpowers `brainstorming`) runs in a repo with the canonical snippet pasted in `CLAUDE.md`
**When** the Producer skill completes its output
**Then** its produced markdown sits in the user's working dialogue (NOT in `spec/ideas/` or `spec/features/`); contains the section headings appropriate for the artifact shape it is drafting (Idea-shaped drafts contain at least `## Problem Statement`, `## Recommended Direction`, `## MVP Scope`, `## Not Doing (and Why)`, `## Key Assumptions to Validate` per the [SpecScore Idea schema](https://specscore.md/idea-specification); Feature-shaped drafts contain at least `## Summary`, `## Problem`, `## Behavior`, `## Acceptance Criteria` per the [SpecScore Feature schema](https://specscore.md/feature-specification)); contains a `## Acceptance Criteria` section using `Given / When / Then` form when drafting a Feature-shaped artifact; ends with an explicit handoff prompt naming `specstudio:ideate` (for Idea-shaped drafts) or `specstudio:specify` (for Feature-shaped drafts). The Producer never commits on the user's behalf and never writes the draft to `spec/`.

### AC: reviewer-registration-and-composition

**Requirements:** third-party-integration#req:reviewer-registration-mechanism, third-party-integration#req:reviewer-registry-entry-shape, third-party-integration#req:reviewer-prompt-location, third-party-integration#req:reviewer-contract, third-party-integration#req:reviewer-composition, third-party-integration#req:reviewer-no-canonical-writes

**Given** zero or more reviewers are registered under the `reviewers:` extension key in `specscore.yaml`, each with `name`, `prompt`, and optional `description`, and each `prompt` path resolves to a file inside the repo working tree
**When** `specstudio:specify` runs its reviewer subagent gate
**Then** every registered reviewer is dispatched in addition to the built-in baseline reviewer; each registered reviewer's prompt file contains an explicit blocker/advisory taxonomy section that consumers can audit before dispatch (a reviewer prompt with no documented taxonomy fails this AC); each reviewer returns `Approved` or `Issues Found` with structured findings categorized as `Blocker` or `Advisory`; AND composition holds (any single `Issues Found` blocks the User Review Gate); failing reviewers are re-dispatched after fixes per `reviewer-composition`'s re-dispatch rules; no reviewer writes to `spec/`. The User Review Gate releases only when every reviewer returns `Approved`.

### AC: capability-invocation-and-output

**Requirements:** third-party-integration#req:capability-invocation-contract, third-party-integration#req:capability-asset-location, third-party-integration#req:capability-no-canonical-writes, third-party-integration#req:capability-single-shape

**Given** a Capability is invoked by `specstudio:specify` during Feature design
**When** SpecStudio calls the Capability with `--out spec/features/<feature-slug>/assets/` (and any Capability-specific flags)
**Then** the Capability runs non-interactively, exits zero on success (or non-zero with stderr on failure), writes its output (PNG / SVG / HTML fragment / mermaid source) to the passed `--out` path, and never writes to `README.md`, `requirements/`, or any path outside `assets/`.

### AC: contract-evolution-discipline

**Requirements:** third-party-integration#req:contract-revise-in-place, third-party-integration#req:contract-supersedes, third-party-integration#req:contract-snippet-version-discipline

**Given** a proposed change to this Feature
**When** the change is evaluated for revise-in-place vs. supersedes
**Then** non-breaking changes (additive REQs, clarifications, new paste targets, new asset types) revise this Feature in place with the snippet version incrementing patch or minor and Status cycling Approved → (additive edit) → Approved; breaking changes (invalidated reviewer entries, changed handoff semantics, restructured taxonomy) create a successor Feature at a new slug declaring `**Supersedes:** third-party-integration`; the snippet version increments major; **and** this Feature's `**Status:**` MUST NOT transition to `Deprecated` until at least 30 days have elapsed since the successor's `**Status:**` first reached `Approved`. Archival before the 30-day deprecation window has elapsed is a contract violation observable in git history (compare the timestamp of the successor's Approved commit against the predecessor's Deprecated commit).

## Rehearse Integration

**No Rehearse stubs scaffolded.** This Feature defines a contract — its acceptance criteria are doc-only normative rules and process discipline, not runtime behavior with CLI / HTTP / pure-function / data / UI-selector / filesystem / event surfaces. The closest testable surface (registry entries in `specscore.yaml` parsing correctly) belongs to a downstream Feature once `specstudio:specify` is updated to consume `reviewers:`. Future Rehearse coverage attaches to that consumer Feature, not to this contract definition.

## Outstanding Questions

- **Concrete schema of additional `reviewers:` entry fields.** This Feature pins `name`, `prompt`, and optional `description`. Future fields (`applies_to:` for scoping which artifact types a reviewer reviews; `severity_overrides:` for tuning blocker/advisory thresholds; `enabled:` for soft-disabling without removing the entry) are deferred until the first concrete reviewer beyond the baseline ships and the consuming need is real. Track signal; revise additively when needed.
- **Where does the `Producer-shape brainstorming integration example` actually live?** This contract enables Superpowers `brainstorming` integration via the pasted snippet, but does not provide an example of what the produced draft looks like or how the handoff prompt renders. Likely a follow-on doc — under `docs/` (prose), not under `spec/` — once the snippet is authored and tested against a real `brainstorming` invocation.
- **Capability sub-shapes signal threshold.** This Feature pins "single Capability contract." If three or more shipped Capability adapters reveal need for sub-shapes (interactive vs. batch, synchronous vs. async), revisit. Track count; revisit at 3.
- **Cross-platform paste-target detection rule.** This Feature documents the paste targets but does not specify the rule for which to update when multiple are present. That rule is owned by [`specstudio:init`](../../ideas/specstudio-init-skill.md) per its current Recommended Direction; this Feature defers to it.
- **Authoring of `snippet.md`.** This Feature defines where the snippet lives, what it must contain, and how it versions, but the snippet's actual text is authored as a separate work item (it is an asset of this Feature, not its primary deliverable). Authoring is gated by approval of this Feature; once approved, the snippet is added as an additive in-place revision with version `0.1.0`. The empirical conformance gate carried over from the originating Idea — author the snippet, run a Producer-shape skill (Superpowers `brainstorming`) against three real prompts in a repo with the snippet pasted, audit ≥80% conformance on artifact shape and 100% on the handoff-prompt signal — gates the version-`0.1.0` snippet ship, not this Feature's approval.

---
*This document follows the https://specscore.md/feature-specification*
