# Feature Specification: Garage Web Portal (garage.bonbon.com.vn)

**Feature Branch**: `001-garage-web`
**Created**: 2026-02-26
**Status**: Draft
**Input**: User description: "Web portal for garage owners to manage registration, profiles, dashboard analytics, transactions, products/services, customers, employees, employee accounts, and showcase portfolio."

## User Scenarios & Testing

### User Story 1 - Garage Registration and Login (Priority: P1)

A garage owner visits garage.bonbon.com.vn to create an account for their car wash business. They register with their phone number, set a password, verify via OTP, and then can log in to access the management dashboard.

**Why this priority**: Without authentication, no other feature is accessible. This is the gateway to the entire platform.

**Independent Test**: Can be fully tested by registering a new phone number, receiving OTP, completing registration, logging out, and logging back in.

**Acceptance Scenarios**:

1. **Given** the registration page is loaded, **When** the owner enters a valid phone number, password, confirms password, completes captcha, and submits, **Then** the system sends an OTP to that phone number.
2. **Given** an OTP has been sent, **When** the owner enters the correct OTP within the validity window, **Then** the account is created and the owner is redirected to the dashboard.
3. **Given** a phone number already registered, **When** someone tries to register with the same number, **Then** the system rejects registration with a clear error message.
4. **Given** the login page is loaded, **When** the owner enters correct phone number, password, and completes captcha, **Then** the owner is redirected to the dashboard.
5. **Given** incorrect credentials, **When** the owner attempts to log in, **Then** the system shows an error without revealing which field is wrong.

---

### User Story 2 - Garage Profile Setup (Priority: P1)

After logging in for the first time, the garage owner fills in their business profile including name, car wash capacity, address, Google Maps location, representative photos, operating hours, and bank QR code for receiving payments.

**Why this priority**: The profile feeds data into the car owner-facing app (public garage page) and provides essential business configuration (capacity, hours, payment info) needed before operations can begin.

**Independent Test**: Can be tested by logging in, filling all profile fields, saving, and verifying the data persists on page reload.

**Acceptance Scenarios**:

1. **Given** the owner is logged in and navigates to Profile, **When** they enter garage name, wash capacity (number), full address, and save, **Then** the profile is persisted and visible on reload.
2. **Given** the profile form, **When** the owner pins a location on the embedded Google Map, **Then** the system captures and stores the exact latitude and longitude.
3. **Given** the photo upload section, **When** the owner selects or drags up to 3 image files, **Then** the images are uploaded to object storage, URLs are saved to the profile, and previews are displayed in a carousel.
3a. **Given** an image is being uploaded, **When** the upload completes successfully, **Then** the system displays a visible success indicator ("Da tai len" badge with green border) for 2 seconds before returning to the normal preview state.
4. **Given** the photo upload section, **When** the owner tries to upload more than 3 images, **Then** the system prevents it with a message indicating the 3-image limit.
4a. **Given** any image upload field, **When** the owner selects a file exceeding 10 MB or a non-image file, **Then** the system rejects with a clear size/type error before uploading.
4b. **Given** an image preview with a "Xoa" (delete) button, **When** the owner clicks the delete button, **Then** the system deletes the actual object from S3 storage via `DELETE /api/v1/upload` and clears the URL reference from the profile form.
5. **Given** the operating hours section, **When** the owner sets open and close times for each day, **Then** those hours are saved and used to determine garage availability.
6. **Given** the payment section, **When** the owner selects or drags a bank QR code image file, **Then** the image is uploaded to object storage and the URL is stored, displayed to employees during payment flow.

---

### User Story 3 - Dashboard Overview (Priority: P1)

The garage owner lands on the dashboard after login and immediately sees key business metrics: today's revenue and visit counts, this week's and this month's aggregated data, plus an emergency close/open toggle.

**Why this priority**: The dashboard is the home screen and the primary tool for daily business monitoring. Revenue and traffic visibility drives garage owner retention on the platform.

**Independent Test**: Can be tested by verifying metric cards render with correct data for today, this week, and this month. Emergency close button can be toggled and verified independently.

**Acceptance Scenarios**:

1. **Given** the owner is logged in, **When** the dashboard loads, **Then** it displays today's revenue, car visit count, and motorcycle visit count.
2. **Given** the dashboard is loaded, **When** viewing weekly metrics, **Then** it shows total weekly revenue, average daily revenue, car visits, and motorcycle visits for the current week.
3. **Given** the dashboard is loaded, **When** viewing monthly metrics, **Then** it shows total monthly revenue, average daily revenue, car visits, and motorcycle visits for the current month.
4. **Given** the garage is currently open, **When** the owner presses the emergency close button, **Then** the garage status changes to closed and is reflected across all apps (bonbon GO cannot receive new cars, car owner web shows garage as closed).
5. **Given** the garage is emergency-closed, **When** the owner presses the button again, **Then** the garage re-opens and operations resume.

---

### User Story 3b - Analytics Dashboard (Priority: P2)

The garage owner navigates to the Analytics page to view a visual breakdown of business performance across multiple dimensions. The page displays KPI cards (total revenue, total transactions, average order value, refund/cancellation rate) and an interactive chart + ranked breakdown panel controlled by the active KPI and dimension tab.

**Why this priority**: Visualization of revenue and transaction data gives garage owners deeper operational insight beyond the daily summary dashboard. Depends on order data existing (P1 order flow).

**Independent Test**: Can be tested by verifying KPI cards render with correct formatted values, switching between metrics re-renders the chart and breakdown, and switching dimension tabs updates both panels.

**Acceptance Scenarios**:

1. **Given** the owner navigates to the Analytics page, **When** the page loads, **Then** four KPI cards are displayed: Total revenue, Total transactions, Avg. order value, and Refund/cancellation rate, each with a formatted value and a delta badge showing percentage change.
2. **Given** the analytics page is loaded, **When** the owner clicks a KPI card, **Then** that card becomes active (highlighted outline) and both the bar chart and breakdown list update to show data for the selected metric.
3. **Given** the analytics page, **When** the owner clicks a dimension tab (Service type, Vehicle type, Payment method, Employee), **Then** the bar chart and breakdown list re-render with data grouped by that dimension.
4. **Given** the bar chart panel, **When** viewing data, **Then** bars have rounded corners, y-axis grid lines, formatted tooltip values, and a custom legend with colored squares.
5. **Given** the breakdown panel, **When** viewing data, **Then** rows are sorted highest to lowest, each showing a colored dot, category name, formatted value, percentage of total, and a horizontal progress bar.
6. **Given** a narrow screen, **When** viewing the analytics page, **Then** KPI cards display in a 2-column grid and the chart/breakdown panels stack vertically.

---

### User Story 3c - Revenue Trend Chart (Priority: P2)

The garage owner sees a line chart on the Tong Quan page that visualizes daily revenue and transaction count over the last 7 or 30 days. A toggle switches between the two time ranges. The chart shows two series (revenue as a filled area, transaction count as a line) with a shared date axis.

**Why this priority**: Time-series visualization allows the owner to spot revenue trends, seasonal dips, and growth patterns that aggregate KPIs alone cannot reveal. Complements the existing daily/weekly/monthly stat cards.

**Independent Test**: Can be tested by verifying the chart renders with the correct number of data points (7 or 30), the toggle switches between ranges, and hovering shows a tooltip with formatted values.

**Acceptance Scenarios**:

1. **Given** the Tong Quan page is loaded, **When** the trend section renders, **Then** a line chart displays daily revenue for the last 7 days by default, with labeled x-axis (dates) and y-axis (revenue).
2. **Given** the trend chart, **When** the owner clicks the "30 ngay" toggle, **Then** the chart re-renders with 30 data points showing the last 30 days of revenue.
3. **Given** the trend chart, **When** hovering over a data point, **Then** a tooltip shows the date, formatted revenue, and transaction count for that day.
4. **Given** no orders exist for some days in the range, **When** the chart renders, **Then** those days show zero revenue instead of being omitted.
5. **Given** the API endpoint `GET /api/v1/garages/me/trend?tz=<timezone>&days=<7|30>`, **When** called, **Then** it returns an array of daily data points sorted by date ascending, each containing date, revenue, and transaction count.

---

### User Story 4 - Transaction History (Priority: P2)

The garage owner views, filters, and exports a detailed history of all service transactions at their garage, including payment breakdowns.

**Why this priority**: Financial tracking is essential for business operations but only becomes meaningful after cars are actually being serviced (requires P1 order flow to exist).

**Independent Test**: Can be tested by creating sample transactions, then filtering by date range and status, verifying correct results, and exporting to Excel.

**Acceptance Scenarios**:

1. **Given** the Transactions page is loaded, **When** no filters are applied, **Then** the system displays all transactions with columns: Time, License Plate, Owner Name, Phone, Services/Products Used, Amount, Transaction Status, Payment Method, Notes, Account that created the order.
2. **Given** the Transactions page, **When** the owner filters by a date range, **Then** only transactions within that range are shown.
3. **Given** the Transactions page, **When** the owner filters by transaction status (e.g., Completed, Cancelled), **Then** only transactions matching that status are shown.
4. **Given** both filters are applied independently, **When** the owner sets a date range AND a status filter, **Then** transactions matching both criteria are shown.
5. **Given** filtered results, **When** the owner clicks "Export Excel", **Then** a downloadable Excel file is generated matching the displayed data.
6. **Given** the Transactions page, **When** viewing the summary header, **Then** totals grouped by payment method (Cash, Transfer, Card, Other) are displayed at the top of the report.

---

### User Story 5 - Products and Services Management (Priority: P2)

The garage owner creates, edits, and organizes their catalog of products (with inventory) and services (without inventory), including bulk import via Excel.

**Why this priority**: The catalog is required for order creation on bonbon GO and for displaying on the car owner marketplace/profile. Depends on basic auth being in place.

**Independent Test**: Can be tested by adding a product manually, adding a service manually, importing via Excel template, searching, and editing an item.

**Acceptance Scenarios**:

1. **Given** the Products/Services page, **When** the owner clicks "Add New" and fills in Name, Price, Description, Image, Category, Type (Product or Service), and saves, **Then** the item appears in the catalog.
2. **Given** the item type is "Product", **When** creating or editing, **Then** an inventory quantity field is visible and required.
3. **Given** the item type is "Service", **When** creating or editing, **Then** no inventory field is shown.
4. **Given** the description field, **When** the owner adds text, images, or videos, **Then** all media types are saved and rendered correctly in the description (rich text editor).
5. **Given** the bulk import flow, **When** the owner downloads the Excel template, fills it out, and uploads it, **Then** all valid items are created and any invalid rows are reported with specific error messages.
6. **Given** the search bar, **When** the owner types a product/service name, **Then** matching results are shown in real time.
7. **Given** an existing item, **When** the owner edits any field and saves, **Then** the changes are persisted.
8. **Given** a product/service, **When** setting status to "Paused", **Then** the item is hidden from the car owner marketplace and bonbon GO quick-select but remains in the catalog.
9. **Given** product images, **When** uploading, **Then** images must conform to a standardized dimension to ensure consistent display on the car owner web.

---

### User Story 6 - Customer Management (Priority: P2)

The garage owner manages customer information organized primarily by license plate, with phone number as the secondary key. The system supports AI-based customer classification, long-term notes, and full order history per customer.

**Why this priority**: Customer data enriches the service experience and supports retention strategies. It becomes valuable once transactions start flowing.

**Independent Test**: Can be tested by adding a vehicle (license plate + owner info), searching by plate/phone/name, viewing customer details, and testing the "car sold" transfer flow.

**Acceptance Scenarios**:

1. **Given** the Customers page, **When** the owner clicks "Add Vehicle" and enters License Plate (required), Owner Name (required), Phone (required), plus optional fields (Brand, Model, Year, Notes, Current KM, Insurance Dates, Inspection Date), **Then** the vehicle record is created.
2. **Given** current KM field, **When** the owner chooses to capture it via photo, **Then** the system accepts an ODO meter photo and extracts the reading.
3. **Given** an existing phone number, **When** adding another vehicle with the same phone, **Then** the system links both vehicles to the same owner (one owner, many vehicles).
4. **Given** an existing license plate, **When** trying to add a duplicate, **Then** the system rejects it with an error indicating the plate already exists.
5. **Given** a vehicle record, **When** the owner marks it as "Car Sold", **Then** the vehicle is archived and its license plate becomes available for re-registration under a new owner.
6. **Given** the search bar, **When** searching by phone number, license plate, or name, **Then** matching customer records are returned showing: Name, Phone, Total Spending at this garage, Time since first visit, AI Customer Classification.
7. **Given** a customer record, **When** viewing details, **Then** the system shows long-term notes, all order history (which may span multiple license plates), and all vehicles under that phone number.
8. **Given** the bulk import flow, **When** uploading an Excel file of customers, **Then** valid records are created and duplicates/errors are reported.
9. **Given** customer data, **When** viewed by a different garage on the platform, **Then** no data is shared. Each garage's customer information is private to that garage.

---

### User Story 7 - Employee Management (Priority: P3)

The garage owner manages employee records with detailed skill tracking, using the national ID (CCCD) as a portable employee identifier across garages.

**Why this priority**: Employee management is an operational optimization feature. Garages can operate initially with a single admin account before needing granular employee tracking.

**Independent Test**: Can be tested by adding an employee manually, verifying skill checkboxes, searching by phone/CCCD, and importing via Excel.

**Acceptance Scenarios**:

1. **Given** the Employee page, **When** the owner clicks "Add Employee" and fills in Name, Phone, CCCD, Date of Birth, Gender, Education, Certifications, Year Entered Industry, Status (Active/Inactive), and selects applicable skill checkboxes, **Then** the employee record is created with CCCD as the unique identifier.
2. **Given** the skill checkboxes, **When** creating/editing an employee, **Then** the following skills are available: Basic Wash, Detail Wash, Glass Stain Removal, Tar Removal, Paint Dust Removal, Interior Cleaning, Engine Bay Cleaning, Polishing, Film Installation, PPF Installation, Decal Installation, Dashcam Installation, Tire Pressure Sensor Installation, Light Modification, Audio Modification.
3. **Given** the search bar, **When** searching by Phone or CCCD, **Then** matching employee records are returned.
4. **Given** an employee record, **When** editing any field and saving, **Then** changes are persisted.
5. **Given** the bulk import flow, **When** uploading a completed Excel template, **Then** valid employee records are created and errors are reported per row.
6. **Given** a CCCD, **When** that employee moves to another garage in the BonBon system, **Then** the new garage can recognize them by CCCD as a returning industry professional.

---

### User Story 8 - Employee Account Management (Priority: P3)

The garage owner creates login accounts for employees so they can use the bonbon GO mobile app to receive cars at the garage.

**Why this priority**: Required for multi-employee garages but a single owner can initially operate bonbon GO themselves.

**Independent Test**: Can be tested by creating an employee, provisioning a login account, verifying the employee can log into bonbon GO, and soft-disabling the account.

**Acceptance Scenarios**:

1. **Given** an existing employee record, **When** the owner creates a "Register" account for them, **Then** a bonbon GO login is created with the employee's phone number as username and an owner-set password.
2. **Given** the Register page, **When** searching by Phone or CCCD, **Then** matching accounts are shown with their ID and status.
3. **Given** an active account, **When** the owner clicks "Deactivate", **Then** the account is soft-disabled (cannot log in, but all historical transaction data tied to that account is preserved).
4. **Given** a deactivated account, **When** viewed in the system, **Then** it still appears in search results with a "Deactivated" status badge.
5. **Given** account deletion is not supported, **When** the owner wants to remove an employee, **Then** deactivation is the only option, ensuring transaction history integrity.

---

### User Story 9 - Showcase Portfolio (Priority: P3)

The garage owner publishes posts showcasing completed detailing work, installations, and service results. These posts appear on the garage's public profile visible to car owners.

**Why this priority**: Marketing/portfolio feature that enhances the garage's public image. Not required for core operations.

**Independent Test**: Can be tested by creating a showcase post with images and text, and verifying it appears on the garage's public profile on bonbon.com.vn.

**Acceptance Scenarios**:

1. **Given** the ShowCase page, **When** the owner creates a new post with title, description, and images, **Then** the post is published and visible on the garage's public profile on the car owner web.
2. **Given** an existing post, **When** the owner edits or deletes it, **Then** the changes are immediately reflected on the public profile.
3. **Given** multiple posts, **When** viewing the ShowCase page, **Then** posts are displayed in reverse chronological order.

---

### Edge Cases

- What happens when the OTP expires before the user enters it? System should allow resending OTP with a cooldown timer (e.g., 60 seconds).
- What happens when the owner uploads an image exceeding the maximum file size (10 MB)? The frontend validates before upload and rejects with a clear size limit error. The backend also enforces the limit and returns HTTP 413.
- What happens when the Google Maps API fails to load during profile setup? System should allow manual entry of latitude/longitude as a fallback.
- What happens when an Excel import file contains partially valid data? System should import valid rows and return a detailed error report for invalid rows with row numbers and field-level errors.
- What happens when the garage is emergency-closed while cars are currently being serviced? Existing orders remain active; only new order intake is blocked.
- What happens when a customer's license plate has non-standard characters? System should normalize plates (strip spaces, uppercase) and validate against Vietnamese plate format.
- What happens when two garages try to register the same phone number? Each phone number maps to exactly one garage owner account. The second registration is rejected.

## Requirements

### Functional Requirements

- **FR-001**: System MUST allow garage owners to register using a phone number with OTP verification and captcha.
- **FR-002**: System MUST enforce one account per phone number.
- **FR-003**: System MUST allow garage owners to log in with phone number, password, and captcha.
- **FR-004**: System MUST allow garage owners to configure their profile: garage name, wash capacity, address, Google Maps coordinates (latitude/longitude), up to 3 carousel images (uploaded via file picker or drag-and-drop), operating hours per day, and bank QR code image (uploaded via file picker or drag-and-drop). All image uploads go to S3-compatible object storage; only URLs are stored in the database.
- **FR-005**: System MUST display dashboard metrics for today (revenue, car count, motorcycle count), this week (revenue, avg daily revenue, car count, motorcycle count), and this month (same breakdown).
- **FR-005b**: System MUST provide an analytics dashboard page with four KPI cards (Total revenue, Total transactions, Avg. order value, Refund/cancellation rate), a bar chart, and a ranked breakdown list. Clicking a KPI card switches both chart and breakdown to that metric. Dimension tabs (Service type, Vehicle type, Payment method, Employee) control the grouping. All rendering is client-side with no page reload. The API endpoint is `GET /api/v1/garages/me/analytics?tz=<timezone>`.
- **FR-005c**: System MUST provide a revenue trend endpoint `GET /api/v1/garages/me/trend?tz=<timezone>&days=<7|30>` returning daily data points (date, revenue, transaction count) for the requested range, with zero-fill for days with no orders.
- **FR-006**: System MUST provide an emergency close/open toggle on the dashboard that immediately stops new order intake across all apps.
- **FR-007**: System MUST display transaction history with columns: Time, License Plate, Owner Name, Phone, Services/Products Used, Amount, Status, Payment Method, Notes, Account.
- **FR-008**: System MUST support independent filtering of transactions by date range and by status.
- **FR-009**: System MUST support exporting filtered transaction data to Excel format.
- **FR-010**: System MUST display transaction totals grouped by payment method at the top of the report.
- **FR-011**: System MUST support creating products/services with fields: Name, Price, Description (rich text with images/video), Image, Category, Type (Product/Service), Inventory (products only), Brand, Status (Active/Paused).
- **FR-012**: System MUST support bulk import of products/services via an Excel template with downloadable template and upload with row-level error reporting.
- **FR-013**: System MUST enforce standardized image dimensions for product/service images. All product/service images are uploaded via file picker or drag-and-drop to object storage.
- **FR-027**: System MUST provide a reusable image upload component with drag-and-drop support, file preview, success indicator (visible for 2 seconds after upload), and client-side validation (max 10 MB, image MIME types only). The upload endpoint is `POST /api/v1/upload` accepting `multipart/form-data`.
- **FR-028**: System MUST delete the actual S3 object when an image is removed from the UI. The delete endpoint is `DELETE /api/v1/upload` accepting `{ "url": "..." }` in the request body. The delete button shows a loading state while the request is in progress.
- **FR-014**: System MUST manage customer records keyed primarily by license plate, with phone number as secondary key.
- **FR-015**: System MUST allow one phone number to be associated with multiple license plates (one owner, many vehicles).
- **FR-016**: System MUST prevent duplicate license plate registration unless the previous plate is marked "Car Sold".
- **FR-017**: System MUST support AI-based customer classification based on spending and visit patterns.
- **FR-018**: System MUST ensure customer data privacy between garages -- no data sharing across garage boundaries.
- **FR-019**: System MUST manage employee records using CCCD (national ID) as a unique, cross-garage portable identifier.
- **FR-020**: System MUST support 15 specific skill checkboxes for employee records.
- **FR-021**: System MUST support bulk employee import via Excel template.
- **FR-022**: System MUST allow garage owners to create bonbon GO login accounts for employees using their phone number as username.
- **FR-023**: System MUST only support soft-deactivation (not deletion) of employee accounts to preserve transaction history.
- **FR-024**: System MUST support showcase posts (text + images) that are published to the garage's public profile on the car owner web.
- **FR-025**: System MUST support ODO meter photo capture for extracting current vehicle kilometer reading.
- **FR-026**: System MUST support insurance (body, mandatory) and inspection expiry date tracking per vehicle, with a "same date" shortcut button for mandatory and body insurance.

### Key Entities

- **Garage**: Represents a car wash business. Key attributes: name, phone, address, coordinates, capacity, operating hours, bank QR, status (open/closed).
- **GarageOwner**: The account holder who manages a garage. Key attributes: phone (unique), password hash, profile link.
- **Product/Service**: Catalog item offered by a garage. Key attributes: name, price, description, category, type (product/service), inventory (products only), status, images.
- **Customer**: A car owner known to a specific garage. Key attributes: phone, name, date of birth, gender, occupation, long-term notes. Scoped per garage.
- **Vehicle**: A car registered at a garage. Key attributes: license plate (unique per garage context), brand, model, year, current KM, insurance dates, inspection date, notes, sold status. Belongs to a Customer.
- **Employee**: A garage staff member. Key attributes: CCCD (unique globally), name, phone, skills, education, certifications, industry start year, status.
- **EmployeeAccount**: Login credentials for bonbon GO. Key attributes: phone (username), password hash, active status, linked employee.
- **Transaction**: A completed service order. Key attributes: time, vehicle, customer, services used, amount, status, payment method, notes, account that created it.
- **ShowcasePost**: A portfolio entry. Key attributes: title, description, images, publish date, garage reference.

## Success Criteria

### Measurable Outcomes

- **SC-001**: Garage owners can complete registration (phone + OTP + password) in under 2 minutes.
- **SC-002**: Garage owners can fully configure their profile (all fields) in under 10 minutes on first setup.
- **SC-003**: Dashboard loads and displays all metrics within 3 seconds of navigation.
- **SC-004**: Transaction filtering and display completes within 2 seconds for up to 10,000 transaction records.
- **SC-005**: Excel export generates and downloads within 5 seconds for up to 10,000 rows.
- **SC-006**: Product/service bulk import processes 500 items within 30 seconds with complete error reporting.
- **SC-007**: Customer search returns results within 1 second across all search modes (phone, plate, name).
- **SC-008**: 90% of garage owners can add their first product/service without external help.
- **SC-009**: Emergency close toggle takes effect across all connected apps within 5 seconds.
- **SC-010**: System supports 500 concurrent garage accounts with no performance degradation on core operations.
