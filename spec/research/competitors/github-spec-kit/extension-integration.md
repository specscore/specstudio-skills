# Spec Kit Extension Integration — Concrete Plug-In Mapping

**Date:** 2026-05-01
**Source:** https://github.com/github/spec-kit/blob/main/extensions/EXTENSION-API-REFERENCE.md (manifest schema 1.0)
**Related docs:** [`README.md`](README.md) (strategic analysis & integrate-vs-compete recommendation)

---

## Purpose

This doc captures the concrete technical surface — manifests, hooks, file layout, namespacing — for shipping SpecScore as a Spec Kit **preset** and Synchestra as a Spec Kit **extension**. It's the implementation companion to [`README.md`](README.md). Read that first for the strategic context.

---

## 1. The Spec Kit Plugin Surface (in 90 seconds)

Spec Kit exposes two plugin types with different roles:

| Type | Role | Mechanism | Use it when… |
|---|---|---|---|
| **Extension** | *Adds* new commands and lifecycle hooks | `extension.yml` manifest with `provides.commands` + `hooks` | You want new verbs (e.g. `speckit.synchestra.claim`) and to react to core events |
| **Preset** | *Overrides* core templates and defaults | Template files in `.specify/presets/templates/` + optional config | You want to change *what Spec Kit produces* without changing *how it flows* |

**Template resolution is layered, top-down first match wins:**
```
.specify/templates/overrides/   ← project-local (highest)
.specify/presets/templates/     ← preset
.specify/extensions/templates/  ← extension
.specify/templates/             ← core defaults (lowest)
```

**Command namespacing is strict:** `speckit.{extension-id}.{command-name}`, all segments matching `^[a-z0-9-]+$`. An extension cannot shadow core or other-extension commands.

**Hook events available** (one before/after pair per core command):
`before_constitution`, `after_constitution`, `before_specify`, `after_specify`, `before_clarify`, `after_clarify`, `before_plan`, `after_plan`, `before_tasks`, `after_tasks`, `before_analyze`, `after_analyze`, `before_checklist`, `after_checklist`, `before_implement`, `after_implement`, plus the `taskstoissues` pair used by Jira-style integrations.

**Hook payload semantics** are *invocation-style*: a hook is "an extension command run optionally after/before a core command, with an optional user prompt." It's not arbitrary code — it's another namespaced command the runtime offers to invoke.

---

## 2. Three Plug-Ins We Should Ship

Listed in priority order. Each is independently shippable; together they cover author → validate → execute.

### 2.1. `speckit-specscore` — preset (★★★★★)

**Goal:** every Spec Kit run produces machine-validatable SpecScore by default.

**Two parts:**
- **Preset templates** that replace `spec-template.md`, `plan-template.md`, `tasks-template.md` with SpecScore-structured equivalents (Markdown body + YAML frontmatter matching the SpecScore schema).
- **A small companion extension** `speckit-specscore-lint` that hooks `after_specify`, `after_plan`, `after_tasks`, `after_analyze` to run `specscore spec lint` and surface drift.

> A preset on its own can't run hooks (presets only override templates). The lint half therefore ships as a thin extension. Keep the names paired so users install both with one command in our docs.

**Preset layout** (preset id `specscore`):
```
.specify/presets/specscore/
├── preset.yml                         # preset manifest
├── templates/
│  ├── spec-template.md                # SpecScore feature spec template
│  ├── plan-template.md                # SpecScore plan template
│  └── tasks-template.md               # SpecScore tasks template
└── README.md
```

**Companion extension manifest** (`extension.yml`, id `specscore-lint`):
```yaml
schema_version: "1.0"
extension:
  id: specscore-lint
  name: SpecScore Lint
  version: 0.1.0
  description: Validate Spec Kit outputs against the SpecScore schema and surface drift.
  author: Synchestra
  repository: https://github.com/specscore/speckit-specscore
  license: Apache-2.0

requires:
  speckit_version: ">=0.8.0"
  tools:
    - name: specscore
      version: ">=0.x"
      required: true

provides:
  commands:
    - name: speckit.specscore.lint
      file: commands/lint.md
      description: Run `specscore spec lint` on the active spec tree and report drift.
      aliases: [speckit.specscore.validate]
    - name: speckit.specscore.fix
      file: commands/fix.md
      description: Apply mechanical fixes that the linter flags (formatting, frontmatter).

hooks:
  after_specify:
    command: speckit.specscore.lint
    optional: false
    description: Validate spec.md against SpecScore schema immediately after authoring.
  after_plan:
    command: speckit.specscore.lint
    optional: false
    description: Validate plan.md and contracts/ after planning.
  after_tasks:
    command: speckit.specscore.lint
    optional: false
    description: Validate tasks.md after task generation.
  after_analyze:
    command: speckit.specscore.lint
    optional: true
    prompt: "Re-run SpecScore lint after analysis?"
    description: Optional re-validation after cross-artifact analysis.

tags: [specscore, validation, schema, lint]
```

**Why this lands:** users keep their familiar Spec Kit workflow but their outputs become a portable open standard from day one. SpecScore goes from "another spec format you have to learn" to "the validated form of the spec you're already writing."

---

### 2.2. `speckit-synchestra` — extension (★★★★★)

**Goal:** turn Spec Kit's single-session `/speckit.implement` into a multi-agent, async, resumable execution plane via Synchestra + Hub.

**Manifest** (`extension.yml`, id `synchestra`):
```yaml
schema_version: "1.0"
extension:
  id: synchestra
  name: Synchestra Coordination
  version: 0.1.0
  description: Multi-agent, async, git-native execution for Spec Kit task lists.
  author: Synchestra
  repository: https://github.com/specscore/speckit-synchestra
  license: Apache-2.0
  homepage: https://synchestra.io

requires:
  speckit_version: ">=0.8.0"
  tools:
    - name: synchestra
      version: ">=0.x"
      required: true
    - name: git
      required: true

provides:
  commands:
    - name: speckit.synchestra.init
      file: commands/init.md
      description: Initialize a Synchestra state repo alongside this Spec Kit project.

    - name: speckit.synchestra.queue
      file: commands/queue.md
      description: Enqueue the current spec's tasks.md into the orchestrator state repo.
      aliases: [speckit.synchestra.enqueue]

    - name: speckit.synchestra.claim
      file: commands/claim.md
      description: Claim the next available task (optimistic-locking via git).

    - name: speckit.synchestra.status
      file: commands/status.md
      description: Show task board and per-task status across the project.

    - name: speckit.synchestra.complete
      file: commands/complete.md
      description: Mark the currently claimed task complete and push.

    - name: speckit.synchestra.release
      file: commands/release.md
      description: Release a claimed task back to the queue.

    - name: speckit.synchestra.dispatch
      file: commands/dispatch.md
      description: Hand off remaining work to the orchestrator Hub for remote async execution.

  config:
    - name: state-repo
      template: synchestra-config.template.yml
      description: Path or remote URL of the orchestrator state repo.
      required: true

hooks:
  after_tasks:
    command: speckit.synchestra.queue
    optional: true
    prompt: "Enqueue these tasks into the orchestrator for parallel/async execution?"
    description: Offer multi-agent execution after Spec Kit generates tasks.md.
    condition: "exists('.synchestra-config.yml')"

  before_implement:
    command: speckit.synchestra.dispatch
    optional: true
    prompt: "Run via Synchestra Hub instead of locally? (close laptop → work continues)"
    description: Offer remote async execution as an alternative to local /speckit.implement.

  after_implement:
    command: speckit.synchestra.status
    optional: true
    prompt: "Show Synchestra task board?"
    description: Surface remaining/blocked tasks after a local implement run.

tags: [synchestra, coordination, multi-agent, async, hub, git-native]

defaults:
  state_repo_pattern: "{project}-synchestra"
  use_hub: false
```

**The user journey this creates:**

1. User installs Spec Kit, runs `specify init my-project --integration claude`.
2. User installs the extension: `specify extension add specscore/speckit-synchestra`.
3. User runs `/speckit.constitution` → `/speckit.specify` → `/speckit.clarify` → `/speckit.plan` → `/speckit.tasks` *as normal*.
4. After `/speckit.tasks`, the `after_tasks` hook prompts: "Enqueue these tasks into Synchestra?" → yes → tasks land in `my-project-synchestra/tasks/`.
5. User can now either:
   - Continue with `/speckit.implement` for a single-session local run (Spec Kit's default), or
   - Run `speckit.synchestra.claim` from multiple agent sessions (parallel local), or
   - Run `speckit.synchestra.dispatch` to send to Hub (close laptop, come back later).

The extension never *replaces* Spec Kit's flow — it adds an off-ramp at exactly the point where Spec Kit's single-agent assumption starts to bind.

---

### 2.3. `speckit-rehearse` — extension (★★★★)

**Goal:** make the acceptance criteria in `/speckit.specify` actually executable. Spec Kit produces ACs as prose; Rehearse turns them into runnable scenarios.

**Manifest** (`extension.yml`, id `rehearse`):
```yaml
schema_version: "1.0"
extension:
  id: rehearse
  name: Rehearse — Markdown-Native AC Verification
  version: 0.1.0
  description: Run Spec Kit acceptance criteria as executable test scenarios.
  author: Synchestra
  repository: https://github.com/specscore/speckit-rehearse
  license: Apache-2.0

requires:
  speckit_version: ">=0.8.0"
  tools:
    - name: rehearse
      version: ">=0.x"
      required: true

provides:
  commands:
    - name: speckit.rehearse.scaffold
      file: commands/scaffold.md
      description: Generate Rehearse scenarios from spec.md acceptance criteria.

    - name: speckit.rehearse.run
      file: commands/run.md
      description: Execute Rehearse scenarios for the active spec.

    - name: speckit.rehearse.verify
      file: commands/verify.md
      description: Verify that all ACs in spec.md are covered by passing scenarios.

hooks:
  after_specify:
    command: speckit.rehearse.scaffold
    optional: true
    prompt: "Scaffold Rehearse scenarios from these acceptance criteria?"

  after_implement:
    command: speckit.rehearse.verify
    optional: false
    description: Verify all spec.md ACs pass before declaring implement done.

tags: [rehearse, testing, acceptance-criteria, verification]
```

---

## 3. Where Spec Kit's Plugin Model Doesn't Reach

What we *cannot* do as a Spec Kit extension, and therefore must keep on the orchestrator side proper:

| Capability | Why extensions can't deliver it |
|---|---|
| **Optimistic-locking task claim** | Requires a separate state repo with its own git remote — outside `.specify/`. Extensions can call out to it but can't define it. |
| **Async / Hub execution** | Hub is a remote daemon + dashboard, not a slash command. The extension can dispatch, but the platform lives outside Spec Kit. |
| **Schema validation at commit time** | Pre-commit/pre-push/CI hooks run on the user's repo, not via Spec Kit's lifecycle. Extensions can lint *during* a flow; only Synchestra/SpecScore tooling can guard *every* commit. |
| **Multi-repo coordination** | Spec Kit assumes one project, one repo. Multi-repo branching/merge is Synchestra-side. |
| **Hierarchical WBS / nested tasks** | Spec Kit's `tasks.md` is flat by design. We can render a flat list; recursion requires the SpecScore-backed Synchestra state repo. |
| **Outstanding Questions lifecycle** | `/speckit.clarify` is one-shot. Persistent OQs that survive across sessions/agents require Synchestra's state repo conventions. |
| **Plugin SPI for Synchestra-on-its-own** | Spec Kit's manifest is for Spec Kit users only. We still need a Synchestra-native plugin SPI for the (large) population who never touch Spec Kit. |

These gaps are not a problem — they're the differentiation. The extension is a *gateway* into the orchestrator substrate, not a replacement for it.

---

## 4. Naming, Distribution, License

| Question | Answer |
|---|---|
| Repo names | `specscore/speckit-specscore`, `specscore/speckit-synchestra`, `specscore/speckit-rehearse` |
| Extension `id` field | `specscore-lint`, `synchestra`, `rehearse` (manifest pattern `^[a-z0-9-]+$`) |
| License | Apache-2.0 (matches the rest of the orchestrator Apache ecosystem; compatible with Spec Kit's MIT for installation, not for code lift) |
| Versioning | Semver. Pin `requires.speckit_version: ">=0.8.0"` initially; tighten if Spec Kit ships breaking manifest changes. |
| Discovery | Submit to Spec Kit's community-extensions catalog and `polarizertech/spec-kit-extensions` incubator. |
| Install command (user-visible) | `specify extension add specscore/speckit-synchestra` |

**Don't use the `speckit-` prefix to imply ownership.** It indicates *integration target*. We are not GitHub.

---

## 5. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Spec Kit ships breaking manifest changes (1.0 → 2.0) | Pin `requires.speckit_version`; keep extensions in their own repos so we can release patches independently of the orchestrator core. |
| GitHub absorbs equivalent functionality (e.g. native multi-agent in Spec Kit core) | Synchestra's substrate value (portable schema, multi-repo, Hub, async) doesn't disappear — it becomes the layer below. We'd lose the "extension is the only way to get this" framing but keep the underlying differentiation. |
| User confusion: "is this Synchestra or Spec Kit?" | Keep the README crystal: Spec Kit = author, SpecScore = format, Synchestra = run. Use Spec Kit's vocabulary in extension docs. |
| Extension becomes the only Synchestra anyone tries | Make sure `orchestrator` standalone CLI ships with first-class onboarding too. The Spec Kit extension is one path, not the path. |
| Maintenance burden of three extensions | Three small repos, shared release tooling. Each extension is < 1k LoC of glue around our existing CLIs — most "code" is markdown command files. |

---

## 6. Implementation Checklist (extracted from `README.md` priorities)

- [ ] Read Spec Kit's `EXTENSION-DEVELOPMENT-GUIDE.md` and `EXTENSION-PUBLISHING-GUIDE.md` end-to-end before writing manifests
- [ ] Stand up `specscore/speckit-specscore` repo with preset templates + `specscore-lint` companion extension
- [ ] Stand up `specscore/speckit-synchestra` repo with manifest above + command markdown files
- [ ] Stand up `specscore/speckit-rehearse` repo with manifest above
- [ ] Verify each manifest installs cleanly via `specify extension add` against latest Spec Kit
- [ ] Add a "Use with GitHub Spec Kit" section to synchestra.io and specscore.org marketing
- [ ] Submit to Spec Kit community-extensions catalog
- [ ] Update Synchestra plugin-SPI design doc to mirror Spec Kit's manifest shape (so a future plugin author can target both ecosystems with one manifest)
