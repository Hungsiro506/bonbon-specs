# 009 — ZNS Default OFF + Order QR Code

## User Story 1: ZNS Default OFF
As a garage owner, I want the ZNS notification toggle to default to OFF so that ZNS messages are not sent unless explicitly enabled, reducing messaging costs.

## User Story 2: QR Code for Order Status
As a garage employee, after creating an order, I want to see a QR code on screen that the car owner can scan to view their order status on the web.

## Acceptance Scenarios

### Scenario 1: ZNS toggle default
**Given** the scan plate screen is opened
**When** the employee views the customer info form
**Then** the ZNS toggle is OFF by default

### Scenario 2: QR code after order creation
**Given** the employee has selected services and confirms the order
**When** the order is successfully created
**Then** an OrderSuccess screen is displayed showing a QR code

### Scenario 3: QR code content
**Given** the OrderSuccess screen is displayed
**When** the car owner scans the QR code
**Then** it opens the order status page at {BONBON_WEB_URL}/status/{order.token}

### Scenario 4: Done navigation
**Given** the OrderSuccess screen is displayed
**When** the employee taps "Xong"
**Then** the app navigates to the order board

### Scenario 5: Order creation failure
**Given** order creation fails
**When** the error occurs
**Then** the error alert is shown and no navigation to OrderSuccess happens
