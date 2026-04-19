# Feature Specification: bonbon GO Mobile App

**Feature Branch**: `002-bonbon-go-app`
**Created**: 2026-02-26
**Status**: Draft
**Input**: User description: "Mobile app for garage employees to log in with provisioned accounts, receive cars by scanning license plates, manage order cards through a status workflow, process payments, and review daily order history."

## User Scenarios & Testing

### User Story 1 - Employee Login (Priority: P1)

A garage employee opens the bonbon GO app and logs in with the account provisioned by the garage owner (phone number + password). After login, they land on the main order board where they can start receiving cars.

**Why this priority**: Authentication is the entry point to all app functionality. Without login, no operations are possible.

**Independent Test**: Can be tested by using credentials created via the Garage Web "Register" feature, logging in, and verifying the main order board loads.

**Acceptance Scenarios**:

1. **Given** the login screen is displayed, **When** the employee enters their phone number and password and taps "Login", **Then** the app authenticates and navigates to the order board.
2. **Given** invalid credentials, **When** the employee attempts to log in, **Then** the app displays an error message without specifying which field is incorrect.
3. **Given** the account has been deactivated by the garage owner, **When** the employee tries to log in, **Then** the app rejects login with a message indicating the account is inactive.
4. **Given** the employee is already logged in, **When** they close and reopen the app, **Then** the session persists and they return to the order board without re-authenticating (until explicit logout or session expiry).

---

### User Story 2 - Receive Car and Create Order (Priority: P1)

The garage employee taps "+" to open the camera, scans the vehicle's license plate, and the system either finds existing vehicle info or prompts for new customer details. The employee selects services, and creates an order card that appears on the main board. A ZNS Zalo notification is sent to the car owner.

**Why this priority**: This is the core value proposition of the app -- converting a physical car arrival into a digital order with automatic customer notification. Every other feature depends on orders existing.

**Independent Test**: Can be tested by scanning a license plate (or manually entering one), selecting services, creating the order, and verifying the card appears on the board and ZNS is triggered.

**Acceptance Scenarios**:

1. **Given** the employee taps "+", **When** the camera opens and a license plate is in frame, **Then** the system uses OCR to recognize and display the detected plate number.
2. **Given** the plate is recognized, **When** a "Retake" button is visible next to the confirm button, **Then** the employee can re-scan if the recognition was incorrect.
3. **Given** the recognized plate, **When** the employee taps directly on the displayed plate number, **Then** they can manually edit/correct it.
4. **Given** a recognized plate that exists in the system, **When** confirmed, **Then** the app displays the customer's Name, Phone (if available), Long-Term Notes (if any), and a "Notes for this visit" input field.
5. **Given** a recognized plate that is new to the system, **When** confirmed, **Then** the app displays empty Name and Phone fields for the employee to fill in, plus Long-Term Notes and "Notes for this visit" fields, and a toggle for ZNS notification (default: ON).
6. **Given** ZNS toggle is ON but no phone number is provided, **When** the employee tries to proceed, **Then** the app shows an error requiring either a phone number or turning off the ZNS toggle.
7. **Given** customer info is confirmed, **When** the service selection screen appears, **Then** the screen displays all active services with a search bar at the top for filtering by name, and an option to add a temporary service (Name + Price only).
8. **Given** services are selected, **When** the employee taps "Receive Car", **Then** an order card is created on the board showing: License Plate, Status (Received), Total Bill, and 4 action buttons: "X" (Cancel), "V" (Start Washing), Bell (Done), "$" (Payment).
9. **Given** a new order with ZNS enabled, **When** the order is created, **Then** the BonBon OA sends a ZNS message to the customer's Zalo with a pre-configured template and a button linking to bonbon.com.vn to view status.
10. **Given** any action button is pressed on a card, **When** tapped, **Then** a confirmation popup appears before executing the action to prevent accidental taps.

---

### User Story 3 - Order Card Management and Status Flow (Priority: P1)

The employee manages active order cards on the board, updating status through the workflow: Received -> Washing -> Done -> Delivered. Cards can be reordered by drag-and-drop to prioritize wash order. Orders can also be edited or cancelled.

**Why this priority**: Order status progression is the operational heartbeat of the garage. Employees and car owners both depend on accurate, real-time status.

**Independent Test**: Can be tested by creating an order, progressing through each status, verifying ZNS is sent on "Done", editing services mid-order, and reordering cards.

**Acceptance Scenarios**:

1. **Given** an order card in "Received" status, **When** the employee taps "V" (Start Washing) and confirms, **Then** the status changes to "Washing".
2. **Given** an order card in "Washing" status, **When** the employee taps the Bell (Done) and confirms, **Then** the status changes to "Done" and a ZNS notification is sent to the customer (if phone number exists).
3. **Given** an order card in "Done" status, **When** the customer returns and the employee taps "$" (Payment), **Then** the payment flow is initiated (see User Story 4).
4. **Given** an order card in "Received" status, **When** the employee taps "X" (Cancel) and confirms, **Then** the status changes to "Cancelled" and the card moves to the completed section.
5. **Given** any active order card, **When** the employee taps on it, **Then** the card expands to show full details: Name, Phone, Long-Term Notes, Notes for This Visit, list of services with prices, and total bill.
6. **Given** an expanded order card, **When** the employee edits services or notes, **Then** the changes are saved and the total bill recalculates.
7. **Given** a car owner adds a service via bonbon.com.vn, **When** the update arrives, **Then** the bonbon GO app shows a notification on the affected order card prompting the employee to review and accept, and a ZNS update is sent to the customer confirming the addition.
8. **Given** multiple active order cards, **When** the employee presses and holds a card for 2 seconds, **Then** the card becomes draggable to reorder wash priority.

---

### User Story 4 - Payment Processing (Priority: P1)

After a service is complete and the customer returns, the employee records the payment method. For bank transfers, the garage's QR code is displayed.

**Why this priority**: Payment recording closes the order lifecycle and feeds the financial data into the Garage Web dashboard/transactions. Without it, orders cannot be completed.

**Independent Test**: Can be tested by completing an order (status: Done), tapping "$", selecting each payment method, and verifying the order moves to "Delivered" status.

**Acceptance Scenarios**:

1. **Given** an order in "Done" status, **When** the employee taps "$" (Payment), **Then** a payment method selection screen appears with options: Cash, Bank Transfer, Card, Other.
2. **Given** the payment method screen, **When** "Bank Transfer" is selected, **Then** the garage's QR code (uploaded in Garage Web profile) is displayed for the customer to scan.
3. **Given** a payment method is selected, **When** the employee confirms, **Then** the order status changes to "Delivered" and the card moves to the completed section.
4. **Given** a completed order, **When** the transaction is recorded, **Then** it appears in the Garage Web transaction history with the correct payment method.

---

### User Story 5 - Order Board Filtering and Search (Priority: P2)

The employee uses status filter chips, a search bar, and a total count indicator on the order board to quickly find and manage orders. The filter chips show the count of orders per status and allow filtering the list to a single status. The search bar filters orders by license plate. The total count is read-only.

**Why this priority**: In a busy garage with many concurrent orders, finding a specific car or seeing only orders at a certain stage saves time and reduces errors. Not blocking core operations but significantly improves daily workflow.

**Independent Test**: Can be tested by creating orders in various statuses via API, loading the board, verifying filter chips show correct counts, tapping a chip filters the list, typing a plate in the search bar narrows results, and the total count reflects the unfiltered order set.

**Acceptance Scenarios**:

1. **Given** the order board is loaded with orders in various statuses, **When** the employee views the board, **Then** a horizontally scrollable row of status filter chips is displayed below the search bar, each showing the status label and its order count (e.g. "Da nhan 3").
2. **Given** the filter chip row, **When** the employee taps a status chip (e.g. "Dang rua"), **Then** only orders with that status are displayed in the list, and the tapped chip is visually highlighted.
3. **Given** a status chip is active, **When** the employee taps "Tat ca", **Then** all orders are shown again without status filtering.
4. **Given** the order board, **When** the employee views the area below the top bar, **Then** a search bar with placeholder "Tim theo bien so..." is visible.
5. **Given** the search bar, **When** the employee types a partial plate number, **Then** the order list is filtered in real-time to show only orders whose license plate contains the search text (case-insensitive).
6. **Given** the search bar has text and a status filter is active, **When** viewing the list, **Then** both filters are applied simultaneously (intersection).
7. **Given** the order board is loaded, **When** the employee views the total count indicator, **Then** it displays the total number of orders in the unfiltered set (e.g. "Tong: 12 don"), and it is read-only (not tappable).
8. **Given** the filter or search yields zero results, **When** viewing the list, **Then** an empty state message is shown.

---

### User Story 6 - Daily Order History (Priority: P2)

The employee taps the history button (top-right corner) to view a summary of all orders they processed today, including revenue breakdown by payment method and vehicle type counts.

**Why this priority**: Provides end-of-day accountability for individual employees. Useful but not required for core operations.

**Independent Test**: Can be tested by processing several orders through completion, then viewing the history screen and verifying summary totals and detail rows match.

**Acceptance Scenarios**:

1. **Given** the main order board, **When** the employee taps the history button (top-right), **Then** the daily report screen opens showing today's data for this employee account.
2. **Given** the daily report, **When** viewing the summary section, **Then** it displays: Total Revenue with sub-lines (Cash Revenue, Transfer Revenue, Card Revenue, Other Revenue), Total Car Count, Total Motorcycle Count.
3. **Given** the daily report, **When** viewing the detail section below the summary, **Then** a list of all orders is shown with columns: Time, License Plate, Services, Status (Received/Washing/Done/Delivered/Cancelled), Amount, Payment Method.
4. **Given** no orders have been processed today, **When** viewing history, **Then** the screen shows zero totals and an empty list with a clear message.

---

### User Story 7 - Logout (Priority: P2)

The employee taps the logout button (top-left corner) to sign out of the app, clearing the session.

**Why this priority**: Basic session management needed for shared devices or employee shifts.

**Independent Test**: Can be tested by logging in, tapping logout, and verifying the session is cleared and the login screen appears.

**Acceptance Scenarios**:

1. **Given** the employee is logged in, **When** they tap the logout button (top-left), **Then** a confirmation dialog appears.
2. **Given** the confirmation dialog, **When** the employee confirms logout, **Then** the session is cleared and the app navigates to the login screen.
3. **Given** active orders exist on the board, **When** the employee logs out and another employee logs in, **Then** the active orders for the garage are still visible (orders belong to the garage, not the employee session).

---

### Edge Cases

- What happens when the camera cannot recognize a license plate (poor lighting, damaged plate)? The employee can manually type the plate number via a keyboard fallback accessible from the scan screen.
- What happens when the app loses network connectivity mid-order? The app should queue status changes locally and sync when connectivity resumes, showing an offline indicator.
- What happens when two employees try to create an order for the same license plate simultaneously? The system should prevent duplicate active orders for the same plate at the same garage, showing a warning that an active order already exists.
- What happens when the garage is emergency-closed via the Garage Web dashboard? bonbon GO should display a "Garage Closed" banner and disable the "+" button to prevent new orders, while allowing existing orders to be completed.
- What happens when a temporary service (custom name + price) is added? It should appear on the order but not be added to the garage's permanent catalog.
- What happens when an employee scans a motorcycle plate vs a car plate? The system should auto-detect vehicle type from the plate format (Vietnamese motorcycle plates have a different format than car plates) and categorize accordingly for reporting.

## Requirements

### Functional Requirements

- **FR-001**: System MUST allow employees to log in using phone number and password provisioned by the garage owner.
- **FR-002**: System MUST persist login sessions until explicit logout or session expiry.
- **FR-003**: System MUST open the device camera for license plate scanning when the employee taps "+".
- **FR-004**: System MUST perform OCR on the camera feed to detect and recognize Vietnamese license plates.
- **FR-005**: System MUST allow manual correction of recognized plates by tapping directly on the plate number.
- **FR-006**: System MUST provide a "Retake" button adjacent to the confirm button during plate scanning.
- **FR-007**: System MUST look up existing vehicle records by plate number and display customer info if found.
- **FR-008**: System MUST display all active services for the garage during service selection, with a search bar always visible at the top that filters services by name in real-time.
- **FR-009**: System MUST support adding temporary services with only Name and Price fields.
- **FR-010**: System MUST create order cards with visual status indicators and 4 action buttons (Cancel, Start Washing, Done, Payment).
- **FR-011**: System MUST enforce the status flow: Received -> Washing -> Done -> Delivered, with Cancel possible from Received status only.
- **FR-012**: System MUST show a confirmation popup before any status-changing action.
- **FR-013**: System MUST support drag-and-drop reordering of order cards (press-hold 2 seconds to initiate).
- **FR-014**: System MUST expand order cards on tap to show full details and allow editing of services and notes.
- **FR-015**: System MUST recalculate total bill when services are added or removed from an order.
- **FR-016**: System MUST display the garage's bank QR code when "Bank Transfer" payment method is selected.
- **FR-017**: System MUST support 4 payment methods: Cash, Bank Transfer, Card, Other.
- **FR-018**: System MUST display daily order history with revenue breakdown by payment method and vehicle type counts.
- **FR-019**: System MUST scope the daily history to the currently logged-in employee account.
- **FR-020**: System MUST provide a ZNS toggle per order (default: ON) with validation that a phone number is required when ON.
- **FR-021**: System MUST trigger ZNS notifications at order creation and at "Done" status transition.
- **FR-022**: System MUST receive and display notifications when a car owner adds services via bonbon.com.vn.
- **FR-023**: System MUST prevent creating duplicate active orders for the same license plate at the same garage.
- **FR-024**: System MUST auto-detect vehicle type (car vs motorcycle) from Vietnamese plate format for reporting categorization.
- **FR-025**: System MUST provide a service search bar on the service selection screen that filters the service list by name as the employee types, regardless of the number of services available.
- **FR-026**: System MUST display a horizontally scrollable row of status filter chips on the order board, each showing the status label and its order count.
- **FR-027**: System MUST filter the order list to a single status when a status chip is tapped, and show all orders when "Tat ca" (All) is tapped.
- **FR-028**: System MUST provide a search bar on the order board that filters orders by license plate in real-time (case-insensitive partial match).
- **FR-029**: System MUST display a read-only total order count indicator on the order board showing the total number of orders in the unfiltered set.

### Key Entities

- **Order**: The central entity. Key attributes: license plate, garage, status (Received/Washing/Done/Delivered/Cancelled), services list, total amount, payment method, notes (long-term, per-visit), ZNS enabled, creating employee account, timestamps per status change.
- **OrderService**: A line item within an order. Key attributes: reference to Product/Service (or temporary name+price), quantity, unit price, subtotal.
- **EmployeeSession**: Active login session. Key attributes: employee account reference, login time, device info, active status.

## Success Criteria

### Measurable Outcomes

- **SC-001**: License plate scanning achieves 90%+ recognition accuracy under normal lighting conditions, with results displayed within 2 seconds.
- **SC-002**: Employee can complete the full "receive car" flow (scan -> select services -> create order) in under 60 seconds.
- **SC-003**: ZNS notification is sent within 5 seconds of order creation or status change.
- **SC-004**: Order card status updates are reflected on-screen within 1 second of tapping the action button.
- **SC-005**: Daily history screen loads within 2 seconds showing accurate totals matching the sum of individual order amounts.
- **SC-006**: 95% of garage employees can operate the core receive-car flow without training beyond a 5-minute walkthrough.
- **SC-007**: Drag-and-drop card reordering initiates within 2 seconds of press-hold and feels responsive (60fps animations).
- **SC-008**: App supports offline operation for status changes, syncing within 10 seconds of connectivity restoration.
