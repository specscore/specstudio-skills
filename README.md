# SpecScore Studio Skills

**Spec-driven development, by Synchestra.**

AI skills (and a coming Web UI) for efficient spec-driven development — the full lifecycle: **ideate ⇒ specify ⇒ plan ⇒ implement ⇒ verify ⇒ recap ⇒ review ⇒ ship.** SpecStudio turns vague ideas into lintable, testable specifications and gates implementation on those specifications being clear, complete, and approved.

This repo (`specstudio-skills`) is the Claude Code plugin surface of SpecScore Studio: skills, commands, and supporting tooling for AI coding agents. The web client lives at [`specstudio-web`](https://github.com/specscore/specstudio-web) (planned) and will deploy to [`specscore.studio`](https://specscore.studio).

## Why a studio

A studio is a workspace where something gets made end-to-end. SpecStudio is the cockpit for working on one project — from raw idea through shipped code, feature by feature — and for keeping the spec and the code honest with each other as both evolve.

Alongside it:

- [**SpecScore**](https://specscore.org/) — the open protocol every spec artifact conforms to.
  - [**Rehearse**](https://rehearse.synchestra.io/) — the markdown-native test framework for SpecScore specs. SpecStudio scaffolds Rehearse test stubs from acceptance criteria.
- [**Synchestra**](https://synchestra.io/) — the engine that runs dispatched work. Headless; you never really "use" it directly.
- [**Synchestra Hub**](https://hub.synchestra.io/) — the portfolio view. When you want to step back from this project and see all your projects and runners, that's Hub.
- [**SpecScore AI Marketplace**](https://github.com/specscore/ai-marketplace) — the dedicated Claude Code marketplace for SpecScore-aligned plugins (`specstudio` and `specscore`). The companion [Sneat AI Marketplace](https://github.com/sneat-co/ai-marketplace) hosts the `synchestra` orchestrator plugin and other unrelated tooling.

## Install

Published on the [SpecScore AI Marketplace](https://github.com/specscore/ai-marketplace). Install into Claude Code in two steps:

```
/plugin marketplace add specscore/ai-marketplace
/plugin install specstudio@specscore
```

The first command registers the marketplace once; the second installs (and later updates) the plugin. Run `/plugin uninstall specstudio` to remove it.

### Dependencies

The `specstudio` plugin declares two dependencies on sibling plugins. Note the dependencies live in different marketplaces:

- **`specscore`** — wraps the `specscore` CLI as agent skills for SpecScore lint, navigation, and lifecycle operations. Lives in the same SpecScore marketplace (`/plugin install specscore@specscore`). Repo: [`ai-plugin-specscore`](https://github.com/specscore/ai-plugin-specscore).
- **`synchestra`** — wraps the `synchestra` CLI as agent skills for task and session orchestration. Lives in the Sneat AI Marketplace (`/plugin install synchestra@sneat-co`; requires `/plugin marketplace add sneat-co/ai-marketplace` first). Repo: [`ai-plugin-synchestra`](https://github.com/synchestra-io/ai-plugin-synchestra).

Both are **installed automatically** when you install `specstudio` — Claude Code resolves the dependency graph by plugin name across any marketplaces the user has registered. If the `synchestra` plugin is not yet resolvable, Claude Code will surface the failure with a clear "add `sneat-co/ai-marketplace`" hint. Uninstalling `specstudio` does not remove the dependencies; run `claude plugin prune` (or `claude plugin uninstall specstudio --prune`) to clean them up if you don't want them around.

Once installed, the three plugins coexist as independent slash-command namespaces:

| Plugin | Slash namespace | Role |
|---|---|---|
| `specstudio` | `specstudio:*` | High-level SDD methodology skills (this plugin) |
| `specscore` | `specscore:*` | SpecScore CLI wrapper |
| `synchestra` | `synchestra:*` | Synchestra CLI wrapper |

## What's in the box

`specstudio-skills` ships as a Claude Code plugin (skills, commands, and supporting tooling) that sits on top of the `specscore` and `synchestra` CLIs. Today:

| Skill | Purpose |
|---|---|
| `specstudio:ideate` | Refine raw ideas into SpecScore Idea artifacts through structured divergent/convergent thinking. Gates on a lint-clean `spec/ideas/<slug>.md` that the user has approved. |
| `specstudio:specify` | Turn an approved Idea into a SpecScore Feature with requirements and `Given / When / Then` acceptance criteria at `spec/features/<slug>/`. Gates implementation until the Feature is lint-clean and approved. |

More skills covering the rest of the lifecycle (implement, verify, recap, review, ship) are on the roadmap, alongside a web authoring UI at [`specscore.studio`](https://specscore.studio).

## Principles

SpecStudio's skills share a common philosophy:

- **Types beat vibes.** If `specscore lint` passes, you can build on it. If it doesn't, you can't.
- **Gates are non-negotiable.** No amount of perceived simplicity bypasses a hard gate. Ideate before specify, specify before plan, plan before code.
- **Unsaved ideation is waste.** If it's worth thinking about, it's worth a lint-clean artifact in `spec/`.
- **Say no to 1,000 things.** The "Not Doing" list is the most valuable part of any artifact.
- **Be honest, not supportive.** Skills push back on weak ideas with specificity and kindness. No yes-machines.

See [`skills/shared/philosophy.md`](./skills/shared/philosophy.md) for the full set.

## Where it fits

| | What it does | Layer |
|---|---|---|
| [SpecScore](https://specscore.org/) | The protocol: feature/requirement/AC format, lint, LSP | Open source |
| [Synchestra](https://synchestra.io/) | The engine: CLI, daemon, runners | Open source |
| **SpecStudio** | Work on one project end-to-end, including spec↔code coherence — AI skills in your IDE today, web authoring UI on the way | **Open source** |
| [Synchestra Hub](https://hub.synchestra.io/) | Portfolio + runtime UI for remote execution | Hosted |

SpecStudio skills work standalone with Claude Code. Paired with Synchestra Hub, the same skills can dispatch long-running or sandboxed work to remote runners without leaving your editor.

## Repository family

The SpecStudio family follows the `specstudio-<role>` stem — every repo in the family is suffixed by its role; no member is unsuffixed:

- `specstudio-skills` (this repo) — Claude Code plugin surface (skills, commands, hooks, supporting tooling)
- `specstudio-web` — web client (planned)
- `specstudio-api` — backend, if not folded into `synchestra-cloud` (TBD)

The wrapper-prefix `ai-plugin-*` (used by [`ai-plugin-synchestra`](https://github.com/synchestra-io/ai-plugin-synchestra) and [`ai-plugin-specscore`](https://github.com/specscore/ai-plugin-specscore)) is reserved for thin CLI wrappers — SpecScore Studio is a product, not a wrapper.

Brand spelling: `SpecScore Studio` (formal copy, first mention, contexts where the SpecScore relationship matters) · `SpecStudio` (casual copy, subsequent mentions, in-product) · `specstudio` (identifier token — repos, namespaces, plugin manifest `name`).

## Dogfooding

`specstudio-skills` specifies and develops itself with its own tools.

**Specified in SpecScore.** Every feature, idea, and acceptance criterion in this repo lives under [`spec/`](./spec/README.md) as a SpecScore artifact:

- [`spec/features/`](./spec/features/README.md) — Feature specs for each skill (`ideate`, `specify`, `plan`, planned `implement`/etc.) with `Given / When / Then` acceptance criteria and Rehearse test stubs.
- [`spec/ideas/`](./spec/ideas/README.md) — Pre-spec one-pagers for skills that haven't been promoted to Features yet.
- [`spec/research/`](./spec/research/README.md) — Long-form analyses that informed key design decisions (e.g., the comparison between SpecStudio's `ideate`/`specify` and `obra/superpowers`'s `brainstorming`).

The whole tree lints clean against `specscore spec lint`.

**Developed with SpecStudio.** Every skill in this repo was authored using its own siblings: `specstudio:ideate` produced the Ideas, `specstudio:specify` promoted them into Features, and the same `specstudio:*` workflow gates implementation on lint-clean specs and explicit user approval. When a skill needs a new behavior, the loop is: ideate → specify → review → implement → land — same loop SpecStudio asks of its users.

If you want to see the methodology applied at scale, this repo is the reference. If something in the spec tree is sloppy, that's also visible — and that's the point.

## Status

**Version 0.0.3 — early.** The `ideate` and `specify` skills are stable enough to use on real work; the rest of the lifecycle is in progress. Expect sharp edges, breaking changes, and active iteration.

## License

MIT. See [`LICENSE`](./LICENSE).
