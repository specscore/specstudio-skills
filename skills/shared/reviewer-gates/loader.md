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
| `<skill>` | The bare skill name of the calling consumer (e.g., `specify`). The plugin-namespace form `specstudio:specify` is NOT used here — `reviewer-gates` REQ `gates-block-location` forbids it in MVP. | Yes |
| `specscore.yaml` | Repo-root file. Read verbatim; preserve key order. | Yes |
| Repo working tree | Used to resolve `prompt:` file paths declared by `type: ai` entries. | Yes |

## Output

On success: an ordered list of reviewer entries, each carrying the fields declared in `specscore.yaml` plus the resolved prompt-file path (for `type: ai`) and effective `min_approvers` value (for `type: human`, always `1` in MVP), **plus the resolved Approve threshold** (a whole letter `A`–`F`, default `B`; see Step 2.5). The list order is exactly the order entries appear under `gates.<skill>.reviewers` — entry order is the dispatch order per [reviewer-gates#req:dispatch-serial](../../../spec/features/reviewer-gates/README.md#req-dispatch-serial).

On failure: no output; the consuming skill halts with the error described above.

## Protocol

Follow the steps in order. Do not skip ahead. The first failed step is terminal.

### Step 1 — Read `specscore.yaml`

Read the repo-root `specscore.yaml`. Preserve the file's key order in your working model — this loader MUST NOT rewrite the file, but a consumer that later writes to `specscore.yaml` for unrelated reasons MUST preserve every key under `gates:` verbatim per [SpecScore Repo Config](https://github.com/specscore/specscore/blob/main/spec/features/repo-config/README.md)'s `unknown-fields-preserved` requirement (see also [reviewer-gates#ac:gates-block-preserved](../../../spec/features/reviewer-gates/README.md#ac-gates-block-preserved)).

If `specscore.yaml` does not exist or cannot be parsed as YAML, refuse with:

> Error: cannot read `specscore.yaml` at repo root. The `<skill>` skill requires a `gates.<skill>` configuration. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md for the canonical schema.

### Step 2 — Resolve `gates.<skill>.reviewers`

Resolve the path `gates.<skill>.reviewers` against the parsed config. Three failure modes — all of which refuse per [reviewer-gates#req:missing-gates-block-refuses](../../../spec/features/reviewer-gates/README.md#req-missing-gates-block-refuses) and verify [reviewer-gates#ac:missing-gates-block-refuses-with-error](../../../spec/features/reviewer-gates/README.md#ac-missing-gates-block-refuses-with-error):

| State | Refusal trigger |
|---|---|
| (a) no top-level `gates:` key | refuse |
| (b) `gates:` present but no `gates.<skill>` sub-key | refuse |
| (c) `gates.<skill>.reviewers` is an empty list (`[]`) | refuse |

In any of these three cases emit:

> Error: `gates.<skill>.reviewers` is missing or empty in `specscore.yaml`. The `<skill>` skill MUST NOT run without a configured reviewer gate (per [reviewer-gates#req:missing-gates-block-refuses](../../../spec/features/reviewer-gates/README.md#req-missing-gates-block-refuses)). Add at minimum one `type: human` entry — see https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md for the canonical schema. Recommended minimal configuration:
>
> ```yaml
> gates:
>   <skill>:
>     reviewers:
>       - name: human-approval
>         type: human
> ```

Halt. Do not dispatch. Do not fall back to any built-in baseline reviewer; do not fall back to any prior "User Review Gate" path.

If `gates.<skill>.reviewers` resolves to a non-list value (e.g., a string, a mapping), refuse with the same minimal-configuration message and an additional sentence: `the value MUST be a YAML list of reviewer-entry objects`.

### Step 2.5 — Resolve the Approve threshold

Resolve the gate's Approve threshold per [reviewer-gates#req:threshold-config](../../../spec/features/reviewer-gates/README.md#req-threshold-config). Resolution order (first present wins):

1. `gates.<skill>.threshold` (per-stage), if present.
2. else the top-level `grade.threshold`, if present.
3. else the built-in default `B`.

The resolved value MUST be one of the whole letters `A`, `B`, `C`, `D`, `F` (case-sensitive; no `E`, no `+`/`-` variants, no numbers). If a `threshold` key is present at either location but its value is outside that set, refuse — citing [reviewer-gates#req:threshold-config](../../../spec/features/reviewer-gates/README.md#req-threshold-config) (verifies [reviewer-gates#ac:invalid-threshold-refused](../../../spec/features/reviewer-gates/README.md#ac-invalid-threshold-refused)):

> Error: `threshold: <value>` in `specscore.yaml` is not one of the allowed grades `{A, B, C, D, F}` (per [reviewer-gates#req:threshold-config](../../../spec/features/reviewer-gates/README.md#req-threshold-config)). The Approve threshold MUST be a whole letter grade; `E`, `+`/`-` variants, and numbers are not allowed. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

Record the resolved threshold; it is returned alongside the validated reviewer list (Step 4) and consumed by the runner to derive the gate verdict (`Approved` iff `grade ≥ threshold`). Resolving the threshold is a pure read; per Step 1 and Note 6 any later write to `specscore.yaml` MUST preserve the `gates:` block and the top-level `grade:` block verbatim. (Verifies [reviewer-gates#ac:threshold-resolution-order](../../../spec/features/reviewer-gates/README.md#ac-threshold-resolution-order).)

### Step 3 — Validate each entry (in list order)

Iterate the `reviewers` list in declared order. For each entry, apply Steps 3a–3e below in order. The first violation in any entry refuses the entire load; the loader MUST NOT skip the bad entry and continue, MUST NOT dispatch any previously-validated entry, and MUST NOT dispatch the offending entry. The check order within an entry is fixed so that error messages are predictable.

After validating every entry, also confirm that all `name:` values are unique within this gate's `reviewers:` list (case-sensitive string comparison). Duplicate names refuse per [reviewer-gates#req:reviewer-entry-required-fields](../../../spec/features/reviewer-gates/README.md#req-reviewer-entry-required-fields) with:

> Error: duplicate reviewer `name:` `<value>` in `gates.<skill>.reviewers`. Names MUST be unique within a gate (per [reviewer-gates#req:reviewer-entry-required-fields](../../../spec/features/reviewer-gates/README.md#req-reviewer-entry-required-fields)). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

#### Step 3a — `name:` required

Every entry MUST declare `name:` as a non-empty string. The value MUST be lowercase plus hyphens only (regex check: `^[a-z][a-z0-9-]*$`). On any violation refuse with:

> Error: reviewer entry at index `<i>` in `gates.<skill>.reviewers` is missing or has an invalid `name:` (required: lowercase + hyphens). Per [reviewer-gates#req:reviewer-entry-required-fields](../../../spec/features/reviewer-gates/README.md#req-reviewer-entry-required-fields). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

#### Step 3b — `type:` required and recognized

Every entry MUST declare an explicit `type:` field. There is no implicit default. The value MUST be exactly one of the MVP type set `{ai, human}`.

- If `type:` is absent, refuse — citing [reviewer-gates#req:no-untyped-entry](../../../spec/features/reviewer-gates/README.md#req-no-untyped-entry) (verifies [reviewer-gates#ac:untyped-entry-refused](../../../spec/features/reviewer-gates/README.md#ac-untyped-entry-refused)):

  > Error: reviewer entry `<name>` in `gates.<skill>.reviewers` has no `type:` field. There is no implicit default — entries MUST declare `type: ai` or `type: human` (per [reviewer-gates#req:no-untyped-entry](../../../spec/features/reviewer-gates/README.md#req-no-untyped-entry)). If this entry was migrated from a legacy flat `reviewers:` registry, add `type: ai` and a `prompt:` path explicitly. See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

- If `type:` is present but not in `{ai, human}`, refuse — citing [reviewer-gates#req:mvp-type-set](../../../spec/features/reviewer-gates/README.md#req-mvp-type-set) (verifies [reviewer-gates#ac:unknown-type-refused](../../../spec/features/reviewer-gates/README.md#ac-unknown-type-refused)):

  > Error: reviewer entry `<name>` declares `type: <value>`, which is outside the MVP type set `{ai, human}` (per [reviewer-gates#req:mvp-type-set](../../../spec/features/reviewer-gates/README.md#req-mvp-type-set)). Unknown types MUST NOT be treated as `ai`. Extension to additional types (`lint`, `security`, `ux`, etc.) is deferred — see https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

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

**3c.iv — Optional `ai` fields.** `model:` (string identifier; opaque to this loader) and `description:` (string ≤ 200 chars) MAY be present. If `description:` is present and exceeds 200 characters, refuse with a short message naming the cap. Unknown extra keys on a `type: ai` entry are NOT permitted in MVP — refuse with a message listing the recognized fields (`name`, `type`, `prompt`, `model`, `description`).

#### Step 3d — If `type: human`, validate the `human` entry shape

These checks verify [reviewer-gates#req:human-entry-shape](../../../spec/features/reviewer-gates/README.md#req-human-entry-shape), [reviewer-gates#ac:human-entry-min-approvers-cap](../../../spec/features/reviewer-gates/README.md#ac-human-entry-min-approvers-cap), and [reviewer-gates#ac:human-entry-rejects-prompt](../../../spec/features/reviewer-gates/README.md#ac-human-entry-rejects-prompt). Apply in order.

**3d.i — No `prompt:` field.** A `type: human` entry MUST NOT declare a `prompt:` field. Humans have no programmatic prompt. If present, refuse:

> Error: reviewer entry `<name>` of `type: human` declares a `prompt:` field. Humans have no programmatic prompt; the `prompt:` field is forbidden on human entries (per [reviewer-gates#req:human-entry-shape](../../../spec/features/reviewer-gates/README.md#req-human-entry-shape)). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3d.ii — `min_approvers:` if present, MUST be exactly `1`.** The field is optional and defaults to `1`. Any integer value ≥ 2 is refused in MVP — multi-approver workflows are deferred. Any non-integer or value < 1 is also refused (typing error). On violation, refuse:

> Error: reviewer entry `<name>` of `type: human` declares `min_approvers: <value>`. MVP pins `min_approvers: 1` — values > 1 are deferred (per [reviewer-gates#req:human-entry-shape](../../../spec/features/reviewer-gates/README.md#req-human-entry-shape) and the Feature's `## Not Doing` list). See https://github.com/specscore/specstudio-skills/blob/main/spec/features/reviewer-gates/README.md.

**3d.iii — Optional `human` fields.** `description:` (string ≤ 200 chars) MAY be present. Unknown extra keys on a `type: human` entry are NOT permitted in MVP — refuse with a message listing the recognized fields (`name`, `type`, `min_approvers`, `description`).

#### Step 3e — Record the validated entry

If Steps 3a–3d pass for this entry, append a normalized record to the output list in declared order. The normalized record carries:

- `name`: as declared.
- `type`: as declared (`ai` or `human`).
- For `type: ai`: `prompt_path` (resolved absolute path inside the working tree), `prompt_repo_relative_path` (the original value, useful for error reporting downstream), and `model` / `description` if present.
- For `type: human`: `min_approvers: 1` (always, in MVP), and `description` if present.

### Step 4 — Return the validated list

After every entry passes Steps 3a–3e and the uniqueness check, return the validated list to the calling skill in declared order, together with the resolved Approve threshold from Step 2.5. The calling skill MUST consume this list as-is — it MUST NOT reorder entries, MUST NOT silently inject any additional reviewer (notably: no hidden built-in baseline reviewer), and MUST NOT silently drop any entry.

## Notes for skill authors

1. **Where to invoke this loader.** Call this loader BEFORE any other gate-related work — before dispatching the first reviewer, before any user-facing prompt about the gate, before any artifact write. The whole point of refusing at load time is to fail before the user invests effort.
2. **No partial output on failure.** If any step fails, the consumer MUST NOT carry a "partial" reviewer list forward. The output is all-or-nothing.
3. **Refusal copy.** The error templates above are recommended copy. A consuming skill MAY adapt wording, but every refusal MUST (a) cite the specific REQ slug from the [reviewer-gates Feature](../../../spec/features/reviewer-gates/README.md), (b) include a link to that Feature, and (c) make clear that no reviewer was dispatched and no artifact was modified.
4. **Halting after first failure inside a single entry.** Within Step 3 for a single entry, the first violation is terminal for that entry (and thus for the load). Do not accumulate multiple errors per entry — surface one clear error and halt. This keeps error messages predictable.
5. **The validator is consumer-agnostic.** Substitute `<skill>` for the calling skill's bare name (e.g., `specify`). The same loader serves any future consumer (`plan`, `implement`, `verify`, `recap`) without contract changes — see the Feature's `## Architecture` section.
6. **`gates:` block preservation across reads.** Reading `specscore.yaml` here is a pure read. If the consuming skill later writes to `specscore.yaml` for unrelated reasons, it MUST preserve the `gates:` block verbatim — every child key, every list-entry order, every field — per the SpecScore Repo Config Feature's `unknown-fields-preserved` requirement. This loader does not itself write to `specscore.yaml`.

## AC verification map

This loader is the implementation of the following acceptance criteria from the [reviewer-gates Feature](../../../spec/features/reviewer-gates/README.md):

| AC | Where verified in this loader |
|---|---|
| [`gates-block-preserved`](../../../spec/features/reviewer-gates/README.md#ac-gates-block-preserved) | Step 1 (preserve key order on read); Note 6 (consumers MUST NOT rewrite the `gates:` block on unrelated writes). |
| [`untyped-entry-refused`](../../../spec/features/reviewer-gates/README.md#ac-untyped-entry-refused) | Step 3b — `type:` absent → refuse, cite `no-untyped-entry`, halt. |
| [`unknown-type-refused`](../../../spec/features/reviewer-gates/README.md#ac-unknown-type-refused) | Step 3b — `type:` not in `{ai, human}` → refuse, cite `mvp-type-set`, halt. |
| [`ai-entry-shape-violations-refused`](../../../spec/features/reviewer-gates/README.md#ac-ai-entry-shape-violations-refused) | Step 3c.i (missing `prompt:`), Step 3c.ii (path outside repo), Step 3c.iii (no documented blocker/advisory taxonomy) — all refuse, cite `ai-entry-shape`, halt. |
| [`human-entry-min-approvers-cap`](../../../spec/features/reviewer-gates/README.md#ac-human-entry-min-approvers-cap) | Step 3d.ii — `min_approvers > 1` → refuse, cite `human-entry-shape`'s MVP cap, halt. |
| [`human-entry-rejects-prompt`](../../../spec/features/reviewer-gates/README.md#ac-human-entry-rejects-prompt) | Step 3d.i — `prompt:` present on `type: human` → refuse, cite `human-entry-shape`'s prohibition, halt. |
| [`missing-gates-block-refuses-with-error`](../../../spec/features/reviewer-gates/README.md#ac-missing-gates-block-refuses-with-error) | Step 2 — all three missing/empty states (no `gates:`, no `gates.<skill>`, empty `reviewers: []`) refuse, recommend minimal `type: human` configuration, halt. |
| [`threshold-resolution-order`](../../../spec/features/reviewer-gates/README.md#ac-threshold-resolution-order) | Step 2.5 — per-stage `gates.<skill>.threshold` → top-level `grade.threshold` → default `B`; resolved threshold returned in Step 4 output. |
| [`invalid-threshold-refused`](../../../spec/features/reviewer-gates/README.md#ac-invalid-threshold-refused) | Step 2.5 — a `threshold` value outside `{A, B, C, D, F}` refuses, cites `threshold-config`, halts. |
