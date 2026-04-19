# Spec 007 — ZNS Full Order Confirmation Pipeline

**Feature Branch**: `zns`
**Created**: 2026-03-22
**Updated**: 2026-03-22
**Status**: Implemented

## 1. Overview

This spec documents the complete implementation of the Zalo ZNS notification pipeline for BonBon order lifecycle events. The feature covers 7 internal trigger events mapped to **4 Zalo templates**, a real Zalo HTTP client with OAuth token management, a background worker, a delivery webhook, and the full test suite.

## 2. Trigger Events → Templates

The 7 internal trigger events are mapped to 4 Zalo templates. Triggers that share identical param structure reuse the same `zalo_template_id` in the `zns_templates` table.

| Internal trigger | Zalo template | Rationale |
|---|---|---|
| `order_created` | **Template A** | Unique — first contact, full order context |
| `status_washing` | **Template B** | Unified status template, `status` = "Đang rửa xe" |
| `status_done` | **Template B** | `status` = "Hoàn thành" |
| `status_delivered` | **Template B** | `status` = "Đã giao xe", `payment_method` populated |
| `status_cancelled` | **Template B** | `status` = "Đã hủy" |
| `service_updated` | **Template C** | Employee proactively changed services |
| `modification_confirmed` | **Template D** | Customer requested → employee approved |

Templates C and D have the **same params** but different message wording — `service_updated` says "garage đã cập nhật dịch vụ", `modification_confirmed` says "yêu cầu của bạn đã được chấp nhận". Two separate Zalo template registrations.

## 3. Template Parameters

### Template A — `order_created`

| Param | Zalo type | Example | Source |
|---|---|---|---|
| `name` | string (30) | `Nguyễn Văn A` | `order.CustomerName` |
| `order_code` | string (30) | `BB-3F9A1C2D` | first 8 hex chars of UUID, uppercased |
| `plate` | string (20) | `51A-12345` | `order.LicensePlate` |
| `garage_name` | string (50) | `Garage Minh Phát` | garageRepo lookup, fallback "BonBon" |
| `service_list` | string | `Rửa xe, Thay nhớt` | comma-joined service names |
| `price` | number (20) | `250000` | `order.TotalAmount` — plain integer, no symbol |
| `status` | string (30) | `Đã tiếp nhận` | `statusVI(order.Status)` |
| `date` | date (20) | `08:30:00 2026/03/22` | `order.ReceivedAt` in ICT (UTC+7) |
| `order_link` | string | `https://order.bonbon.com.vn/s/abc123` | `bonbonBaseURL/status/{token}` |

> **Note**: The recipient phone (`84xxxxxxxxx`) is the top-level `"phone"` field in the Zalo API call — it is not a `template_data` param. Zalo uses it only as a routing identifier, not message content.

### Template B — Status change (washing / done / delivered / cancelled)

All 4 status triggers send the same params. The single Zalo template uses `{status}` and `{payment_method}` as variables.

| Param | washing | done | delivered | cancelled |
|---|---|---|---|---|
| `name` | ✓ | ✓ | ✓ | ✓ |
| `order_code` | ✓ | ✓ | ✓ | ✓ |
| `plate` | ✓ | ✓ | ✓ | ✓ |
| `garage_name` | ✓ | ✓ | ✓ | ✓ |
| `service_list` | ✓ | ✓ | ✓ | ✓ |
| `price` | ✓ | ✓ | ✓ | ✓ |
| `payment_method` | `-` | `-` | `cash`/`transfer` | `-` |
| `status` | `Đang rửa xe` | `Hoàn thành` | `Đã giao xe` | `Đã hủy` |
| `date` | `WashingAt` | `DoneAt` | `DeliveredAt` | `CancelledAt` |
| `order_link` | ✓ | ✓ | ✓ | ✓ |

`payment_method` is `"-"` for non-delivered statuses so Zalo's required-param validation passes.
`date` uses the actual timestamp of each status change, formatted as `HH:MM:SS YYYY/MM/DD` ICT.

### Templates C & D — Service change (service_updated / modification_confirmed)

| Param | Zalo type | Example | Source |
|---|---|---|---|
| `name` | string (30) | `Nguyễn Văn A` | `order.CustomerName` |
| `order_code` | string (30) | `BB-3F9A1C2D` | first 8 hex chars of UUID |
| `plate` | string (20) | `51A-12345` | `order.LicensePlate` |
| `garage_name` | string (50) | `Garage Minh Phát` | garageRepo lookup |
| `service_list` | string | `Rửa xe, Thay nhớt, Thay lốp` | updated list after change |
| `price` | number (20) | `350000` | updated `order.TotalAmount` |
| `status` | string (30) | `Đang rửa xe` | current order status in Vietnamese |
| `date` | date (20) | `09:15:00 2026/03/22` | `time.Now()` in ICT |
| `order_link` | string | `https://order.bonbon.com.vn/s/abc123` | tracking link |

> Service change ZNS only fires when the **total amount changes**. Silent updates (e.g. notes) do not trigger a notification.

## 4. Architecture

```
Order Service (usecase)
  └─ queueZNS(ctx, order, event, params)   ← never blocks, errors logged only
        └─ notificationSvc.Queue(ctx, ...)
              └─ DB: INSERT zns_notifications (status=queued)

Background Worker (cmd/worker, every 2s)
  └─ notificationSvc.ProcessPending(ctx)
        ├─ loadAccessToken (Redis cache → DB decrypt)
        ├─ checkQuota     (Redis: zns:quota:{oaID}:{date})
        ├─ isPhoneBlocked (Redis SET: zns:blocked_phones)
        └─ processSingle  → zaloClient.SendZNS → DB UPDATE

Token Refresh Worker (cmd/worker, every 30min)
  └─ notificationSvc.RefreshTokenIfNeeded(ctx)
        └─ if expiring soon → zaloClient.RefreshToken → DB UPSERT + Redis cache

Webhook Handler  POST /api/v1/webhooks/zalo/zns
  └─ notificationSvc.UpdateDeliveryStatus(trackingID, delivered)
        └─ DB UPDATE zns_notifications

Admin Bootstrap  POST /api/v1/admin/zns/authorize
  └─ notificationSvc.BootstrapToken(authCode)
        └─ zaloClient.ExchangeAuthCode → DB UPSERT + Redis cache
```

## 5. Param Helpers (order/service.go)

| Helper | Returns | Notes |
|---|---|---|
| `znsStatusParams(order, garageName, orderLink)` | `map[string]string` | Unified params for Template B (all 4 status events) |
| `znsServiceParams(order, garageName, orderLink)` | `map[string]string` | Params for Templates C & D (service changes) |
| `shortOrderCode(uuid.UUID)` | `string` | `"BB-" + upper(first 8 hex chars)` |
| `formatPrice(int64)` | `string` | Plain integer string, e.g. `"250000"` (Zalo number type) |
| `formatVND(int64)` | `string` | `"250000 vnđ"` — human-readable, used in logs only |
| `zaloDate(time.Time)` | `string` | `"HH:MM:SS YYYY/MM/DD"` in ICT (UTC+7) |
| `statusVI(OrderStatus)` | `string` | EN status → Vietnamese label |
| `serviceListString([]OrderService)` | `string` | Comma-joined service names |

## 6. Retry Policy

| Attempt | Backoff |
|---|---|
| 1 (immediate) | 0s |
| 2 | +30s |
| 3 | +2min |
| 4 (final) | +15min |

After 4 failed attempts → `failed` permanently.

**Special Zalo error codes:**
| Code | Behaviour |
|---|---|
| `-124` invalid token | marked `token_expired`, retried after next token refresh |
| `-201` not on Zalo | marked `undeliverable`, no retry |
| `-210` user blocked OA | marked `blocked`, phone added to Redis blocked set |
| `-216` quota exceeded | `next_retry_at` = next midnight ICT, attempt count unchanged |

## 7. Database Schema

### `zns_oa_tokens` (new table)
| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| oa_id | VARCHAR UNIQUE | Platform OA identifier |
| access_token_encrypted | TEXT | AES-256-GCM in production (stub in dev) |
| refresh_token_encrypted | TEXT | |
| access_token_expires_at | TIMESTAMPTZ | |
| refresh_token_expires_at | TIMESTAMPTZ | |
| last_refreshed_at | TIMESTAMPTZ | |

### `zns_notifications` (columns added by migration 000003)
| New column | Type | Notes |
|---|---|---|
| tracking_id | VARCHAR NOT NULL UNIQUE | `{orderID}_{triggerEvent}` |
| zalo_msg_id | VARCHAR | Returned by Zalo on success |
| next_retry_at | TIMESTAMPTZ | NULL = retry immediately |

Migration: `000003_zns_full_pipeline.up.sql`

## 8. Configuration (env vars)

| Var | Default | Description |
|---|---|---|
| `ZALO_APP_ID` | — | Zalo application ID |
| `ZALO_OA_ID` | — | Official Account ID |
| `ZALO_SECRET_KEY` | — | Secret key for OAuth calls |
| `ZALO_BASE_URL` | `https://openapi.zalo.me` | API base |
| `ZALO_OAUTH_URL` | `https://oauth.zaloapp.com/v4/oa/access_token` | OAuth endpoint |
| `ZALO_DAILY_QUOTA` | `500` | Max messages/day |
| `ZALO_WEBHOOK_VERIFY_TOKEN` | — | Shared secret for `X-Zalo-Webhook-Token` |
| `BONBON_BASE_URL` | `https://order.bonbon.com.vn` | Generates `order_link` |
| `ZALO_TEMPLATE_ORDER_CREATED` | — | Template A ID |
| `ZALO_TEMPLATE_STATUS_WASHING` | — | Template B ID (same as done/delivered/cancelled) |
| `ZALO_TEMPLATE_STATUS_DONE` | — | Template B ID |
| `ZALO_TEMPLATE_STATUS_DELIVERED` | — | Template B ID |
| `ZALO_TEMPLATE_STATUS_CANCELLED` | — | Template B ID |
| `ZALO_TEMPLATE_MODIFICATION_CONFIRMED` | — | Template D ID |
| `ZALO_TEMPLATE_SERVICE_UPDATED` | — | Template C ID |

Set all 4 status env vars to the same Zalo template ID once Template B is approved.

## 9. Deployment Checklist

1. Run migration `000003_zns_full_pipeline.up.sql`
2. Get authorization_code from Zalo OA Dashboard → Settings → API
3. Call `POST /api/v1/admin/zns/authorize` with `{"auth_code": "..."}` (one-time bootstrap)
4. Submit 4 templates to Zalo for approval
5. Once approved, insert template IDs into `zns_templates` (or set env vars)
6. Replace `encrypt()`/`decrypt()` stubs in `notification/service.go` with AES-256-GCM before production
7. Deploy `cmd/worker` alongside `cmd/api`

## 10. E2E Test Flow (Manual / Staging)

### Step 1 — Bootstrap OA token
```bash
curl -X POST .../api/v1/admin/zns/authorize \
  -H "Authorization: Bearer <garage_owner_token>" \
  -H "Content-Type: application/json" \
  -d '{"auth_code": "<zalo_auth_code>"}'
# Expected: 200 {"message": "Zalo OA token bootstrapped successfully"}
```

### Step 2 — Register templates
```sql
-- Template A
INSERT INTO zns_templates (id, zalo_template_id, internal_name, trigger_event, is_approved)
VALUES (gen_random_uuid(), '<template_a_id>', 'Order Created', 'order_created', true);

-- Template B (insert same zalo_template_id for all 4 status events)
INSERT INTO zns_templates (id, zalo_template_id, internal_name, trigger_event, is_approved) VALUES
  (gen_random_uuid(), '<template_b_id>', 'Status Change', 'status_washing',   true),
  (gen_random_uuid(), '<template_b_id>', 'Status Change', 'status_done',      true),
  (gen_random_uuid(), '<template_b_id>', 'Status Change', 'status_delivered', true),
  (gen_random_uuid(), '<template_b_id>', 'Status Change', 'status_cancelled', true);

-- Template C
INSERT INTO zns_templates (id, zalo_template_id, internal_name, trigger_event, is_approved)
VALUES (gen_random_uuid(), '<template_c_id>', 'Service Updated', 'service_updated', true);

-- Template D
INSERT INTO zns_templates (id, zalo_template_id, internal_name, trigger_event, is_approved)
VALUES (gen_random_uuid(), '<template_d_id>', 'Modification Confirmed', 'modification_confirmed', true);
```

### Step 3 — Full lifecycle test
```bash
# 1. Create order → verify order_created notification queued
# 2. Worker runs → verify delivery_status = 'sent', zalo_msg_id populated
# 3. Transition status → verify status_washing notification queued + sent
# 4. Transition to done → status_done
# 5. Record payment → status_delivered (payment_method populated)
# 6. Zalo webhook callback → delivery_status = 'delivered'

SELECT trigger_event, delivery_status, attempt_count, zalo_msg_id
FROM zns_notifications
WHERE order_id = '<order_id>'
ORDER BY created_at;
```

### Failure scenarios
| Scenario | Expected |
|---|---|
| Invalid token (corrupt Redis key) | `token_expired` → re-sent after token refresh |
| Phone not on Zalo (-201) | `undeliverable`, no retry |
| User blocked OA (-210) | `blocked`, phone added to Redis SET |
| Quota exceeded (-216) | retry = next midnight ICT |
| Transient error | backoff 0s/30s/2m/15m → `failed` after 4 attempts |

## 11. Test Coverage

| Layer | File | Cases |
|---|---|---|
| Notification svc | `usecase/notification/service_test.go` | 20 unit tests |
| Zalo HTTP client | `repo/apiclient/zalo/client_test.go` | 12 unit tests |
| Webhook handler | `handler/http/v1/webhook_handler_test.go` | 9 unit tests |
| Mocks | `testutil/mocks.go` | GetByTrackingID, MockZNSOATokenRepo |
| Integration | `testdata/integration/zns_test.go` | 4 tests (skip if no DB) |
