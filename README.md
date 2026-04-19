# bonbon-specs

Product specs for the [BonBon](https://github.com/Hungsiro506/bonbon) platform — one spec per feature, written as abstract user stories with stable acceptance-criterion IDs. Bonbon's engineering repo consumes this repo as a submodule; tests in bonbon link to ACs here by `(spec, id)` pairs and stay in sync via tooling.

## Audience

This repo is for:

- **Product managers / stakeholders** writing acceptance criteria.
- **QA** reviewing coverage against acceptance criteria.
- **Engineers** cross-referencing what a test is supposed to prove.

It is intentionally kept separate from the bonbon engineering repo so non-developers can contribute without touching code.

## Contents

```
001-garage-web/            garage portal (web)
002-bonbon-go-app/         employee mobile app
003-car-owner-web/         car-owner portal (web)
004-order-lifecycle/       cross-cutting: full order flow across all 3 apps
005-zns-notifications/     Zalo ZNS pipeline
006-api-gateway-domains/   api / domain boundaries
007-unified-web/           unified web app spec
007-zns-full-pipeline/     ZNS end-to-end pipeline spec
008-plate-crop/            license-plate recognition UI
009-order-qr-code/         order QR-code access
010-auth-hardening/        auth / session security
010-car-owner-order-history/ car-owner order history view
```

Top-level files (`backend-architecture.md`, `plan.md`, `golang-*`, …) are cross-cutting references, not feature specs.

## How specs are structured

Every feature spec follows the same template (`specs/<nnn>-<slug>/spec.md`):

```markdown
# Feature Specification: <Feature Name>

## User Scenarios & Testing

### User Story 1 - <title> (Priority: P1)

<narrative>

**Acceptance Scenarios**:

1. **Given** <context>, **When** <action>, **Then** <expected outcome>.
2. **Given** ..., **When** ..., **Then** ...
...

### User Story 2 - <title> (Priority: P2)
...

## Requirements *(mandatory)*

- **FR-001**: System MUST ...
- **FR-002**: System MUST ...
```

**AC IDs**:

- User stories are numbered 1, 2, 3, … ; acceptance scenarios under each are numbered 1, 2, 3, … The `N`th scenario under User Story `k` has ID `USk.ACN` (e.g. `US1.AC3`).
- Functional requirements have explicit IDs: `FR-001`, `FR-002`, … Unique within the spec.
- **AC IDs are scoped per spec.** `US1.AC3` in `004-order-lifecycle` is a different AC from `US1.AC3` in `001-garage-web`. The bonbon engineering repo references ACs as `(spec, id)` pairs.

## The **append-only** rule

**Never renumber or delete an existing AC.** Test code in bonbon pins specific `(spec, id)` pairs; renumbering silently changes what those tests are supposed to prove.

- To change what an AC says: edit the prose, keep the number.
- To remove an AC: mark it `**DEPRECATED (retained for ID stability)**` and leave the numbered item in place. Tests referencing it must be removed in bonbon in the same cycle.
- To add an AC: **append** at the end of the list. Do not insert in the middle.

This is the single hardest rule to enforce by review alone. Please flag it in every PR review.

## Contributing

Anyone can open a PR. See [CONTRIBUTING.md](./CONTRIBUTING.md) for the review process. Merge requires **at least 1 approval**. The PR template will prompt you to confirm the append-only rule.

## How bonbon consumes this repo

Bonbon mounts this repo as a submodule at `specs/`, pinned to a specific commit SHA. Engineers bump the SHA with:

```bash
# in bonbon
git submodule update --remote specs
git add specs && git commit -m "chore: bump specs to $(cd specs && git rev-parse --short HEAD)"
```

Tests in bonbon use `scenario({ spec: "<slug>", id: "<AC-ID>", given, when, then }, fn)` to link to ACs. Tooling (`pnpm scenarios:sync`, `pnpm scenarios:validate`) keeps the link honest.

For the full workflow see [bonbon/docs/testing/test-first-with-ai.md](https://github.com/Hungsiro506/bonbon/blob/main/docs/testing/test-first-with-ai.md).

## License

Internal use only.
