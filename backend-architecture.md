# Backend Architecture: BonBon Platform

**Date**: 2026-02-27 | **Stack**: Go 1.22+ / PostgreSQL 16 / Redis 7 | **Pattern**: Monorepo, multi-component, clean architecture

## 1. C4 Model

### Level 1 -- System Context

```
                  +------------------+
                  |   Car Owner      |
                  |   (Person)       |
                  +--------+---------+
                           |
                           | Uses bonbon.com.vn
                           v
+----------------+   +-----------+   +------------------+
| Garage Owner   |-->| BonBon    |<--| Garage Employee  |
| (Person)       |   | Platform  |   | (Person)         |
+----------------+   +-----+-----+   +------------------+
  Uses garage.           |   |         Uses bonbon GO
  bonbon.com.vn          |   |         mobile app
                         v   v
                  +------+---+------+
                  | Zalo ZNS API    |
                  | (External)      |
                  +-----------------+
```

Three user types interact with one platform. The platform sends transactional notifications via Zalo ZNS API.

### Level 2 -- Container Diagram

```
+----------------------------------------------------+
|                    BonBon Platform                   |
|                                                      |
|  +-----------+  +-----------+  +------------------+  |
|  | garage-web|  |car-owner- |  | bonbon GO        |  |
|  | (Next.js) |  |web        |  | (React Native)   |  |
|  | :3001     |  |(Next.js)  |  |                  |  |
|  |           |  | :3002     |  |                  |  |
|  +-----+-----+  +-----+-----+  +--------+---------+  |
|        |              |                  |            |
|        +--------------+------------------+            |
|                       |                               |
|                       v                               |
|              +--------+--------+                      |
|              | BonBon API      |                      |
|              | (Go, Fiber)     |                      |
|              | :25502           |                      |
|              +--------+--------+                      |
|                       |                               |
|           +-----------+-----------+                   |
|           |                       |                   |
|           v                       v                   |
|    +------+------+         +------+------+            |
|    | PostgreSQL  |         | Redis       |            |
|    | :5432       |         | :6379       |            |
|    +-------------+         +-------------+            |
+----------------------------------------------------+
```

A single Go API server serves all three frontend apps. PostgreSQL is the source of truth. Redis handles caching, real-time pub/sub for order status, and session storage.

### Level 3 -- Component Diagram (inside BonBon API)

The API is a single deployable binary, but internally organized into 6 components. Each component owns its domain, use cases, handlers, and repository. Components communicate through well-defined Go interfaces (in-process), not HTTP.

```
+------------------------------------------------------------------+
|                         BonBon API                                |
|                                                                    |
|  +-------------+  +-------------+  +-------------+                |
|  |    Auth      |  |   Garage    |  |   Catalog   |                |
|  |  Component   |  |  Component  |  |  Component  |                |
|  |             |  |             |  |             |                |
|  | - Login     |  | - Profile   |  | - Products  |                |
|  | - Register  |  | - Dashboard |  | - Services  |                |
|  | - OTP       |  | - Emergency |  | - Import    |                |
|  | - Sessions  |  | - Hours     |  | - Categories|                |
|  +------+------+  +------+------+  +------+------+                |
|         |                |                |                        |
|  +------+------+  +------+------+  +------+------+                |
|  |   Order      |  |  Customer   |  | Notification|                |
|  |  Component   |  |  Component  |  |  Component  |                |
|  |             |  |             |  |             |                |
|  | - Create    |  | - Vehicles  |  | - ZNS Send  |                |
|  | - Status    |  | - Employees |  | - Templates |                |
|  | - Payment   |  | - Accounts  |  | - Retry     |                |
|  | - Modify    |  | - Showcase  |  | - Quota     |                |
|  | - History   |  | - Search    |  | - Audit log |                |
|  +-------------+  +-------------+  +-------------+                |
|                                                                    |
|  Shared: pkg/logger, pkg/httpserver, pkg/postgres, pkg/redis,      |
|          pkg/auth (JWT), pkg/validator                             |
+------------------------------------------------------------------+
```

**Component responsibilities:**

| Component | Owns | Depends On |
|-----------|------|------------|
| **Auth** | Garage owner accounts, employee accounts, car owner OTP sessions, JWT tokens | Redis (sessions) |
| **Garage** | Garage profiles, operating hours, emergency status, dashboard metrics | Auth, Order (metrics), Redis (cache) |
| **Catalog** | Products, services, categories, bulk import | Auth, Garage |
| **Order** | Orders, order services, status transitions, payments, service modification requests | Auth, Catalog, Customer, Notification |
| **Customer** | Customers, vehicles, vehicle history, employee records, showcase posts | Auth, Garage |
| **Notification** | ZNS messages, templates, delivery tracking, quota | Zalo ZNS API (external) |

**Dependency direction:** Auth <- Garage <- Catalog, Auth <- Customer, (Auth, Catalog, Customer, Notification) <- Order. Notification has no inbound dependencies from other components (Order calls Notification, not the other way).

### Shared Infrastructure: Object Storage

The platform uses an S3-compatible object storage service for all file uploads (images, QR codes, etc.). Domain models store only the resulting public URL; binary data is never stored in PostgreSQL.

```
+------------------------------------------------------------------+
|                     Object Storage (S3-compatible)                |
|                                                                    |
|  pkg/objectstorage/                                               |
|    objectstorage.go       # Client interface + S3 implementation  |
|                                                                    |
|  Bucket structure:                                                |
|    garage/{garage_id}/images/     -- carousel photos              |
|    garage/{garage_id}/bank-qr/    -- bank QR code                 |
|    garage/{garage_id}/products/   -- product/service images       |
|    garage/{garage_id}/customers/  -- customer avatars             |
|    garage/{garage_id}/showcase/   -- showcase post images         |
|    car-owner/{account_id}/avatar/ -- car owner avatars            |
+------------------------------------------------------------------+
```

| Config | Env Variable | Default |
|--------|-------------|---------|
| Endpoint | `S3_ENDPOINT` | `http://localhost:9000` |
| Region | `S3_REGION` | `us-east-1` |
| Bucket | `S3_BUCKET` | `bonbon` |
| Access Key | `S3_ACCESS_KEY` | `minioadmin` |
| Secret Key | `S3_SECRET_KEY` | `minioadmin` |
| Public URL | `S3_PUBLIC_URL` | `http://localhost:9000/bonbon` |

For local development, MinIO runs as a Docker container. In production, any S3-compatible service (AWS S3, DigitalOcean Spaces, etc.) can be used.

**Upload API**: `POST /api/v1/upload` accepts `multipart/form-data` with fields `file` (the binary), `folder` (path prefix). Returns `{ "url": "https://..." }`. Max file size: 10 MB. Allowed MIME types: `image/jpeg`, `image/png`, `image/webp`, `image/gif`.

**Delete API**: `DELETE /api/v1/upload` accepts `{ "url": "https://..." }` in the JSON request body. Extracts the S3 object key from the URL path and deletes the object from storage. Returns `{ "ok": true }`. Both endpoints require JWT authentication.

---

## 2. Domain Models

### 2.1 Auth Component

```go
type GarageOwner struct {
    ID           uuid.UUID
    Phone        string       // unique, Vietnamese format 0XXXXXXXXX
    PasswordHash string
    GarageID     uuid.UUID    // 1:1 with Garage
    Status       AccountStatus
    CreatedAt    time.Time
    UpdatedAt    time.Time
}

type AccountStatus string
const (
    AccountStatusActive   AccountStatus = "active"
    AccountStatusInactive AccountStatus = "inactive"
)

type EmployeeAccount struct {
    ID           uuid.UUID
    Phone        string       // username, unique within garage
    PasswordHash string
    EmployeeID   uuid.UUID
    GarageID     uuid.UUID
    Status       AccountStatus
    CreatedAt    time.Time
    UpdatedAt    time.Time
}

type CarOwnerAccount struct {
    ID        uuid.UUID
    Phone     string          // unique, login via OTP
    Name      *string
    Email     *string
    Gender    *Gender
    DOB       *time.Time
    AvatarURL *string
    CreatedAt time.Time
    UpdatedAt time.Time
}

type Gender string
const (
    GenderMale   Gender = "male"
    GenderFemale Gender = "female"
    GenderOther  Gender = "other"
)

type AuthSession struct {
    Token     string
    AccountID uuid.UUID
    Role      AuthRole        // garage_owner | employee | car_owner
    GarageID  *uuid.UUID      // nil for car_owner
    ExpiresAt time.Time
}

type AuthRole string
const (
    RoleGarageOwner AuthRole = "garage_owner"
    RoleEmployee    AuthRole = "employee"
    RoleCarOwner    AuthRole = "car_owner"
)
```

### 2.2 Garage Component

```go
type Garage struct {
    ID               uuid.UUID
    Name             string
    Phone            string
    Address          string
    Latitude         *float64
    Longitude        *float64
    Capacity         int
    YearEstablished  *int
    BankQRImageURL   *string
    Status           GarageStatus
    Images           []string         // max 3 URLs
    OperatingHours   []OperatingHour
    CreatedAt        time.Time
    UpdatedAt        time.Time
}

type GarageStatus string
const (
    GarageStatusOpen            GarageStatus = "open"
    GarageStatusClosed          GarageStatus = "closed"
    GarageStatusEmergencyClosed GarageStatus = "emergency_closed"
)

type OperatingHour struct {
    DayOfWeek int    // 0=Monday, 6=Sunday
    OpenTime  string // HH:mm
    CloseTime string // HH:mm
}

func (g *Garage) IsCurrentlyOpen(now time.Time) bool {
    // pure domain logic: check day + time against OperatingHours
    // returns false if emergency_closed regardless of hours
}

type DashboardMetrics struct {
    Today DashboardPeriod
    Week  DashboardPeriod
    Month DashboardPeriod
}

type DashboardPeriod struct {
    Revenue     int64 // VND, no decimals
    AvgDaily    int64
    CarCount    int
    MotoCount   int
}
```

### 2.3 Catalog Component

```go
type Product struct {
    ID          uuid.UUID
    GarageID    uuid.UUID
    Name        string
    Price       int64          // VND
    Description *string        // rich text / HTML
    ImageURL    *string
    Category    string
    Type        ProductType
    Inventory   *int           // nil for services
    Brand       *string
    Status      ProductStatus
    CreatedAt   time.Time
    UpdatedAt   time.Time
}

type ProductType string
const (
    ProductTypeProduct ProductType = "product"
    ProductTypeService ProductType = "service"
)

type ProductStatus string
const (
    ProductStatusActive ProductStatus = "active"
    ProductStatusPaused ProductStatus = "paused"
)

func (p *Product) Validate() error {
    // if Type == ProductTypeProduct, Inventory must not be nil
    // Price must be > 0
    // Name must not be empty
}

type BulkImportResult struct {
    SuccessCount int
    ErrorCount   int
    Errors       []ImportRowError
}

type ImportRowError struct {
    Row     int
    Field   string
    Message string
}
```

### 2.4 Order Component

```go
type Order struct {
    ID              uuid.UUID
    Token           string        // unique, non-guessable, for status page URL
    GarageID        uuid.UUID
    LicensePlate    string
    VehicleType     VehicleType
    CustomerName    string
    CustomerPhone   *string
    EmployeeID      uuid.UUID
    Status          OrderStatus
    Services        []OrderService
    TotalAmount     int64         // VND, auto-calculated
    PaymentMethod   *PaymentMethod
    ZNSEnabled      bool
    LongTermNotes   string
    VisitNotes      string
    PublicExpiresAt time.Time     // CreatedAt + 72h; anonymous access expires after this
    ReceivedAt      time.Time
    WashingAt       *time.Time
    DoneAt          *time.Time
    DeliveredAt     *time.Time
    CancelledAt     *time.Time
    CreatedAt       time.Time
    UpdatedAt       time.Time
}

type OrderStatus string
const (
    OrderStatusReceived  OrderStatus = "received"
    OrderStatusWashing   OrderStatus = "washing"
    OrderStatusDone      OrderStatus = "done"
    OrderStatusDelivered OrderStatus = "delivered"
    OrderStatusCancelled OrderStatus = "cancelled"
)

type VehicleType string
const (
    VehicleTypeCar        VehicleType = "car"
    VehicleTypeMotorcycle  VehicleType = "motorcycle"
)

type PaymentMethod string
const (
    PaymentCash     PaymentMethod = "cash"
    PaymentTransfer PaymentMethod = "transfer"
    PaymentCard     PaymentMethod = "card"
    PaymentOther    PaymentMethod = "other"
)

type OrderService struct {
    ID          uuid.UUID
    OrderID     uuid.UUID
    ProductID   *uuid.UUID    // nil for temporary services
    Name        string
    Price       int64
    Quantity    int
    IsTemporary bool
}

func (o *Order) CanTransitionTo(target OrderStatus) bool {
    switch o.Status {
    case OrderStatusReceived:
        return target == OrderStatusWashing || target == OrderStatusCancelled
    case OrderStatusWashing:
        return target == OrderStatusDone
    case OrderStatusDone:
        return target == OrderStatusDelivered
    default:
        return false
    }
}

func (o *Order) CanModifyServices() bool {
    return o.Status == OrderStatusReceived || o.Status == OrderStatusWashing
}

const PublicLinkTTL = 72 * time.Hour

func (o *Order) IsPublicAccessExpired(now time.Time) bool {
    return !o.PublicExpiresAt.IsZero() && now.After(o.PublicExpiresAt)
}

func (o *Order) RecalculateTotal() {
    var total int64
    for _, svc := range o.Services {
        total += svc.Price * int64(svc.Quantity)
    }
    o.TotalAmount = total
}

type OrderStatusTransition struct {
    ID            uuid.UUID
    OrderID       uuid.UUID
    FromStatus    OrderStatus
    ToStatus      OrderStatus
    EmployeeID    uuid.UUID
    TriggerSource string      // "employee" | "system"
    CreatedAt     time.Time
}

type ServiceModificationRequest struct {
    ID          uuid.UUID
    OrderID     uuid.UUID
    ProductID   uuid.UUID
    ServiceName string
    Price       int64
    Source      ModificationSource
    Status      ModificationStatus
    RequestedAt time.Time
    ResolvedAt  *time.Time
}

type ModificationSource string
const (
    ModSourceCarOwner ModificationSource = "car_owner"
    ModSourceEmployee ModificationSource = "employee"
)

type ModificationStatus string
const (
    ModStatusPending  ModificationStatus = "pending"
    ModStatusAccepted ModificationStatus = "accepted"
    ModStatusDeclined ModificationStatus = "declined"
)
```

### 2.5 Customer Component

```go
type Customer struct {
    ID             uuid.UUID
    GarageID       uuid.UUID
    Phone          string
    Name           string
    DOB            *time.Time
    Gender         *Gender
    Occupation     *string
    LongTermNotes  string
    Classification *CustomerClassification
    CreatedAt      time.Time
    UpdatedAt      time.Time
}

type CustomerClassification string
const (
    ClassNew      CustomerClassification = "new"
    ClassRegular  CustomerClassification = "regular"
    ClassVIP      CustomerClassification = "vip"
    ClassInactive CustomerClassification = "inactive"
)

type Vehicle struct {
    ID                      uuid.UUID
    GarageID                uuid.UUID
    CustomerID              uuid.UUID
    LicensePlate            string     // unique within garage
    Brand                   *string
    Model                   *string
    Year                    *int
    CurrentKM               *int
    BodyInsuranceExpiry     *time.Time
    MandatoryInsuranceExpiry *time.Time
    InspectionExpiry        *time.Time
    Notes                   string
    IsSold                  bool
    CreatedAt               time.Time
    UpdatedAt               time.Time
}

func (v *Vehicle) DetectType() VehicleType {
    // Vietnamese plate format detection
    // Motorcycle: 2 digits - letter digit(s) space 3digits.2digits (e.g., 29-X1 234.56)
    // Car: 2 digits letter - 3digits.2digits (e.g., 30A-123.45)
}

type Employee struct {
    ID              uuid.UUID
    GarageID        uuid.UUID
    Name            string
    Phone           string
    CCCD            string        // national ID, globally unique
    DOB             *time.Time
    Gender          *Gender
    Education       *string
    Certifications  *string
    IndustryStartYear *int
    Skills          []string      // from predefined set of 15
    Status          EmployeeStatus
    CreatedAt       time.Time
    UpdatedAt       time.Time
}

type EmployeeStatus string
const (
    EmployeeStatusActive   EmployeeStatus = "active"
    EmployeeStatusInactive EmployeeStatus = "inactive"
)

var ValidSkills = []string{
    "basic_wash", "detail_wash", "glass_stain", "tar_removal",
    "paint_dust", "interior_cleaning", "engine_bay",
    "polishing", "film_install", "ppf_install",
    "decal_install", "dashcam_install", "tire_pressure_sensor",
    "light_modification", "audio_modification",
}

type ShowcasePost struct {
    ID          uuid.UUID
    GarageID    uuid.UUID
    Title       string
    Description string
    Images      []string
    PublishedAt time.Time
    CreatedAt   time.Time
    UpdatedAt   time.Time
}
```

### 2.6 Notification Component

```go
type ZNSNotification struct {
    ID             uuid.UUID
    OrderID        uuid.UUID
    TriggerEvent   ZNSTriggerEvent
    RecipientPhone string
    TemplateID     string
    Params         map[string]string
    TrackingID     string
    ZaloMsgID      *string
    DeliveryStatus ZNSDeliveryStatus
    AttemptCount   int
    LastAttemptAt  *time.Time
    NextRetryAt    *time.Time
    ErrorDetail    *string
    CreatedAt      time.Time
    UpdatedAt      time.Time
}

type ZNSTriggerEvent string
const (
    ZNSTriggerOrderCreated          ZNSTriggerEvent = "order_created"
    ZNSTriggerStatusWashing         ZNSTriggerEvent = "status_washing"
    ZNSTriggerStatusDone            ZNSTriggerEvent = "status_done"
    ZNSTriggerStatusDelivered       ZNSTriggerEvent = "status_delivered"
    ZNSTriggerStatusCancelled       ZNSTriggerEvent = "status_cancelled"
    ZNSTriggerModificationConfirmed ZNSTriggerEvent = "modification_confirmed"
    ZNSTriggerServiceUpdated        ZNSTriggerEvent = "service_updated"
)

type ZNSDeliveryStatus string
const (
    ZNSStatusQueued        ZNSDeliveryStatus = "queued"
    ZNSStatusSent          ZNSDeliveryStatus = "sent"
    ZNSStatusDelivered     ZNSDeliveryStatus = "delivered"
    ZNSStatusFailed        ZNSDeliveryStatus = "failed"
    ZNSStatusUndeliverable ZNSDeliveryStatus = "undeliverable"
    ZNSStatusBlocked       ZNSDeliveryStatus = "blocked"
    ZNSStatusTokenExpired  ZNSDeliveryStatus = "token_expired"
)

type ZNSTemplate struct {
    ID             uuid.UUID
    ZaloTemplateID string
    InternalName   string
    TriggerEvent   ZNSTriggerEvent
    TagType        int
    ParamSchema    []string
    IsApproved     bool
    ApprovedAt     *time.Time
}

type ZNSOAToken struct {
    ID                     uuid.UUID
    OAID                   string
    AccessTokenEncrypted   string
    RefreshTokenEncrypted  string
    AccessTokenExpiresAt   time.Time
    RefreshTokenExpiresAt  time.Time
    LastRefreshedAt        time.Time
    CreatedAt              time.Time
    UpdatedAt              time.Time
}

func (t *ZNSOAToken) IsAccessTokenExpiringSoon(now time.Time) bool {
    return now.Add(5 * time.Minute).After(t.AccessTokenExpiresAt)
}

func (t *ZNSOAToken) IsRefreshTokenExpiringSoon(now time.Time) bool {
    return now.Add(7 * 24 * time.Hour).After(t.RefreshTokenExpiresAt)
}

type ZNSQuotaTracker struct {
    OAID              string
    DailyLimit        int
    CurrentUsage      int
    LastResetAt       time.Time
    WarningTriggered  bool
}

func (q *ZNSQuotaTracker) CanSend() bool {
    return q.CurrentUsage < q.DailyLimit
}

func (q *ZNSQuotaTracker) IsWarningThreshold() bool {
    return float64(q.CurrentUsage) >= float64(q.DailyLimit)*0.8
}
```

---

## 3. Database Schema (PostgreSQL)

### 3.1 Auth Tables

```sql
CREATE TABLE garage_owners (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone           VARCHAR(15) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    garage_id       UUID NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_garage_owners_phone ON garage_owners(phone);

CREATE TABLE employee_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone           VARCHAR(15) NOT NULL,
    password_hash   VARCHAR(255) NOT NULL,
    employee_id     UUID NOT NULL,
    garage_id       UUID NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(phone, garage_id)
);
CREATE INDEX idx_employee_accounts_phone ON employee_accounts(phone);
CREATE INDEX idx_employee_accounts_garage ON employee_accounts(garage_id);

CREATE TABLE car_owner_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    phone           VARCHAR(15) NOT NULL UNIQUE,
    name            VARCHAR(100),
    email           VARCHAR(255),
    gender          VARCHAR(10),
    dob             DATE,
    avatar_url      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_car_owners_phone ON car_owner_accounts(phone);
```

### 3.2 Garage Tables

```sql
CREATE TABLE garages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(200) NOT NULL,
    phone               VARCHAR(15),
    address             TEXT,
    latitude            DOUBLE PRECISION,
    longitude           DOUBLE PRECISION,
    capacity            INT NOT NULL DEFAULT 0,
    year_established    INT,
    bank_qr_image_url   TEXT,
    status              VARCHAR(30) NOT NULL DEFAULT 'open',
    images              TEXT[] DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE operating_hours (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    garage_id   UUID NOT NULL REFERENCES garages(id),
    day_of_week INT NOT NULL,
    open_time   VARCHAR(5) NOT NULL,
    close_time  VARCHAR(5) NOT NULL,
    UNIQUE(garage_id, day_of_week)
);
CREATE INDEX idx_operating_hours_garage ON operating_hours(garage_id);
```

### 3.3 Catalog Tables

```sql
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    garage_id   UUID NOT NULL REFERENCES garages(id),
    name        VARCHAR(200) NOT NULL,
    price       BIGINT NOT NULL,
    description TEXT,
    image_url   TEXT,
    category    VARCHAR(100) NOT NULL DEFAULT '',
    type        VARCHAR(20) NOT NULL,
    inventory   INT,
    brand       VARCHAR(100),
    status      VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_product_type CHECK (type IN ('product', 'service')),
    CONSTRAINT chk_product_status CHECK (status IN ('active', 'paused')),
    CONSTRAINT chk_product_inventory CHECK (
        (type = 'service' AND inventory IS NULL) OR
        (type = 'product' AND inventory IS NOT NULL AND inventory >= 0)
    )
);
CREATE INDEX idx_products_garage ON products(garage_id);
CREATE INDEX idx_products_garage_status ON products(garage_id, status);
CREATE INDEX idx_products_name_gin ON products USING gin(to_tsvector('simple', name));
```

### 3.4 Order Tables

```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    token           VARCHAR(64) NOT NULL UNIQUE,
    garage_id       UUID NOT NULL REFERENCES garages(id),
    license_plate   VARCHAR(20) NOT NULL,
    vehicle_type    VARCHAR(15) NOT NULL,
    customer_name   VARCHAR(100) NOT NULL DEFAULT '',
    customer_phone  VARCHAR(15),
    employee_id     UUID NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'received',
    total_amount    BIGINT NOT NULL DEFAULT 0,
    payment_method  VARCHAR(20),
    zns_enabled     BOOLEAN NOT NULL DEFAULT true,
    long_term_notes TEXT NOT NULL DEFAULT '',
    visit_notes     TEXT NOT NULL DEFAULT '',
    public_expires_at TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '72 hours',
    received_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    washing_at      TIMESTAMPTZ,
    done_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_order_status CHECK (status IN ('received','washing','done','delivered','cancelled')),
    CONSTRAINT chk_payment_method CHECK (payment_method IS NULL OR payment_method IN ('cash','transfer','card','other')),
    CONSTRAINT chk_vehicle_type CHECK (vehicle_type IN ('car','motorcycle'))
);
CREATE INDEX idx_orders_token ON orders(token);
CREATE INDEX idx_orders_garage ON orders(garage_id);
CREATE INDEX idx_orders_garage_status ON orders(garage_id, status);
CREATE INDEX idx_orders_garage_date ON orders(garage_id, created_at DESC);
CREATE INDEX idx_orders_plate_garage ON orders(license_plate, garage_id);

CREATE TABLE order_services (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id    UUID NOT NULL REFERENCES orders(id),
    product_id  UUID REFERENCES products(id),
    name        VARCHAR(200) NOT NULL,
    price       BIGINT NOT NULL,
    quantity    INT NOT NULL DEFAULT 1,
    is_temporary BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_order_services_order ON order_services(order_id);

CREATE TABLE order_status_transitions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id),
    from_status     VARCHAR(20) NOT NULL,
    to_status       VARCHAR(20) NOT NULL,
    employee_id     UUID NOT NULL,
    trigger_source  VARCHAR(20) NOT NULL DEFAULT 'employee',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_transitions_order ON order_status_transitions(order_id);

CREATE TABLE service_modification_requests (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id    UUID NOT NULL REFERENCES orders(id),
    product_id  UUID NOT NULL,
    service_name VARCHAR(200) NOT NULL,
    price       BIGINT NOT NULL,
    source      VARCHAR(20) NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'pending',
    requested_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at  TIMESTAMPTZ,
    CONSTRAINT chk_mod_source CHECK (source IN ('car_owner', 'employee')),
    CONSTRAINT chk_mod_status CHECK (status IN ('pending', 'accepted', 'declined'))
);
CREATE INDEX idx_mod_requests_order ON service_modification_requests(order_id);
```

### 3.5 Customer Tables

```sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    garage_id       UUID NOT NULL REFERENCES garages(id),
    phone           VARCHAR(15) NOT NULL,
    name            VARCHAR(100) NOT NULL,
    dob             DATE,
    gender          VARCHAR(10),
    occupation      VARCHAR(100),
    long_term_notes TEXT NOT NULL DEFAULT '',
    classification  VARCHAR(20),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(garage_id, phone)
);
CREATE INDEX idx_customers_garage ON customers(garage_id);
CREATE INDEX idx_customers_phone ON customers(phone);

CREATE TABLE vehicles (
    id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    garage_id                   UUID NOT NULL REFERENCES garages(id),
    customer_id                 UUID NOT NULL REFERENCES customers(id),
    license_plate               VARCHAR(20) NOT NULL,
    brand                       VARCHAR(50),
    model                       VARCHAR(50),
    year                        INT,
    current_km                  INT,
    body_insurance_expiry       DATE,
    mandatory_insurance_expiry  DATE,
    inspection_expiry           DATE,
    notes                       TEXT NOT NULL DEFAULT '',
    is_sold                     BOOLEAN NOT NULL DEFAULT false,
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(garage_id, license_plate) WHERE (is_sold = false)
);
CREATE INDEX idx_vehicles_garage ON vehicles(garage_id);
CREATE INDEX idx_vehicles_customer ON vehicles(customer_id);
CREATE INDEX idx_vehicles_plate ON vehicles(license_plate);

CREATE TABLE employees (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    garage_id           UUID NOT NULL REFERENCES garages(id),
    name                VARCHAR(100) NOT NULL,
    phone               VARCHAR(15) NOT NULL,
    cccd                VARCHAR(20) NOT NULL,
    dob                 DATE,
    gender              VARCHAR(10),
    education           VARCHAR(100),
    certifications      TEXT,
    industry_start_year INT,
    skills              TEXT[] DEFAULT '{}',
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_employees_cccd ON employees(cccd);
CREATE INDEX idx_employees_garage ON employees(garage_id);

CREATE TABLE showcase_posts (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    garage_id   UUID NOT NULL REFERENCES garages(id),
    title       VARCHAR(200) NOT NULL,
    description TEXT NOT NULL DEFAULT '',
    images      TEXT[] DEFAULT '{}',
    published_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_showcase_garage ON showcase_posts(garage_id);
```

### 3.6 Notification Tables

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

CREATE TABLE zns_templates (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    zalo_template_id VARCHAR(100) NOT NULL UNIQUE,
    internal_name    VARCHAR(100) NOT NULL,
    trigger_event    VARCHAR(30) NOT NULL,
    tag_type         INT NOT NULL DEFAULT 1,
    param_schema     TEXT[] DEFAULT '{}',
    is_approved      BOOLEAN NOT NULL DEFAULT false,
    approved_at      TIMESTAMPTZ
);

CREATE TABLE zns_oa_tokens (
    id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    oa_id                       VARCHAR(100) NOT NULL UNIQUE,
    access_token_encrypted      TEXT NOT NULL,
    refresh_token_encrypted     TEXT NOT NULL,
    access_token_expires_at     TIMESTAMPTZ NOT NULL,
    refresh_token_expires_at    TIMESTAMPTZ NOT NULL,
    last_refreshed_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 3.7 Redis Keys

```
session:{token}                  -> JSON AuthSession          TTL: 24h (garage/employee), 30d (car_owner)
garage:{id}:status               -> GarageStatus string       TTL: none (invalidate on change)
garage:{id}:dashboard            -> JSON DashboardMetrics     TTL: 60s
garage:{id}:products             -> JSON []Product            TTL: 300s
order:{id}:status                -> OrderStatus string        TTL: none (invalidate on change)
order:token:{token}              -> order_id UUID             TTL: 90d
zns:access_token:{oa_id}         -> encrypted access token    TTL: 55 minutes
zns:quota:{oa_id}:{date}         -> int counter               TTL: 25h
zns:blocked_phones               -> SET of phone numbers       TTL: none
```

---

## 4. REST API Contracts

Base URL: `/api/v1`

### 4.1 Auth API

```
POST   /auth/garage/register        Register garage owner + garage
POST   /auth/garage/login           Login garage owner (phone + password + captcha)
POST   /auth/employee/login         Login employee (phone + password)
POST   /auth/car-owner/otp/request  Request OTP for car owner (phone)
POST   /auth/car-owner/otp/verify   Verify OTP and login (phone + otp)
POST   /auth/logout                 Logout (invalidate session)
GET    /auth/me                     Get current session info
```

| Endpoint | Request Body | Response | Auth |
|----------|-------------|----------|------|
| `POST /auth/garage/register` | `{ phone, password, garageName }` | `{ token, garage }` | None |
| `POST /auth/garage/login` | `{ phone, password }` | `{ token, garage }` | None |
| `POST /auth/employee/login` | `{ phone, password }` | `{ token, garageId, employeeId }` | None |
| `POST /auth/car-owner/otp/request` | `{ phone }` | `{ success: true }` | None |
| `POST /auth/car-owner/otp/verify` | `{ phone, otp }` | `{ token, user }` | None |

### 4.2 Garage API

```
GET    /garages/me                  Get own garage profile
PUT    /garages/me                  Update garage profile
PATCH  /garages/me/emergency        Toggle emergency close/open
GET    /garages/me/dashboard        Get dashboard metrics
GET    /garages/me/hours            Get operating hours
PUT    /garages/me/hours            Update operating hours
GET    /garages/:id/public          Get public garage profile (no auth)
```

| Endpoint | Request | Response | Auth |
|----------|---------|----------|------|
| `GET /garages/me` | -- | `{ garage }` | garage_owner |
| `PUT /garages/me` | `{ name, phone, address, lat, lng, capacity, images, bankQR }` | `{ garage }` | garage_owner |
| `PATCH /garages/me/emergency` | `{ action: "close" \| "open" }` | `{ status }` | garage_owner |
| `GET /garages/me/dashboard` | `?tz=Asia/Ho_Chi_Minh` | `{ today, week, month }` | garage_owner |
| `GET /garages/:id/public` | -- | `{ garage, hours, showcase, catalog }` | None |

### 4.3 Catalog API

```
GET    /products                    List products/services (own garage)
POST   /products                    Create product/service
GET    /products/:id                Get product/service by ID
PUT    /products/:id                Update product/service
PATCH  /products/:id/status         Activate/pause product
POST   /products/import             Bulk import from Excel
GET    /products/export-template    Download Excel template
```

| Endpoint | Key Fields | Auth |
|----------|-----------|------|
| `GET /products` | `?search=&type=&status=&page=&limit=` | garage_owner |
| `POST /products` | `{ name, price, type, category, description, inventory, brand, imageUrl }` | garage_owner |
| `POST /products/import` | `multipart/form-data` with Excel file | garage_owner |

### 4.4 Order API

```
POST   /orders                      Create order (from bonbon GO)
GET    /orders                      List orders (own garage, filterable)
GET    /orders/:id                  Get order detail
PATCH  /orders/:id/status           Transition order status
PUT    /orders/:id/services         Update services on order
PATCH  /orders/:id/payment          Record payment
GET    /orders/:id/transitions      Get status transition history
POST   /orders/:id/modification     Request service addition (car owner)
PATCH  /orders/:id/modification/:mid Accept/decline modification
GET    /orders/history/daily        Daily summary for employee
GET    /status/:token               Get order by token (public, no auth)
```

| Endpoint | Request | Response | Auth |
|----------|---------|----------|------|
| `POST /orders` | `{ licensePlate, customerName, customerPhone, services[], visitNotes, znsEnabled }` | `{ order }` | employee |
| `PATCH /orders/:id/status` | `{ status: "washing"\|"done"\|"cancelled" }` | `{ order }` | employee |
| `PATCH /orders/:id/payment` | `{ method: "cash"\|"transfer"\|"card"\|"other" }` | `{ order }` | employee |
| `POST /orders/:id/modification` | `{ productId, serviceName, price }` | `{ request }` | car_owner |
| `PATCH /orders/:id/modification/:mid` | `{ action: "accept"\|"decline" }` | `{ request, order }` | employee |
| `GET /orders/history/daily` | `?date=2026-02-27` | `{ summary, orders[] }` | employee |
| `GET /status/:token` | -- | `{ order, services, garage, upsell[] }` or 410 if link expired (72h) | None (rate limited: 30/min per IP) |
| `GET /orders` | `?status=&from=&to=&page=&limit=` | `{ orders[], total, page }` | garage_owner |

### 4.5 Customer API

```
GET    /customers                   List customers (own garage)
POST   /customers                   Create customer
GET    /customers/:id               Get customer detail
PUT    /customers/:id               Update customer
GET    /customers/:id/vehicles      List vehicles for customer

POST   /vehicles                    Create vehicle
PUT    /vehicles/:id                Update vehicle
PATCH  /vehicles/:id/sold           Mark vehicle as sold

GET    /employees                   List employees (own garage)
POST   /employees                   Create employee
PUT    /employees/:id               Update employee
POST   /employees/import            Bulk import employees
POST   /employees/:id/account       Create bonbon GO account for employee
PATCH  /employee-accounts/:id/status Activate/deactivate account

GET    /showcase                    List showcase posts (own garage)
POST   /showcase                    Create showcase post
PUT    /showcase/:id                Update showcase post
DELETE /showcase/:id                Delete showcase post
```

| Endpoint | Key Fields | Auth |
|----------|-----------|------|
| `GET /customers` | `?search=&classification=&page=&limit=` (search by phone, plate, name) | garage_owner |
| `POST /vehicles` | `{ licensePlate, customerPhone, name, brand, model, year, km, insurances, notes }` | garage_owner |
| `POST /employees` | `{ name, phone, cccd, dob, gender, education, certifications, industryStartYear, skills[] }` | garage_owner |

### 4.6 Notification API (internal + admin)

```
GET    /notifications/orders/:id    List ZNS notifications for order
GET    /notifications/quota         Get current ZNS quota usage
POST   /webhooks/zalo/zns           Receive ZNS delivery status callback from Zalo
```

| Endpoint | Request | Response | Auth |
|----------|---------|----------|------|
| `GET /notifications/orders/:id` | -- | `{ notifications[] }` | garage_owner, employee |
| `GET /notifications/quota` | -- | `{ dailyLimit, currentUsage, warningTriggered }` | garage_owner |
| `POST /webhooks/zalo/zns` | `{ event_name, msg_id, tracking_id, delivery_time }` | `200 OK` | Zalo verify token |

### 4.7 Car Owner API (authenticated car owner)

```
GET    /me/profile                  Get car owner profile
PUT    /me/profile                  Update car owner profile
GET    /me/vehicles                 List all vehicles across garages
GET    /me/vehicles/:plate          Get vehicle detail with history
GET    /marketplace                 Browse marketplace
GET    /marketplace/:id             Marketplace item detail
GET    /news                        List articles
GET    /news/:slug                  Get article detail
```

### 4.8 Real-time: SSE (Server-Sent Events)

```
GET    /sse/orders/:id              Stream order status changes
GET    /sse/garage/orders           Stream all order updates for a garage
```

Redis Pub/Sub channels:
- `order:{garage_id}` -- publishes order status changes, new orders, service modifications
- Clients connect via SSE, server subscribes to Redis and forwards events

---

## 5. Go Project Structure

Following `specs/golang-project-structure.md`:

```
backend/
  cmd/
    api/main.go                         # HTTP API server entry point
    migrate/main.go                     # Database migration runner
    worker/main.go                      # Background worker (ZNS retry, quota reset)

  config/
    config.go                           # Env-based config: App, HTTP, DB, Redis, ZNS

  internal/
    app/
      app.go                            # Bootstrap, wire deps, start Fiber + SSE + worker

    deps/
      deps.go                           # DI container: all repos, usecases, handlers

    domain/
      auth.go                           # GarageOwner, EmployeeAccount, CarOwnerAccount, AuthSession
      garage.go                         # Garage, OperatingHour, GarageStatus, DashboardMetrics
      product.go                        # Product, ProductType, ProductStatus, BulkImportResult
      order.go                          # Order, OrderService, OrderStatus, PaymentMethod,
                                        #   OrderStatusTransition, ServiceModificationRequest
      customer.go                       # Customer, Vehicle, CustomerClassification
      employee.go                       # Employee, EmployeeStatus, ValidSkills
      notification.go                   # ZNSNotification, ZNSTemplate, ZNSQuotaTracker
      showcase.go                       # ShowcasePost
      errors.go                         # Domain errors: ErrNotFound, ErrDuplicate, ErrInvalidTransition, etc.

    handler/
      http/
        middleware/
          auth.go                       # JWT extraction, role-based guard
          logging.go                    # Request/response logging
          cors.go                       # CORS for web apps
          recovery.go                   # Panic recovery
          ratelimit.go                  # Per-IP rate limiting for public endpoints
        v1/
          router.go                     # Mount all route groups
          auth_handler.go               # POST /auth/*
          garage_handler.go             # GET/PUT /garages/*
          product_handler.go            # CRUD /products/*
          order_handler.go              # CRUD /orders/*, GET /status/:token
          customer_handler.go           # CRUD /customers/*, /vehicles/*
          employee_handler.go           # CRUD /employees/*, /employee-accounts/*
          showcase_handler.go           # CRUD /showcase/*
          notification_handler.go       # GET /notifications/*, POST /webhooks/zalo/zns
          car_owner_handler.go          # GET /me/*, /marketplace, /news
          sse_handler.go                # SSE /sse/*
          request/                      # Request DTOs with validation tags
          response/                     # Response DTOs

    repo/
      contracts.go                      # All repository interfaces

      persistent/
        auth/repo.go, model.go          # garage_owners, employee_accounts, car_owner_accounts
        garage/repo.go, model.go        # garages, operating_hours
        product/repo.go, model.go       # products
        order/repo.go, model.go         # orders, order_services, transitions, mod_requests
        customer/repo.go, model.go      # customers, vehicles
        employee/repo.go, model.go      # employees
        notification/repo.go, model.go  # zns_notifications, zns_templates, zns_oa_tokens
        showcase/repo.go, model.go      # showcase_posts

      apiclient/
        zalo/client.go                  # Zalo ZBS Template Message API: SendZNS, RefreshToken
        zalo/token.go                   # OAuth token management: refresh, encrypt/decrypt, cache

    usecase/
      contracts.go                      # All use case interfaces

      auth/service.go                   # Register, login, OTP, session management
      garage/service.go                 # Profile CRUD, emergency toggle, dashboard aggregation
      product/service.go                # CRUD, bulk import, search
      order/service.go                  # Create, status transition (with state machine validation),
                                        #   payment, service modification, daily history
      customer/service.go               # Customer CRUD, vehicle management, search
      employee/service.go               # Employee CRUD, account provisioning
      notification/service.go           # Queue ZNS (7 trigger events), process retries,
                                        #   quota tracking, token management, webhook handling
      showcase/service.go               # Post CRUD

  pkg/
    logger/logger.go                    # slog-based structured logger
    httpserver/server.go                # Fiber HTTP server wrapper with graceful shutdown
    postgres/postgres.go                # GORM PostgreSQL connection with pool config
    redis/redis.go                      # go-redis client wrapper
    auth/jwt.go                         # JWT sign/verify, claims extraction
    validator/validator.go              # Request validation helpers
    sse/broker.go                       # SSE broker backed by Redis Pub/Sub
    crypto/password.go                  # bcrypt password hashing
    crypto/token.go                     # Secure random token generation (order tokens)
    phone/phone.go                      # Vietnamese phone normalization (+84 <-> 0)
    plate/plate.go                      # License plate normalization, vehicle type detection
    excel/importer.go                   # Excel parsing for bulk imports

  migrations/
    000001_init.up.sql                  # All CREATE TABLE statements
    000001_init.down.sql                # All DROP TABLE statements

  mocks/                                # Generated by mockery

  Makefile
  go.mod
  go.sum
  .env.example
```

### Key Libraries

| Purpose | Library | Why |
|---------|---------|-----|
| HTTP framework | `gofiber/fiber/v2` | Fast, Express-like, low memory, good for high-throughput |
| ORM | `gorm.io/gorm` + `gorm.io/driver/postgres` | Mature, convention over config, migration support |
| Redis | `redis/go-redis/v9` | Full Redis feature set, pub/sub, pipelining |
| Config | `caarlos0/env/v9` | Env-only config, no YAML/TOML complexity |
| JWT | `golang-jwt/jwt/v5` | Standard JWT library |
| Validation | `go-playground/validator/v10` | Struct tag validation |
| Migration | `golang-migrate/migrate/v4` | SQL-file based migrations |
| Testing | `stretchr/testify` + `vektra/mockery` | Assertions + mock generation |
| Logging | `log/slog` (stdlib) | Structured logging, no external dep |
| UUID | `google/uuid` | Standard UUID generation |
| Excel | `qax-os/excelize/v2` | Excel read/write for bulk imports |
| Password | `golang.org/x/crypto/bcrypt` | Industry standard password hashing |

---

## 6. Cross-Cutting Concerns

### Authentication Flow

```
Garage Owner:  phone + password + captcha -> JWT (role: garage_owner, garageId in claims)
Employee:      phone + password -> JWT (role: employee, garageId, employeeId in claims)
Car Owner:     phone -> OTP -> JWT (role: car_owner, no garageId)
Public:        /status/:token (rate limited, 72h expiry), /garages/:id/public, /marketplace, /news -- no auth
```

JWT stored in Redis with TTL. Every request validates token against Redis (allows instant revocation on logout/deactivation).

### Real-time Order Updates (SSE + Redis Pub/Sub)

1. Employee changes order status on bonbon GO -> `PATCH /orders/:id/status`
2. Order use case updates DB, publishes to Redis channel `order:{garage_id}`
3. SSE handler for `/sse/orders/:id` subscribes to that channel, filters by order ID
4. Car owner's browser receives SSE event, updates status page UI
5. Garage web dashboard receives same event stream via `/sse/garage/orders`

### ZNS Notification Flow

ZNS is triggered at every order state transition (7 events), not just creation and completion.

1. Order use case calls `notification.Service.Queue(orderID, triggerEvent)` at each trigger point (order_created, status_washing, status_done, status_delivered, status_cancelled, modification_confirmed, service_updated)
2. Notification service checks `order.ZNSEnabled && order.CustomerPhone != nil`, then creates a `zns_notifications` row with status `queued`, tracking_id `{order_id}_{trigger_event}`, and template params built from order + garage data
3. Background worker (Send Loop, every 2 seconds) polls for queued/retryable notifications
4. Worker checks quota (Redis), blocked phone list, then loads access token from Redis cache (falls back to DB + decrypt)
5. Worker calls `POST https://openapi.zalo.me/v2.0/oa/message` via `apiclient/zalo` with phone (84-format), template_id, template_data, tracking_id
6. On success (error=0): update status to `sent`, store `zalo_msg_id`, increment quota counter
7. On token error (-124): trigger immediate token refresh, retry once
8. On phone-not-on-zalo: update status to `undeliverable`
9. On user-blocked-OA: update status to `blocked`, add phone to `zns:blocked_phones` Redis set
10. On other errors: increment attempt_count, set next_retry_at (immediate, +30s, +2min, +15min), max 4 attempts then status `failed`
11. Token Refresh Loop (every 30 minutes): refresh access token 5 minutes before expiry, alert if refresh token expires within 7 days
12. Webhook (`POST /api/v1/webhooks/zalo/zns`): receives delivery callbacks from Zalo, updates notification status from `sent` to `delivered` using `tracking_id` / `zalo_msg_id` correlation

### Data Isolation

Each garage's data is scoped by `garage_id`. All repository queries for garage-owned data include `WHERE garage_id = ?`. Car owner data is cross-garage (vehicles/history across all garages visible to the car owner via their phone number).
