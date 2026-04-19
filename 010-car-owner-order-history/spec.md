# Feature Specification: Car Owner Order History

**Feature Branch**: `010-car-owner-order-history`
**Created**: 2026-04-18
**Status**: Draft
**Input**: User description: "After login, a car owner can view their past orders. Users never input an order ID manually. There are exactly two ways to view an order's details: (1) the existing public one-time token link, and (2) a logged-in order-history flow that lists the owner's orders and lets them open a detail page the backend authorizes by account identity — not by garage and not by the order's random token."

## User Scenarios & Testing

### User Story 1 - View Order History After Login (Priority: P1)

A logged-in car owner opens their account on bonbon.com.vn and sees a list of every order ever created for their phone number, across every BonBon-connected garage, newest first. The list is the only entry point into order detail for authenticated browsing — there is no "enter order ID" form and no way to reach an order that is not on this list.

**Why this priority**: This is the primary reason a car owner logs in at all. Today the login provides no order access; the only way a car owner reaches an order after the ZNS link expires or is lost is to have the token, which defeats the purpose of having an account. This feature closes that gap and makes login meaningful.

**Independent Test**: Log in as a car owner whose phone number has orders at two different garages, navigate to the history page, and verify both garages' orders appear interleaved by date with the correct per-order summary (garage name, plate, status, total, date).

**Acceptance Scenarios**:

1. **Given** a car owner has completed orders at multiple garages under the same phone number, **When** they navigate to their history page, **Then** all orders across all garages are displayed in reverse chronological order.
2. **Given** the history list, **When** the owner scrolls, **Then** the page continues loading older orders until the full history is shown (paginated, no arbitrary cutoff).
3. **Given** the history list, **When** the owner taps an order row, **Then** they are taken to that order's detail page without ever typing or pasting an identifier.
4. **Given** the history list, **When** the owner filters by status (e.g., only "Delivered") or by date range, **Then** the list is narrowed without losing chronological ordering within the filter.
5. **Given** the car owner has never had any order, **When** they open the history page, **Then** an empty state is displayed explaining that orders appear here after their first visit to a BonBon-connected garage.
6. **Given** an unauthenticated visitor, **When** they attempt to open the history page, **Then** they are redirected to the OTP login flow and returned to the history page after successful login.

---

### User Story 2 - View Order Detail From History (Priority: P1)

From the history list, the car owner taps an order and sees the full order detail: garage info, services, total, status timeline, payment method (if paid), and any modification requests they made. Access is authorized by the logged-in account — not by the order's random public token and not by garage membership.

**Why this priority**: This is the pair to User Story 1. Without a detail view, the history list is only a teaser. The authenticated detail view also removes the 72-hour expiry of the public link for orders that belong to the logged-in user — their own orders never expire for them.

**Independent Test**: Log in, open the history list, tap an order, and verify the detail page renders without calling the public `/status/{token}` endpoint and without requiring the token to be known client-side.

**Acceptance Scenarios**:

1. **Given** a car owner is logged in and viewing their history list, **When** they tap an order row, **Then** the detail page renders the full order without requiring the public one-time token.
2. **Given** the order belongs to the authenticated car owner (order's customer phone matches the account's phone), **When** the detail endpoint is called, **Then** the backend returns the order data.
3. **Given** the car owner tries to open a detail page for an order whose customer phone does NOT match their account phone, **When** the request reaches the backend, **Then** it is denied in a way that does not reveal whether the order exists.
4. **Given** an order whose public link has expired (older than 72 hours), **When** the logged-in owner opens it from history, **Then** the detail page still works because expiry applies only to the public-token route.
5. **Given** the detail page is open, **When** the order is one the owner created via bonbon GO as a walk-in without a Zalo link, **Then** it is still reachable from history as long as the customer phone was captured at check-in.
6. **Given** the detail page, **When** the order is still active (Received or Washing), **Then** the "Add upsell service" action remains available exactly as on the public status page, reusing the existing modification flow.

---

### User Story 3 - Existing Public Token Flow Remains Untouched (Priority: P1)

The public one-time link (`/status/{token}`) keeps working exactly as today for unauthenticated access (SMS/ZNS recipients, QR scans). This feature does not change the public flow; it only adds an authenticated flow alongside it.

**Why this priority**: The public link is the primary acquisition surface — it arrives in ZNS when the car is received, before the owner has even thought about logging in. Breaking it would break the core notification loop. This story is defensive: "nothing regresses."

**Independent Test**: Open a fresh ZNS link (or simulated `/status/{token}` URL) without logging in and verify the page renders identically to before this feature.

**Acceptance Scenarios**:

1. **Given** a ZNS recipient opens a status link, **When** not logged in, **Then** the existing public status page renders with no login wall and no behavioral change.
2. **Given** a logged-in car owner opens a status link for one of their own orders, **When** the page loads, **Then** the public token flow is used (not the authenticated one) — this story does not merge the two flows.
3. **Given** a logged-in car owner opens a status link for someone else's order (e.g., a friend shared it), **When** the page loads, **Then** it still works because the public token is a bearer capability independent of who is logged in.
4. **Given** the public link has expired (past 72 hours), **When** anyone — logged in or not — opens it via the token URL, **Then** the existing 410 "link expired" response continues to be returned. (Logged-in owners reach their own expired orders through the history flow, not through this URL.)

---

### User Story 4 - Access Boundaries for Garage Staff (Priority: P2)

Garage employees and garage owners continue to use the existing authenticated order endpoints (scoped to their garage). The new car-owner order endpoints are not available to them, and vice versa. A car owner cannot list or open orders of other car owners even if they happen to know an order ID.

**Why this priority**: This is a security guardrail, not a new capability. It must be tested but does not add user-visible value on its own.

**Independent Test**: Attempt each cross-role scenario and verify the backend denies access with a response that does not leak whether the resource exists.

**Acceptance Scenarios**:

1. **Given** an employee is logged in, **When** they call the car-owner history endpoint, **Then** the backend denies the request based on role.
2. **Given** a car owner is logged in, **When** they call the employee-scoped order endpoints, **Then** the backend continues to deny them based on role (no regression).
3. **Given** two different car owners A and B with different phone numbers, **When** A requests the detail of one of B's orders by ID, **Then** the backend responds identically to "no such order" without leaking existence.
4. **Given** an order row exists in the database but has no customer phone captured (walk-in with `customer_phone` null), **When** any car owner queries it via the authenticated detail endpoint, **Then** access is denied — such orders can only be reached via the public token.

---

### Edge Cases

- What happens when an order row has a customer phone stored in a non-normalized format (e.g., raw `+84912345678` vs. `0912345678`) while the account phone is stored normalized? The system must match across both forms; otherwise a legitimate owner would be locked out of their own order.
- What happens when a car owner changes their phone number in the future? Orders are matched by the phone captured at check-in, not by account ID. If phone-number change is ever allowed, historical orders tied to the old phone must either be re-linked or remain accessible — this must be documented explicitly.
- What happens when the same phone number belongs to two CarOwnerAccounts? This should not be possible by the existing unique-phone constraint; the history endpoint MUST fail closed if it somehow encounters ambiguity rather than returning someone else's orders.
- What happens when a garage is deleted or disabled? Orders still appear in the owner's history with a clearly labeled garage-name fallback; the owner can read the record but the "Add upsell" action is disabled.
- What happens when an order is cancelled? It appears in history with a "Cancelled" badge and shows zero revenue; it is not hidden.
- What happens when the history list is very long (hundreds of orders over years)? Pagination must keep page load under the 2-second SLA established in 003-car-owner-web. No client-side-only filter may require loading the whole list first.
- What happens when an order's customer phone was captured at check-in before the owner ever created an account, then they log in later? FR-007 of 003-car-owner-web already states that first login auto-links historical records by phone, so those orders appear in the history with no additional action.
- What happens when the user was viewing their order history, loses their session (e.g., token expired), and then taps a detail link? They are redirected to OTP login and, on success, returned to the detail page for the same order.

## Requirements

### Functional Requirements

- **FR-001**: System MUST provide an authenticated history endpoint that returns every order whose captured customer phone matches the authenticated car owner's phone, across all garages, with no garage filter.
- **FR-002**: System MUST return history results in reverse chronological order by the order's received-at timestamp.
- **FR-003**: System MUST paginate history results with a stable page size and MUST support continuing beyond the first page until all orders are retrieved.
- **FR-004**: System MUST allow filtering history by status and by date range on the server side.
- **FR-005**: System MUST enrich each history list entry with: order id, status, license plate, total amount, payment method (if any), received-at, garage id, garage name, and a short service summary (count or first service name).
- **FR-006**: System MUST provide an authenticated detail endpoint that returns a single order by id ONLY when the order's customer phone matches the authenticated car owner's phone.
- **FR-007**: System MUST return the same response shape for "not found" and "not yours" so that existence is not leaked across car-owner accounts.
- **FR-008**: System MUST NOT apply the 72-hour public expiry on the authenticated detail endpoint; the account-owned route is identity-authorized, not capability-authorized.
- **FR-009**: System MUST preserve the existing public `/status/{token}` endpoint with no behavioral change, including the 72-hour expiry and the unauthenticated access pattern.
- **FR-010**: System MUST deny access to the new car-owner endpoints for non-car-owner roles (employee, garage owner) and MUST continue to deny car owners access to the existing garage-scoped order endpoints.
- **FR-011**: System MUST normalize phone numbers consistently on both sides of the match (account phone and order customer phone) so that the same human maps to the same orders regardless of entry format.
- **FR-012**: System MUST allow the car owner to trigger the existing upsell/modification flow from the authenticated detail view for orders that are still in a modifiable status, reusing the same backend modification endpoint with no new authorization path.
- **FR-013**: System MUST exclude from history any order whose customer phone is null (walk-in without phone capture); such orders are reachable only via the public token.
- **FR-014**: Frontend MUST NOT expose any UI for entering or pasting an order ID, and MUST NOT accept order IDs from query strings except as the direct result of navigation from the history list (or from the public token page, which uses the token URL).
- **FR-015**: System MUST return a 401/redirect response when the history or detail endpoint is called without a valid session, and the frontend MUST redirect to OTP login and return to the original destination after success.
- **FR-016**: System MUST log a distinct reason code when the detail endpoint denies access due to phone mismatch versus not-found, so that operators can debug without the client-visible response revealing it.

### Key Entities

- **OrderHistoryEntry**: A summarized order record returned by the history list. Key attributes: order id, status, license plate, total amount, payment method (optional), received-at, garage id, garage name, service summary. Does NOT include the public token or long-form fields.
- **OrderDetailView (Authenticated)**: The full order as viewed by the authenticated owner. Key attributes: same fields as the existing public detail payload, but without the 72-hour expiry check and without the public token appearing in the response body.
- **CarOwnerIdentity**: Derived from the JWT at request time. Key attributes: account id, normalized phone. This is the authorization primitive for the two new endpoints.

## Success Criteria

### Measurable Outcomes

- **SC-001**: 95% of logged-in car owners with a phone number that has at least one order see that order in the history list within 2 seconds of landing on the history page, on a 4G connection.
- **SC-002**: Zero cross-account order data leaks across car-owner accounts in automated tests, including the "known id for another owner" case (FR-007).
- **SC-003**: Public token flow page load time and response shape are identical before and after this feature (regression check).
- **SC-004**: Car owners reach order detail from history in at most 2 taps (open history → tap order).
- **SC-005**: The new endpoints reject non-car-owner roles with the same shape as the existing role-required middleware (no regression, no new error codes to learn).
- **SC-006**: Logged-in car owners can open orders older than 72 hours from history (direct verification that the public-expiry cutoff does not apply to the authenticated flow).
- **SC-007**: The frontend ships with no "enter order id" control and no route that accepts an order id from a source other than the history list.
