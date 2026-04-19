# Implementation Plan: BonBon Platform

**Branch**: `bonbon-platform` | **Date**: 2026-02-26 | **Specs**: [001](001-garage-web/spec.md), [002](002-bonbon-go-app/spec.md), [003](003-car-owner-web/spec.md), [004](004-order-lifecycle/spec.md), [005](005-zns-notifications/spec.md)

## Summary

BonBon is a platform connecting car owners with car wash garages in Vietnam, targeting 500 garages as the initial scale. The platform consists of three client applications: a garage management web portal, a mobile app for garage employees, and a customer-facing web app. This plan covers UI-first development with mocked backend APIs. Backend design is deferred to a follow-up phase.

## Technical Context

**Language/Version**: TypeScript 5.x across all projects
**Primary Dependencies**: Next.js 14+ (web apps), React Native / Expo SDK 52+ (mobile app), Tailwind CSS 4 (web styling)
**Storage**: Deferred (backend phase) -- UI uses mock data via MSW (Mock Service Worker) or static JSON fixtures
**Testing**: Vitest (unit), Playwright (E2E web), Detox (E2E mobile)
**Target Platform**: Web (garage.bonbon.com.vn, bonbon.com.vn), iOS 15+ and Android 10+ (bonbon GO)
**Project Type**: Multi-app monorepo (2 web apps + 1 mobile app + shared packages)
**Performance Goals**: < 2s page load (web on 4G), < 1s screen transitions (mobile), 60fps animations
**Constraints**: Must work on mid-range Android devices common in Vietnam, mobile-first responsive design for bonbon.com.vn
**Scale/Scope**: 500 garages, ~50 screens total across 3 apps

## Project Structure

### Documentation

```text
specs/
  001-garage-web/
    spec.md
    checklists/requirements.md
  002-bonbon-go-app/
    spec.md
    checklists/requirements.md
  003-car-owner-web/
    spec.md
    checklists/requirements.md
  004-order-lifecycle/
    spec.md
  005-zns-notifications/
    spec.md
  plan.md                          # This file
```

### Source Code

```text
bonbon/
  apps/
    garage-web/                     # Next.js -- garage.bonbon.com.vn
      src/
        app/                        # Next.js App Router pages
          (auth)/                   # Login, Register pages (public)
          (dashboard)/              # Protected layout with sidebar
            page.tsx                # Dashboard home
            transactions/
            products/
            customers/
            employees/
            accounts/
            showcase/
            profile/
        components/                 # Garage Web specific components
        hooks/                      # Custom hooks
        lib/                        # Utilities, API client, mock data
      public/
      tailwind.config.ts
      next.config.ts

    car-owner-web/                  # Next.js -- bonbon.com.vn
      src/
        app/
          status/[token]/           # Order status page (public, no auth required)
          (auth)/                   # OTP login
          (main)/                   # Protected layout
            profile/
            garage/[id]/            # Public garage profile
            marketplace/
            news/
        components/
        hooks/
        lib/
      public/
      tailwind.config.ts
      next.config.ts

    bonbon-go/                      # React Native (Expo) -- mobile app
      src/
        screens/
          LoginScreen.tsx
          OrderBoardScreen.tsx      # Main screen with order cards
          ScanPlateScreen.tsx       # Camera + OCR
          ServiceSelectScreen.tsx
          PaymentScreen.tsx
          OrderHistoryScreen.tsx
        components/                 # Reusable mobile components
          OrderCard.tsx
          StatusBadge.tsx
          ServiceList.tsx
        hooks/
        lib/
        navigation/                 # React Navigation config
      app.json
      eas.json

  packages/
    shared-types/                   # Shared TypeScript interfaces
      src/
        order.ts                    # Order, OrderStatus, OrderService
        garage.ts                   # Garage, GarageProfile
        customer.ts                 # Customer, Vehicle
        employee.ts                 # Employee, EmployeeAccount
        product.ts                  # Product, Service, Category
        notification.ts             # ZNSNotification, ZNSTemplate
        transaction.ts              # Transaction, PaymentMethod
        auth.ts                     # Auth tokens, session types
      package.json
      tsconfig.json

    ui-kit/                         # Shared design tokens
      src/
        tokens/
          colors.ts                 # Color palette
          typography.ts             # Font scales
          spacing.ts                # Spacing scale
        index.ts
      package.json

  turbo.json                        # Turborepo configuration
  package.json                      # Root workspace package.json
  tsconfig.base.json                # Shared TypeScript config
```

**Structure Decision**: Turborepo monorepo with 3 apps and 2 shared packages. This enables shared TypeScript interfaces between apps (critical for order lifecycle consistency) and shared design tokens. Each app is independently deployable.

## Screen Inventory

### Garage Web (garage.bonbon.com.vn) -- 14 screens

| # | Screen | Route | Spec User Story |
|---|--------|-------|-----------------|
| 1 | Registration | /register | US1 |
| 2 | Login | /login | US1 |
| 3 | Dashboard | / | US3 |
| 4 | Profile Setup | /profile | US2 |
| 5 | Transaction List | /transactions | US4 |
| 6 | Product/Service List | /products | US5 |
| 7 | Product/Service Form | /products/new, /products/[id]/edit | US5 |
| 8 | Product Import | /products/import | US5 |
| 9 | Customer List | /customers | US6 |
| 10 | Customer Detail | /customers/[id] | US6 |
| 11 | Vehicle Form | /customers/vehicles/new | US6 |
| 12 | Employee List | /employees | US7 |
| 13 | Employee Account Management | /accounts | US8 |
| 14 | ShowCase Management | /showcase | US9 |

### bonbon GO Mobile -- 6 screens

| # | Screen | Navigation | Spec User Story |
|---|--------|-----------|-----------------|
| 1 | Login | Stack: Login | US1 |
| 2 | Order Board (main) | Stack: Main/Board | US2, US3 |
| 3 | Scan License Plate | Modal over Board | US2 |
| 4 | Service Selection | Modal over Board | US2 |
| 5 | Payment | Modal over Board | US4 |
| 6 | Daily Order History | Modal over Board | US5 |

### Car Owner Web (bonbon.com.vn) -- 8 screens

| # | Screen | Route | Spec User Story |
|---|--------|-------|-----------------|
| 1 | Service Status | /status/[token] | US1 |
| 2 | Login (OTP) | /login | US2 |
| 3 | Car Owner Profile | /profile | US3 |
| 4 | Vehicle Detail | /profile/vehicles/[plate] | US3 |
| 5 | Garage Public Profile | /garage/[id] | US4 |
| 6 | Marketplace | /marketplace | US5 |
| 7 | News List | /news | US6 |
| 8 | News Article | /news/[slug] | US6 |

**Total: 28 screens**

## UI Component Library Strategy

Rather than building a full design system upfront, shared design tokens (colors, typography, spacing) live in `packages/ui-kit/`. Each app builds its own components using Tailwind CSS (web) or React Native StyleSheet (mobile) with those tokens.

### Shared Tokens

```text
Colors:     primary (#2563EB), secondary (#10B981), destructive (#EF4444),
            warning (#F59E0B), background, surface, text, muted
Typography: heading-1 through heading-4, body, body-small, caption
Spacing:    4px base unit, scale: 1, 2, 3, 4, 6, 8, 12, 16, 24, 32
Radius:     sm (4px), md (8px), lg (12px), xl (16px), full (9999px)
```

### Key Reusable Components Per App

**Garage Web**: DataTable (transactions, customers, employees), FilterBar, StatCard (dashboard metrics), RichTextEditor (product descriptions), FileUploader (images, Excel), MapPicker (Google Maps embed)

**bonbon GO**: OrderCard (draggable, expandable), StatusBadge (color-coded per status), CameraScanner (plate OCR), ServiceChip (quick-select service), ConfirmationPopup

**Car Owner Web**: StatusTracker (timeline visual), ServiceSuggestionCard (upsell), GarageCard (marketplace listing), ArticleCard (news), VehicleCard (profile)

## Mock Data Strategy

During UI-first development, all API calls are intercepted and served from mock data. This allows full UI development and testing without a backend.

**Approach**: MSW (Mock Service Worker) for web apps, custom fetch interceptor for React Native.

**Mock Data Files**: Located in each app's `src/lib/mocks/` directory with realistic Vietnamese data (names, phone numbers in +84 format, Vietnamese license plate formats, Vietnamese addresses, VND currency).

**API Contract**: Typed request/response interfaces in `packages/shared-types/`. These interfaces will be the contract for the future backend implementation.

## Implementation Phases

### Phase 1: Setup (Week 1)

**Goal**: Monorepo scaffold, shared packages, dev tooling

- Initialize Turborepo with pnpm workspaces
- Create `packages/shared-types/` with all entity interfaces from specs
- Create `packages/ui-kit/` with design tokens
- Scaffold `apps/garage-web/` with Next.js 14 (App Router, Tailwind CSS)
- Scaffold `apps/car-owner-web/` with Next.js 14 (App Router, Tailwind CSS)
- Scaffold `apps/bonbon-go/` with Expo (React Native, TypeScript)
- Configure ESLint, Prettier, TypeScript strict mode across all projects
- Set up MSW mock handlers skeleton for each app

### Phase 2: Garage Web MVP -- Auth + Dashboard + Profile (Weeks 2-3)

**Goal**: Garage owners can register, log in, see dashboard, and configure profile

**Screens**: Registration, Login, Dashboard, Profile Setup (4 screens)

- Registration page: phone input, password, confirm, captcha placeholder, OTP input
- Login page: phone, password, captcha placeholder
- Dashboard layout: sidebar navigation, stat cards (today/week/month), emergency close toggle
- Profile page: form fields, Google Maps embed for pin drop, image carousel upload (3 images), operating hours grid, QR code upload
- Mock data: 3 sample garages with varied metrics

### Phase 3: bonbon GO MVP -- Login + Receive Car + Payment (Weeks 3-4)

**Goal**: Employees can log in, scan plates, create orders, manage status, take payment

**Screens**: Login, Order Board, Scan Plate, Service Selection, Payment (5 screens)

- Login screen: phone + password fields
- Order Board: scrollable list of OrderCards with status badges and 4 action buttons, drag-to-reorder via long press
- Scan Plate screen: camera view with plate recognition overlay, retake button, manual edit
- Service Selection: top 10 frequent services grid, search bar, temporary service input
- Payment screen: 4 payment method buttons, QR code display for bank transfer
- Confirmation popup component (reused across all status transitions)
- Mock data: 10 sample orders in various statuses, 20 sample services

### Phase 4: Car Owner Web MVP -- Status Page (Weeks 4-5)

**Goal**: Car owners can follow a link and see real-time service status with upsell

**Screens**: Service Status, Login OTP (2 screens)

- Status page (public, no auth): order status timeline, service list, total, 2 upsell suggestions with "Add" button, login prompt popup
- OTP login: phone input, OTP input, auto-link history
- Real-time status simulation via mock WebSocket/polling
- Mock data: sample order with status transitions, 2 recommended services

### Phase 5: Garage Web P2 -- Products, Customers, Transactions (Weeks 5-7)

**Goal**: Full catalog management, customer CRM, financial history

**Screens**: Product List, Product Form, Product Import, Customer List, Customer Detail, Vehicle Form, Transaction List (7 screens)

- Product/Service CRUD: list with search/filter, create/edit form with rich text editor, image upload with dimension validation, Excel template download/upload
- Customer management: searchable list, detail view with spending/classification, linked vehicles, order history, long-term notes, "Car Sold" action
- Transaction history: DataTable with dual filters (date range + status), payment method summary header, Excel export button
- Mock data: 50 products/services, 100 customers with vehicles, 500 transactions

### Phase 6: bonbon GO P2 + Car Owner Web P2 (Weeks 7-9)

**Goal**: Complete remaining screens for both apps

**bonbon GO**: Daily Order History (1 screen)
- Summary section: revenue by payment type, vehicle type counts
- Detail table: today's orders for this employee

**Car Owner Web**: Profile, Garage Profile, Marketplace, News (4 screens)
- Car Owner Profile: personal info form, vehicle list with expiry date cards
- Garage Public Profile: hero images, info block, portfolio gallery, service catalog
- Marketplace: category nav, search, proximity sort placeholder, service/product cards
- News: article list with category filter, article detail with rich content, SEO meta tags

### Phase 7: Garage Web P3 -- Employees, Accounts, ShowCase (Weeks 9-10)

**Goal**: Complete all remaining Garage Web features

**Screens**: Employee List, Account Management, ShowCase (3 screens)

- Employee management: list with search, create/edit form with 15 skill checkboxes, Excel import
- Account management: create account from employee, search, deactivate (soft-delete)
- ShowCase: post list, create/edit post with images and text, preview as public page

### Phase 8: Polish and Cross-Cutting (Week 10-11)

- Responsive design audit for all web pages (mobile-first for bonbon.com.vn)
- Accessibility pass (ARIA labels, keyboard navigation, screen reader support)
- Loading states, error states, empty states for all screens
- Vietnamese localization review (all UI text in Vietnamese)
- Offline indicator and queued-action visual for bonbon GO
- Cross-app flow smoke testing with mock data
- Performance optimization (image lazy loading, code splitting, list virtualization)

## Dependencies and Execution Order

```text
Phase 1 (Setup)
  |
  +---> Phase 2 (Garage Web Auth + Dashboard)
  |       |
  |       +---> Phase 5 (Garage Web Products, Customers, Transactions)
  |               |
  |               +---> Phase 7 (Garage Web Employees, Accounts, ShowCase)
  |
  +---> Phase 3 (bonbon GO MVP)
  |       |
  |       +---> Phase 6a (bonbon GO Order History)
  |
  +---> Phase 4 (Car Owner Web Status)
          |
          +---> Phase 6b (Car Owner Web Profile, Marketplace, News)

Phase 8 (Polish) -- after all feature phases
```

Phases 2, 3, and 4 can run in parallel after Phase 1 (different apps, different codebases). Within each app, phases are sequential.

## Parallel Opportunities

With multiple developers:

- **Developer A**: Garage Web (Phases 2 -> 5 -> 7)
- **Developer B**: bonbon GO (Phases 3 -> 6a)
- **Developer C**: Car Owner Web (Phases 4 -> 6b)
- All developers: Phase 8 (Polish)

Phase 1 (Setup) should be done by one person, then all three app tracks can proceed independently.

## Key Technical Decisions

### Why Turborepo Monorepo

Shared TypeScript types are critical for order lifecycle consistency across 3 apps. Turborepo provides efficient builds with caching and minimal configuration overhead compared to Nx. Easy to hire for -- it is just pnpm workspaces with a build orchestrator.

### Why Next.js App Router

Server-side rendering for SEO (bonbon.com.vn news/marketplace) and fast initial loads. App Router is the current standard for new Next.js projects. Large ecosystem and hiring pool in Vietnam.

### Why Expo for React Native

Managed workflow reduces native build complexity. EAS Build handles iOS/Android builds without local Xcode/Android Studio setup. Camera and image APIs are well-supported. Expo SDK 52+ supports all required features (camera, file upload, notifications).

### Why MSW for Mocking

Industry standard for API mocking in frontend development. Works in both browser and Node.js. Mocks at the network level so the actual fetch/axios calls in the app code remain identical to production. Zero changes needed when switching from mock to real backend.

### Why Tailwind CSS

Utility-first approach speeds up UI development significantly. Built-in responsive design utilities. Consistent design tokens via configuration. Very large community and easy to hire for.

## Risk Register

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| License plate OCR accuracy on Vietnamese plates | High | Medium | Evaluate multiple OCR providers early. Build manual fallback prominently in UX. |
| Zalo ZNS template approval delays | High | Medium | Submit templates for approval in Week 1, parallel to development. Have fallback SMS provider. |
| React Native camera performance on low-end Android | Medium | Medium | Test on target devices from Week 3. Optimize camera resolution/frame rate. |
| Monorepo build times as codebase grows | Low | Low | Turborepo remote caching. Strict package boundaries. |
| Scope creep from Vietnamese market expectations | Medium | High | Specs are locked. Changes go through spec amendment process. |

## Notes

- All UI text must be in Vietnamese from the start (not English with i18n later)
- Currency is VND (no decimals, use dot separator for thousands: 150.000d)
- Phone numbers are Vietnamese format: 10 digits starting with 0 (display) or +84 (storage)
- License plate format: Vietnamese standard (e.g., 30A-123.45 for cars, 29-X1 123.45 for motorcycles)
- Date format: DD/MM/YYYY (Vietnamese convention)
- Time format: HH:mm (24-hour)
