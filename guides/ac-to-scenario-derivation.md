# Guide: From Abstract AC to Concrete Test Scenarios

> **Audience**: engineers writing tests, PMs reviewing coverage, anyone bridging spec and implementation.
> **Companion to**: [`test-first-with-ai.md`](./test-first-with-ai.md) (the end-to-end workflow) and the enforceable contract in the bonbon repo: [`.claude/rules/e2e-scenarios.md`](https://github.com/Hungsiro506/bonbon/blob/main/.claude/rules/e2e-scenarios.md).
> **Goal**: answer the question *"I have a spec acceptance criterion. How do I turn it into the right number of good test scenarios?"*

---

## 1. The mental model

A spec AC and a test scenario are **two different things**, linked by an ID. They are not copies.

| Layer | What it is | Who owns it | Example |
|-------|-----------|-------------|---------|
| **Abstract AC** — in `bonbon-specs` | A contract: what *should* be true for a class of cases | PM / product | "Given an active order, when the employee adds a service, then the total is recalculated" |
| **Concrete scenario** — in `bonbon/e2e/**/*.spec.ts` | An observable proof: a specific case with real values | Engineer | "Given plate 99Z-EDIT.10 with 1 × Basic Wash (150,000 VND) in Received status, when the employee adds 1 × Interior Clean (150,000 VND), then total_amount becomes 300,000" |

The abstract AC is **a universal quantifier** ("for any active order, …"). A concrete scenario is **an instance** of that quantifier. One universal ⇒ potentially many instances. **1 : N.**

## 2. The operation model

Three roles, one contract:

- **PM** writes ACs in `bonbon-specs`. Never opens a `.ts` file. Never concretizes.
- **Engineer** bumps the submodule SHA, reads the AC, derives N concrete scenarios, implements each with `e2e/flows/` verbs, commits.
- **AI (Claude)** assists at two points:
  - `pnpm scenarios:ai` → emits a scaffold from a `(spec, id)` pair with TODO placeholders.
  - The `/test-from-spec` skill or the portable prompt in [`prompts/scenario-derivation.md`](./prompts/scenario-derivation.md) → drafts N concrete scenarios + fn bodies.

Human always has the last word. AI drafts; engineer reviews, concretizes, and signs the commit.

## 3. The process — from AC to a committed test

For **each** AC you want to cover:

1. **Read the abstract AC out loud.** If you can't paraphrase it in one sentence without the spec in front of you, the AC itself is too broad — flag it to the PM and split it.
2. **Enumerate the hidden variables.** Every stand-in noun ("an active order") and abstract verb ("adds a service") hides a dimension. Write them down.
3. **Cross off the dimensions that are irrelevant to this AC.** An AC about *totals* doesn't care about *customer phone formats*. Don't test a dimension just because it's there.
4. **Pick 2–5 meaningful combinations** that exercise different *code paths*, not just different *data*. Stop when you can't name a realistic bug the next combination would catch.
5. **Scaffold**: `pnpm scenarios:ai -- --ac <spec>#<id>` (fast, no AI) or `/test-from-spec <spec>#<id>` in Claude Code (richer, drafts fn body).
6. **Concretize** each scaffold: plate, amounts, payment method, assertions on exact numbers.
7. **Implement** the `async () => { ... }` body with 3–7 flow-verb calls. If you reach for `fetch()` or `loginAs*`, stop and add a flow first.
8. `pnpm scenarios:sync && pnpm scenarios:validate && pnpm scenarios:report` — commit the test + the regenerated `SCENARIOS.md`.

## 4. Principles — what makes a scenario good

### 4.1 Concrete over abstract, always

The whole point of a scenario is to pin down what the spec left open.

```ts
// BAD — abstract, just echoes the spec
given: "an active order with a service",

// GOOD — concrete, you know exactly what's set up
given: "an order with plate 99Z-EDIT.10 and 1 x Basic Wash (150,000 VND) in Received status",
```

If your `given/when/then` could describe ten different test runs, it's abstract. Make it describe exactly one.

### 4.2 Cover code paths, not states alone

"Try all three states" isn't a principle. "Try each distinct branch in the implementation" is.

- Adding a **catalog** service vs. a **temporary** service → two different branches (`product_id` path vs. null).
- Adding **before Washing** vs. **during Washing** → same code branch, probably, unless the state machine gates it.

Before writing a second test, ask: **what different thing runs if this input changes?** If nothing, skip it.

### 4.3 Stop when bug-naming fails

For every candidate scenario, answer: *"If I delete this test, what realistic bug could slip through?"*

If you can't name one, delete the candidate. 2 good tests beat 7 formulaic ones.

### 4.4 One primary AC per scenario

A scenario has exactly one `id`. If one test incidentally proves more than its primary AC (e.g. adding a service during Washing proves both US4.AC1 *and* touches FR-009's inverse), note it as a trailing comment:

```ts
scenario(
  { spec: "004-order-lifecycle", id: "US4.AC1", ... },
  // also: FR-009 (proves services CAN be added pre-Done)
  async () => { ... },
);
```

Primary IDs make coverage greppable. Secondary notes keep context.

### 4.5 Assert specifically

`toBe(150000)` beats `toBeGreaterThanOrEqual(0)`. Use loose matchers only when **concurrent state** (like today's dashboard aggregate) makes an exact number fragile.

```ts
// GOOD — exact
expect(updated.total_amount).toBe(DEFAULT_SERVICE.price + 150000);

// ACCEPTABLE — for concurrent aggregates
expect(after.today.revenue - before.today.revenue).toBeGreaterThanOrEqual(DEFAULT_SERVICE.price);

// BAD — useless
expect(updated.total_amount).toBeGreaterThan(0);
```

### 4.6 Name plates after the AC

A plate suffix like `99Z-EDIT.10`, `99Z-DASH.11`, `99Z-SM.15` makes it trivial to trace a CI failure back to an AC, and prevents two scenarios from colliding on the same license plate in a shared test DB.

Pattern: `99Z-<4-letter-ac-tag>.<NN>`. Pick a tag per file/feature area, keep it consistent.

### 4.7 Flows, not HTTP

If your test body has `fetch(`, `Bearer `, or `loginAs*`, you're plumbing. Move it into `e2e/flows/<aggregate>.ts` and call a verb. Rule of thumb: **a reviewer should understand the test by reading only the test body**, without opening any helper.

## 5. Worked example — deriving from `US4.AC1` and `US4.AC2`

Starting with these two ACs from `specs/004-order-lifecycle/spec.md`:

> **US4.AC1**: *Given an active order on bonbon GO, When the employee expands the card and adds a service, Then the order total is recalculated and the change is visible on bonbon.com.vn.*
>
> **US4.AC2**: *Given an active order, When the employee removes a service, Then the total is recalculated and the change is reflected on bonbon.com.vn.*

### Step 2 — variables hidden in "an active order" + "adds/removes a service"

| Variable | Values that exist |
|---|---|
| Starting status | Received, Washing, Done (FR-009 forbids after Done; "active" excludes Delivered/Cancelled) |
| Service type | Catalog (has `product_id`) vs. Temporary (`product_id = null`) |
| Quantity | 1 vs. > 1 |
| What's removed | An added service vs. the seed service vs. the last remaining service |
| Cross-app visibility | `bonbon GO` order-read vs. `bonbon.com.vn` car-owner status page |

### Step 3 — what's irrelevant to this AC

- **Exact service price** is irrelevant to US4.AC1/AC2 — FR-005 owns the math. We pick one price and one quantity for each test; we don't sweep the price axis here.
- **Cross-app visibility** is mentioned in both ACs but is really covered by FR-001 (cross-app sync ≤ 5s). Our flow layer doesn't expose the car-owner status-page read yet, so we cover this indirectly via `fetchOrder` and leave a note to add `fetchCarOwnerStatus` to flows when that API lands.

### Step 4 — chosen scenarios

**US4.AC1** — 3 scenarios:

1. **Catalog service, Received state, qty 1** — happy path, most common branch.
2. **Temporary service** — different code path (`product_id = null`).
3. **Add during Washing** — proves mid-service edit is allowed pre-Done (touches FR-009 inverse).

**US4.AC2** — 2 scenarios:

1. **Remove an added service, seed remains** — common case.
2. **Remove the seeded service, only temporaries remain** — catches "what if we remove the anchor?" bugs.

Five scenarios total across two ACs. If we added a 6th (e.g. "qty = 5") we couldn't name a new bug it would catch — stop.

### Step 5 — the resulting file

See [`test-first-with-ai.md`](./test-first-with-ai.md) for the multi-scenario shape, and the [rule doc in bonbon](https://github.com/Hungsiro506/bonbon/blob/main/.claude/rules/e2e-scenarios.md) for the required `scenario({...})` fields. A complete example for US4.AC1/AC2 lives at [`e2e/bonbon-go/order-mid-service-edit.integration.spec.ts`](https://github.com/Hungsiro506/bonbon/blob/main/e2e/bonbon-go/order-mid-service-edit.integration.spec.ts) in the bonbon repo (once the port PR lands).

## 6. When to split an AC instead of writing more scenarios

If an AC spawns **more than 5 genuine, non-redundant scenarios**, the AC is probably two ACs pretending to be one. File a PR on `bonbon-specs` to split it. Examples:

- "The dashboard updates for any completed order" → likely split into cash-payment, transfer-payment, and multi-line-item cases.
- "The system handles all payment methods" → one AC per method.

Splitting in the spec makes coverage auditable per AC — a single AC with 7 scenarios hides gaps.

## 7. Anti-patterns to avoid

- **Parrot tests**: `given` that echoes the spec verbatim. Pin values down.
- **Formulaic multiplication**: "let me try all 3 states × 2 service types × 2 quantities = 12 scenarios". 80% of them catch nothing.
- **Mixed primary IDs**: a single test claiming to prove both US4.AC1 and US4.AC2. Split it; use `// also:` for incidentals.
- **Loose assertions**: `toBeGreaterThan(0)` on something whose exact value is known.
- **Plate collisions**: two scenarios with the same plate — integration runs will clash in a shared DB.
- **Hand-editing the abstract block**: `@spec-begin/@spec-end` is machine-owned. Edit the spec instead.

## 8. Checklist (before you push)

- [ ] Each scenario has a single primary `id`.
- [ ] Every `given/when/then` names concrete values (plate, amount, status, payment method).
- [ ] Every plate is unique across the repo for that test file's prefix.
- [ ] Fn bodies call `e2e/flows/` verbs only — no raw `fetch`, no token plumbing.
- [ ] Assertions use exact values where possible.
- [ ] `pnpm scenarios:sync` writes 0 files on a re-run (idempotent).
- [ ] `pnpm scenarios:validate` is green.
- [ ] `pnpm scenarios:report` regenerated; `e2e/SCENARIOS.md` in the commit.
- [ ] For each AC you touched, you wrote the **minimum** number of scenarios to catch realistic bugs — not the maximum.
