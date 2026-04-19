# Feature Specification: Car Owner Web (bonbon.com.vn)

**Feature Branch**: `003-car-owner-web`
**Created**: 2026-02-26
**Status**: Draft
**Input**: User description: "Web application for car owners to view service status in real time, manage their vehicle profiles, browse a marketplace of garage services, and read automotive news. Entry point is via ZNS Zalo notification link, with frictionless OTP-only authentication."

## User Scenarios & Testing

### User Story 1 - Service Status Tracking (Priority: P1)

A car owner receives a ZNS Zalo notification when their car is checked in at a garage. They tap the link and land on bonbon.com.vn where they can see the real-time status of their vehicle's service. The page also suggests 2 relevant upsell services they can add with a single tap.

**Why this priority**: This is the primary entry point for car owners and the core value loop -- garage checks in car, owner gets notified, owner tracks progress. It drives first-time engagement and enables the upsell mechanism that increases garage revenue.

**Independent Test**: Can be tested by creating an order on bonbon GO, following the ZNS link, and verifying the status page shows the correct current status and upsell suggestions.

**Acceptance Scenarios**:

1. **Given** a car owner receives a ZNS message on Zalo, **When** they tap the link, **Then** they land on bonbon.com.vn and see the current status of their vehicle (Received, Washing, Done, etc.) without needing to log in.
2. **Given** the status page is loaded, **When** the vehicle status changes on bonbon GO (e.g., from Received to Washing), **Then** the status page updates in real time without requiring a page refresh.
3. **Given** the status page is displayed, **When** viewing below the status, **Then** 2 quick care services are recommended based on the vehicle's profile and the garage's catalog.
4. **Given** the upsell suggestions, **When** the car owner taps "Add this service", **Then** a notification is sent to the bonbon GO app for the employee to review and accept, and the car owner sees a "Pending confirmation" state.
5. **Given** the employee accepts the added service on bonbon GO, **When** confirmed, **Then** the car owner receives a ZNS update showing the updated service list and revised total.
6. **Given** the status page, **When** the car owner has not logged in, **Then** the page still functions (status viewing and upsell) but a non-blocking popup suggests "Log in now to save your history."

---

### User Story 2 - Car Owner Login (Priority: P1)

When visiting bonbon.com.vn for the first time (via ZNS link), the car owner is gently prompted to log in. Authentication is OTP-only via their phone number -- no password required -- to maximize simplicity.

**Why this priority**: Login enables persistent history, profile management, and personalized experiences. It must be frictionless to avoid losing first-time visitors who arrived via a notification link.

**Independent Test**: Can be tested by visiting bonbon.com.vn, entering a phone number, receiving OTP, completing login, and verifying the session persists on subsequent visits.

**Acceptance Scenarios**:

1. **Given** the car owner is viewing the service status page without logging in, **When** a soft popup appears suggesting login, **Then** it is dismissible and does not block the current view.
2. **Given** the car owner taps "Log in", **When** they enter their phone number, **Then** an OTP is sent to that number.
3. **Given** the OTP is received, **When** the car owner enters the correct OTP, **Then** they are logged in and the session is established.
4. **Given** the car owner is now logged in, **When** they revisit bonbon.com.vn later, **Then** their session persists and they see their personalized dashboard.
5. **Given** the first-time login flow, **When** completed, **Then** all historical service records for that phone number are automatically linked to the new account.

---

### User Story 3 - Car Owner Profile (Priority: P2)

The logged-in car owner manages their personal information, views all registered vehicles, tracks upcoming insurance/inspection deadlines, and browses their complete service history across all garages in the BonBon system.

**Why this priority**: Profile gives car owners a reason to return to bonbon.com.vn between garage visits. It creates stickiness through vehicle management and deadline reminders.

**Independent Test**: Can be tested by logging in, editing personal info, viewing linked vehicles, checking service history, and verifying date-based reminders are displayed.

**Acceptance Scenarios**:

1. **Given** the car owner is logged in, **When** they navigate to their profile, **Then** they see their personal information: Avatar, Name, Phone, Email, Gender, Date of Birth.
2. **Given** the profile page, **When** the car owner edits any personal field and saves, **Then** the changes are persisted.
3. **Given** the vehicle section, **When** viewing, **Then** all vehicles linked to this phone number across all garages are displayed with: License Plate, Brand, Model Name.
4. **Given** a specific vehicle, **When** expanded, **Then** it shows: Mandatory Insurance Expiry Date, Body Insurance Expiry Date, Inspection Expiry Date, Estimated Next Maintenance Date, and full service history within the BonBon system.
5. **Given** one phone number can have multiple vehicles, **When** viewing the vehicle list, **Then** all vehicles are listed and each can be expanded independently.
6. **Given** upcoming deadlines (insurance, inspection), **When** a deadline is within 30 days, **Then** the profile highlights it as a reminder.

---

### User Story 4 - Garage Profile (Public Page) (Priority: P2)

The car owner can view a public-facing garage profile showing the garage's credibility, completed work portfolio, and service catalog.

**Why this priority**: Builds trust between car owners and garages. Enables discovery when car owners are deciding where to go for services.

**Independent Test**: Can be tested by navigating to a garage's profile URL and verifying all sections render correctly with data from the garage's configuration.

**Acceptance Scenarios**:

1. **Given** the car owner navigates to a garage profile, **When** the page loads, **Then** it displays: Garage Avatar/Photos, Name, Address, Phone Number, Year Established.
2. **Given** the garage profile, **When** scrolling to the portfolio section, **Then** completed projects (from the garage's ShowCase) are displayed in a gallery format.
3. **Given** the garage profile, **When** scrolling to the services section, **Then** all active products and services from the garage's catalog are listed with names, prices, and images.
4. **Given** the garage has set operating hours, **When** viewing the profile, **Then** current open/closed status is displayed based on the garage's configured hours.

---

### User Story 5 - Marketplace (Priority: P3)

A marketplace aggregating products and services from all garages in the BonBon system, allowing car owners to browse, compare, and discover nearby options. The experience is similar to Tuhu.cn.

**Why this priority**: Marketplace is a growth feature that drives traffic and cross-garage discovery. It requires a critical mass of garages and products to be valuable.

**Independent Test**: Can be tested by browsing the marketplace, searching/filtering products, and viewing results from multiple garages.

**Acceptance Scenarios**:

1. **Given** the car owner navigates to the Marketplace, **When** the page loads, **Then** it displays a categorized listing of products and services from all active garages.
2. **Given** the marketplace, **When** the car owner searches by service name or product name, **Then** results from all garages are returned, sorted by relevance.
3. **Given** search results, **When** viewing a product/service card, **Then** it displays: Name, Price, Garage Name, Garage Location, Image.
4. **Given** a product/service card, **When** tapped, **Then** it navigates to the garage's profile page with the specific item highlighted.
5. **Given** location permissions are granted, **When** browsing the marketplace, **Then** results are prioritized by proximity to the car owner's current location.

---

### User Story 6 - News and Sharing (Priority: P3)

An editorial section with articles about car care knowledge, product reviews, service tips, and automotive news. This section serves dual purpose: providing value to car owners and driving SEO traffic to bonbon.com.vn.

**Why this priority**: Content marketing feature that drives organic discovery. Important for long-term growth but not required for core operations.

**Independent Test**: Can be tested by publishing an article in the CMS, verifying it appears on the news page, and checking that SEO meta tags are correctly generated.

**Acceptance Scenarios**:

1. **Given** the car owner navigates to News, **When** the page loads, **Then** articles are displayed in reverse chronological order with title, excerpt, featured image, and publish date.
2. **Given** the article list, **When** the car owner taps an article, **Then** the full article is displayed with rich content (text, images, embedded media).
3. **Given** article pages, **When** crawled by search engines, **Then** proper SEO meta tags (title, description, Open Graph, structured data) are present for indexing.
4. **Given** the news page, **When** browsing, **Then** articles are categorized by topic (e.g., car care tips, product reviews, industry news) and filterable by category.

---

### Edge Cases

- What happens when a car owner taps a ZNS link for an order that has already been completed (Delivered status)? The page should show the final order summary with a "Service completed" badge rather than an error.
- What happens when the car owner's phone number is associated with vehicles at multiple garages? The profile should show all vehicles and service history across all garages, with clear garage attribution per record.
- What happens when a car owner tries to add an upsell service but the garage has emergency-closed? The upsell buttons should be disabled with a message that the garage is temporarily closed.
- What happens when the car owner has no vehicles linked to their phone? The profile should display an empty state with an explanation that vehicle records are created when they visit a BonBon-connected garage.
- What happens when a ZNS link is opened on a desktop browser? The experience should be fully responsive and work well on both mobile and desktop viewports.
- What happens when a garage removes a product that was previously shown on the marketplace? The item should be removed from the marketplace listing; existing orders referencing it are unaffected.

## Requirements

### Functional Requirements

- **FR-001**: System MUST allow car owners to view real-time service status via a link received through ZNS Zalo notification, without requiring login.
- **FR-002**: System MUST update service status in real time on the status page when changes occur on bonbon GO.
- **FR-003**: System MUST display 2 recommended upsell services on the status page, tailored to the vehicle and garage catalog.
- **FR-004**: System MUST send a notification to bonbon GO when a car owner selects an upsell service, and update the car owner with a ZNS message upon employee confirmation.
- **FR-005**: System MUST authenticate car owners via OTP-only (no password) using their phone number.
- **FR-006**: System MUST display a non-blocking login prompt to unauthenticated users viewing service status.
- **FR-007**: System MUST automatically link all historical service records to a car owner's account upon first login based on phone number matching.
- **FR-008**: System MUST display car owner profile with editable personal info: Avatar, Name, Phone (read-only), Email, Gender, Date of Birth.
- **FR-009**: System MUST display all vehicles linked to the car owner's phone number across all garages, with per-vehicle details: License Plate, Brand, Model, Insurance Dates, Inspection Date, Maintenance Schedule, Service History.
- **FR-010**: System MUST highlight upcoming deadlines (insurance, inspection) within 30 days as reminders.
- **FR-011**: System MUST display public garage profiles with: photos, name, address, phone, year established, portfolio, and service catalog.
- **FR-012**: System MUST display current open/closed status on garage profiles based on configured operating hours and emergency close state.
- **FR-013**: System MUST provide a marketplace aggregating products and services from all active garages, with search and category-based browsing.
- **FR-014**: System MUST support proximity-based sorting in the marketplace when location permissions are granted.
- **FR-015**: System MUST provide a news/editorial section with categorized articles, rich content, and proper SEO meta tags.
- **FR-016**: System MUST be fully responsive across mobile and desktop viewports.
- **FR-017**: System MUST handle completed-order ZNS links gracefully by showing a final summary rather than an error.

### Key Entities

- **CarOwnerAccount**: Authenticated car owner. Key attributes: phone (unique, serves as login identifier), name, email, gender, date of birth, avatar.
- **VehicleView**: Aggregated view of a vehicle across all garages. Key attributes: license plate, brand, model, insurance dates, inspection date, service history (cross-garage), linked car owner.
- **GaragePublicProfile**: Public-facing projection of a garage. Key attributes: name, photos, address, phone, year established, operating hours, open/closed status, portfolio posts, active catalog.
- **MarketplaceListing**: Searchable catalog item. Key attributes: product/service name, price, garage name, garage location, image, category.
- **Article**: Editorial content. Key attributes: title, body (rich text), featured image, category, publish date, SEO meta data, author.
- **ServiceStatusSession**: Temporary session for a specific order status tracking. Key attributes: order reference, current status, services list, total, upsell suggestions, garage info.

## Success Criteria

### Measurable Outcomes

- **SC-001**: Car owners can view their vehicle's service status within 3 seconds of tapping the ZNS link.
- **SC-002**: Real-time status updates appear on the car owner's screen within 5 seconds of the employee changing status on bonbon GO.
- **SC-003**: OTP login flow completes in under 30 seconds from phone number entry to authenticated state.
- **SC-004**: 80% of first-time visitors (via ZNS link) successfully view their service status without confusion or drop-off.
- **SC-005**: Upsell service conversion rate of at least 5% among car owners who view the status page.
- **SC-006**: Marketplace search returns results within 2 seconds for any search query.
- **SC-007**: News articles are indexed by search engines within 48 hours of publication.
- **SC-008**: Page load time under 2 seconds on 4G mobile connections for all primary pages (status, profile, marketplace).
