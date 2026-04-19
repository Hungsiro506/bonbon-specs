# Technical Specification: Unified Web App (apps/web)

**Created**: 2026-04-17
**Status**: Implemented
**Supersedes**: Deployment architecture of `apps/garage-web` and `apps/car-owner-web` (source code only — business specs 001 and 003 remain authoritative for feature requirements)

## Context

The BonBon platform has two web-facing products:

- **Garage Web** (`garage.bonbon.com.vn`) — Dashboard for garage owners to manage their business: analytics, products/services, customers, employees, transactions, and showcase portfolio.
- **Car Owner Web** (`bonbon.com.vn`) — Consumer app for car owners: real-time service tracking via ZNS links, OTP login, vehicle profiles, marketplace, and editorial content.

These two products were originally implemented as separate Next.js applications (`apps/garage-web` and `apps/car-owner-web`). They have been consolidated into a single Next.js application at `apps/web` to reduce infrastructure and maintenance overhead while preserving full feature isolation.

## Important: Business Identity Unchanged

From a product, user, and domain perspective, **Garage Web and Car Owner Web remain distinct products**. They have:

- Different target users (garage owners vs car owners)
- Different authentication flows (password vs OTP)
- Different UI layouts (desktop sidebar vs mobile bottom nav)
- Different feature sets (admin CRUD vs consumer tracking)
- Different subdomains (`gara.bonbon.com.vn` vs `bonbon.com.vn`)

The merge is purely a **source code and deployment optimization**. Business specs [001-garage-web](../001-garage-web/spec.md) and [003-car-owner-web](../003-car-owner-web/spec.md) remain the authoritative feature specifications. When implementing new features, continue to think of them as separate products that happen to share a codebase.

## Architecture

### Subdomain Routing via Middleware

A single Next.js app serves both products. Middleware (`src/middleware.ts`) inspects the `Host` header and rewrites the URL path:

```
gara.bonbon.com.vn/login     → /garage/login     → app/garage/(auth)/login/page.tsx
bonbon.com.vn/login           → /car-owner/login  → app/car-owner/(auth)/login/page.tsx
order.bonbon.com.vn/status/x  → /car-owner/status/x
```

In local development, the `NEXT_PUBLIC_PORTAL` env var overrides subdomain detection:

```bash
NEXT_PUBLIC_PORTAL=garage pnpm --filter web dev    # force garage portal
NEXT_PUBLIC_PORTAL=car-owner pnpm --filter web dev # force car-owner portal
```

Without the env var, the middleware defaults to `car-owner` for non-garage hostnames.

### Directory Structure

```
apps/web/
├── src/
│   ├── middleware.ts                  # Subdomain → path rewrite
│   ├── app/
│   │   ├── layout.tsx                 # Root: font, lang, globals.css
│   │   ├── not-found.tsx              # Global 404
│   │   │
│   │   ├── car-owner/                 # All car-owner routes (bonbon.com.vn)
│   │   │   ├── layout.tsx             # CarOwnerAuthProvider + i18n
│   │   │   ├── page.tsx               # Redirect → /status
│   │   │   ├── error.tsx
│   │   │   ├── (auth)/
│   │   │   │   └── login/page.tsx     # OTP login
│   │   │   ├── (main)/               # BottomNav layout
│   │   │   │   ├── status/page.tsx
│   │   │   │   ├── profile/page.tsx
│   │   │   │   ├── profile/vehicles/[plate]/page.tsx
│   │   │   │   ├── marketplace/page.tsx
│   │   │   │   ├── garage/[id]/page.tsx
│   │   │   │   └── news/...
│   │   │   └── status/[token]/page.tsx  # Public (no auth)
│   │   │
│   │   └── garage/                    # All garage routes (gara.bonbon.com.vn)
│   │       ├── layout.tsx             # GarageAuthProvider + i18n
│   │       ├── not-found.tsx
│   │       ├── (auth)/
│   │       │   ├── login/page.tsx     # Password login
│   │       │   └── register/page.tsx
│   │       └── (dashboard)/           # Sidebar layout
│   │           ├── page.tsx           # Dashboard overview
│   │           ├── products/...
│   │           ├── customers/...
│   │           ├── employees/...
│   │           ├── transactions/...
│   │           ├── profile/page.tsx
│   │           ├── accounts/page.tsx
│   │           └── showcase/page.tsx
│   │
│   ├── components/
│   │   ├── Providers.tsx              # Shared: I18nProvider
│   │   ├── car-owner/
│   │   │   └── BottomNav.tsx          # Mobile bottom navigation
│   │   └── garage/
│   │       ├── Sidebar.tsx            # Desktop sidebar navigation
│   │       ├── DataTable.tsx          # Generic data table
│   │       ├── StatCard.tsx           # KPI metric card
│   │       ├── ImageUpload.tsx        # Drag-drop image upload
│   │       ├── RevenueTrendChart.tsx   # Chart.js line chart
│   │       └── AnalyticsDashboard.tsx  # Chart.js bar + breakdown
│   │
│   └── lib/
│       ├── api.ts                     # Unified API client
│       ├── auth/
│       │   ├── index.ts               # Barrel re-exports (backward compat aliases)
│       │   ├── garage.tsx             # GarageAuthProvider (password + redirect)
│       │   └── car-owner.tsx          # CarOwnerAuthProvider (OTP)
│       ├── mock-data/
│       │   ├── garage.ts             # Mock data for garage portal
│       │   └── car-owner.ts          # Mock data for car-owner portal
│       ├── analytics.ts              # Chart data types and formatters
│       └── transactions.ts           # Order filtering, CSV export
```

### Token Isolation

Each portal uses a separate `localStorage` key to avoid token conflicts:

| Portal | Token Key | Set By |
|--------|-----------|--------|
| Garage | `bonbon_token` | `GarageAuthProvider` calls `setActiveTokenKey("bonbon_token")` |
| Car Owner | `bonbon_car_owner_token` | `CarOwnerAuthProvider` calls `setActiveTokenKey("bonbon_car_owner_token")` |

The API client (`lib/api.ts`) uses a module-level `activeTokenKey` that each auth provider sets on mount. This means the API client automatically uses the correct token for the active portal.

### Auth Provider Isolation

Each portal has its own auth context. They never share state:

- `useGarageAuth()` — for garage pages (also aliased as `useAuth()` for backward compat)
- `useCarOwnerAuth()` — for car-owner pages

The barrel file at `lib/auth/index.ts` re-exports both, including backward-compatible aliases so existing page imports (`import { useAuth } from "@/lib/auth"`) continue to work.

## Infrastructure Mapping

### Before (2 services)

```
gara.bonbon.com.vn     → Caddy → garage-web:3001    (Docker container)
car-owner.bonbon.com.vn → Caddy → car-owner-web:3002 (Docker container)
order.bonbon.com.vn     → Caddy → car-owner-web:3002
```

### After (1 service)

```
gara.bonbon.com.vn     → Caddy → web:25501  → middleware → /garage/*
car-owner.bonbon.com.vn → Caddy → web:25501  → middleware → /car-owner/*
order.bonbon.com.vn     → Caddy → web:25501  → middleware → /car-owner/*
```

Benefits:
- 1 Docker image instead of 2 (smaller deploy, faster CI)
- 1 port instead of 2 (simpler firewall rules)
- Shared bundle optimization (common dependencies loaded once)
- Single build pipeline

## How to Add a New Feature

### For Garage Portal (spec 001)

1. Create page at `apps/web/src/app/garage/(dashboard)/your-feature/page.tsx`
2. Add navigation link in `components/garage/Sidebar.tsx`
3. Use `useGarageAuth()` for auth context
4. Add mock data in `lib/mock-data/garage.ts`

### For Car Owner Portal (spec 003)

1. Create page at `apps/web/src/app/car-owner/(main)/your-feature/page.tsx`
2. Add navigation link in `components/car-owner/BottomNav.tsx`
3. Use `useCarOwnerAuth()` for auth context
4. Add mock data in `lib/mock-data/car-owner.ts`

### Shared Code

Components or utilities used by both portals go in:
- `components/` (root, not under `garage/` or `car-owner/`)
- `lib/` (e.g., shared API helpers, formatting utilities)
- `packages/shared-types/` (TypeScript interfaces)
- `packages/ui-kit/` (design tokens)

## Development Commands

```bash
# Run unified web app (middleware routes by Host header)
pnpm dev:web

# Force a specific portal in local dev
pnpm dev:web:garage     # NEXT_PUBLIC_PORTAL=garage
pnpm dev:web:owner      # NEXT_PUBLIC_PORTAL=car-owner

# Build
pnpm --filter web build

# Unit tests
pnpm --filter web test

# E2E tests
./run.sh test:e2e:web        # both portals
./run.sh test:e2e:garage     # garage portal only
./run.sh test:e2e:owner      # car-owner portal only
```

## Migration Notes

The original `apps/garage-web/` and `apps/car-owner-web/` directories are retained temporarily for reference. They are no longer built or deployed. All new development happens in `apps/web/`.

To fully remove the old apps:
1. Delete `apps/garage-web/` and `apps/car-owner-web/`
2. Run `pnpm install` to update the lockfile
3. Verify `pnpm build` passes
