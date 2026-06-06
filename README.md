# SpecScore Studio Skills

**Spec-driven development.**

AI skills (and a coming Web UI) for efficient spec-driven development — the full lifecycle: **ideate ⇒ specify ⇒ plan ⇒ implement ⇒ verify ⇒ recap ⇒ ship.** SpecStudio turns vague ideas into lintable, testable specifications and gates implementation on those specifications being clear, complete, and approved. Reviews are stage-internal — see [Reviewer Gates](spec/features/reviewer-gates/README.md).

This repo (`specstudio-skills`) is the AI-agent plugin surface of SpecScore Studio: skills, commands, and supporting tooling for Claude Code and Codex. The web client lives at [`specstudio-web`](https://github.com/specscore/specstudio-web) (planned) and will deploy to [`specscore.studio`](https://specscore.studio).

## Why a studio

A studio is a workspace where something gets made end-to-end. SpecStudio is the cockpit for working on one project — from raw idea through shipped code, feature by feature — and for keeping the spec and the code honest with each other as both evolve.

Alongside it:

- [**SpecScore**](https://specscore.org/) — the open protocol every spec artifact conforms to.
  - **Rehearse** — the markdown-native test framework for SpecScore specs. SpecStudio scaffolds Rehearse test stubs from acceptance criteria.
- [**SpecScore AI Marketplace**](https://github.com/specscore/ai-marketplace) — the dedicated marketplace for SpecScore-aligned AI-agent plugins (`specstudio` and `specscore`).

## Install

### Claude Code

Published on the [SpecScore AI Marketplace](https://github.com/specscore/ai-marketplace). Install into Claude Code in two steps:

```
/plugin marketplace add specscore/ai-marketplace
/plugin install specstudio@specscore
```

The first command registers the marketplace once; the second installs (and later updates) the plugin. Run `/plugin uninstall specstudio` to remove it.

### Codex

This repo also includes a Codex plugin manifest at [`.codex-plugin/plugin.json`](./.codex-plugin/plugin.json). A Codex marketplace can point at this plugin source and ingest the existing [`skills/`](./skills/) tree.

For a local Codex marketplace entry, use plugin name `specstudio` with a local source path that resolves to this repo. For a repo/team marketplace following Codex's conventional layout, place or symlink this repo at `plugins/specstudio` and add:

```json
{
  "name": "specstudio",
  "source": {
    "source": "local",
    "path": "./plugins/specstudio"
  },
  "policy": {
    "installation": "AVAILABLE",
    "authentication": "ON_INSTALL"
  },
  "category": "Productivity"
}
```

### Dependencies

The `specstudio` plugin declares one dependency on a sibling plugin:

- **`specscore`** — wraps the `specscore` CLI as agent skills for SpecScore lint, navigation, and lifecycle operations. Lives in the same SpecScore marketplace (`/plugin install specscore@specscore`). Repo: [`ai-plugin-specscore`](https://github.com/specscore/ai-plugin-specscore).

In Claude Code, it's **installed automatically** when you install `specstudio` — Claude Code resolves the dependency graph by plugin name across any marketplaces the user has registered. Uninstalling `specstudio` does not remove the dependency; run `claude plugin prune` (or `claude plugin uninstall specstudio --prune`) to clean it up if you don't want it around.

In Codex, install the `specscore` companion plugin from the same marketplace when available. The Codex manifest intentionally omits a dependency field because Codex plugin validation does not currently accept Claude-style plugin dependencies.

Once installed, the two plugins coexist as independent slash-command namespaces:

| Plugin | Slash namespace | Role |
|---|---|---|
| `specstudio` | `specstudio:*` | High-level SDD methodology skills (this plugin) |
| `specscore` | `specscore:*` | SpecScore CLI wrapper |

## What's in the box

`specstudio-skills` ships as an AI-agent plugin (skills, commands, and supporting tooling) that sits on top of the `specscore` CLI. Today:

| Skill | Purpose |
|---|---|
| `specstudio:ideate` | Refine raw ideas into SpecScore Idea artifacts through structured divergent/convergent thinking. Gates on a lint-clean `spec/ideas/<slug>.md` that the user has approved. |
| `specstudio:specify` | Turn an approved Idea into a SpecScore Feature with requirements and `Given / When / Then` acceptance criteria at `spec/features/<slug>/`. Gates implementation until the Feature is lint-clean and approved. |

More skills covering the rest of the lifecycle (implement, verify, recap, ship) are on the roadmap, alongside a web authoring UI at [`specscore.studio`](https://specscore.studio).

## For AI agents working on this repo

> **Read [`PRINCIPLES.md`](./PRINCIPLES.md) before starting any task — especially before invoking `specstudio:ideate` or `specstudio:specify`.**

These are the two slowest, highest-leverage phases of the lifecycle: bad decisions here propagate into every downstream Plan, commit, and review. The principles in that document orient *how we work* on this repo — which decisions involve the user, how to batch questions, how to use the user's idle time productively, how to push back honestly. They override skill defaults where they conflict.

Two-document split, read both in order:

1. [`PRINCIPLES.md`](./PRINCIPLES.md) — **how we work** (user attention economy, question cadence, parallel work). Project-wide; applies to every task.
2. [`skills/shared/philosophy.md`](./skills/shared/philosophy.md) — **how skills behave** (lint discipline, hard gates, scope decomposition, YAGNI). Skill-specific; applies to skill authoring and skill execution.

If you're a human reader, the same documents tell you what to expect from the AI agents working alongside you on this repo.

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
| **SpecStudio** | Work on one project end-to-end, including spec↔code coherence — AI skills in your IDE today, web authoring UI on the way | **Open source** |

SpecStudio skills work standalone with Claude Code.

## Repository family

The SpecStudio family follows the `specstudio-<role>` stem — every repo in the family is suffixed by its role; no member is unsuffixed:

- `specstudio-skills` (this repo) — Claude Code plugin surface (skills, commands, hooks, supporting tooling)
- `specstudio-web` — web client (planned)
- `specstudio-api` — backend (planned)

The wrapper-prefix `ai-plugin-*` (used by [`ai-plugin-specscore`](https://github.com/specscore/ai-plugin-specscore)) is reserved for thin CLI wrappers — SpecScore Studio is a product, not a wrapper.

Brand spelling: `SpecScore Studio` (formal copy, first mention, contexts where the SpecScore relationship matters) · `SpecStudio` (casual copy, subsequent mentions, in-product) · `specstudio` (identifier token — repos, namespaces, plugin manifest `name`).

## Dogfooding

`specstudio-skills` specifies and develops itself with its own tools.

**Specified in SpecScore.** Every feature, idea, and acceptance criterion in this repo lives under [`spec/`](./spec/README.md) as a SpecScore artifact:

- [`spec/features/`](./spec/features/README.md) — Feature specs for each skill (`ideate`, `specify`, `plan`, planned `implement`/etc.) with `Given / When / Then` acceptance criteria and Rehearse test stubs.
- [`spec/ideas/`](./spec/ideas/README.md) — Pre-spec one-pagers for skills that haven't been promoted to Features yet.
- [`spec/research/`](./spec/research/README.md) — Long-form analyses that informed key design decisions (e.g., the comparison between SpecStudio's `ideate`/`specify` and `obra/superpowers`'s `brainstorming`).

The whole tree lints clean against `specscore spec lint`.

**Developed with SpecStudio.** Every skill in this repo was authored using its own siblings: `specstudio:ideate` produced the Ideas, `specstudio:specify` promoted them into Features, and the same `specstudio:*` workflow gates implementation on lint-clean specs and explicit user approval. When a skill needs a new behavior, the loop is: ideate → specify → implement → land — same loop SpecStudio asks of its users.

If you want to see the methodology applied at scale, this repo is the reference. If something in the spec tree is sloppy, that's also visible — and that's the point.

## Status

**Version 0.0.7 — early.** The `ideate`, `specify`, `plan`, `implement`, `verify`, and `recap` skills are active enough to dogfood on real work. Expect sharp edges, breaking changes, and active iteration.

## Contributing

Contributions are welcome. See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for the contribution workflow and required reading, and [`CHANGELOG.md`](./CHANGELOG.md) for release history.

## License

MIT. See [`LICENSE`](./LICENSE).
