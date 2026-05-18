---
type: rehearse-stub
status: pending
ac: back-link-section-created-when-absent
feature: sidekick-capture
---

# Rehearse: back-link-section-created-when-absent

## Scenario (from AC)

**Given** a source artifact whose markdown body has no `## Sidekick Seeds Generated` section
**When** the skill writes a seed pointing at that artifact
**Then** the section is created with exactly one heading line (`## Sidekick Seeds Generated`) and one entry bullet beneath it; the section is positioned immediately before the SpecScore footer line if present, otherwise at end-of-file.

## Verification approach

Pre-create two fixtures: (a) a source artifact with a footer and no existing section, (b) a source artifact with no footer and no existing section. Invoke against each. Assert: in case (a), the new section is immediately before the footer; in case (b), the new section is at end-of-file. In both cases, the section heading is exactly `## Sidekick Seeds Generated` and contains exactly one bullet beneath it.

---
*This document follows the https://specscore.md/scenario-specification*
