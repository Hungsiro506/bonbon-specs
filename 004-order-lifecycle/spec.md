# Feature Specification: Cross-Cutting -- Order Lifecycle

**Feature Branch**: `004-order-lifecycle`
**Created**: 2026-02-26
**Status**: Draft
**Input**: User description: "Documents the full order lifecycle that spans all 3 BonBon applications: bonbon GO (mobile), Garage Web (garage.bonbon.com.vn), and Car Owner Web (bonbon.com.vn). Covers order creation, status transitions, service modifications, payment, and data flow between apps."

## User Scenarios & Testing

### User Story 1 - Order Creation and Customer Notification (Priority: P1)

A customer drives their car into a garage. The garage employee uses bonbon GO to scan the license plate and create an order. The system automatically sends a ZNS Zalo notification to the car owner with a link to track the service on bonbon.com.vn. The order simultaneously appears on the Garage Web dashboard and transaction ledger.

**Why this priority**: This is the foundational flow that activates all three applications. Without order creation and notification, the platform has no value.

**Independent Test**: Can be tested end-to-end by: (1) creating an order on bonbon GO, (2) verifying ZNS is delivered, (3) opening the ZNS link and seeing the status on bonbon.com.vn, (4) checking the order appears on Garage Web dashboard.

**Acceptance Scenarios**:

1. **Given** an employee on bonbon GO scans a license plate and selects services, **When** they tap "Receive Car", **Then** the order is created with status "Received" and is visible on all three apps within 5 seconds.
2. **Given** the order is created with ZNS enabled, **When** the order creation succeeds, **Then** a ZNS Zalo message is sent to the car owner's registered phone number containing a personalized link to bonbon.com.vn/status/{order-token}.
3. **Given** the order is created, **When** the Garage Web dashboard is open, **Then** today's car/motorcycle count and revenue metrics update to reflect the new order.
4. **Given** ZNS is disabled for this order (toggle off or no phone number), **When** the order is created, **Then** no notification is sent but the order is still fully tracked across Garage Web and bonbon GO.

---

### User Story 2 - Status Progression Flow (Priority: P1)

The employee progresses the order through its lifecycle: Received -> Washing -> Done -> Delivered. Each transition is reflected in real time on the car owner's status page and recorded in the Garage Web transaction history.

**Why this priority**: Status flow is the operational core that keeps all three apps synchronized. Real-time updates are the key differentiator from a manual process.

**Independent Test**: Can be tested by progressing an order through each status on bonbon GO while simultaneously monitoring bonbon.com.vn status page and Garage Web transactions for real-time updates.

**Acceptance Scenarios**:

1. **Given** an order in "Received" status on bonbon GO, **When** the employee taps "Start Washing" and confirms, **Then** the status changes to "Washing" on bonbon GO, on bonbon.com.vn (within 5 seconds), and in Garage Web transaction records.
2. **Given** an order in "Washing" status, **When** the employee taps "Done" and confirms, **Then** the status changes to "Done" across all apps, and a ZNS notification is sent to the car owner (if phone number exists and ZNS is enabled).
3. **Given** an order in "Done" status, **When** the employee completes payment and selects a method, **Then** the status changes to "Delivered", payment method is recorded, and the transaction appears as completed in Garage Web.
4. **Given** an order in "Received" status, **When** the employee taps "Cancel" and confirms, **Then** the status changes to "Cancelled" across all apps, the car owner's status page shows "Cancelled", and the transaction is recorded with zero revenue.
5. **Given** any status transition, **When** it occurs, **Then** a timestamp is recorded for that transition to enable time-tracking analytics.

---

### User Story 3 - Mid-Service Modification by Car Owner (Priority: P1)

While the car is being serviced, the car owner views the status page on bonbon.com.vn and decides to add an additional service (upsell). This triggers a notification to the bonbon GO app. The employee reviews and accepts, updating the order. A confirmation ZNS is sent back to the car owner.

**Why this priority**: This cross-app interaction is a key revenue driver for garages and a unique value proposition of the platform -- allowing customers to add services remotely.

**Independent Test**: Can be tested by: (1) creating an order, (2) on bonbon.com.vn selecting an upsell service, (3) verifying notification appears on bonbon GO, (4) accepting on bonbon GO, (5) verifying ZNS update sent, (6) verifying updated total on all apps.

**Acceptance Scenarios**:

1. **Given** a car owner is viewing their active order status on bonbon.com.vn, **When** they tap "Add this service" on a recommended service, **Then** a service addition request is sent to the bonbon GO app.
2. **Given** the bonbon GO app receives a service addition request, **When** a notification appears on the relevant order card, **Then** the employee can review the requested service and its price.
3. **Given** the employee reviews the request, **When** they accept it, **Then** the service is added to the order, the total bill is recalculated, and the car owner receives a ZNS message confirming the addition with the updated service list and total.
4. **Given** the employee reviews the request, **When** they decline it, **Then** the car owner's status page updates to show the service was not added, with no ZNS sent for declined requests.
5. **Given** the order total has changed due to added services, **When** viewing the Garage Web transaction records, **Then** the updated total is reflected.

---

### User Story 4 - Mid-Service Modification by Employee (Priority: P2)

The employee on bonbon GO edits an active order by adding or removing services or updating notes. Changes are reflected on the car owner's status page.

**Why this priority**: Operational flexibility for employees to adjust orders based on discoveries during service (e.g., finding additional issues while washing). Less complex than the cross-app upsell flow.

**Independent Test**: Can be tested by modifying services on an active order via bonbon GO and verifying changes appear on bonbon.com.vn and Garage Web.

**Acceptance Scenarios**:

1. **Given** an active order on bonbon GO, **When** the employee expands the card and adds a service, **Then** the order total is recalculated and the change is visible on bonbon.com.vn.
2. **Given** an active order, **When** the employee removes a service, **Then** the total is recalculated and the change is reflected on bonbon.com.vn.
3. **Given** an active order, **When** the employee updates notes (long-term or per-visit), **Then** the notes are saved and visible in Garage Web customer records.
4. **Given** an employee modification, **When** the change results in a different total, **Then** a ZNS update is sent to the car owner with the revised service list and total.

---

### User Story 5 - Payment Completion and Financial Recording (Priority: P1)

The employee records the payment method on bonbon GO, closing the order. The transaction is finalized with accurate financial data visible on the Garage Web dashboard and transaction history.

**Why this priority**: Payment closes the order loop and is essential for financial tracking -- the primary reason garage owners use the Garage Web portal.

**Independent Test**: Can be tested by completing payment on bonbon GO and verifying the transaction appears in Garage Web with correct amounts and payment method, and dashboard metrics update.

**Acceptance Scenarios**:

1. **Given** an order in "Done" status on bonbon GO, **When** the employee taps "$" and selects a payment method, **Then** the order moves to "Delivered" status and the payment method is recorded.
2. **Given** the payment is recorded, **When** the Garage Web transaction history is viewed, **Then** the completed order appears with: Time, License Plate, Owner Name, Phone, Services Used, Amount, Status (Delivered), Payment Method, Notes, Employee Account.
3. **Given** the payment is recorded, **When** the Garage Web dashboard is viewed, **Then** today's revenue, weekly revenue, and monthly revenue totals are updated to include this transaction.
4. **Given** the payment is recorded, **When** the bonbon GO daily history is viewed, **Then** the order appears with the correct payment method and amount.
5. **Given** the payment was via "Bank Transfer", **When** the "$" button was pressed, **Then** the garage's QR code was displayed on bonbon GO for the customer to scan before recording the payment.

---

### Edge Cases

- What happens when multiple employees at the same garage are viewing the order board simultaneously? All boards must stay synchronized in real time -- status changes by one employee are immediately visible to others.
- What happens when the car owner adds a service on bonbon.com.vn but the employee has already completed the service (status: Done)? The system should reject the addition and show the car owner a message that service is already complete.
- What happens when network connectivity drops during a status transition? The transition should be queued locally and replayed when connectivity resumes. Other apps should see the last known status until sync completes.
- What happens when a car owner opens the ZNS link days after the order was completed? The page should show the final order summary with a "Completed" badge, not an expired/error state.
- What happens if the garage emergency-closes while orders are active? Active orders continue through their lifecycle (existing orders can be completed), but no new orders can be created. The car owner's status page shows the garage as temporarily closed but their order is unaffected.
- What happens when a car owner's phone number changes between visits? The old phone number retains its history. The new phone number starts fresh. Manual migration is not supported in this phase.

## Requirements

### Functional Requirements

- **FR-001**: System MUST synchronize order state across bonbon GO, Garage Web, and Car Owner Web within 5 seconds of any state change.
- **FR-002**: System MUST enforce the order status state machine: Received -> Washing -> Done -> Delivered, with Cancel available only from Received.
- **FR-003**: System MUST record timestamps for every status transition for analytics and audit purposes.
- **FR-004**: System MUST support bidirectional service modifications: car owner adding via bonbon.com.vn (requires employee approval) and employee adding via bonbon GO (immediate).
- **FR-005**: System MUST recalculate and propagate order totals across all apps whenever services are added or removed.
- **FR-006**: System MUST trigger ZNS notifications at every order state transition: order creation, Received->Washing, Washing->Done, Done->Delivered, Received->Cancelled, service modification confirmation (upsell accepted), and employee-initiated service change (when total changes). See 005-zns-notifications/spec.md for the full trigger event matrix.
- **FR-007**: System MUST record the payment method (Cash, Transfer, Card, Other) and associate it with the completed order for financial reporting.
- **FR-008**: System MUST update Garage Web dashboard metrics (daily/weekly/monthly revenue, vehicle counts) in near-real-time as orders are completed.
- **FR-009**: System MUST prevent service additions on orders that are in Done, Delivered, or Cancelled status.
- **FR-010**: System MUST handle ZNS link access for completed orders gracefully by showing a final order summary.
- **FR-011**: System MUST support offline status transitions on bonbon GO with automatic sync upon connectivity restoration.

### Key Entities

- **Order**: Central entity tying all apps together. Lifecycle: Created (on bonbon GO) -> Status transitions -> Payment -> Finalized. Attributes: unique order token, garage ID, vehicle (license plate), customer (phone), employee account, service list, total amount, payment method, status, ZNS enabled flag, per-status timestamps, notes (long-term, per-visit).
- **OrderStatusTransition**: Audit log entry. Attributes: order reference, from-status, to-status, timestamp, employee account, trigger source (employee/system).
- **ServiceModificationRequest**: Cross-app request. Attributes: order reference, requested service, source (car_owner or employee), status (pending/accepted/declined), requested at, resolved at.

## Success Criteria

### Measurable Outcomes

- **SC-001**: Order state changes propagate to all three apps within 5 seconds, 99% of the time.
- **SC-002**: The complete order lifecycle (creation to payment) can be performed in under 5 minutes for a single-service wash.
- **SC-003**: Cross-app upsell flow (car owner request -> employee confirmation -> ZNS update) completes within 30 seconds when the employee is actively using the app.
- **SC-004**: Garage Web dashboard revenue totals match the sum of individual transaction amounts with 100% accuracy.
- **SC-005**: Zero data loss during network interruptions -- all queued transitions are successfully replayed upon reconnection.
- **SC-006**: System supports 50 concurrent active orders per garage without performance degradation.
