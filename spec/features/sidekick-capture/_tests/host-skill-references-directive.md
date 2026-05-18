---
type: rehearse-stub
status: pending
ac: host-skill-references-directive
feature: sidekick-capture
---

# Rehearse: host-skill-references-directive

## Scenario (from AC)

**Given** the latest SKILL.md files of `specstudio:ideate` and `specstudio:specify` after this Feature ships
**When** each file is read
**Then** each contains a markdown link to `skills/shared/sidekick-capture.md` located in the skill's checklist section; neither file copies the directive body inline.

## Verification approach

Grep `skills/ideate/SKILL.md` and `skills/specify/SKILL.md` for the link `../shared/sidekick-capture.md`; assert both contain it inside their `## Checklist` section.

---
*This document follows the https://specscore.md/scenario-specification*
