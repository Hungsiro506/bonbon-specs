# Contributing to bonbon-specs

This repo is the canonical source of product acceptance criteria for BonBon. Every AC here is potentially referenced by test code in [bonbon](https://github.com/Hungsiro506/bonbon), so stability matters.

## Who can contribute

Anyone. PRs are reviewed by product / engineering leads. **1 approval is required** before merge (GitHub branch protection enforces this).

## How to change a spec

### Adding a new feature spec

1. Pick the next unused 3-digit prefix (highest existing + 1).
2. Create `<nnn>-<slug>/spec.md` using the template below.
3. Open a PR. Describe in the body: the feature, who asked for it, why.

### Adding a new acceptance criterion

1. Open `<nnn>-<slug>/spec.md`.
2. Under the right User Story's `**Acceptance Scenarios**:` block, **append** a new numbered item — do not insert in the middle.
3. Under `## Requirements`, **append** the next unused `FR-xxx` ID if applicable.
4. Open a PR. Confirm the append-only rule in the PR checklist.

### Changing existing AC prose

1. Edit the text in place. Keep the numbering unchanged.
2. Note the change in the PR description so the engineering team can review downstream test impact.

### Deprecating an AC

1. Do **not** delete the numbered item.
2. Replace the prose with `**DEPRECATED (retained for ID stability)**` + a short reason.
3. Flag the engineering team — tests referencing the deprecated AC must be removed in bonbon.

## Why append-only

Tests in bonbon pin `(spec, id)` pairs. Renumbering silently changes what those tests are supposed to prove. Appending is safe; renumbering is a stealth regression.

## Feature spec template

```markdown
# Feature Specification: <Feature Name>

**Feature Branch**: `<nnn>-<slug>`
**Created**: YYYY-MM-DD
**Status**: Draft | Approved | Implemented
**Input**: User description: "..."

## User Scenarios & Testing

### User Story 1 - <short title> (Priority: P1)

<2-4 sentences describing the user flow.>

**Why this priority**: <one sentence>

**Independent Test**: <how this scenario can be tested in isolation>

**Acceptance Scenarios**:

1. **Given** <context>, **When** <action>, **Then** <observable outcome>.
2. **Given** ..., **When** ..., **Then** ...

### User Story 2 - ...

## Requirements *(mandatory)*

- **FR-001**: System MUST <behavior>.
- **FR-002**: System MUST <behavior>.

## Non-Functional

- Performance, SLAs, security, localization, etc.

## Out of Scope

- What this spec does *not* cover, to prevent scope creep.
```

## Review checklist (for reviewers)

- [ ] AC numbers are append-only (nothing renumbered, nothing deleted).
- [ ] User story title is clear; priority is set.
- [ ] Acceptance scenarios are observable (Given/When/Then, not implementation details).
- [ ] FR IDs are unique within the spec.
- [ ] No identifiers in Vietnamese — the spec uses English for role names, entity names, state labels (user-facing strings in bonbon itself go through i18n separately).
