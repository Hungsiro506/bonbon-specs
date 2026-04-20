# Guide: Test-First Development with AI

> **Companion**: [`ac-to-scenario-derivation.md`](./ac-to-scenario-derivation.md) — the thinking guide for picking N scenarios per AC.
> **Companions in bonbon**: [`.claude/rules/e2e-scenarios.md`](https://github.com/Hungsiro506/bonbon/blob/main/.claude/rules/e2e-scenarios.md) (enforceable contract) · [`docs/testing/spec-synced-tests-plan.md`](https://github.com/Hungsiro506/bonbon/blob/main/docs/testing/spec-synced-tests-plan.md) (rollout plan).
> **Audience**: PMs writing acceptance criteria, engineers implementing tests, reviewers.

## The role split

| Role | Writes | Reads | Tools |
|------|--------|-------|-------|
| PM / product | Abstract ACs in the spec repo | Test run output, `SCENARIOS.md` | Markdown editor, spec repo web UI |
| Engineer | Concrete `scenario({...})` + fn body | Spec AC, flows, existing tests | `pnpm scenarios:ai`, `scenarios:sync`, IDE |
| AI (Claude) | First-pass scenario + fn body from an AC ID | Spec, flows, fixtures | `/test-from-spec` skill |
| Reviewer | Comments | Abstract block + concrete scenario in the test file | GitHub diff |

**Spec is abstract; test is concrete.** One AC (1) spawns many scenarios (N). The link is an ID. Prose is never copied.

## Golden path — adding a new test for a new AC

### Step 1 — PM adds an AC to the spec repo

In `bonbon-specs` (the public spec repo), open a PR adding a new acceptance scenario under the relevant User Story. Keep it abstract:

```markdown
### User Story 1 - Order Creation and Customer Notification (Priority: P1)

**Acceptance Scenarios**:

1. Given ..., When ..., Then ...
2. ...
3. Given the order is created, When the Garage Web dashboard is open, Then today's car/motorcycle count and revenue metrics update to reflect the new order.
```

The numbered position is the AC ID: the 3rd scenario under US1 is `US1.AC3`. **Do not renumber existing items** — append only. If you need to retire an AC, mark it `**DEPRECATED**` and leave the number in place.

PR template reminds contributors of the append-only rule. 1 reviewer approves; merge.

### Step 2 — Engineer pulls the new spec

```bash
# In bonbon
git submodule update --remote specs     # bump to latest spec SHA
git add specs && git commit -m "chore: bump specs to $(cd specs && git rev-parse --short HEAD)"
```

### Step 3 — Engineer generates a test scenario skeleton

Using the `test-from-spec` Claude skill:

```
/test-from-spec US1.AC3
```

Claude:
1. Reads `specs/004-order-lifecycle/spec.md` and extracts US1.AC3.
2. Picks an existing e2e test file matching the feature area (`e2e/bonbon-go/garage-dashboard-sync.integration.spec.ts`) or creates a new one.
3. Inserts a new `scenario({ id: "US1.AC3", given, when, then }, async () => { /* TODO */ })` block — with concrete placeholder values drawn from fixtures (plate, service ID, prices).
4. Drafts a first-pass fn body using `e2e/flows/` verbs.
5. Runs `pnpm scenarios:sync` so the abstract block is populated.

Example output:

```ts
// @spec-begin US1.AC3 — from specs/004-order-lifecycle/spec.md
// ABSTRACT: Given the order is created, When the Garage Web dashboard is
// open, Then today's car/motorcycle count and revenue metrics update to
// reflect the new order.
// @spec-end
scenario(
  {
    spec:  "004-order-lifecycle",
    id:    "US1.AC3",
    given: "an order with plate 99Z-DASH.10 and 1 x Basic Wash (150,000 VND)",
    when:  "employee progresses it to Done and records a cash payment",
    then:  "today.revenue increases by 150,000 and today.car_count by 1",
  },
  async () => {
    const before = await fetchOwnerDashboard();
    const order  = await placeOrder({ plate: "99Z-DASH.10" });
    await completeAndPay(order, "cash");
    const after  = await fetchOwnerDashboard();
    expect(after.today.revenue - before.today.revenue).toBeGreaterThanOrEqual(DEFAULT_SERVICE.price);
    expect(after.today.car_count - before.today.car_count).toBeGreaterThanOrEqual(1);
  },
);
```

### Step 4 — Engineer reviews + tweaks

AI-generated is a draft, never final. Engineer:

- Adjusts concrete values (plate, payment method) to avoid collisions with other scenarios.
- Validates the fn body logic — *especially* the assertions.
- Adds edge-case scenarios for the same AC (1:N). E.g. `US1.AC3 @ transfer-payment`, `US1.AC3 @ negative-amount`.

### Step 5 — Engineer verifies locally

```bash
pnpm scenarios:validate     # ids resolve, abstract blocks in sync
pnpm scenarios:report       # regenerates e2e/SCENARIOS.md
pnpm test:integration       # runs against real backend
```

### Step 6 — PR

Commit all three: the submodule bump, the test file, and the regenerated `e2e/SCENARIOS.md`. Reviewer sees:

- Submodule SHA — what the spec pinned to.
- Abstract block — what the AC says (machine-synced, cannot drift).
- Concrete scenario — what this test covers with real values.
- `e2e/SCENARIOS.md` — human-readable list of all scenarios in the repo.

## Multiple scenarios for one AC (1:N)

One AC often has multiple instances worth testing. They share `id` but differ in `given/when/then`:

```ts
scenario(
  {
    spec:  "004-order-lifecycle",
    id:    "US1.AC3",
    given: "an order with plate 99Z-DASH.20 and 1 x Basic Wash (150,000 VND)",
    when:  "employee completes and records a CASH payment",
    then:  "today.revenue increases by 150,000",
  },
  async () => { /* ... */ },
);

scenario(
  {
    spec:  "004-order-lifecycle",
    id:    "US1.AC3",
    given: "an order with plate 99Z-DASH.21 and 1 x Premium Wash (300,000 VND)",
    when:  "employee completes and records a TRANSFER payment",
    then:  "today.revenue increases by 300,000",
  },
  async () => { /* ... */ },
);
```

Both are valid — 1 AC, 2 concrete scenarios. The abstract block above each is identical (it's the same AC).

## Test cases MUST be concrete

PMs write abstractions. Engineers write **specific, detailed, concrete** tests:

- Use real fixture values: `SVC1_ID`, `SVC1_PRICE`, named plates (`99Z-DASH.20`).
- Quote exact amounts in prose: "150,000 VND", "today.revenue increases by exactly 150,000".
- Name the payment method, the status transition, the assertion target.

"An order with a service" is not a test case. "An order with plate 99Z-DASH.20 and 1 x Basic Wash priced at 150,000 VND, progressed to Done, paid by cash" is.

## The bypass escape hatch (discouraged)

Internal-only tests that don't map to a spec AC use the `INT-` prefix:

```ts
scenario(
  {
    spec:  "004-order-lifecycle", // or omit — INT- prefix makes spec optional
    id:    "INT-42",
    given: "an internal invariant: order IDs are unique UUIDs",
    when:  "two orders are created in parallel",
    then:  "no ID collision occurs",
  },
  async () => { /* ... */ },
);
```

Validator allows `INT-*` but emits a warning. Use sparingly — the whole point of this system is spec traceability.

## Commands cheat sheet

```bash
# Scaffold a scenario block from an AC (prints to stdout)
pnpm scenarios:ai -- --ac 004-order-lifecycle#US1.AC3

# Same, but also append to a file and auto-sync
pnpm scenarios:ai -- --ac 004-order-lifecycle#US1.AC3 --file e2e/bonbon-go/foo.integration.spec.ts

# Sync spec → test (rewrite abstract blocks)
pnpm scenarios:sync

# Check ids resolve + abstract blocks in sync
pnpm scenarios:validate

# Regenerate e2e/SCENARIOS.md
pnpm scenarios:report

# Use Claude to generate a scenario AND draft the fn body from an AC ID
# (via Claude Code — richer than scenarios:ai, which only scaffolds)
/test-from-spec 004-order-lifecycle#US1.AC3
```

## Troubleshooting

**"Validator says my AC doesn't exist"** — either the AC was renumbered in the spec (shouldn't happen; file a spec-repo issue) or you typo'd the ID. Check `specs/<spec>/spec.md`.

**"The abstract block keeps reverting after I edit it"** — that's by design. Edit the spec, not the block. The block is machine-owned.

**"My test is for a cross-cutting concern with no single AC"** — list multiple AC IDs in a `// also: US5.AC3` trailing comment, pick one as the primary `id`. Or use `INT-*` if truly not spec-linked.

**"Submodule commands confuse me"** — see bonbon's [`docs/infrastructure.md`](https://github.com/Hungsiro506/bonbon/blob/main/docs/infrastructure.md) or ask in the engineering channel.
