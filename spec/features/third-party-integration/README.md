---
format: https://specscore.md/feature-specification
status: Approved
---

# Feature: Third-Party Skill Integration

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/third-party-integration?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/third-party-integration?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/third-party-integration?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/specstudio-skills/spec/features/third-party-integration?op=request-change) |

**Status:** Approved
**Source Ideas:** third-party-skill-integration-contracts

## Summary

Defines the contract for integrating third-party agent skills (Superpowers `brainstorming`, addyosmani `idea-refine`, future capabilities) with SpecScore artifacts. Classifies every third-party skill into one of two shapes covered here — **Producer** and **Capability** — and pins normative MUST / SHOULD / MUST-NOT requirements per shape. Capabilities become a first-class integration path with a concrete contract. Shape-3 third-party Producers (skills that write canonical SpecScore Idea or Feature artifacts directly) are an explicit non-goal: their output is draft input to `specstudio:ideate` or `specstudio:specify`, never a finished artifact. The Reviewer shape is carved out into the [`reviewer-gates`](../reviewer-gates/README.md) Feature, which owns reviewer registration, the type discriminator, dispatch contract, and AND-composition.

## Problem

SpecStudio adopters increasingly run third-party agent skills (Superpowers, addyosmani agent-skills, etc.) alongside SpecStudio in the same repository. Without an explicit contract, two failure modes surface: (1) third-party Producer-class skills write artifacts that look canonical but fail `specscore spec lint` and lack lifecycle integration (events, status transitions, Synchestra reconciliation); (2) Capability-class tools (visual companions, diagram renderers, accessibility auditors) have no defined invocation surface, so each integration becomes an ad-hoc adapter.

This Feature replaces ad-hoc handling with a typed contract that adopters and skill authors can target. The Reviewer shape — registration, dispatch, AND-composition — is owned by [`reviewer-gates`](../reviewer-gates/README.md).

## Behavior

### Shape taxonomy and Producer non-goal

Every third-party agent skill SpecStudio interoperates with classifies into exactly one shape. Shape determines the integration mechanism, not the other way around. This Feature pins the **Producer** and **Capability** shapes; the **Reviewer** shape is owned by [`reviewer-gates`](../reviewer-gates/README.md).

#### REQ: shape-classification

Every third-party agent skill SpecStudio interoperates with MUST be classified as exactly one of: **Producer** (writes a draft artifact intended to be canonicalized by SpecStudio later), **Reviewer** (returns a structured verdict on a candidate artifact — see [`reviewer-gates`](../reviewer-gates/README.md)), or **Capability** (emits a static asset or runs a runtime tool consumed by SpecStudio). A skill that does not fit any of the three shapes MUST be rejected as out of scope for integration; the contract does not stretch to accommodate novel shapes via reinterpretation.

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

Breaking revisions — changes that invalidate the snippet's required handoff semantics, restructure the shape taxonomy, or break the Capability invocation contract — MUST create a successor Feature with `**Supersedes:** third-party-integration` declared. The successor MUST publish a deprecation window before the predecessor is archived; the window length is NOT pinned by this Feature (governance decision per breaking change), but MUST be at least 30 days from successor approval to predecessor archival.

#### REQ: contract-snippet-version-discipline

The canonical snippet's embedded version (per `producer-snippet-versioning`) increments synchronously with this Feature's revisions. Patch increments cover editorial-only changes (typo fixes, link updates) that do not change the instructions agents follow. Minor increments cover additive content (new paste targets, additional handoff guidance). Major increments cover breaking semantics (changed handoff prompt format, changed canonicalization target) — major increments accompany a `supersedes:` Feature transition.

## Architecture and Components

This Feature defines a contract, not a runtime. Its "components" are two artifacts and the rules connecting them. (The Reviewer registry lives in [`reviewer-gates`](../reviewer-gates/README.md).)

### 1. Canonical Producer-shape instruction snippet

- **What:** A single markdown file at `spec/features/third-party-integration/snippet.md` containing the platform-neutral instructions text + an embedded version comment.
- **How used:** Adopters (or `specstudio:init`) paste the snippet into the platform agent-instructions file (`CLAUDE.md` / `AGENTS.md` / `GEMINI.md` / etc.) at repo root.
- **Depends on:** Nothing in the runtime — it's static text. Conceptually depends on the `specstudio:ideate` and `specstudio:specify` Features (the canonical producers it points handoffs to).

### 2. Capability invocation contract

- **What:** An interface specification — output path injection, completion signal, supported asset types — that any Capability-shape tool conforms to.
- **How used:** `specstudio:specify` invokes registered Capabilities as subprocesses during Feature design (e.g., to render a mockup, generate a diagram). The Capability writes assets to the path SpecStudio passes via `--out`.
- **Depends on:** Nothing in the local runtime — it's an interface definition. Capabilities themselves are external (the Superpowers visual companion, future tools).

The two artifacts are independent — adopters can use only Producer-shape instruction injection, only Capabilities, or both. The contract does not require both to be in use simultaneously.

### Write-permission matrix

At-a-glance summary of which shape may write to which path. Authoritative requirements are in the relevant REQs (`producer-non-goal`, `capability-no-canonical-writes`); this table is a digest, not a redefinition. Reviewer write-permissions are governed by [`reviewer-gates`](../reviewer-gates/README.md).

| Path | Producer (third-party) | Capability (third-party) | Canonical SpecStudio skills |
|---|---|---|---|
| `spec/ideas/<slug>.md` | ✗ MUST NOT | ✗ MUST NOT | ✓ `specstudio:ideate` only |
| `spec/features/<slug>/README.md` | ✗ MUST NOT | ✗ MUST NOT | ✓ `specstudio:specify` only |
| `spec/features/<slug>/requirements/` | ✗ MUST NOT | ✗ MUST NOT | ✓ `specstudio:specify` only |
| `spec/features/<slug>/assets/` | ✗ MUST NOT | ✓ at injected `--out` only | ✓ |
| `spec/plans/<slug>/` | ✗ MUST NOT | ✗ MUST NOT | ✓ future `specstudio:plan` only |
| User's working dialogue / chat output | ✓ this is the only output channel | ✗ — assets only | ✓ |
| `CLAUDE.md` / `AGENTS.md` / `GEMINI.md` / `.cursor/rules/*` | ✗ MUST NOT (adopter-edited; `specstudio:init` may write here) | ✗ MUST NOT | ✓ `specstudio:init` only |

## Interaction with Other Features

| Feature | Interaction |
|---|---|
| [Reviewer Gates](../reviewer-gates/README.md) | Owns the Reviewer shape carved out of this Feature. Reviewer registration, type discriminator, dispatch contract, and AND-composition live there. This Feature retains only the Producer and Capability shapes. |
| [Specify Skill](../skills/specify/README.md) | `specstudio:specify` consumes Capabilities defined here during Feature design. Reviewer dispatch is owned by [`reviewer-gates`](../reviewer-gates/README.md), not this Feature. |
| [Ideate Skill](../skills/ideate/README.md) | Producer non-goal codifies that `specstudio:ideate` is the only sanctioned Idea producer. Third-party Producers MAY produce Idea-shaped drafts but MUST hand off to `ideate` for canonicalization. |
| [SpecScore Repo Config](https://github.com/specscore/specscore/blob/main/spec/features/repo-config/README.md) | This Feature relies on Repo Config's `unknown-fields-preserved` requirement for any extension keys it (or downstream consumers like [`reviewer-gates`](../reviewer-gates/README.md)) introduce. The reverse dependency does not exist: Repo Config is a generic schema; this Feature is one of its consumers. |
| `specstudio-init-skill` (Idea, [`spec/ideas/specstudio-init-skill.md`](../../ideas/specstudio-init-skill.md)) | Consumes the canonical snippet at `spec/features/third-party-integration/snippet.md` and installs it into the right platform agent-instructions file per its detection rule. Init's `--update` flow reads the snippet's embedded version comment to detect drift. The init Feature is downstream — it cannot be specified before this Feature defines the snippet artifact. |
| Synchestra Events | This Feature defines a contract, not a runtime. It emits no events directly. Consuming features (notably `specstudio:specify`) emit their own events; Capability invocation is observable through those existing event streams. |
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

### AC: capability-invocation-and-output

**Requirements:** third-party-integration#req:capability-invocation-contract, third-party-integration#req:capability-asset-location, third-party-integration#req:capability-no-canonical-writes, third-party-integration#req:capability-single-shape

**Given** a Capability is invoked by `specstudio:specify` during Feature design
**When** SpecStudio calls the Capability with `--out spec/features/<feature-slug>/assets/` (and any Capability-specific flags)
**Then** the Capability runs non-interactively, exits zero on success (or non-zero with stderr on failure), writes its output (PNG / SVG / HTML fragment / mermaid source) to the passed `--out` path, and never writes to `README.md`, `requirements/`, or any path outside `assets/`.

### AC: contract-evolution-discipline

**Requirements:** third-party-integration#req:contract-revise-in-place, third-party-integration#req:contract-supersedes, third-party-integration#req:contract-snippet-version-discipline

**Given** a proposed change to this Feature
**When** the change is evaluated for revise-in-place vs. supersedes
**Then** non-breaking changes (additive REQs, clarifications, new paste targets, new asset types) revise this Feature in place with the snippet version incrementing patch or minor and Status cycling Approved → (additive edit) → Approved; breaking changes (changed handoff semantics, restructured taxonomy, broken Capability invocation contract) create a successor Feature at a new slug declaring `**Supersedes:** third-party-integration`; the snippet version increments major; **and** this Feature's `**Status:**` MUST NOT transition to `Deprecated` until at least 30 days have elapsed since the successor's `**Status:**` first reached `Approved`. Archival before the 30-day deprecation window has elapsed is a contract violation observable in git history (compare the timestamp of the successor's Approved commit against the predecessor's Deprecated commit).

## Rehearse Integration

**No Rehearse stubs scaffolded.** This Feature defines a contract — its acceptance criteria are doc-only normative rules and process discipline, not runtime behavior with CLI / HTTP / pure-function / data / UI-selector / filesystem / event surfaces. The closest testable surfaces (snippet drift detection by `specstudio:init --update`; Capability subprocess invocation by `specstudio:specify`) belong to those downstream consumer Features. Future Rehearse coverage attaches to those consumer Features, not to this contract definition.

## Open Questions

- **Where does the `Producer-shape brainstorming integration example` actually live?** This contract enables Superpowers `brainstorming` integration via the pasted snippet, but does not provide an example of what the produced draft looks like or how the handoff prompt renders. Likely a follow-on doc — under `docs/` (prose), not under `spec/` — once the snippet is authored and tested against a real `brainstorming` invocation.
- **Capability sub-shapes signal threshold.** This Feature pins "single Capability contract." If three or more shipped Capability adapters reveal need for sub-shapes (interactive vs. batch, synchronous vs. async), revisit. Track count; revisit at 3.
- **Cross-platform paste-target detection rule.** This Feature documents the paste targets but does not specify the rule for which to update when multiple are present. That rule is owned by [`specstudio:init`](../../ideas/specstudio-init-skill.md) per its current Recommended Direction; this Feature defers to it.
- **Authoring of `snippet.md`.** This Feature defines where the snippet lives, what it must contain, and how it versions, but the snippet's actual text is authored as a separate work item (it is an asset of this Feature, not its primary deliverable). Authoring is gated by approval of this Feature; once approved, the snippet is added as an additive in-place revision with version `0.1.0`. The empirical conformance gate carried over from the originating Idea — author the snippet, run a Producer-shape skill (Superpowers `brainstorming`) against three real prompts in a repo with the snippet pasted, audit ≥80% conformance on artifact shape and 100% on the handoff-prompt signal — gates the version-`0.1.0` snippet ship, not this Feature's approval.

---
*This document follows the https://specscore.md/feature-specification*
