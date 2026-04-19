# 010 — Auth Hardening (Enterprise Baseline)

**Created:** 2026-04-18
**Status:** Draft
**Depends on:** none
**Related:** [`docs/auth-design.md`](../../docs/auth-design.md), memory `project_auth_audit.md`
**Explicit non-goal for this spec:** Removing the `"555555"` OTP bypass in `backend/internal/usecase/auth/service.go:212`. That change is deferred and tracked separately.

## Context

The BonBon platform authenticates three actors (car owner, garage owner, employee) with JWT-based sessions backed by Redis. An audit (see `docs/auth-design.md`) identified gaps that block enterprise use:

- OTP code generated with `math/rand` (not CSPRNG).
- No rate limiting on `/auth/*` endpoints.
- 24-hour access tokens with no refresh mechanism and no revocation on password change.
- Client-side JWT stored in browser `localStorage` (XSS-reachable); mobile stores in memory (no persistence).
- No structured auth audit log.
- Default `JWT_SECRET` accepted even in production.

This spec defines the phased hardening plan to close those gaps while preserving current UX and API shapes where possible.

## User Stories

### US-1 — As a **car owner**, my OTP is unguessable
I want the 6-digit OTP I receive to be generated with a cryptographically secure RNG so that attackers can't predict future codes by observing prior ones.

### US-2 — As any **actor**, my account is protected from brute force
I want repeated wrong passwords or wrong OTPs from the same IP or against the same phone to be throttled so an attacker can't try millions of combinations.

### US-3 — As a **garage owner or employee**, I stay logged in across days without a long-lived access token
I want my app to stay signed in for weeks, but the access token used on each request should expire in minutes so a stolen token has a short blast radius.

### US-4 — As any **actor**, logging out and changing my password really ends every session
I want logout and password change to invalidate *all* existing sessions for my account, not just wait for the 24-hour TTL.

### US-5 — As a **car owner / garage owner** using the web app, my token can't be stolen by a malicious script
I want my refresh token stored in an httpOnly cookie so JavaScript on the page can't read it.

### US-6 — As a **car owner** using the mobile app, I don't re-login every time I open the app
I want my session persisted in the platform's secure keystore so the app resumes my session after restart.

### US-7 — As a **platform operator**, I can audit authentication activity
I want every login success, login failure, OTP request, OTP verify, logout, and authz-denied event to be logged as a structured event with account ID, role, IP, UA, and request correlation ID.

### US-8 — As a **platform operator**, production refuses to boot with an unsafe JWT secret
I want the backend to fail fast at startup in `env=production` if `JWT_SECRET` is missing, default, or shorter than 32 bytes.

### US-9 — As a **garage owner**, I can recover from a forgotten password
I want an OTP-gated password reset flow so I don't lose access to my business data.

## Acceptance Scenarios

### Scenario 1: CSPRNG OTP
**Given** a car owner requests an OTP
**When** the backend generates the 6-digit code
**Then** it is produced via `crypto/rand.Int` (not `math/rand`) and is still 6 digits, 5-minute TTL

### Scenario 2: IP rate limit on login
**Given** the same IP sends 11 `POST /auth/login/garage` requests inside 60 seconds
**When** the 11th request arrives
**Then** the backend responds `429 Too Many Requests` with `Retry-After`, and emits an `auth.ratelimit.ip` audit event

### Scenario 3: Per-phone OTP limit
**Given** the same phone has received 5 OTP codes in the last 15 minutes
**When** a 6th `POST /auth/otp/request` arrives for that phone
**Then** the request is rejected with `429` and no new OTP is stored in Redis

### Scenario 4: OTP verify attempt cap
**Given** a stored OTP for phone X
**When** the 6th `POST /auth/otp/verify` with a wrong code arrives
**Then** the stored OTP is deleted, further verifies return `401 invalid_or_expired`, and the user must request a new OTP
**And** the `"555555"` dev bypass still works (out of scope for this spec)

### Scenario 5: Short-lived access + refresh rotation
**Given** a user just logged in
**When** they call a protected endpoint 20 minutes later with the original access token
**Then** the API returns `401 token_expired`
**And** the client calls `POST /auth/refresh` with its refresh token
**Then** the API returns a new access token (15-min TTL) and a new refresh token; the old refresh token is deleted
**And** a second use of the old refresh token revokes the entire session chain for the account

### Scenario 6: Logout revokes session
**Given** a user is logged in with access token A and refresh token R
**When** they call `POST /auth/logout`
**Then** R is deleted from Redis and subsequent `POST /auth/refresh` with R returns `401`

### Scenario 7: Password change revokes all sessions
**Given** a garage owner has three active sessions across devices
**When** they change their password
**Then** all three refresh tokens are deleted from Redis and every device is forced back to login

### Scenario 8: httpOnly refresh cookie on web
**Given** a successful login via garage-web or car-owner-web
**When** the browser receives the response
**Then** `Set-Cookie: bonbon_refresh=...; HttpOnly; Secure; SameSite=Strict; Path=/auth` is present
**And** `document.cookie` in devtools does not expose that cookie

### Scenario 9: Mobile persists session
**Given** a car owner has logged in on bonbon-go
**When** they kill and relaunch the app
**Then** the app silently refreshes via the refresh token stored in `expo-secure-store` and lands on the logged-in screen

### Scenario 10: Auth audit log
**Given** any of login success/failure, otp request/verify, logout, role denied
**When** the event occurs
**Then** a single structured log line is emitted with `evt`, `account_id` (if known), `role`, `ip`, `ua`, `session_id`, `rid` fields
**And** the event is keyed so an operator can filter by `evt=auth.login.failure`

### Scenario 11: Boot-time secret check
**Given** `APP_ENV=production`
**When** the backend starts with `JWT_SECRET` unset, equal to the dev default, or shorter than 32 bytes
**Then** the process panics at startup with a clear error message and does not serve traffic

### Scenario 12: Password reset for garage owner
**Given** a garage owner has forgotten their password
**When** they call `POST /auth/reset/request { phone }` and then `POST /auth/reset/confirm { phone, otp, new_password }`
**Then** the password is replaced, all existing sessions are revoked, and they can log in with the new password

## Out of Scope (tracked separately)

- Removing the `"555555"` OTP bypass — deferred per product request.
- TOTP / 2FA for garage owners.
- Device fingerprinting and new-device alerts.
- Migrating JWT signing to RS256 + JWKS rotation.
- Admin / super-user role.
- GDPR export/delete endpoints.

## Risks

1. **Refresh rotation must be atomic.** A non-atomic swap leaves a window where both tokens are valid or neither is — both break UX. Implementation must use Redis Lua or `WATCH/MULTI/EXEC`.
2. **Mobile secure-store migration.** Existing users have in-memory tokens that will be lost on deploy — first launch after upgrade re-prompts login. Acceptable.
3. **Cookie-based refresh on web requires CSRF posture review.** Refresh endpoint should accept the cookie only for `POST` with a custom header (double-submit not needed if `SameSite=Strict`).
4. **Rate-limit false positives behind NAT.** Cafes, offices share an IP. Per-IP limits must be generous enough (60/min per IP for `/auth/*`) and per-phone/per-account limits do the heavy lifting.
