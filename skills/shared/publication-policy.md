# Publication Policy Protocol

**Status:** Canonical
**Applies to:** Producer skills that create, edit, stage, commit, push, or emit lifecycle events.

SpecStudio skills do not own the durable publication config schema. The schema lives in SpecScore config, and deterministic mutation and resolution live behind the `specscore publication` CLI group. Skills own the conversational pieces: collecting first-run preferences, tracking the path manifest for the current checkpoint, preserving approval gates, disclosing the resolved policy, and invoking the CLI helpers.

## Checkpoint Protocol

At every publication checkpoint, the skill MUST:

1. Build a manifest of paths created or edited for this checkpoint. Include any paths returned by `specscore` CLI helpers in machine-readable `touched_paths`.
2. Resolve policy for the current context with `specscore publication resolve` when available. Pass the command name, event name or milestone name, task override when present, session override when present, project path, and current branch.
3. If no effective policy exists at any scope, ask the first-run preference prompt in `## First-Run Prompt`.
4. Reject invalid action sequences before any git operation. Valid actions are an ordered subset of `[stage, commit, push]`; `commit` requires `stage`, and `push` requires both `stage` and `commit`.
5. Disclose the checkpoint before acting. Name the event or milestone, resolved actions, allowed actions after branch safety, policy source when known, manifest paths, and any blocked action reason.
6. Apply only the allowed actions:
   - Empty list: leave edits unstaged.
   - `stage`: stage only manifest paths.
   - `commit`: run the manifest safety check in `## Commit Safety`, then commit only after the relevant human/reviewer gate has released for that checkpoint.
   - `push`: run branch safety first; push only when allowed by the resolved branch policy and the current branch has an upstream.
7. Record the publication result on any emitted event, using the common extension in [events.md](events.md).

Publication policy is never an approval policy. A configured `commit` or `push` action runs only after the artifact lifecycle event or milestone is legitimately reached. It must not transition Draft to Approved, release a reviewer gate, skip a user approval phrase, or move to a downstream skill.

## First-Run Prompt

When no task, session, command, event, milestone, project, user, or built-in policy applies, ask the user for the workflow before the first publication checkpoint acts:

| User-facing option | Resolved actions |
|---|---|
| `just edit` | `[]` |
| `stage` | `[stage]` |
| `commit` | `[stage, commit]` |
| `commit & push` | `[stage, commit, push]` |

Highlight `stage` as the default option, because it preserves the historical SpecStudio behavior.

In the same prompt, ask where the preference applies:

| Scope | Behavior |
|---|---|
| This run only | Keep in the current skill invocation only. |
| This session | Keep in the current agent session only. |
| User default | Persist through `specscore publication set --scope user ...`. |
| Project default | Persist through `specscore publication set --scope project ...`. |

Durable saves MUST be delegated to `specscore publication set`. Do not hand-edit `specscore.yaml` or the user config for publication settings when the CLI mutation exists. If the needed CLI command is unavailable, keep the preference at run or session scope and tell the user durable persistence is blocked until the companion CLI command is available.

## Resolution Context

Prefer event policy over command defaults. Skills should resolve policy for the artifact lifecycle event they are about to emit whenever there is a canonical event, for example `idea.drafted`, `idea.approved`, `idea.updated`, `feature.specified`, `feature.approved`, `feature.updated`, `plan.drafted`, `plan.approved`, `plan.updated`, `verify.completed`, or `recap.completed`.

Use a milestone only when the checkpoint has no canonical event. Stable milestone names are cataloged in [events.md](events.md). Do not invent a configurable milestone name in a skill file without adding it to that catalog.

Resolution precedence is:

1. Task override.
2. Session override.
3. Command-scoped event or milestone override.
4. Event or milestone override.
5. Command default.
6. Project default.
7. User default.
8. Built-in default.

Branch deny rules are safety constraints. A more specific action preference must not weaken a user or project branch denial.

## Manifest Discipline

The manifest is the intent boundary for staging and committing. It contains only paths this checkpoint created or edited, plus paths the CLI explicitly reports as touched by helper commands.

- Stage only manifest paths unless the user explicitly approves broadening the manifest.
- Do not stage unrelated files just because they are present in the worktree.
- If a helper mutates status or config and does not report touched paths, add only the paths you can prove it changed; otherwise stop before publication and ask for direction.

## Commit Safety

Before any skill-created commit:

1. Compare the manifest to the staged index.
2. If unrelated staged paths are present, stop and ask whether to include them in the manifest, unstage them, or abort.
3. Do not silently commit unrelated staged changes.
4. Unstaged unrelated changes may remain in the worktree. They are not included unless the user explicitly adds them to the manifest.

`implement` still has its batch approval gate and `Verifies:` trailer discipline. Publication policy may supply the post-approval action list, but it does not allow subagents to commit, does not skip the consolidated diff approval, and does not remove the requirement that implementation commits carry the required trailer.

## Push Safety

Before `push`, resolve the current branch and apply branch policy. Refuse push on detached HEAD, missing upstream, denied branch names, or branches that match no allow pattern when an allow list is configured. Treat `main`, `master`, and `release/*` as denied by default unless project policy explicitly configures otherwise through the companion schema. Report the reason and record the blocked action in the publication result.

## CLI Fallback

If `specscore publication resolve` is unavailable, the skill may apply only run/session preferences it collected in the conversation and the built-in default prompt behavior above. It must not parse or mutate durable publication config by hand.

If `specscore publication set` is unavailable, durable user/project saves are unavailable for this run. Keep the user's choice at run or session scope only.

If CLI commit or push helpers are unavailable, use ordinary git commands only after performing the manifest, unrelated-index, approval-gate, and branch-safety checks described here.

## Checkpoint Summary

Every producer handoff message should include a concise checkpoint line after the policy acts:

```text
Publication: event=<event-or-milestone>, actions=<resolved>, executed=<executed>, skipped=<action: reason>, manifest=<paths>, commit=<sha-or-none>, push=<target-or-none>
```

This line is user-facing disclosure; emitted events carry the machine-readable publication result.
