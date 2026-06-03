# Reviewer-Gates Config Loader

**Status:** Contract — shared instructional document for skills that consume a reviewer gate from `specscore.yaml`.
**Owned by:** [reviewer-gates](../../../spec/features/reviewer-gates/README.md) Feature.

## Purpose

This file tells a consuming skill (currently `specstudio:specify`; future consumers MUST follow the same protocol) how to load and validate a per-stage reviewer-gate configuration from the project's `specscore.yaml` BEFORE dispatching any reviewer. The output of following this protocol is a validated, ordered list of reviewer entries ready for the gate runner.

The loader's job is **load-time validation only** — it does not dispatch reviewers, aggregate verdicts, or implement the rerun policy. Those concerns live in the gate runner (see [reviewer-gates Feature](../../../spec/features/reviewer-gates/README.md) REQs `dispatch-serial`, `and-composition`, `rerun-policy`).

When validation fails for **any** reason described below, the consuming skill MUST:

1. Stop immediately. Do NOT dispatch any reviewer in the gate. Do NOT write or modify any artifact under `spec/`.
2. Emit a clear error message to the user. The message MUST cite the specific REQ slug from the [reviewer-gates Feature](../../../spec/features/reviewer-gates/README.md) that the configuration violates, and MUST include a link to that Feature.
3. Exit non-zero (in the Markdown-driven-skill sense: surface the failure as the skill's final outcome; do not continue past the gate; do not emit any success/approval event).

There is no auto-repair, no silent fallback to a built-in baseline reviewer, and no implicit default for any field.

## Inputs

| Input | Source | Required |
|---|---|---|
| `<event>` | The **gate-point event identifier** the consumer guards — either a canonical artifact-lifecycle event from [`events.md`](../events.md) (e.g., `feature.approved`, `idea.approved`, `plan.approved`) or a canonical pre-action gate-point event (e.g., `implementation.pre_commit`, `implementation.pre_push`). This is the key under `gates:`. A bare skill/command name (e.g., `specify`) is NOT a valid `<event>` and is rejected per [reviewer-gates#req:migration-to-event-keys](../../../spec/features/reviewer-gates/README.md#req-migration-to-event-keys) (see Step 1.5). For example, `specstudio:specify` passes `<event> = feature.approved`. | Yes |
| `specscore.yaml` | Repo-root file. Read verbatim; preserve key order. | Yes |
| Repo working tree | Used to resolve `prompt:` file paths declared by `type: ai` entries. | Yes |

## Output

On success: an ordered list of reviewer entries, each carrying the fields declared in `specscore.yaml` plus the resolved prompt-file path (for `type: ai`) and effective `min_approvers` value (for `type: human`, always `1` in MVP), **plus the resolved Approve threshold** (a whole letter `A`–`F`, default `B`; see Step 2.5). The list order is exactly the order entries appear under `gates.<event>.reviewers` — entry order is the dispatch order per [reviewer-gates#req:dispatch-serial](../../../spec/features/reviewer-gates/README.md#req-dispatch-serial).

On failure: no output; the consuming skill halts with the error described above.

## Protocol

Follow the steps in order. Do not skip ahead. The first failed step is terminal.

### Step 1 — Read `specscore.yaml`

Read the repo-root `specscore.yaml`. Preserve the file's key order in your working model — this loader MUST NOT rewrite the file, but a consumer that later writes to `specscore.yaml` for unrelated reasons MUST preserve every key under `gates:` verbatim per [SpecScore Repo Config](https://github.com/specscore/specscore/blob/main/spec/features/repo-config/README.md)'s `unknown-fields-preserved` requirement (see also [reviewer-gates#ac:gates-block-preserved](../../../spec/features/reviewer-gates/README.md#ac-gates-block-preserved)).

If `specscore.yaml` does not exist or cannot be parsed as YAML, refuse with:

> Error: cannot read `specscore.yaml` at repo root. The `<skill>` skill requires a `gates.<event>` configuration. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md for the canonical schema.

### Step 1.5 — Reject legacy command-keyed gate keys

This revision replaces command/skill-keyed gates (`gates.<skill>`, e.g. `gates.specify`) with **event-keyed** gates (`gates.<event>`) as a clean break — there is no back-compat window (per [reviewer-gates#req:migration-to-event-keys](../../../spec/features/reviewer-gates/README.md#req-migration-to-event-keys)).

Before resolving the gate, inspect every child key under the top-level `gates:` block. Each child key MUST be a registered gate-point event identifier — either a canonical artifact-lifecycle event from [`events.md`](../events.md) (e.g., `feature.approved`, `idea.approved`, `plan.approved`) or a canonical pre-action gate-point event (`implementation.pre_commit`, `implementation.pre_push`; see [reviewer-gates#req:gate-point-events-and-multi-fire](../../../spec/features/reviewer-gates/README.md#req-gate-point-events-and-multi-fire)). A child key that is a bare skill/command name (a name with no `.` segment that matches a known skill/command, e.g. `specify`, `plan`, `implement`) rather than a registered event identifier MUST be rejected. The loader MUST refuse, dispatch nothing, and halt — citing [reviewer-gates#req:migration-to-event-keys](../../../spec/features/reviewer-gates/README.md#req-migration-to-event-keys) and naming the event key to migrate to (verifies [reviewer-gates#ac:legacy-command-key-rejected](../../../spec/features/reviewer-gates/README.md#ac-legacy-command-key-rejected)):

> Error: gate key `gates.<key>` in `specscore.yaml` is a legacy command/skill-keyed gate. Gate keys are now **event-keyed** — there is no back-compat window (per [reviewer-gates#req:migration-to-event-keys](../../../spec/features/reviewer-gates/README.md#req-migration-to-event-keys)). Migrate `gates.specify` to `gates.feature.approved`. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md for the canonical event keys.

The mapping for the MVP consumer is `gates.specify` → `gates.feature.approved`. Halt here before Step 2 — do NOT dispatch any reviewer, and do NOT fall back to resolving the legacy key.

### Step 2 — Resolve `gates.<event>.reviewers`

Resolve the path `gates.<event>.reviewers` against the parsed config. Three failure modes — all of which refuse per [reviewer-gates#req:missing-gates-block-refuses](../../../spec/features/reviewer-gates/README.md#req-missing-gates-block-refuses) and verify [reviewer-gates#ac:missing-gates-block-refuses-with-error](../../../spec/features/reviewer-gates/README.md#ac-missing-gates-block-refuses-with-error):

| State | Refusal trigger |
|---|---|
| (a) no top-level `gates:` key | refuse |
| (b) `gates:` present but no `gates.<event>` sub-key | refuse |
| (c) `gates.<event>.reviewers` is an empty list (`[]`) | refuse |

In any of these three cases emit:

> Error: `gates.<event>.reviewers` is missing or empty in `specscore.yaml`. The `<skill>` skill MUST NOT run without a configured reviewer gate (per [reviewer-gates#req:missing-gates-block-refuses](../../../spec/features/reviewer-gates/README.md#req-missing-gates-block-refuses)). Add at minimum one `type: human` entry — see https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md for the canonical schema. Recommended minimal configuration:
>
> ```yaml
> gates:
>   <event>:
>     reviewers:
>       - name: human-approval
>         type: human
> ```

Halt. Do not dispatch. Do not fall back to any built-in baseline reviewer; do not fall back to any prior "User Review Gate" path.

If `gates.<event>.reviewers` resolves to a non-list value (e.g., a string, a mapping), refuse with the same minimal-configuration message and an additional sentence: `the value MUST be a YAML list of reviewer-entry objects`.

### Step 2.5 — Resolve the Approve threshold

Resolve the gate's Approve threshold per [reviewer-gates#req:threshold-config](../../../spec/features/reviewer-gates/README.md#req-threshold-config). Resolution order (first present wins):

1. `gates.<event>.threshold` (per-stage), if present.
2. else the top-level `grade.threshold`, if present.
3. else the built-in default `B`.

The resolved value MUST be one of the whole letters `A`, `B`, `C`, `D`, `F` (case-sensitive; no `E`, no `+`/`-` variants, no numbers). If a `threshold` key is present at either location but its value is outside that set, refuse — citing [reviewer-gates#req:threshold-config](../../../spec/features/reviewer-gates/README.md#req-threshold-config) (verifies [reviewer-gates#ac:invalid-threshold-refused](../../../spec/features/reviewer-gates/README.md#ac-invalid-threshold-refused)):

> Error: `threshold: <value>` in `specscore.yaml` is not one of the allowed grades `{A, B, C, D, F}` (per [reviewer-gates#req:threshold-config](../../../spec/features/reviewer-gates/README.md#req-threshold-config)). The Approve threshold MUST be a whole letter grade; `E`, `+`/`-` variants, and numbers are not allowed. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

Record the resolved threshold; it is returned alongside the validated reviewer list (Step 4) and consumed by the runner to derive the gate verdict (`Approved` iff `grade ≥ threshold`). Resolving the threshold is a pure read; per Step 1 and Note 6 any later write to `specscore.yaml` MUST preserve the `gates:` block and the top-level `grade:` block verbatim. (Verifies [reviewer-gates#ac:threshold-resolution-order](../../../spec/features/reviewer-gates/README.md#ac-threshold-resolution-order).)

### Step 3 — Validate each entry (in list order)

Iterate the `reviewers` list in declared order. For each entry, apply Steps 3a–3e below in order. The first violation in any entry refuses the entire load; the loader MUST NOT skip the bad entry and continue, MUST NOT dispatch any previously-validated entry, and MUST NOT dispatch the offending entry. The check order within an entry is fixed so that error messages are predictable.

After validating every entry, compute each entry's **effective name** — its declared `name:` if present, else its `type:` value (Step 3b) — and set the normalized record's `name` to that effective name (so downstream consumers, e.g. the runner's verdict map, always see a concrete name). Then confirm all effective names are unique within this gate's `reviewers:` list (case-sensitive string comparison). This means two entries that share a `type:` and both omit `name:` collide and MUST be disambiguated with an explicit `name:`. Duplicate effective names refuse per [reviewer-gates#req:reviewer-entry-required-fields](../../../spec/features/reviewer-gates/README.md#req-reviewer-entry-required-fields) (verifies [reviewer-gates#ac:duplicate-effective-name-refused](../../../spec/features/reviewer-gates/README.md#ac-duplicate-effective-name-refused)) with:

> Error: duplicate reviewer effective name `<value>` in `gates.<event>.reviewers` (the effective name is the declared `name:`, or the `type:` when `name:` is omitted). Effective names MUST be unique within a gate; give same-type entries an explicit `name:` to disambiguate (per [reviewer-gates#req:reviewer-entry-required-fields](../../../spec/features/reviewer-gates/README.md#req-reviewer-entry-required-fields)). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

#### Step 3a — `name:` optional (defaults to `type:`)

`name:` is OPTIONAL (verifies [reviewer-gates#ac:name-defaults-to-type](../../../spec/features/reviewer-gates/README.md#ac-name-defaults-to-type)). When present, it MUST be a non-empty string of lowercase plus hyphens only (regex check: `^[a-z][a-z0-9-]*$`). When omitted, the entry's **effective name** defaults to its `type:` value — computed and uniqueness-checked in the post-entry effective-name step above. On a present-but-invalid `name:` refuse with:

> Error: reviewer entry at index `<i>` in `gates.<event>.reviewers` has an invalid `name:` — when present it MUST be lowercase + hyphens (`^[a-z][a-z0-9-]*$`); omit it to default the effective name to `type:`. Per [reviewer-gates#req:reviewer-entry-required-fields](../../../spec/features/reviewer-gates/README.md#req-reviewer-entry-required-fields). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

#### Step 3b — `type:` required and recognized

Every entry MUST declare an explicit `type:` field. There is no implicit default. The value MUST be exactly one of the MVP type set `{ai, human, deterministic, auto-approve}`.

- If `type:` is absent, refuse — citing [reviewer-gates#req:no-untyped-entry](../../../spec/features/reviewer-gates/README.md#req-no-untyped-entry) (verifies [reviewer-gates#ac:untyped-entry-refused](../../../spec/features/reviewer-gates/README.md#ac-untyped-entry-refused)):

  > Error: reviewer entry `<name>` in `gates.<event>.reviewers` has no `type:` field. There is no implicit default — entries MUST declare one of `type: ai`, `type: human`, `type: deterministic`, or `type: auto-approve` (per [reviewer-gates#req:no-untyped-entry](../../../spec/features/reviewer-gates/README.md#req-no-untyped-entry)). If this entry was migrated from a legacy flat `reviewers:` registry, add `type: ai` and a `prompt:` path explicitly. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

- If `type:` is present but not in `{ai, human, deterministic, auto-approve}`, refuse — citing [reviewer-gates#req:mvp-type-set](../../../spec/features/reviewer-gates/README.md#req-mvp-type-set) (verifies [reviewer-gates#ac:unknown-type-refused](../../../spec/features/reviewer-gates/README.md#ac-unknown-type-refused)):

  > Error: reviewer entry `<name>` declares `type: <value>`, which is outside the MVP type set `{ai, human, deterministic, auto-approve}` (per [reviewer-gates#req:mvp-type-set](../../../spec/features/reviewer-gates/README.md#req-mvp-type-set)). Unknown types MUST NOT be treated as `ai`. A tool-backed check (lint, security scanner) is expressed as `type: deterministic` with a `run:` command, not as a bespoke type; further specialized types (`ux`, `peer-review-bot`, etc.) are deferred — see https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

#### Step 3c — If `type: ai`, validate the `ai` entry shape

These checks verify [reviewer-gates#req:ai-entry-shape](../../../spec/features/reviewer-gates/README.md#req-ai-entry-shape) and [reviewer-gates#ac:ai-entry-shape-violations-refused](../../../spec/features/reviewer-gates/README.md#ac-ai-entry-shape-violations-refused). Apply in order.

**3c.i — `prompt:` field present.** The entry MUST declare `prompt:` as a non-empty string. If absent or empty, refuse:

> Error: reviewer entry `<name>` of `type: ai` is missing the required `prompt:` field (per [reviewer-gates#req:ai-entry-shape](../../../spec/features/reviewer-gates/README.md#req-ai-entry-shape)). Provide a repo-relative path to a prompt file containing a documented blocker/advisory taxonomy section. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3c.ii — `prompt:` path is repo-relative and resolves inside the working tree.** The path MUST be expressed relative to the repo root; absolute paths and network URLs are forbidden. Reject any of:

- Path starts with `/` (absolute filesystem path).
- Path starts with a URL scheme (e.g., `http://`, `https://`, `file://`).
- After resolving against the repo root and normalizing `..` segments, the resulting absolute path is NOT a descendant of the repo root.
- Path resolves to a file that does not exist, or is not a regular file (e.g., a directory or symlink that escapes the working tree).

On any of the above, refuse:

> Error: reviewer entry `<name>` declares `prompt: <value>`, which does not resolve to a file inside the repo working tree (per [reviewer-gates#req:ai-entry-shape](../../../spec/features/reviewer-gates/README.md#req-ai-entry-shape)). Prompts MUST be repo-relative paths to files inside this repo — absolute filesystem paths and network URLs are forbidden. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3c.iii — prompt file contains a documented blocker/advisory taxonomy section.** Read the resolved prompt file. The contents MUST contain an explicit section (heading or clearly-labeled block) documenting which finding categories the reviewer treats as `Blocker` versus `Advisory`. A reasonable heuristic for "documented taxonomy section" is: the file contains BOTH the literal word `Blocker` and the literal word `Advisory` (case-sensitive, as section labels — not as casual prose), AND the words appear in a section heading or in a labeled list/table that maps finding categories to severities. If the file contains neither word in this structural sense, refuse:

> Error: reviewer entry `<name>`'s prompt file at `<path>` contains no documented blocker/advisory taxonomy section (per [reviewer-gates#req:ai-entry-shape](../../../spec/features/reviewer-gates/README.md#req-ai-entry-shape)). The prompt MUST explicitly state which finding categories are `Blocker` vs. `Advisory`. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3c.iv — Optional `ai` fields.** `model:` (string identifier; opaque to this loader) and `description:` (string ≤ 200 chars) MAY be present. If `description:` is present and exceeds 200 characters, refuse with a short message naming the cap. Unknown extra keys on a `type: ai` entry are NOT permitted in MVP — refuse with a message listing the recognized fields (`name`, `type`, `prompt`, `model`, `description`, `when`). (`when:` is the optional branch condition validated in Step 3-when.)

#### Step 3d — If `type: human`, validate the `human` entry shape

These checks verify [reviewer-gates#req:human-entry-shape](../../../spec/features/reviewer-gates/README.md#req-human-entry-shape), [reviewer-gates#ac:human-entry-min-approvers-cap](../../../spec/features/reviewer-gates/README.md#ac-human-entry-min-approvers-cap), and [reviewer-gates#ac:human-entry-rejects-prompt](../../../spec/features/reviewer-gates/README.md#ac-human-entry-rejects-prompt). Apply in order.

**3d.i — No `prompt:` field.** A `type: human` entry MUST NOT declare a `prompt:` field. Humans have no programmatic prompt. If present, refuse:

> Error: reviewer entry `<name>` of `type: human` declares a `prompt:` field. Humans have no programmatic prompt; the `prompt:` field is forbidden on human entries (per [reviewer-gates#req:human-entry-shape](../../../spec/features/reviewer-gates/README.md#req-human-entry-shape)). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3d.ii — `min_approvers:` if present, MUST be exactly `1`.** The field is optional and defaults to `1`. Any integer value ≥ 2 is refused in MVP — multi-approver workflows are deferred. Any non-integer or value < 1 is also refused (typing error). On violation, refuse:

> Error: reviewer entry `<name>` of `type: human` declares `min_approvers: <value>`. MVP pins `min_approvers: 1` — values > 1 are deferred (per [reviewer-gates#req:human-entry-shape](../../../spec/features/reviewer-gates/README.md#req-human-entry-shape) and the Feature's `## Not Doing` list). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3d.iii — Optional `human` fields.** `description:` (string ≤ 200 chars) MAY be present. Unknown extra keys on a `type: human` entry are NOT permitted in MVP — refuse with a message listing the recognized fields (`name`, `type`, `min_approvers`, `description`, `when`). (`when:` is the optional branch condition validated in Step 3-when — the home for per-branch autonomy masks.)

#### Step 3d-det — If `type: deterministic`, validate the `deterministic` entry shape

These checks verify [reviewer-gates#req:deterministic-entry-shape](../../../spec/features/reviewer-gates/README.md#req-deterministic-entry-shape) and [reviewer-gates#ac:deterministic-verdict-from-exit](../../../spec/features/reviewer-gates/README.md#ac-deterministic-verdict-from-exit). A `type: deterministic` entry runs a repo-local tool (linter, security scanner, conflict check) whose verdict is derived from the command's exit code at runtime (see [`runner.md`](./runner.md) Step 2-det). Apply in order.

**3d-det.i — `run:` field present.** The entry MUST declare `run:` as a non-empty string — a command resolvable in the repo (e.g., a script path or Make target). If absent or empty, refuse:

> Error: reviewer entry `<name>` of `type: deterministic` is missing the required `run:` field (per [reviewer-gates#req:deterministic-entry-shape](../../../spec/features/reviewer-gates/README.md#req-deterministic-entry-shape)). Provide a command resolvable in the repo (a script path or Make target). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3d-det.ii — No `prompt:` or `model:` field.** A `type: deterministic` entry runs a tool, not an LLM, and MUST NOT declare `prompt:` or `model:`. If either is present, refuse:

> Error: reviewer entry `<name>` of `type: deterministic` declares a `prompt:` or `model:` field. A deterministic check runs a tool via `run:`, not an LLM; `prompt:`/`model:` are forbidden on deterministic entries (per [reviewer-gates#req:deterministic-entry-shape](../../../spec/features/reviewer-gates/README.md#req-deterministic-entry-shape)). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3d-det.iii — No network commands.** Network commands are forbidden in `run:`. The loader SHOULD reject a `run:` value that is plainly a network fetch (e.g., begins with `curl`, `wget`, or contains a URL scheme such as `http://`/`https://`). On a plainly-network `run:`, refuse citing `deterministic-entry-shape`'s network prohibition.

**3d-det.iv — Optional `deterministic` fields.** `description:` (string ≤ 200 chars) MAY be present. Unknown extra keys on a `type: deterministic` entry are NOT permitted in MVP — refuse with a message listing the recognized fields (`name`, `type`, `run`, `description`, `when`). (`when:` is the optional branch condition validated in Step 3-when.)

#### Step 3d-auto-approve — If `type: auto-approve`, validate the `auto-approve` entry shape

These checks verify [reviewer-gates#req:auto-approve-entry-shape](../../../spec/features/reviewer-gates/README.md#req-auto-approve-entry-shape) and [reviewer-gates#ac:auto-approve-always-approves](../../../spec/features/reviewer-gates/README.md#ac-auto-approve-always-approves). A `type: auto-approve` entry dispatches nothing and always returns `Approved` with no findings — the explicit auto-approve placeholder that lets an event's gate be configured as "no review at this checkpoint" without removing the gate key.

**3d-auto-approve.i — No `prompt:`, `model:`, or `run:` field.** A `type: auto-approve` entry MUST NOT declare `prompt:`, `model:`, or `run:` — it dispatches nothing. If any is present, refuse:

> Error: reviewer entry `<name>` of `type: auto-approve` declares a `prompt:`, `model:`, or `run:` field. A `auto-approve` entry dispatches nothing and always approves; those fields are forbidden (per [reviewer-gates#req:auto-approve-entry-shape](../../../spec/features/reviewer-gates/README.md#req-auto-approve-entry-shape)). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3d-auto-approve.ii — Optional `auto-approve` fields.** `description:` (string ≤ 200 chars) MAY be present (e.g., the rationale for auto-approving this checkpoint). Unknown extra keys on a `type: auto-approve` entry are NOT permitted in MVP — refuse with a message listing the recognized fields (`name`, `type`, `description`, `when`). (`when:` is the optional branch condition validated in Step 3-when.)

#### Step 3-when — Validate the optional `when:` branch condition (any type)

This check verifies [reviewer-gates#req:gate-entry-when-condition](../../../spec/features/reviewer-gates/README.md#req-gate-entry-when-condition) and [reviewer-gates#ac:when-condition-masks-by-branch](../../../spec/features/reviewer-gates/README.md#ac-when-condition-masks-by-branch). The `when:` field is **optional on a reviewer entry of any type** (`ai`, `human`, `deterministic`, `auto-approve`) and is the home for per-branch autonomy masks (e.g., [approval-autonomy#req:branch-mask-via-gate-when](../../../spec/features/approval-autonomy/README.md#req-branch-mask-via-gate-when)). It is therefore an allowed extra key alongside each type's recognized-field set (Steps 3c.iv, 3d.iii, 3d-det.iv, 3d-auto-approve.ii — `when` is permitted on top of those lists and MUST NOT trip the unknown-extra-key rejection).

- If `when:` is **absent**, the entry always participates — record nothing for it (default unconditioned behavior) and continue.
- If `when:` is **present**, its value MUST be a string of the form `branch =~ <regex>`: the literal token `branch`, optional whitespace, the operator `=~`, optional whitespace, then a non-empty regular expression matched (anchored) against the current branch name. The grammar is **anchored regex** — there is no glob alternative (per [reviewer-gates#req:gate-entry-when-condition](../../../spec/features/reviewer-gates/README.md#req-gate-entry-when-condition)). The loader validates the **shape** only (the `branch =~ <non-empty-regex>` form); the runner resolves the current branch and applies the match at dispatch time (see [`runner.md`](./runner.md) Step 1.5). On a malformed `when:` — not a string, missing the `branch =~` prefix, or an empty/absent right-hand-side regex — refuse:

  > Error: reviewer entry `<name>` declares a malformed `when:` condition `<value>`. The `when:` field MUST be a string of the form `branch =~ <anchored-regex>` (e.g., `when: "branch =~ ^(main|master|release/)"`) — anchored regex, no glob alternative (per [reviewer-gates#req:gate-entry-when-condition](../../../spec/features/reviewer-gates/README.md#req-gate-entry-when-condition)). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

#### Step 3e — Record the validated entry

If Steps 3a–3d (and Step 3-when) pass for this entry, append a normalized record to the output list in declared order. The normalized record carries:

- `name`: as declared.
- `type`: as declared (`ai`, `human`, `deterministic`, or `auto-approve`).
- `when`: the original `when:` string if present (the `branch =~ <regex>` condition the runner applies per branch); absent when the entry carries no `when:` (the entry then always participates).
- For `type: ai`: `prompt_path` (resolved absolute path inside the working tree), `prompt_repo_relative_path` (the original value, useful for error reporting downstream), and `model` / `description` if present.
- For `type: human`: `min_approvers: 1` (always, in MVP), and `description` if present.
- For `type: deterministic`: `run` (the command string, as declared), and `description` if present.
- For `type: auto-approve`: `description` if present (no other fields).

### Step 4 — Return the validated list

After every entry passes Steps 3a–3e and the uniqueness check, return the validated list to the calling skill in declared order, together with the resolved Approve threshold from Step 2.5. The calling skill MUST consume this list as-is — it MUST NOT reorder entries, MUST NOT silently inject any additional reviewer (notably: no hidden built-in baseline reviewer), and MUST NOT silently drop any entry.

## Notes for skill authors

1. **Where to invoke this loader.** Call this loader BEFORE any other gate-related work — before dispatching the first reviewer, before any user-facing prompt about the gate, before any artifact write. The whole point of refusing at load time is to fail before the user invests effort.
2. **No partial output on failure.** If any step fails, the consumer MUST NOT carry a "partial" reviewer list forward. The output is all-or-nothing.
3. **Refusal copy.** The error templates above are recommended copy. A consuming skill MAY adapt wording, but every refusal MUST (a) cite the specific REQ slug from the [reviewer-gates Feature](../../../spec/features/reviewer-gates/README.md), (b) include a link to that Feature, and (c) make clear that no reviewer was dispatched and no artifact was modified.
4. **Halting after first failure inside a single entry.** Within Step 3 for a single entry, the first violation is terminal for that entry (and thus for the load). Do not accumulate multiple errors per entry — surface one clear error and halt. This keeps error messages predictable.
5. **The validator is consumer-agnostic.** Substitute `<event>` for the gate-point event the calling skill guards (e.g., `feature.approved` for `specstudio:specify`; `implementation.pre_commit` for an implement-time gate). The same loader serves any future consumer (`plan`, `implement`, `verify`, `recap`) without contract changes — see the Feature's `## Architecture` section. Gate keys are event identifiers, never bare skill/command names (Step 1.5).
6. **`gates:` block preservation across reads.** Reading `specscore.yaml` here is a pure read. If the consuming skill later writes to `specscore.yaml` for unrelated reasons, it MUST preserve the `gates:` block verbatim — every child key, every list-entry order, every field — per the SpecScore Repo Config Feature's `unknown-fields-preserved` requirement. This loader does not itself write to `specscore.yaml`.

## AC verification map

This loader is the implementation of the following acceptance criteria from the [reviewer-gates Feature](../../../spec/features/reviewer-gates/README.md):

| AC | Where verified in this loader |
|---|---|
| [`gates-block-preserved`](../../../spec/features/reviewer-gates/README.md#ac-gates-block-preserved) | Step 1 (preserve key order on read); Note 6 (consumers MUST NOT rewrite the `gates:` block on unrelated writes). |
| [`legacy-command-key-rejected`](../../../spec/features/reviewer-gates/README.md#ac-legacy-command-key-rejected) | Step 1.5 — a bare skill/command name as a `gates:` child key (e.g., `gates.specify`) → refuse, cite `migration-to-event-keys`, name the event key (`gates.feature.approved`), dispatch nothing, halt. |
| [`untyped-entry-refused`](../../../spec/features/reviewer-gates/README.md#ac-untyped-entry-refused) | Step 3b — `type:` absent → refuse, cite `no-untyped-entry`, halt. |
| [`unknown-type-refused`](../../../spec/features/reviewer-gates/README.md#ac-unknown-type-refused) | Step 3b — `type:` not in `{ai, human, deterministic, auto-approve}` → refuse, cite `mvp-type-set`, halt. |
| [`ai-entry-shape-violations-refused`](../../../spec/features/reviewer-gates/README.md#ac-ai-entry-shape-violations-refused) | Step 3c.i (missing `prompt:`), Step 3c.ii (path outside repo), Step 3c.iii (no documented blocker/advisory taxonomy) — all refuse, cite `ai-entry-shape`, halt. |
| [`human-entry-min-approvers-cap`](../../../spec/features/reviewer-gates/README.md#ac-human-entry-min-approvers-cap) | Step 3d.ii — `min_approvers > 1` → refuse, cite `human-entry-shape`'s MVP cap, halt. |
| [`human-entry-rejects-prompt`](../../../spec/features/reviewer-gates/README.md#ac-human-entry-rejects-prompt) | Step 3d.i — `prompt:` present on `type: human` → refuse, cite `human-entry-shape`'s prohibition, halt. |
| [`deterministic-verdict-from-exit`](../../../spec/features/reviewer-gates/README.md#ac-deterministic-verdict-from-exit) | Step 3d-det — validate the `type: deterministic` shape (`run:` required; `prompt:`/`model:` forbidden) at load time; the exit-code→verdict mapping itself is the runner's job ([`runner.md`](./runner.md) Step 2-det). |
| [`auto-approve-always-approves`](../../../spec/features/reviewer-gates/README.md#ac-auto-approve-always-approves) | Step 3d-auto-approve — validate the `type: auto-approve` shape (no `prompt:`/`model:`/`run:`) at load time; the always-approve-dispatch-nothing behavior is the runner's job ([`runner.md`](./runner.md) Step 2-auto-approve). |
| [`missing-gates-block-refuses-with-error`](../../../spec/features/reviewer-gates/README.md#ac-missing-gates-block-refuses-with-error) | Step 2 — all three missing/empty states (no `gates:`, no `gates.<event>`, empty `reviewers: []`) refuse, recommend minimal `type: human` configuration, halt. |
| [`when-condition-masks-by-branch`](../../../spec/features/reviewer-gates/README.md#ac-when-condition-masks-by-branch) | Step 3-when — validate the optional `when: "branch =~ <anchored-regex>"` shape (allowed on any type); a malformed `when:` refuses citing `gate-entry-when-condition`; the validated `when:` is carried through to the normalized record (Step 3e). The per-branch match itself is applied by the runner ([`runner.md`](./runner.md) Step 1.5). |
| [`threshold-resolution-order`](../../../spec/features/reviewer-gates/README.md#ac-threshold-resolution-order) | Step 2.5 — per-stage `gates.<event>.threshold` → top-level `grade.threshold` → default `B`; resolved threshold returned in Step 4 output. |
| [`invalid-threshold-refused`](../../../spec/features/reviewer-gates/README.md#ac-invalid-threshold-refused) | Step 2.5 — a `threshold` value outside `{A, B, C, D, F}` refuses, cites `threshold-config`, halts. |
