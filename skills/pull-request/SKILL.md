---
name: pull-request
description: |
  Gate-and-create-one-PR twin of the ship skill. Enforces the project's
  pre-PR gates on the current branch via the shared reviewer-gates layer,
  then creates exactly one pull request — built-in git push + gh pr create
  by default, or a project-configured delegate skill — and emits
  pull_request.created. It gates, creates one PR, and records; it never
  merges, deploys, stacks multiple PRs, retries a delegate, or orchestrates.
  Trigger: "pull-request", "/pull-request", "open a PR", "specstudio:pull-request".
aliases: [pull-request, pr]
---

# Pull Request

Open a pull request for the current branch only after the project's pre-PR
gates pass. Pull-request **gates, creates one PR, and records** — it never
merges, deploys, stacks, retries, or orchestrates. The verify gate runs
*before* the PR exists, so a branch below the project's bar (e.g. failing
tests or coverage) never reaches an open PR.

Implements the [Pull Request Skill Feature](../../spec/features/skills/pull-request/README.md).

## When to Use

- The user is on a feature branch with commits ahead of the base branch and wants to open a pull request.
- The user wants the project's verify gate (tests, coverage) plus any configured review to run before the PR is created, not after CI flags it.

**Refuse and redirect when:**

- The current branch is the repository's default branch → print the reason and recommend creating a feature branch first. Create no PR; exit non-zero. (AC: `rejects-default-branch`)
- The current branch has no commits ahead of the base branch → print "nothing to open a PR for" and create no PR; exit non-zero. (AC: `refuses-when-no-commits-ahead`)
- The user asks to create the PR before the reviewer gate releases → refuse; the PR is available only after `pull_request.pre_dispatch` releases `Approved`.

## Pre-Flight

Pre-flight refusals exit immediately, write nothing, create no PR, and dispatch no reviewer and no delegate. The skill operates on the **current branch only** and creates **at most one** pull request per invocation — it never accepts, batches, or coordinates multiple branches or stacked PRs. (AC: `rejects-default-branch`)

### Machine gates

These are hard gates the skill enforces itself, before any reviewer or delegate is dispatched.

1. **Default-branch guard.** Resolve the repository's default branch and the current branch. Refuse when they are the same — there is no PR from the default branch into itself. Print the reason and recommend creating a feature branch. (AC: `rejects-default-branch`)

2. **Commits-ahead guard.** Refuse when the current branch has zero commits ahead of the base branch (`git rev-list --count <base>..HEAD` is `0`) — there is nothing to open a PR for. Print the reason; create no PR. (AC: `refuses-when-no-commits-ahead`)

## Checklist

Create a task for each and complete in order:

1. **Pre-flight machine gates** (Pre-Flight steps 1–2): default-branch guard, then commits-ahead guard. Any refusal exits immediately.
2. **Reviewer gate** (`## Reviewer Gate`): fire `pull_request.pre_dispatch`, load and run `gates.pull_request.pre_dispatch` reviewers (AND-composed, including the `type: deterministic` verify reviewer). Halt on `Issues Found`; proceed only on release.
3. **PR creation** (`## PR Creation`): if `pull_request.delegate` is configured, dispatch it once; otherwise run the built-in `git push` + `gh pr create` path. Derive and confirm the title/body first.
4. **Emit** (`## Lifecycle Event`): on successful creation, apply publication policy and emit `pull_request.created` exactly once.

## Reviewer Gate

After the pre-flight machine gates pass, the skill fires the `pull_request.pre_dispatch` gate-point event and evaluates the reviewer gate keyed on it — `gates.pull_request.pre_dispatch` in `specscore.yaml` — via the shared reviewer-gates [loader](../shared/reviewer-gates/loader.md) and [runner](../shared/reviewer-gates/runner.md). The skill carries **no** hardcoded baseline reviewer; the reviewer list comes exclusively from the gate config.

1. **Fire and load.** Fire `pull_request.pre_dispatch` (single-fire per run; see [events.md](../shared/events.md)). Load and validate `gates.pull_request.pre_dispatch.reviewers` via the loader with event key `pull_request.pre_dispatch`. If the gate is unconfigured or invalid, the loader refuses per its contract — surface its error verbatim and halt; create no PR.
2. **Run.** Dispatch the validated reviewer list via the runner: serial, in declared list order, AND-composed. The gate releases only when **every** entry returns `Approved`. (AC: `gate-releases-only-on-all-approved`)
3. **Verify reviewer.** The project verify check (tests, coverage) is expressed as an existing `type: deterministic` reviewer whose `run:` command's exit code is the verdict (zero → `Approved`; non-zero → `Issues Found` with diagnostics captured as `Blocker`(s)). The skill introduces **no new reviewer type** — it uses only the types reviewer-gates already defines (`ai`, `human`, `deterministic`, `auto-approve`). The configured `run:` command must be read-only with respect to `spec/`; side effects outside `spec/` (e.g. a `cover.out` at the repo root) are conformant.
4. **Outcome.** On release (`Approved`), proceed to PR creation. On `Issues Found`, halt: surface the failing reviewer's `Blocker` findings, run no `git push` or `gh pr create`, and open no PR. A non-zero verify command therefore blocks PR creation before any push runs. (AC: `verify-gate-blocks-pr-below-threshold`)

## PR Creation

When the reviewer gate releases, the skill creates exactly **one** pull request. It dispatches once and reacts to a single outcome — it never sequences, retries, or orchestrates.

### Title and body

Before creating the PR, derive a draft title and body from the current branch's commits — title from the latest/squash subject; body summarizing the commits — and present them to the user for confirmation or edit. Create the PR with the **confirmed** title and body. (AC: `derives-and-confirms-title-body`)

### Built-in path (default)

When **no** `pull_request.delegate` is configured, the skill creates the PR itself:

1. Push the current branch to its remote (`git push -u origin <branch>`).
2. Open exactly one **ready-for-review** (non-draft) pull request whose base is the repository's default branch, using the confirmed title/body (`gh pr create --base <default-branch> --head <branch> --title … --body …`).

It creates the PR exactly once and interposes no orchestration. (AC: `creates-ready-pr-against-default-branch`)

### Config schema (`pull_request:` in `specscore.yaml`)

```yaml
pull_request:
  delegate:
    skill: <skill-name>     # the PR-creation skill to dispatch (e.g. commit-push-pr)
    args: <string>          # opaque args handed to the delegate verbatim
```

`delegate.skill` and `delegate.args` are the **only** recognized fields. The `pull_request:` schema MUST NOT grow any field expressing sequencing, retry, merge, or multi-PR behavior.

### Delegate override

When `pull_request.delegate` is configured, the skill dispatches that single delegate skill **instead of** the built-in path, passing `delegate.args`. It invokes the delegate **exactly once** and does not run the built-in `git push` / `gh pr create` path, sequence multiple delegates, or retry a failed delegate. (AC: `dispatches-configured-delegate`)

## Lifecycle Event

Runs **only** on successful PR creation — the built-in path completing, or a configured delegate returning explicit success.

1. **Publication + event.** Build the checkpoint manifest, apply [publication-policy.md](../shared/publication-policy.md) for the `pull_request.created` checkpoint, then emit `pull_request.created` **exactly once** with `publication_result` per [events.md](../shared/events.md). The event carries the created PR's identifier (URL/number) and the base/head branches. If creation fails, emit no event. (AC: `emits-created-once`)

## Architectural Boundary

This boundary is load-bearing and applies across the whole skill: **pull-request gates, creates one PR, and records — it never orchestrates.** Concretely, the skill MUST NOT:

- merge, auto-merge, or approve the PR;
- deploy, build, or release;
- stack, batch, or coordinate multiple PRs in one run;
- retry a failed delegate, or interpose orchestration between itself and the delegate.

Those belong to other skills (`ship`, the merge step) or to the delegate, never to this skill. The `pull_request:` config block MUST expose **only** `delegate.skill` and `delegate.args`. (AC: `bars-orchestration`)

## Red Flags

- Running `git push` or `gh pr create` before the `pull_request.pre_dispatch` gate releases (`Approved`).
- Carrying a hardcoded reviewer instead of loading `gates.pull_request.pre_dispatch`.
- Introducing a new reviewer type for the verify gate instead of the existing `type: deterministic`.
- Opening the PR as a draft, or against a base other than the repository's default branch, without explicit configuration.
- Creating the PR without first confirming the derived title/body with the user.
- Merging, auto-merging, deploying, stacking multiple PRs, or retrying a delegate.
- Adding any `pull_request:` config field beyond `delegate.skill` / `delegate.args`.
- Emitting `pull_request.created` when creation did not succeed.

## References

- [Feature: Pull Request Skill](../../spec/features/skills/pull-request/README.md) — the SpecScore Feature this skill implements.
- [ship SKILL.md](../ship/SKILL.md) — the sibling gate-and-dispatch skill whose structure this mirrors.
- [reviewer-gates/loader.md](../shared/reviewer-gates/loader.md), [reviewer-gates/runner.md](../shared/reviewer-gates/runner.md) — load + run the `gates.pull_request.pre_dispatch` reviewer list.
- [events.md](../shared/events.md) — the `pull_request.pre_dispatch` and `pull_request.created` event payloads this skill fires/emits.
