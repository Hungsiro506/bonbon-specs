# Implementation Plan: 010 — Car Owner Order History

**Feature Branch**: `010-car-owner-order-history` | **Date**: 2026-04-18 | **Spec**: [spec.md](spec.md)

## Summary

Close the gap where a logged-in car owner has no way to view their past orders. Add two authenticated endpoints (`GET /me/orders`, `GET /me/orders/{id}`) on the Go backend, authorized by phone-number match against the JWT-bound car owner account. Surface those endpoints in the unified `apps/web/` app under `car-owner/(main)/orders/`, reachable from the car-owner bottom nav, and a detail page reached only by tapping a history row. The existing public `/status/{token}` flow is not touched.

The only two paths to an order detail after this feature:

1. **Public token URL** — unauthenticated capability, 72 h TTL, unchanged.
2. **Authenticated history → detail** — identity-authorized, no TTL, new.

No UI ever accepts an order id as user input.

## Technical Context

**Language/Version**: Go 1.x (backend), TypeScript 5.x + Next.js 14 App Router (unified `apps/web/`, car-owner routes under `src/app/car-owner/`).
**Primary Dependencies**: Fiber v2, GORM, existing JWT middleware (`backend/pkg/auth`), existing phone normalization package (`backend/pkg/phone`).
**Storage**: Existing Postgres schema — no new tables. Matching is done via `orders.customer_phone` ↔ `car_owner_accounts.phone`.
**Testing**: Go `testing` + testify for unit, existing integration harness in `backend/internal/testdata/integration`, Playwright under `e2e/car-owner-web/` for E2E (both mock `*.spec.ts` and integration `*.integration.spec.ts` flavors per the developer guide).
**Target Platform**: Same as existing — web responsive, Vietnamese locale.
**Project Type**: Backend feature + matching frontend page. No new packages, no new infrastructure.
**Performance Goals**: History endpoint p95 ≤ 300 ms for owners with ≤ 500 orders; list page LCP ≤ 2 s on 4G.
**Constraints**: Must not regress the public `/status/{token}` route or the employee-scoped `/orders` routes.
**Scale/Scope**: ≤ 500 orders per owner realistically; one new backend endpoint pair, one new frontend page pair, one new repo method, one new service method.

## Architectural Decisions

### AD-001 — Phone is the authorization primitive, not account id

The `orders` table has no `car_owner_account_id` column. Orders are created at check-in with `customer_phone` (nullable) and a per-garage `customer_id` pointer; the car owner account may not even exist yet. The cleanest authorization rule is therefore **"order is yours if `orders.customer_phone` normalizes to the same value as your account phone"** rather than an account-id join.

**Consequences**:
- Walk-ins without a phone (FR-013) are invisible to the authenticated flow — by design.
- If a car owner ever changes phone, ownership of prior orders does not follow automatically. This is called out in Edge Cases; out of scope here.
- We avoid a schema migration, which keeps this feature small and reversible.

### AD-002 — Phone normalization enforced on both sides of the query

Today `auth/service.go` normalizes `CarOwnerAccount.Phone` via `pkg/phone.Normalize` (drops spaces, converts `+84` to `0`). `order/service.go:Create` does **not** normalize `order.customer_phone` before persisting it. If a check-in stored `+84912…` but the owner registered as `0912…`, the two would not match.

Approach:
- The new repo query normalizes the *account* phone to the canonical `0…` form before comparing.
- We also generate the canonical form at rest by having `order/service.go:Create` normalize `CustomerPhone` when non-nil. This is a small correctness fix aligned with FR-011.
- Defensive: the repo query uses a predicate that matches either format (`customer_phone IN (normalized, international)`) so legacy rows still work even before a backfill.

A backfill migration for pre-existing unnormalized rows is listed as an optional follow-up (see Risks). It is **not** required for the feature to ship correctly — the dual-form predicate handles legacy data.

### AD-003 — Two new endpoints, mounted under existing `/me` group

`router.go` already has `meGroup := v1.Group("/me", authMiddleware, carOwnerRole)`. The new endpoints slot in naturally:

```
meGroup.Get("/orders",      handlers.CarOwner.ListOrderHistory)
meGroup.Get("/orders/:id",  handlers.CarOwner.GetOrderDetail)
```

This keeps the role check (`car_owner`), auth middleware, and URL shape consistent with the existing `/me/profile` and `/me/vehicles` endpoints. We do **not** reuse `ordersGroupEmployee` because it requires `employee`/`garage_owner` role and a garage id.

### AD-004 — Not-found and not-yours collapse to the same response

Per FR-007, the detail endpoint returns an identical 404 payload whether:

- the order id does not exist in the DB, or
- it exists but its `customer_phone` does not match the authenticated account.

Internal logs distinguish the two reasons (FR-016) so operators can debug; the client cannot tell them apart. This closes the cross-account existence oracle that the employee-scoped route has by design (where the 404 is used for cross-garage, not cross-account).

### AD-005 — Handler sits on `CarOwnerHandler`, not `OrderHandler`

The `OrderHandler` is coupled to `middleware.GetGarageID` and employee-scoped behavior. Putting the car-owner variants on `CarOwnerHandler` keeps responsibilities clean: `CarOwnerHandler` owns the `/me/*` surface and reads the car owner's identity from context; it depends on the order service but does not share code paths with the employee handler. This also avoids accidentally inheriting the garage-id 404 bug discussed in prior work.

### AD-006 — Repo gains a phone-scoped list, service gains two methods

New on `repo.OrderRepo`:

```go
ListByCustomerPhone(ctx, phones []string, status *OrderStatus, from, to *time.Time, page, limit int) ([]*domain.Order, int64, error)
GetByIDForPhones(ctx, id uuid.UUID, phones []string) (*domain.Order, error)  // nil, nil when not matching
```

The `phones` slice lets the service pass both normalized `0…` and `+84…` forms (see AD-002) in one query. The repo stays a thin Gorm wrapper; logic lives in the service.

New on `usecase.OrderService`:

```go
ListForCarOwner(ctx, account *domain.CarOwnerAccount, status *OrderStatus, from, to *time.Time, page, limit int) ([]*domain.Order, int64, error)
GetForCarOwner(ctx, account *domain.CarOwnerAccount, orderID uuid.UUID) (*domain.Order, error)
```

Both accept the authenticated `CarOwnerAccount` rather than a raw phone string, so phone canonicalization happens in exactly one place.

### AD-007 — Frontend: new route tree under `(main)/orders` and `(main)/orders/[id]`

`apps/web/src/app/car-owner/(main)/` already has the authenticated layout. Add:

- `car-owner/(main)/orders/page.tsx` — history list, server-component shell + client list
- `car-owner/(main)/orders/[id]/page.tsx` — detail page calling `/me/orders/:id`

The `car-owner/(main)/status/` existing page (and the public `car-owner/status/[token]/`) remain a "track an order by token" utility for users who know a token. The ID form is not re-added anywhere.

### AD-008 — Detail response shape

For consistency with the existing public `/status/{token}` response, the authenticated detail endpoint returns the **same payload shape**, minus the `token` field (never echo capability tokens on identity-authorized endpoints). Clients reuse the existing `StatusResponse` type with an optional `token`. The upsell/modification action reuses `POST /orders/:id/modification`, which already allows `car_owner` role.

## Project Structure

### Documentation

```text
specs/
  010-car-owner-order-history/
    spec.md
    plan.md                     # this file
    tasks.md
    checklists/requirements.md
```

### Source Code Touchpoints

```text
backend/
  internal/
    repo/
      contracts.go                              # + ListByCustomerPhone, GetByIDForPhones on OrderRepo
      persistent/order/repo.go                  # implement the two new methods
    usecase/
      contracts.go                              # + ListForCarOwner, GetForCarOwner on OrderService
      order/service.go                          # implement both; normalize CustomerPhone on Create
      order/service_test.go                     # unit tests
    handler/
      http/v1/car_owner_handler.go              # + ListOrderHistory, GetOrderDetail
      http/v1/router.go                         # wire routes under meGroup
      http/v1/handler_test.go                   # handler tests + role-denial tests
  pkg/
    phone/phone.go                              # (reuse, no changes expected)

apps/web/                                       # unified app: garage + car-owner
  src/
    app/car-owner/(main)/orders/page.tsx        # history list
    app/car-owner/(main)/orders/[id]/page.tsx   # detail page
    components/car-owner/BottomNav.tsx          # add "Orders" entry
    lib/api.ts                                  # + me.orders.list(), me.orders.get()
e2e/car-owner-web/                              # Playwright: list + detail specs
```

## Implementation Phases

### Phase 1 — Backend foundation (data access)

**Goal**: Query orders by phone correctly and safely.

1. Add `ListByCustomerPhone` and `GetByIDForPhones` to `repo.OrderRepo` interface.
2. Implement both in `backend/internal/repo/persistent/order/repo.go`. Use `customer_phone IN (?)` with a pre-built phone-variants slice. Reuse `getServices` for join.
3. Add indexed column check: verify `orders(customer_phone)` is indexed in migrations; if not, add a new migration. Confirm before adding to keep migration churn minimal.
4. Normalize `CustomerPhone` in `order/service.go:Create` using `pkg/phone.Normalize`. Add one unit test for the new normalization path.
5. Unit-test both repo methods via the existing integration harness (`backend/internal/testdata/integration`) with fixtures covering both normalized and `+84` rows.

**Exit criteria**: Go unit + integration tests green; `go vet` and linter clean.

### Phase 2 — Backend service + handlers

**Goal**: Two identity-authorized endpoints that deny cleanly.

1. Add `ListForCarOwner` and `GetForCarOwner` to `usecase.OrderService`. Inside, canonicalize to `[normalized, international]` slice, call the repo, collapse not-found and not-matching into the same result.
2. Implement `CarOwnerHandler.ListOrderHistory` and `GetOrderDetail`. Parse `status`, `from`, `to`, `page`, `limit` from query; parse `id` from path. Call auth service for the account (already cached in context? if not, load by `GetAccountID`). Return enriched list items (garage name is looked up via `garageRepo.GetByID`, same pattern as existing public `GetByToken`).
3. Wire routes in `router.go` under `meGroup`.
4. Add handler tests in `handler_test.go` including: happy path, foreign-order id (expect identical-to-not-found), unknown id, role denial (employee calling `/me/orders`), session-missing case.
5. Log a distinct internal reason code on phone-mismatch vs. not-found (structured log field, e.g., `deny_reason=phone_mismatch|not_found`).

**Exit criteria**: All existing tests remain green. New handler tests cover every acceptance scenario in spec User Stories 1, 2, 4.

### Phase 3 — Frontend list page

**Goal**: Logged-in owner lands on `/orders` and sees their history.

1. Extend `apps/web/src/lib/api.ts` with `me.orders.list({ status, from, to, page, limit })`.
2. Create `(main)/orders/page.tsx`. Use a client component that paginates via infinite scroll or an explicit "Load more" button (decision deferred to implementation — both meet SC-001).
3. Card layout per row: garage name, license plate, status badge, total, received-at, small service summary. Reuse `StatusBadge`-like component from 003.
4. Empty state matches edge-case 5: explain that orders appear after a visit.
5. Add an "Orders" entry to the main-group navigation.
6. Localize all strings via `@bonbon/i18n`, Vietnamese primary.

**Exit criteria**: `pnpm --filter @bonbon/web test` (or closest project-scoped command) + relevant Playwright specs green; manual smoke on a seeded account.

### Phase 4 — Frontend detail page

**Goal**: Tapping an order opens its detail authorized by the session.

1. Create `(main)/orders/[id]/page.tsx`. Fetch via the new `me.orders.get(id)` rather than `/status/{token}`.
2. Reuse the existing status-page components (timeline, service list, total, upsell cards) — extract shared parts into `components/` if needed; do not fork into a second copy.
3. If the order is still modifiable, render the upsell section wired to the existing `POST /orders/:id/modification`.
4. On 401/expired session, redirect to `/login?redirect=/orders/{id}`; on 404, show a neutral "order not available" state (do not say "this is not yours").

**Exit criteria**: Acceptance scenarios 1, 2, 4, 6 of User Story 2 pass end-to-end.

### Phase 5 — Guardrails and hardening

**Goal**: FR-007 leak resistance and cross-role denial confirmed.

1. Add a security-focused test file asserting: (a) 404-shape equivalence across not-found and not-yours; (b) employee JWT gets role-denied on `/me/orders`; (c) car-owner JWT gets role-denied on `/orders` employee routes (regression).
2. Add a log-assertion test that the distinct internal reason code is present.
3. Confirm no regression on `/status/{token}` — re-run existing public-token tests.

**Exit criteria**: `backend/` test suites + `e2e/car-owner-web/` Playwright specs green. Public token flow unchanged.

### Phase 6 — Docs and rollout

1. Update `docs/` (API reference section, if present) with the two new endpoints. Keep spec cross-references.
2. Update the main `specs/plan.md` table-of-contents if new-feature numbering is tracked there.
3. No feature flag needed — the endpoints are additive and gated by the existing `car_owner` role; exposure is controlled by landing page links.

## Dependencies and Execution Order

```text
Phase 1 (Backend data access)
  └─ Phase 2 (Backend service + handlers)
        ├─ Phase 3 (Frontend list)
        │     └─ Phase 4 (Frontend detail)
        └─ Phase 5 (Guardrails) ── can start after Phase 2
              │
              └─ Phase 6 (Docs) ── after all above
```

Phases 1 and 2 are strictly sequential. Phase 3 and Phase 5 can run in parallel once Phase 2 lands. Phase 4 depends on Phase 3 (shared layout + nav).

## Key Technical Decisions Recap

| ID | Decision | Alternative rejected |
|----|----------|----------------------|
| AD-001 | Authorize by phone match | Add `car_owner_account_id` FK on `orders` — rejected: needs migration + backfill for data we can already derive. |
| AD-002 | Normalize on both sides + dual-form predicate | Rely on at-rest normalization only — rejected: legacy rows would silently lock owners out. |
| AD-003 | Mount under `/me` | New top-level `/car-owner/orders` — rejected: duplicates role-scope already encoded in `/me`. |
| AD-004 | Collapse not-found and not-yours | Separate 403 for not-yours — rejected: leaks existence. |
| AD-005 | Put handlers on `CarOwnerHandler` | Add methods to `OrderHandler` — rejected: couples to employee-scoped helpers. |
| AD-007 | Keep `/status/:token` untouched | Unify token + identity flows — rejected: token is a bearer capability, identity isn't, mixing them surprises operators. |

## Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Legacy rows with unnormalized `customer_phone` | Owners cannot see their old orders | Medium | Dual-form predicate in the repo (AD-002); optional one-shot backfill migration if dual-form matches turn out to be slow. |
| Owners with very large histories (1000+ orders) slow the list | P95 latency miss | Low | Pagination (FR-003), existing `received_at` order clause, verify `orders(customer_phone)` index exists. |
| Adding a nav item named "Orders" collides with other apps' terminology | Minor UX confusion | Low | Use Vietnamese label from i18n ("Lịch sử đơn" or equivalent); align with copy used in status page. |
| `CarOwnerAccount` phone mutability in future | Silent loss of history when phone changes | Low | Documented as out-of-scope edge case in spec; note to revisit if profile-edit surfaces phone. |
| Handler accidentally surfaces public `token` in detail payload | Leaks capability to identity-authorized clients | Low | AD-008 strips `token`; add a test asserting the JSON has no `token` key. |

## Notes

- Phone normalization is the highest-risk correctness surface — invest test coverage there.
- The `/status/{token}` flow must stay byte-compatible; add a golden test if one does not already exist.
- No new infrastructure, no new secrets, no new feature flags. This is a pure code change on top of the current schema.
