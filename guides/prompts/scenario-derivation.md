# AI Prompt: Derive E2E scenarios from a spec AC

A portable prompt for generating concrete `scenario({...})` blocks from an abstract acceptance criterion. Paste into Claude, ChatGPT, Gemini, or any capable LLM. Fill the three `{{PLACEHOLDERS}}` at the bottom. Review the output — AI drafts; a human signs the commit.

> Prefer this over free-form "write me a test" prompts. It encodes the bonbon-specific contract (flows, fixtures, rules) so outputs are directly usable.
>
> **In Claude Code**: invoke the [`test-from-spec`](https://github.com/Hungsiro506/bonbon/blob/main/.claude/skills/test-from-spec/SKILL.md) skill in the bonbon repo instead — it has this same contract baked in and can also write to a file.

---

## Copy everything below this line

You are helping a BonBon engineer derive concrete end-to-end test scenarios from an abstract spec acceptance criterion. BonBon is a car-wash garage platform with three apps: Garage Web, bonbon GO (employee mobile), and Car Owner Web. E2E integration tests in the bonbon repo link to ACs in a separate `bonbon-specs` repo by a `(spec, id)` pair.

### The test shape (strict)

Every scenario is a single `scenario({ spec, id, given, when, then }, fn)` call, no bare `test(...)`:

```ts
scenario(
  {
    spec:  "<spec-folder-slug>",                          // e.g. "004-order-lifecycle"
    id:    "<stable-ac-id>",                              // e.g. "US1.AC3" or "FR-008"
    given: "concrete pre-conditions with plate, amount, state",
    when:  "a single action in active voice",
    then:  "a concrete, observable outcome with numbers",
  },
  async () => {
    // Implementation using flow verbs only. No raw fetch(). No loginAs*.
  },
);
```

The Playwright title is auto-generated from `${id} | ${then}` — so write `then` to be informative.

### Flow verbs available (use these, never raw HTTP)

From `e2e/flows/order-lifecycle.ts`:

- `placeOrder({ plate, customerName?, customerPhone?, services?, znsEnabled? })` → `PlacedOrder`
- `startWashing(order)` — Received → Washing
- `markDone(order)` — Received → Washing → Done
- `completeAndPay(order, "cash" | "transfer")` — Received → Washing → Done → Delivered
- `cancelOrder(order)` — Received → Cancelled
- `addService(order, { name, price, quantity, isTemporary? })` → `AddedService`
- `removeService(order, serviceId)` → void
- `fetchOrder(order)` → `OrderSnapshot` with `total_amount`, `status`, `services`
- `fetchOrderTransitions(order)` → `TransitionRecord[]` (`fromStatus`, `toStatus`, `hasTimestamp`, `hasEmployee`)
- `fetchOwnerDashboard()` → `DashboardMetrics` (`today | week | month`, each with `revenue` + `car_count`)
- `tryTransition(order, status)` → raw `Response` (for 4xx assertions)
- `tryAddService(order, partial?)` → raw `Response` (for 4xx assertions)
- `DEFAULT_SERVICE` — Basic Wash, 150,000 VND, catalog ID `SVC1_ID`

If a required action isn't in this list, **say so explicitly in your output** and propose the signature that should be added. Do not invent flows.

### Fixture conventions

- Plate format: `99Z-<4-LETTER-TAG>.NN`, where TAG is the feature area (e.g. `DASH` for dashboard, `SM` for state-machine, `EDIT` for mid-service edits, `RCAL` for recalc, `TRN` for transitions). Pick a NEW tag if none applies, explain it in one sentence.
- Unique plate per scenario. Never reuse a plate across scenarios.
- Amounts in VND, always quoted as an integer (e.g. `150000`), formatted in prose as `150,000 VND`.
- Payment methods: `"cash" | "transfer"` only.

### Rules (strict — the validator enforces these)

1. **Concrete over abstract.** `given/when/then` must pin down concrete values. "An active order" is wrong; "an order with plate 99Z-EDIT.10 and 1 × Basic Wash (150,000 VND) in Received status" is right.
2. **One primary `id` per scenario.** Secondary AC coverage goes in a `// also: <spec>#<id>` comment on the scenario.
3. **No raw HTTP in fn bodies.** No `fetch(`, no `Bearer `, no `loginAs*`, no `createOrder()`. Only flow verbs from the list above.
4. **Exact assertions where possible.** `expect(x).toBe(150000)` beats `toBeGreaterThan(0)`. Use `toBeGreaterThanOrEqual` only when concurrent state (today's dashboard aggregate) makes exact values fragile.
5. **1:N with a stop criterion.** Aim for 2–5 scenarios per AC. Stop when you can't name a realistic bug the next candidate would catch. More isn't better.
6. **Do not hand-edit the `@spec-begin/@spec-end` abstract block above a scenario.** It's machine-owned by `pnpm scenarios:sync`. Don't emit it — the sync script adds it after your code lands.
7. **Do not emit import statements.** Assume the consuming `.integration.spec.ts` already imports `test`, `expect`, `scenario`, and the needed flows.

### Derivation procedure

For the AC I give you, follow these steps and show your work:

1. **Enumerate hidden variables.** List the dimensions the abstract AC leaves open (state, service type, quantity, payment method, etc). Max ~6 bullets.
2. **Cross off the irrelevant ones.** For each dimension, say in ≤10 words whether it matters to this AC or is covered by another AC / FR.
3. **Pick scenarios.** For each meaningful combination, justify in one line: *"this scenario catches [specific bug class]"*. Stop at ≤5.
4. **Emit TypeScript.** Output a single `scenario(...)` block per chosen scenario, each compilable, with realistic plates and amounts.
5. **Flag gaps.** If a required flow is missing, or the AC is too broad (>5 genuine scenarios), say so instead of forcing output.

### Output format

First a short **Variables & picks** section (markdown, ≤ 20 lines) documenting steps 1–3. Then a **Scenarios** section with one code block containing the generated `scenario(...)` calls, ready to paste into a `test.describe(...)` block.

No preamble, no apologies, no "here's your code". Just the two sections.

---

## Fill in and paste to the AI

- **Spec folder slug**: `{{SPEC_SLUG}}`
  — *(e.g. `004-order-lifecycle`)*
- **AC ID(s)**: `{{AC_IDS}}`
  — *(one or more, e.g. `US4.AC1` or `US4.AC1, US4.AC2`)*
- **Abstract AC prose (paste from spec.md)**:
  ```
  {{AC_PROSE}}
  ```
- **Any constraints, known bugs, or edge cases to specifically cover** (optional):
  ```
  {{CONSTRAINTS}}
  ```

---

## After the AI returns

1. Paste the scenarios into the right `e2e/<app>/<feature>.integration.spec.ts` (inside a `test.describe(...)`).
2. Check AI didn't invent a flow. If it did, either add the flow to `e2e/flows/<aggregate>.ts` or rewrite the scenario.
3. `pnpm scenarios:sync` — injects the abstract blocks from spec.
4. `pnpm scenarios:validate` — ids resolve, blocks in sync.
5. `pnpm scenarios:report` — regenerate `e2e/SCENARIOS.md`.
6. `pnpm test:integration` (with backend running) — prove the scenarios actually pass.
7. Commit the test file + regenerated report.
