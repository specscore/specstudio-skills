```yaml
feature: ideate
revision: d3eba7b
verdicts:
  - ac: ideate#ac:hard-gate-enforced
    verdict: unmapped
    justification: "no commits reference this AC"
  - ac: ideate#ac:artifact-conformance
    verdict: unmapped
    justification: "no commits reference this AC"
  - ac: ideate#ac:phase-discipline
    verdict: unmapped
    justification: "no commits reference this AC"
  - ac: ideate#ac:cli-vs-fallback
    verdict: unmapped
    justification: "no commits reference this AC"
  - ac: ideate#ac:lifecycle-events
    verdict: unmapped
    justification: "no commits reference this AC"
  - ac: ideate#ac:approval-detection
    verdict: unmapped
    justification: "no commits reference this AC"
  - ac: ideate#ac:open-questions-resolution
    verdict: pass
    justification: "SKILL.md User Review Gate adds an open-questions resolution step covering all seven scenarios (skip placeholder, count+list+menu, fold/remove, chat-in-order, annotate-no-section, skip-untouched-no-event, re-lint+status-appropriate event)."
  - ac: ideate#ac:lint-failure-recovery
    verdict: unmapped
    justification: "no commits reference this AC"
  - ac: ideate#ac:skip-condition-respected
    verdict: unmapped
    justification: "no commits reference this AC"
  - ac: ideate#ac:promotion-boundary-held
    verdict: unmapped
    justification: "no commits reference this AC"
```

# Verify Report — ideate @ d3eba7b

**Tally:** 1 passed, 0 failed, 9 unmapped, 0 errored (10 ACs total).

## AC: hard-gate-enforced

**Verdict:** unmapped

**Justification:** No commits reference this AC.

**Commits:**
- No commits reference this AC.

**Evidence:**
- (none)

## AC: artifact-conformance

**Verdict:** unmapped

**Justification:** No commits reference this AC.

**Commits:**
- No commits reference this AC.

**Evidence:**
- (none)

## AC: phase-discipline

**Verdict:** unmapped

**Justification:** No commits reference this AC.

**Commits:**
- No commits reference this AC.

**Evidence:**
- (none)

## AC: cli-vs-fallback

**Verdict:** unmapped

**Justification:** No commits reference this AC.

**Commits:**
- No commits reference this AC.

**Evidence:**
- (none)

## AC: lifecycle-events

**Verdict:** unmapped

**Justification:** No commits reference this AC.

**Commits:**
- No commits reference this AC.

**Evidence:**
- (none)

## AC: approval-detection

**Verdict:** unmapped

**Justification:** No commits reference this AC.

**Commits:**
- No commits reference this AC.

**Evidence:**
- (none)

## AC: open-questions-resolution

**Verdict:** pass

**Justification:** The `## User Review Gate` section of `skills/ideate/SKILL.md` adds a "Resolve open questions first" step that covers all seven scenarios: skip when the section is the `None at this time.` placeholder (exact match); state the count + list + offer wizard/chat/skip via `AskUserQuestion` before approval; wizard presents 2–4 candidate answers and folds/removes the resolved question (last resolved → body set to `None at this time.`); chat asks one at a time in listed order; answers with no fitting section are annotated inline rather than discarded; skip leaves the section untouched with no event; folding re-lints with one-shot `--fix` and emits the status-appropriate `idea.drafted`/`idea.updated` event with no new event type.

**Commits:**
- d3eba7b — feat(ideate): resolve open questions at the review gate

**Evidence:**
- skills/ideate/SKILL.md (commit d3eba7b), "## User Review Gate" → "Resolve open questions first" steps 1–4 + "Request approval"
- skills/ideate/SKILL.md checklist + verification updates referencing the open-questions resolution

## AC: lint-failure-recovery

**Verdict:** unmapped

**Justification:** No commits reference this AC.

**Commits:**
- No commits reference this AC.

**Evidence:**
- (none)

## AC: skip-condition-respected

**Verdict:** unmapped

**Justification:** No commits reference this AC.

**Commits:**
- No commits reference this AC.

**Evidence:**
- (none)

## AC: promotion-boundary-held

**Verdict:** unmapped

**Justification:** No commits reference this AC.

**Commits:**
- No commits reference this AC.

**Evidence:**
- (none)
