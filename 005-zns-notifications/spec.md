# Feature Specification: Cross-Cutting -- ZNS Zalo Notification System

**Feature Branch**: `005-zns-notifications`
**Created**: 2026-02-26
**Updated**: 2026-03-01
**Status**: Draft
**Input**: User description: "Documents the Zalo ZNS (Zalo Notification Service) integration that connects all BonBon apps. ZNS is used to send transactional notifications from the BonBon Official Account (OA) to car owners' Zalo, providing service status updates and linking them to bonbon.com.vn."

## Zalo ZNS/ZBS API Integration

### Overview

As of January 1, 2026, Zalo consolidated ZNS and UID-based messaging into **ZBS Template Message**. BonBon uses ZBS Template Message **via phone number** (not UID) to send transactional notifications to car owners. This is the correct method because car owners have not followed or interacted with the BonBon OA before their first order -- we only have their phone number from the garage employee.

**Single OA, multi-garage architecture**: One BonBon platform OA sends all ZNS messages on behalf of every garage. The garage name is a template parameter inside the message body (e.g., "BonBon -- {garage_name}"), not a separate OA per garage. This means one set of credentials, one token lifecycle, one quota pool, and one webhook endpoint for the entire platform.

All ZNS messages are **Tag 1 (IN_TRANSACTION)** -- they confirm/update transaction information (order creation, status changes, service modifications). This tag permits order notifications and is compliant with Zalo's anti-spam policies.

### Authentication & Token Management

Zalo OA API uses OAuth 2.0 with short-lived access tokens.

```
Token Lifecycle:
  Authorization Code (from Zalo OA dashboard) ---> Access Token (1 hour TTL)
  Access Token expired ---> Refresh Token (3 months TTL, single-use) ---> New Access Token + New Refresh Token
  Refresh Token expired ---> Manual re-authorization via Zalo OA dashboard
```

**Token endpoint**: `POST https://oauth.zaloapp.com/v4/oa/access_token`

```
Request (application/x-www-form-urlencoded):
  grant_type=authorization_code (initial) | refresh_token (renewal)
  app_id={ZALO_APP_ID}
  code={authorization_code}         -- only for initial grant
  refresh_token={refresh_token}     -- only for renewal

Headers:
  secret_key: {ZALO_SECRET_KEY}

Response:
  {
    "access_token": "...",
    "refresh_token": "...",
    "expires_in": 3600
  }
```

**Token storage**: Access token and refresh token are stored in the `zns_oa_tokens` table (encrypted at rest). A background job refreshes the access token 5 minutes before expiry. If the refresh token is within 7 days of expiry, the system logs a critical alert for manual re-authorization.

### Send ZNS API

**Endpoint**: `POST https://openapi.zalo.me/v2.0/oa/message`

```
Headers:
  Content-Type: application/json
  access_token: {access_token}

Request Body:
  {
    "phone": "84901234567",
    "template_id": "12345",
    "template_data": {
      "garage_name": "BonBon Car Wash Q7",
      "vehicle_plate": "51F-123.45",
      "service_list": "Rua xe co ban, Hut bui noi that",
      "total": "150.000d",
      "order_link": "https://bonbon.com.vn/status/abc123token"
    },
    "tracking_id": "bonbon_order_uuid_event_type"
  }

Response (success):
  {
    "error": 0,
    "message": "Success",
    "data": {
      "msg_id": "zns_msg_id_from_zalo"
    }
  }

Response (failure):
  {
    "error": -124,
    "message": "Access token is invalid or has expired"
  }
```

**Phone number format**: Zalo requires the international format without the `+` prefix. The system normalizes Vietnamese phone `0901234567` to `84901234567` before calling the API.

### Delivery Status Webhook

Zalo sends delivery status callbacks to a configured webhook URL when a ZNS message is received or seen by the user.

**Webhook endpoint** (BonBon side): `POST /api/v1/webhooks/zalo/zns`

```
Callback payload from Zalo:
  {
    "event_name": "user_received_message",
    "msg_id": "zns_msg_id_from_zalo",
    "delivery_time": "1709312400000",
    "tracking_id": "bonbon_order_uuid_event_type"
  }

  delivery statuses: "received", "seen", "failed"
```

The webhook updates the `zns_notifications.delivery_status` from `sent` to `delivered` (on "received"/"seen") or marks delivery details on failure.

### Template Registration

Templates must be submitted for approval via the Zalo OA admin portal before use. Approval typically takes 1-3 business days. Templates have a 400-character limit and support text with placeholders. CTA buttons (links) are supported.

### Quota & Rate Limits

- **Starting daily quota: 500 messages/day** (lowest Zalo tier, sufficient for ~125 orders/day across all garages at ~4 ZNS per order)
- Zalo quota tiers: 500 -> 2,000 -> 5,000 -> 10,000 -> 20,000 -> 50,000 -> 100,000 -> 500,000 messages/day
- Upgrading is an administrative process via Zalo OA portal (no code changes). On our side, only the `ZALO_ZNS_DAILY_QUOTA` env var needs updating.
- Rate limit: Zalo does not publish a hard per-second rate limit, but the worker processes messages sequentially with a 100ms minimum gap to avoid throttling
- Quota resets daily at 00:00 ICT (UTC+7)
- The system tracks usage in Redis (`zns:quota:{oa_id}:{date}`) and warns at 80% threshold (400 messages at the 500 tier)

### Configuration (Environment Variables)

```
ZALO_APP_ID=...
ZALO_SECRET_KEY=...
ZALO_OA_ID=...
ZALO_INITIAL_AUTH_CODE=...        # one-time, from OA dashboard
ZALO_ZNS_DAILY_QUOTA=500
ZALO_WEBHOOK_VERIFY_TOKEN=...     # shared secret to validate webhook origin
BONBON_BASE_URL=https://bonbon.com.vn
```

---

## User Scenarios & Testing

### User Story 1 - Order Created Notification (Priority: P1)

When a garage employee creates an order on bonbon GO for a car owner's vehicle, the BonBon Official Account (OA) on Zalo sends a ZNS message to the car owner's Zalo containing order details and a link to track service status on bonbon.com.vn.

**Why this priority**: This is the primary mechanism for connecting car owners to the platform. Without this notification, car owners have no awareness of bonbon.com.vn and no entry point to track their service.

**Independent Test**: Can be tested by creating an order on bonbon GO with a valid phone number and ZNS enabled, then verifying the ZNS message appears in the car owner's Zalo chat with the BonBon OA.

**Acceptance Scenarios**:

1. **Given** an employee creates an order on bonbon GO with ZNS toggle ON and a valid phone number, **When** the order is successfully created, **Then** a ZNS message is sent from the BonBon OA to the car owner's Zalo within 5 seconds.
2. **Given** the ZNS message is received, **When** the car owner views it, **Then** it contains: garage name, vehicle license plate, list of selected services, and a button/link opening bonbon.com.vn/status/{order-token}.
3. **Given** the ZNS message contains a link, **When** the car owner taps it, **Then** they are taken to the service status page for their specific order.
4. **Given** the ZNS toggle is OFF for an order, **When** the order is created, **Then** no ZNS message is sent and the order proceeds normally without notification.
5. **Given** the ZNS toggle is ON but no phone number is provided, **When** the employee attempts to create the order, **Then** bonbon GO blocks order creation and prompts the employee to either enter a phone number or turn off the ZNS toggle.

---

### User Story 2 - Status Change Notifications (Priority: P1)

When the employee transitions an order through the lifecycle (Received -> Washing -> Done -> Delivered), a ZNS message is sent to the car owner at each state change. Each message reflects the current status and includes a link to the live status page.

**Why this priority**: Status-driven notifications are the core value loop. The car owner stays informed without calling the garage. "Done" (pickup) notification directly reduces car dwell time. Washing and Delivered notifications complete the transparency story.

**Independent Test**: Can be tested by progressing an order through each status on bonbon GO and verifying the ZNS message is received on the car owner's Zalo at each transition.

**Acceptance Scenarios**:

1. **Given** an order transitions from "Received" to "Washing", **When** the employee confirms the transition on bonbon GO, **Then** a ZNS message is sent to the car owner within 5 seconds with template `status_washing` containing: garage name, vehicle plate, current services, and status page link.
2. **Given** an order transitions from "Washing" to "Done", **When** the employee confirms the transition, **Then** a ZNS message is sent with template `status_done` containing: garage name, vehicle plate, a "Your car is ready for pickup" message, and status page link.
3. **Given** an order transitions from "Done" to "Delivered", **When** the employee records payment, **Then** a ZNS message is sent with template `status_delivered` containing: garage name, vehicle plate, total amount, payment method, and a "Thank you" message with status page link.
4. **Given** an order is cancelled (Received -> Cancelled), **When** the employee confirms cancellation, **Then** a ZNS message is sent with template `status_cancelled` containing: garage name, vehicle plate, cancellation notice, and status page link.
5. **Given** the order was created without a phone number (ZNS was OFF), **When** any status change occurs, **Then** no ZNS is sent and no error occurs.
6. **Given** multiple status changes occur rapidly (e.g., Received -> Washing within seconds of creation), **When** each transition fires, **Then** each ZNS is sent independently -- the system does not batch or skip intermediate notifications.

---

### User Story 3 - Service Modification Confirmation Notification (Priority: P2)

When a car owner requests an additional service via bonbon.com.vn and the employee accepts it on bonbon GO, a ZNS message is sent confirming the updated service list and revised total.

**Why this priority**: Provides closed-loop communication for the upsell feature. Without this, car owners have no confirmation that their service addition was accepted.

**Independent Test**: Can be tested by requesting a service on bonbon.com.vn, accepting it on bonbon GO, and verifying the ZNS confirmation is received.

**Acceptance Scenarios**:

1. **Given** a car owner adds a service on bonbon.com.vn and the employee accepts on bonbon GO, **When** the acceptance is confirmed, **Then** a ZNS message is sent to the car owner within 5 seconds.
2. **Given** the modification ZNS, **When** the car owner views it, **Then** it contains: garage name, license plate, updated list of all services, new total amount, and a link to bonbon.com.vn/status/{order-token}.
3. **Given** the employee declines the service addition, **When** the decline is processed, **Then** no ZNS is sent for the declined request.

---

### User Story 4 - Employee-Initiated Service Update Notification (Priority: P2)

When an employee modifies an active order on bonbon GO (adds/removes services resulting in a different total), a ZNS message notifies the car owner of the change.

**Why this priority**: Keeps the car owner informed of any cost changes initiated by the garage, building trust and transparency.

**Independent Test**: Can be tested by modifying services on an active order via bonbon GO and verifying the car owner receives a ZNS update.

**Acceptance Scenarios**:

1. **Given** an employee adds or removes services on an active order on bonbon GO, **When** the modification results in a changed total, **Then** a ZNS message is sent to the car owner within 5 seconds (if phone number exists).
2. **Given** the update ZNS, **When** the car owner views it, **Then** it contains: garage name, license plate, updated service list, new total, and a link to the status page.
3. **Given** the order has no associated phone number, **When** services are modified, **Then** no ZNS is sent and no error occurs.

---

### Edge Cases

- What happens when the Zalo ZNS API is temporarily unavailable? The system should queue failed messages and retry with exponential backoff (max 3 retries over 15 minutes). If all retries fail, log the failure but do not block order operations.
- What happens when a phone number is not registered on Zalo? The ZNS delivery will fail silently (Zalo API returns an error). The system should log this and mark the notification as "undeliverable" without affecting the order.
- What happens when ZNS daily quota is exceeded? Zalo enforces quotas per OA (starting at 500/day). The system tracks quota usage and logs warnings at 80% threshold (400). When exceeded, notifications are queued with status "queued" until the next quota reset at 00:00 ICT. Upgrading the quota tier is an admin process via Zalo OA portal -- only the `ZALO_ZNS_DAILY_QUOTA` env var changes on our side.
- What happens when the car owner blocks the BonBon OA on Zalo? ZNS delivery fails. The system logs this and marks future notifications for this phone number as "blocked" to avoid repeated failed attempts.
- What happens when multiple status changes occur rapidly (e.g., Received -> Washing within seconds)? Each transition that triggers a ZNS should be sent independently. The system should not batch or skip intermediate notifications.
- What happens when the car owner's Zalo account uses a different phone number than the one on file? ZNS is delivered to the phone number stored in the order. If that number's Zalo account differs from the car owner's primary Zalo, they may not receive the message. This is documented as a known limitation.
- What happens when the Zalo OA access token expires during a send attempt? The worker detects the 401/invalid token response, triggers an immediate token refresh using the stored refresh token, and retries the send. This counts as one retry attempt.
- What happens when the refresh token expires (every 3 months)? The system cannot auto-renew. A critical alert is raised 7 days before expiry. An admin must re-authorize via the Zalo OA dashboard to obtain a new authorization code. During the gap, all ZNS messages are queued with status "token_expired" and sent once a new token is configured.

## End-to-End ZNS Flow

### Flow 1: Employee Creates Order -> Car Owner Receives ZNS

```
Employee (bonbon GO)              BonBon API                    Background Worker              Zalo API
       |                              |                              |                           |
       |-- POST /orders              |                              |                           |
       |   { licensePlate,            |                              |                           |
       |     customerPhone,           |                              |                           |
       |     services[],              |                              |                           |
       |     znsEnabled: true }       |                              |                           |
       |                              |                              |                           |
       |                         [1] Create order                    |                           |
       |                         [2] Normalize phone                 |                           |
       |                             0901234567 -> 84901234567       |                           |
       |                         [3] Insert zns_notifications        |                           |
       |                             trigger: order_created          |                           |
       |                             status: queued                  |                           |
       |                         [4] Publish SSE event               |                           |
       |                              |                              |                           |
       |<-- 201 { order }             |                              |                           |
       |                              |                              |                           |
       |                              |                         [5] Poll queued notifications    |
       |                              |                             (every 2 seconds)            |
       |                              |                              |                           |
       |                              |                         [6] Check quota (Redis)          |
       |                              |                         [7] Load template params         |
       |                              |                         [8] Build template_data:         |
       |                              |                             garage_name, plate,          |
       |                              |                             service_list, link           |
       |                              |                              |                           |
       |                              |                         [9] POST /v2.0/oa/message ------>|
       |                              |                             { phone, template_id,        |
       |                              |                               template_data,             |
       |                              |                               tracking_id }              |
       |                              |                              |                           |
       |                              |                         [10] error: 0 (success) <--------|
       |                              |                              |                           |
       |                              |                         [11] Update notification:        |
       |                              |                              status: sent                |
       |                              |                              msg_id: {zalo_msg_id}       |
       |                              |                         [12] Increment quota counter     |
       |                              |                              |                           |
       |                              |                              |  (async, later)           |
       |                              |  POST /webhooks/zalo/zns <---|---------------------------|
       |                              |  { event: "received",        |                           |
       |                              |    msg_id, tracking_id }     |                           |
       |                              |                              |                           |
       |                              |  [13] Update notification:   |                           |
       |                              |       status: delivered      |                           |
```

### Flow 2: Status Change -> ZNS at Each Transition

```
Employee (bonbon GO)              BonBon API                    Background Worker              Zalo API
       |                              |                              |                           |
       |-- PATCH /orders/:id/status   |                              |                           |
       |   { status: "washing" }      |                              |                           |
       |                              |                              |                           |
       |                         [1] Validate transition             |                           |
       |                             (received -> washing OK)        |                           |
       |                         [2] Update order status             |                           |
       |                         [3] Record transition log           |                           |
       |                         [4] Check: order.zns_enabled        |                           |
       |                             AND order.customer_phone        |                           |
       |                         [5] Insert zns_notifications        |                           |
       |                             trigger: status_washing         |                           |
       |                             status: queued                  |                           |
       |                         [6] Publish SSE event               |                           |
       |                              |                              |                           |
       |<-- 200 { order }             |                              |                           |
       |                              |                         [7] Poll -> find queued          |
       |                              |                         [8] Send ZNS (same as Flow 1)    |
       |                              |                              |-------------------------->|
       |                              |                              |<--------------------------|
       |                              |                         [9] Update status: sent          |

  ... same pattern repeats for washing->done, done->delivered, received->cancelled ...
```

### Flow 3: Retry on Failure

```
Background Worker                                    Zalo API
       |                                                |
       |-- POST /v2.0/oa/message ---------------------->|
       |                                                |
       |<-- error: -124 (invalid token) ----------------|
       |                                                |
       |  [1] Refresh access token                      |
       |      POST oauth.zaloapp.com/v4/oa/access_token |
       |  [2] Store new tokens                          |
       |  [3] Retry send (attempt 2/3)                  |
       |-- POST /v2.0/oa/message ---------------------->|
       |                                                |
       |<-- error: 0 (success) -------------------------|
       |                                                |
       |  [4] Update: status=sent, attempt_count=2      |


Retry Schedule (exponential backoff):
  Attempt 1: immediate
  Attempt 2: +30 seconds
  Attempt 3: +2 minutes
  Attempt 4 (final): +15 minutes
  After attempt 4: status = "failed", log error, no more retries
```

## ZNS Trigger Event Matrix

| Order Event | Trigger | Template | Condition |
|---|---|---|---|
| Order created | `POST /orders` | `order_created` | `zns_enabled=true AND customer_phone IS NOT NULL` |
| Received -> Washing | `PATCH /orders/:id/status {washing}` | `status_washing` | `zns_enabled=true AND customer_phone IS NOT NULL` |
| Washing -> Done | `PATCH /orders/:id/status {done}` | `status_done` | `zns_enabled=true AND customer_phone IS NOT NULL` |
| Done -> Delivered | `PATCH /orders/:id/payment` | `status_delivered` | `zns_enabled=true AND customer_phone IS NOT NULL` |
| Received -> Cancelled | `PATCH /orders/:id/status {cancelled}` | `status_cancelled` | `zns_enabled=true AND customer_phone IS NOT NULL` |
| Upsell accepted by employee | `PATCH /orders/:id/modification/:mid {accept}` | `modification_confirmed` | `zns_enabled=true AND customer_phone IS NOT NULL` |
| Employee modifies services | `PUT /orders/:id/services` (total changed) | `service_updated` | `zns_enabled=true AND customer_phone IS NOT NULL` |

## ZNS Message Templates

Seven templates are required (must be pre-approved by Zalo as Tag 1 -- IN_TRANSACTION):

### 1. `order_created`

```
BonBon -- {garage_name}
Xe {vehicle_plate} da duoc tiep nhan.
Dich vu: {service_list}
Theo doi trang thai tai day:
[Xem trang thai] -> {order_link}
```

Parameters: `garage_name`, `vehicle_plate`, `service_list`, `order_link`

### 2. `status_washing`

```
BonBon -- {garage_name}
Xe {vehicle_plate} dang duoc rua.
Dich vu: {service_list}
Theo doi tai day:
[Xem trang thai] -> {order_link}
```

Parameters: `garage_name`, `vehicle_plate`, `service_list`, `order_link`

### 3. `status_done`

```
BonBon -- {garage_name}
Xe {vehicle_plate} da hoan thanh!
Vui long den nhan xe.
[Xem chi tiet] -> {order_link}
```

Parameters: `garage_name`, `vehicle_plate`, `order_link`

### 4. `status_delivered`

```
BonBon -- {garage_name}
Xe {vehicle_plate} da giao.
Tong: {total}
Thanh toan: {payment_method}
Cam on quy khach!
[Xem hoa don] -> {order_link}
```

Parameters: `garage_name`, `vehicle_plate`, `total`, `payment_method`, `order_link`

### 5. `status_cancelled`

```
BonBon -- {garage_name}
Don hang cho xe {vehicle_plate} da bi huy.
[Xem chi tiet] -> {order_link}
```

Parameters: `garage_name`, `vehicle_plate`, `order_link`

### 6. `modification_confirmed`

```
BonBon -- {garage_name}
Dich vu cho xe {vehicle_plate} da duoc cap nhat.
Dich vu moi: {service_list}
Tong moi: {total}
[Xem chi tiet] -> {order_link}
```

Parameters: `garage_name`, `vehicle_plate`, `service_list`, `total`, `order_link`

### 7. `service_updated`

```
BonBon -- {garage_name}
Dich vu cho xe {vehicle_plate} da thay doi.
Dich vu hien tai: {service_list}
Tong: {total}
[Xem chi tiet] -> {order_link}
```

Parameters: `garage_name`, `vehicle_plate`, `service_list`, `total`, `order_link`

## Requirements

### Functional Requirements

- **FR-001**: System MUST integrate with the Zalo ZBS Template Message API (`POST https://openapi.zalo.me/v2.0/oa/message`) to send transactional notifications via phone number from the BonBon Official Account.
- **FR-002**: System MUST send ZNS notifications at these trigger points: (a) Order Created, (b) Received->Washing, (c) Washing->Done, (d) Done->Delivered, (e) Received->Cancelled, (f) Upsell Modification Confirmed, (g) Employee-Initiated Service Change (when total changes).
- **FR-003**: System MUST include in every ZNS message: garage name, vehicle license plate, and a CTA button/link to `bonbon.com.vn/status/{order-token}`.
- **FR-004**: System MUST use pre-approved ZNS message templates (Tag 1 -- IN_TRANSACTION) registered with Zalo. Seven templates are required.
- **FR-005**: System MUST respect the per-order ZNS toggle -- when OFF, no ZNS is sent for any trigger point on that order.
- **FR-006**: System MUST validate that a phone number exists before attempting ZNS delivery, and block order creation if ZNS is ON but phone is missing.
- **FR-007**: System MUST implement retry logic for failed ZNS deliveries: exponential backoff (immediate, +30s, +2min, +15min), max 4 attempts.
- **FR-008**: System MUST log all ZNS delivery attempts with status (queued, sent, delivered, failed, undeliverable, blocked, token_expired) for auditing.
- **FR-009**: System MUST track Zalo OA quota usage in Redis and log warnings at 80% threshold.
- **FR-010**: System MUST NOT block order operations if ZNS delivery fails -- orders proceed independently of notification success.
- **FR-011**: System MUST generate unique, non-guessable order tokens for status page deep links to prevent unauthorized access to order details.
- **FR-012**: System MUST manage Zalo OA access tokens automatically -- refresh 5 minutes before expiry using the stored refresh token.
- **FR-013**: System MUST raise a critical alert when the refresh token is within 7 days of expiry, requiring manual re-authorization.
- **FR-014**: System MUST normalize Vietnamese phone numbers from `0XXXXXXXXX` to `84XXXXXXXXX` before calling the Zalo API.
- **FR-015**: System MUST expose a webhook endpoint (`POST /api/v1/webhooks/zalo/zns`) to receive delivery status callbacks from Zalo and update notification records.
- **FR-016**: System MUST validate webhook origin using a shared verify token to prevent spoofed callbacks.
- **FR-017**: System MUST use `tracking_id` (format: `{order_id}_{trigger_event}`) in all ZNS API calls to correlate Zalo callbacks with internal notification records.

### Key Entities

- **ZNSNotification**: A notification record. Key attributes: order reference, trigger event (order_created, status_washing, status_done, status_delivered, status_cancelled, modification_confirmed, service_updated), recipient phone number (normalized to 84 format), message template ID, template parameters (garage name, plate, services, total, payment method, link), Zalo msg_id (from response), tracking_id, delivery status (queued, sent, delivered, failed, undeliverable, blocked, token_expired), attempt count, last attempt timestamp, next retry at, error details.
- **ZNSTemplate**: Pre-registered Zalo template. Key attributes: template ID (from Zalo), internal name, trigger event type, parameter schema, tag type (always Tag 1), approval status, last approved date.
- **ZNSOAToken**: Zalo OA OAuth tokens. Key attributes: OA ID, access token (encrypted), refresh token (encrypted), access token expires at, refresh token expires at, last refreshed at.
- **ZNSQuotaTracker**: Quota monitoring (Redis-backed). Key attributes: OA ID, daily quota limit, current usage count, last reset timestamp, warning threshold reached flag.

### ZNS Trigger Event Enum

```
order_created
status_washing
status_done
status_delivered
status_cancelled
modification_confirmed
service_updated
```

The original 4 events are expanded to 7 to cover every state transition in the order lifecycle.

## Database Additions

### zns_oa_tokens table

```sql
CREATE TABLE zns_oa_tokens (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    oa_id                   VARCHAR(100) NOT NULL UNIQUE,
    access_token_encrypted  TEXT NOT NULL,
    refresh_token_encrypted TEXT NOT NULL,
    access_token_expires_at TIMESTAMPTZ NOT NULL,
    refresh_token_expires_at TIMESTAMPTZ NOT NULL,
    last_refreshed_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Updated zns_notifications table

```sql
CREATE TABLE zns_notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    trigger_event   VARCHAR(30) NOT NULL,
    recipient_phone VARCHAR(15) NOT NULL,
    template_id     VARCHAR(100) NOT NULL,
    params          JSONB NOT NULL DEFAULT '{}',
    tracking_id     VARCHAR(200) NOT NULL,
    zalo_msg_id     VARCHAR(200),
    delivery_status VARCHAR(20) NOT NULL DEFAULT 'queued',
    attempt_count   INT NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    next_retry_at   TIMESTAMPTZ,
    error_detail    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_trigger CHECK (trigger_event IN (
        'order_created','status_washing','status_done',
        'status_delivered','status_cancelled',
        'modification_confirmed','service_updated'
    )),
    CONSTRAINT chk_delivery CHECK (delivery_status IN (
        'queued','sent','delivered','failed',
        'undeliverable','blocked','token_expired'
    ))
);
CREATE INDEX idx_zns_order ON zns_notifications(order_id);
CREATE INDEX idx_zns_status ON zns_notifications(delivery_status);
CREATE INDEX idx_zns_retry ON zns_notifications(delivery_status, next_retry_at)
    WHERE delivery_status IN ('queued', 'failed', 'token_expired');
CREATE INDEX idx_zns_tracking ON zns_notifications(tracking_id);
CREATE INDEX idx_zns_msg_id ON zns_notifications(zalo_msg_id);
```

### Updated Redis Keys

```
zns:access_token:{oa_id}         -> encrypted access token       TTL: 55 minutes (refresh 5min before expiry)
zns:quota:{oa_id}:{date}         -> int counter                  TTL: 25h
zns:blocked_phones               -> SET of phone numbers          TTL: none (manual cleanup)
```

## Background Worker: ZNS Processor

The ZNS processor runs as part of `cmd/worker/main.go` with two loops:

### Send Loop (every 2 seconds)

1. Query `zns_notifications` where `delivery_status IN ('queued', 'failed', 'token_expired') AND (next_retry_at IS NULL OR next_retry_at <= NOW())`
2. For each notification (processed sequentially, 100ms gap between sends):
   a. Check quota (`zns:quota:{oa_id}:{date}` < daily limit)
   b. Check if phone is in `zns:blocked_phones` set
   c. Load access token from Redis cache (or DB fallback)
   d. Build request body from template params
   e. Call `POST https://openapi.zalo.me/v2.0/oa/message`
   f. On success (error=0): update status to `sent`, store `zalo_msg_id`, increment quota
   g. On token error (-124): trigger token refresh, retry once immediately
   h. On phone-not-on-zalo error: update status to `undeliverable`
   i. On user-blocked-OA error: update status to `blocked`, add phone to `zns:blocked_phones`
   j. On other errors: increment attempt_count, set `next_retry_at` based on backoff schedule, if max attempts exceeded set status to `failed`

### Token Refresh Loop (every 30 minutes)

1. Query `zns_oa_tokens` where `access_token_expires_at <= NOW() + interval '5 minutes'`
2. Call Zalo token refresh endpoint
3. Store new access token + refresh token (encrypted)
4. Update Redis cache
5. If refresh token expires within 7 days: log CRITICAL alert

## Success Criteria

### Measurable Outcomes

- **SC-001**: ZNS messages are sent within 5 seconds of the triggering event, 95% of the time.
- **SC-002**: ZNS delivery success rate of 90%+ for car owners with valid Zalo-registered phone numbers.
- **SC-003**: Failed ZNS deliveries never block or delay order operations.
- **SC-004**: 100% of ZNS delivery attempts are logged with complete audit trail.
- **SC-005**: Quota warnings are triggered before 80% usage, giving operators time to manage capacity.
- **SC-006**: Deep links in ZNS messages load the correct order status page 100% of the time with no broken links.
- **SC-007**: The car owner taps the ZNS link and sees their status page within 3 seconds (end-to-end from tap to rendered page).
- **SC-008**: Access token refresh succeeds automatically 100% of the time when the refresh token is valid.
- **SC-009**: ZNS notifications are sent for every order state transition (7 trigger events) without the employee taking any extra action beyond the normal workflow.
