# Tasks: 010 — Car Owner Order History

Tasks are grouped by the phases defined in [plan.md](plan.md). Order within a stream is sequential; streams B/C may run in parallel after Stream A completes.

## Stream A: Backend — Data access

- [x] A01 **Repo**: Add `ListByCustomerPhone(ctx, phones []string, status *OrderStatus, from, to *time.Time, page, limit int)` and `GetByIDForPhones(ctx, id uuid.UUID, phones []string)` to `repo.OrderRepo` interface in `backend/internal/repo/contracts.go`.
- [x] A02 **Repo**: Implement both in `backend/internal/repo/persistent/order/repo.go`. Reuse `getServices` for the join. Query shape: `WHERE customer_phone IN (?) [AND status = ?] [AND received_at BETWEEN ? AND ?] ORDER BY received_at DESC LIMIT ? OFFSET ?`.
- [x] A03 **Mocks**: Extend `backend/internal/testutil/mocks.MockOrderRepo` to cover the two new methods.
- [x] A04 **Migration check**: Verify `orders(customer_phone)` is indexed. If not, add a new migration file `00000X_orders_customer_phone_index.up.sql` with the matching `.down.sql`. Confirm numbering against the existing `backend/migrations/` sequence before writing.
- [x] A05 **Service**: In `backend/internal/usecase/order/service.go:Create`, normalize `order.CustomerPhone` via `backend/pkg/phone.Normalize` when non-nil. Update `service_test.go` to assert normalization happens at create-time.
- [x] A06 **Unit tests**: Add repo-level tests under `backend/internal/testdata/integration/order_test.go` with fixtures that include both normalized (`0912…`) and international (`+84912…`) phones to confirm the dual-form predicate works.

## Stream B: Backend — Service + handlers

- [x] B01 **Contracts**: Add `ListForCarOwner(ctx, account *domain.CarOwnerAccount, status *OrderStatus, from, to *time.Time, page, limit int)` and `GetForCarOwner(ctx, account *domain.CarOwnerAccount, orderID uuid.UUID)` to `usecase.OrderService` in `backend/internal/usecase/contracts.go`.
- [x] B02 **Service**: Implement both in `backend/internal/usecase/order/service.go`. Canonicalize to `phones := []string{phone.Normalize(p), phone.ToInternational(p)}` before calling the repo. Collapse "no row" into `(nil, nil)` for the detail method and let the handler convert to 404.
- [x] B03 **Service tests**: Extend `backend/internal/usecase/order/service_test.go` to cover: happy path list, empty list, status filter, date filter, detail happy path, detail for foreign phone (expect nil), detail for unknown id (expect nil).
- [x] B04 **Handler**: Add `ListOrderHistory(c *fiber.Ctx)` and `GetOrderDetail(c *fiber.Ctx)` on `CarOwnerHandler` in `backend/internal/handler/http/v1/car_owner_handler.go`. Resolve `*domain.CarOwnerAccount` via `h.auth.GetCarOwnerProfile(ctx, middleware.GetAccountID(c))`. Parse `status`, `from`, `to`, `page`, `limit` from query. On detail, look up garage name (existing `garageRepo.GetByID` pattern) and strip the `token` field from the response body.
- [x] B05 **Routes**: In `backend/internal/handler/http/v1/router.go`, add `meGroup.Get("/orders", handlers.CarOwner.ListOrderHistory)` and `meGroup.Get("/orders/:id", handlers.CarOwner.GetOrderDetail)` after the existing `/me/vehicles/:id` line.
- [x] B06 **Wiring**: Ensure `CarOwnerHandler` receives the order service dependency (constructor signature update in `NewCarOwnerHandler`). Update `backend/cmd/server/main.go` (or wherever handlers are wired) accordingly.
- [x] B07 **Handler tests**: In `backend/internal/handler/http/v1/handler_test.go` add: happy path list, empty list, pagination, status filter, date filter, detail happy path, detail for foreign id (assert identical body/status to the unknown-id case — FR-007), detail for unknown id, role denial (employee JWT calling `/me/orders` → same shape as existing role-required denial), missing session (401).
- [x] B08 **Leak-shape test**: Dedicated test asserting the `/me/orders/:id` response for "not yours" and "not found" are byte-for-byte identical (status, body, headers).
- [x] B09 **Log fields**: Emit structured log field `deny_reason=phone_mismatch` vs. `deny_reason=not_found` on the service side. Add a test that captures logs (existing logger test helper) and asserts both branches produce their respective fields.
- [x] B10 **Regression**: Run the existing public `/status/{token}` test file — no modifications expected, must stay green.

## Stream C: Frontend — Car owner web

- [x] C01 **API client**: In `apps/web/src/lib/api.ts`, add `me.orders.list({ status?, from?, to?, page?, limit? })` and `me.orders.get(id)` helpers. Use the existing `api` wrapper to inherit auth + redirect behavior. Add mock handlers in `src/lib/mock-data/car-owner.ts` for `NEXT_PUBLIC_MOCK_MODE=true`.
- [x] C02 **Types**: Mirror the backend response shapes in a new `apps/web/src/lib/car-owner-orders.ts` (list entry and detail view). Detail type must not include `token`. Consider exporting from `@bonbon/shared-types` if reused elsewhere.
- [x] C03 **Nav**: In `apps/web/src/components/car-owner/BottomNav.tsx` (or equivalent), add an "Orders" entry (Vietnamese: "Đơn của tôi").
- [x] C04 **History page**: Create `apps/web/src/app/car-owner/(main)/orders/page.tsx`. Fetch first page on mount, paginate via "Load more" button. Card layout: garage name + plate + status badge + total + date + service summary. Empty state and loading state.
- [x] C05 **Detail page**: Create `apps/web/src/app/car-owner/(main)/orders/[id]/page.tsx`. Fetch via `me.orders.get(id)`. Reuse status-page components; extract shared pieces into `apps/web/src/components/car-owner/order/` if duplication appears. Redirect to `/car-owner/login?redirect=/car-owner/orders/{id}` on 401.
- [x] C06 **Upsell reuse**: On the detail page, if order status is modifiable, render the existing upsell-card component and wire it to `POST /orders/:id/modification` exactly as the car-owner `/status/[token]` page does. No new endpoint, no new auth path.
- [x] C07 **i18n**: Add keys for history title, empty state, filters, and detail labels to the `@bonbon/i18n` Vietnamese bundle.
- [x] C08 **Ban order-id inputs**: Audit `apps/web/src/app/car-owner/` for any route or form that would accept an order id from the user. Remove/guard any found. Confirm the token-entry form (`car-owner/(main)/status/`) is untouched (tokens are permitted, ids are not).

## Stream D: E2E and docs

- [x] D01 **Playwright — list (mock)**: `e2e/car-owner-web/orders.spec.ts` — mock-based, uses `e2e/fixtures/api-mock.ts` to intercept `/me/orders`, asserts at least one entry rendered and card contents.
- [x] D02 **Playwright — detail (mock)**: `e2e/car-owner-web/orders-detail.spec.ts` — taps an order from the list, asserts detail page renders via mocked `/me/orders/:id`, never via user-entered id.
- [x] D03 **Playwright — cross-account (integration)**: `e2e/car-owner-web/orders.integration.spec.ts` — logs in as seeded account A and hits `/car-owner/orders/{id}` for account B's order; asserts identical "not found" UX as for an unknown id.
- [x] D04 **Playwright — expired-link resilience (integration)**: verifies an owner can open an order older than 72 h from history; complements SC-006. Seeds or ages an order accordingly.
- [x] D05 **Regression**: Existing `/car-owner/status/[token]` Playwright specs — must stay green with no changes.
- [x] D06 **Docs**: Update `docs/` (or the closest API reference) with the two new endpoints, their query/path params, auth requirement, and response shape. Cross-link from this spec.
- [x] D07 **Checklist**: Tick items in `checklists/requirements.md` as each one is satisfied; note any items deferred or waived with a short justification.
