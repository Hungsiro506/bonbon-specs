# 010 — Auth Hardening: Implementation Plan

**Goal:** Take the current JWT-in-Redis auth to an enterprise baseline — CSPRNG OTP, layered rate limiting, short-lived access + rotating refresh, full session revocation, structured audit log, safe secrets, secure client storage — without removing the `"555555"` OTP bypass (deferred).

**Architecture conventions (from `docs/developer-guide.md`):**

- Clean architecture: `domain → repo → usecase → handler`, dependencies inward.
- Contracts in `internal/repo/contracts.go`, `internal/usecase/contracts.go`.
- Wiring in `internal/app/app.go`.
- Numbered, paired migrations — never edit applied ones.
- No inline comments; docstrings on exported symbols only.
- Phone normalization in `pkg/phone` at the service layer.
- E2E naming: `*.spec.ts` (mock) vs `*.integration.spec.ts` (real backend).
- Seed creds: garage `0901000001 / owner123`, employee `0901000002 / emp123`, car owner `0912000001` (OTP).

## Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Access token TTL | 15 min | Short enough to bound theft, long enough to avoid chatty refreshes |
| Refresh token TTL | 30d idle, 90d absolute | Matches mobile expectation of staying signed in |
| Refresh transport | `httpOnly; Secure; SameSite=Strict` cookie on web; `expo-secure-store` on mobile | XSS-resistant on web; keystore-backed on mobile |
| Refresh shape | Opaque token (256-bit random), not JWT | Enables atomic rotation and revocation via Redis |
| Refresh rotation | On every use; reuse detection revokes entire chain | Standard OWASP guidance |
| Signing algorithm | Keep HS256 for now, `kid` claim added | RS256/JWKS is out of scope for this spec |
| Rate-limit store | Redis, sliding window via Lua | Single source of truth, works across replicas |
| Rate-limit keys | `rl:ip:{ip}:{bucket}`, `rl:phone:{phone}:{bucket}`, `rl:acct:{account_id}:{bucket}` | Layered defense |
| Session storage | One Redis key per session: `session:{session_id}` + index `sessions:{account_id}` | Enables bulk revoke on password change |
| OTP attempt cap | 5 wrong verifies → delete stored OTP | Prevents 1M-space brute force |
| Password reset | OTP-gated, same primitive as car-owner login | Reuse existing OTP delivery |
| Audit log sink | Existing structured logger (`log.InfoCtx/WarnCtx`) | No new infra; grep-able |
| CSPRNG OTP | `crypto/rand.Int(rand.Reader, big.NewInt(1_000_000))` | Replace `math/rand.Intn` |
| Boot-time secret check | `APP_ENV=production` + `JWT_SECRET` length ≥ 32 bytes, not equal to known dev default | Fail fast |
| Backwards compat | Old access-token-only clients keep working during Phase 1 rollout by treating their 24h token as both access + refresh behind a feature flag | Zero-downtime rollout |

## Backend Architecture Mapping

Following the layer order from `developer-guide.md#adding-a-new-feature`:

### Domain (`backend/internal/domain/`)

- New types:
  - `domain/session.go`: `Session{ID, AccountID, Role, GarageID, EmployeeID, DeviceID, IP, UA, CreatedAt, LastUsedAt, AbsoluteExpiresAt}`
  - `domain/auth.go`: add `RefreshTokenRequest`, `RefreshTokenResponse`, `ResetRequest`, `ResetConfirm`, plus validators.
  - `domain/errors.go`: add sentinels — `ErrRateLimited`, `ErrRefreshReused`, `ErrResetOTPInvalid`, `ErrSecretMisconfigured`.
- Strengthen password policy: `ValidatePassword` requires ≥ 10 chars and rejects top-N commons.

### Repo (`backend/internal/repo/`)

- `contracts.go`: add `SessionRepo`:
  ```go
  type SessionRepo interface {
      Create(ctx, s Session) error
      Get(ctx, id string) (Session, error)
      Rotate(ctx, oldID, newID string, newTTL time.Duration) error     // atomic via Lua
      Delete(ctx, id string) error
      DeleteAllForAccount(ctx, accountID uuid.UUID) (int, error)
      MarkReused(ctx, id string) error                                  // trips chain revocation
  }
  ```
- Implementation: `repo/redis/session/repo.go`. Keys:
  - `session:{session_id}` → JSON of Session
  - `sessions:{account_id}` → SET of session_ids (for bulk revoke)
  - `session:used:{session_id}` → tombstone with short TTL for reuse detection
- Add `RateLimitRepo` (Redis sliding-window) in `repo/redis/ratelimit/repo.go`.
- Add `AuthAuditRepo` — thin wrapper over the logger so handlers/usecases have a contract, not a direct logger call (keeps domain free of logger dependency).

### Use case (`backend/internal/usecase/auth/`)

- Split `auth/service.go` if it grows too large — introduce `auth/session_service.go` for refresh/rotation logic.
- New methods:
  - `Refresh(ctx, refreshToken, deviceID, ip, ua) (access, refresh, error)`
  - `LogoutAll(ctx, accountID)`
  - `ChangePassword(ctx, accountID, old, new)` — revokes sessions after swap
  - `RequestReset(ctx, phone)` / `ConfirmReset(ctx, phone, otp, newPassword)`
- Every login path now creates a Session row and emits `auth.login.success`.
- OTP verify uses `crypto/rand`; attempt counter stored at `otp:attempts:{phone}`.
- `"555555"` bypass **remains untouched** in this spec.

### Handler (`backend/internal/handler/http/`)

- Middleware additions (`middleware/`):
  - `authratelimit.go` → `AuthIPLimiter(r RateLimitRepo)`, `AuthPhoneLimiter(r)`, `AuthAccountLimiter(r)`.
  - Update `auth.go` → `AuthRequired` now re-reads `session:{jti}` and rejects if missing (restores behavior that was reverted per memory).
- Handler additions (`v1/auth_handler.go`):
  - `POST /auth/refresh`
  - `POST /auth/logout/all`
  - `POST /auth/password/change`
  - `POST /auth/reset/request`, `POST /auth/reset/confirm`
- Cookie helpers (`v1/cookie.go`): centralized `setRefreshCookie`, `clearRefreshCookie` with `httpOnly; Secure; SameSite=Strict; Path=/api/v1/auth`.

### Router (`backend/internal/handler/http/v1/router.go`)

Re-apply the hardening reverted on `feat/authorization` (see memory `project_auth_audit.md`) plus new limits:

```go
authGroup := app.Group("/auth", middleware.AuthIPLimiter(rl))
authGroup.Post("/register",      authPhoneLimit, authHandler.Register)
authGroup.Post("/login/garage",  authPhoneLimit, authHandler.LoginGarage)
authGroup.Post("/login/employee",authPhoneLimit, authHandler.LoginEmployee)
authGroup.Post("/otp/request",   authPhoneLimit, authHandler.RequestOTP)
authGroup.Post("/otp/verify",    authPhoneLimit, authHandler.VerifyOTP)
authGroup.Post("/refresh",       authHandler.Refresh)
authGroup.Post("/logout",        middleware.AuthRequired(jwt, sessions), authHandler.Logout)
authGroup.Post("/logout/all",    middleware.AuthRequired(jwt, sessions), authHandler.LogoutAll)
authGroup.Post("/password/change", middleware.AuthRequired(jwt, sessions), authHandler.ChangePassword)
authGroup.Post("/reset/request", authPhoneLimit, authHandler.RequestReset)
authGroup.Post("/reset/confirm", authPhoneLimit, authHandler.ConfirmReset)

// Re-apply role guards:
productsGroup.Use(middleware.RoleRequired("garage_owner")) // writes
employeesGroup.Use(middleware.RoleRequired("garage_owner"))
showcaseGroup.Use(middleware.RoleRequired("garage_owner"))
```

`SetupRouter(... , rl RateLimitRepo, sessions SessionRepo, ...)` gains new dependencies wired in `app.go`.

### Config (`backend/config/config.go`)

- Remove the dev-default value for `JWT_SECRET` — or keep the default but add boot-time guard:
  ```go
  if cfg.App.Env == "production" {
      if len(cfg.JWT.Secret) < 32 || cfg.JWT.Secret == "bonbon-dev-secret-change-me" {
          return nil, fmt.Errorf("JWT_SECRET must be ≥32 bytes in production")
      }
  }
  ```
- New keys:
  - `JWT_ACCESS_TTL` default `15m`
  - `JWT_REFRESH_TTL` default `720h`
  - `AUTH_RL_IP_PER_MIN` default `60`
  - `AUTH_RL_PHONE_PER_15MIN` default `5`
  - `AUTH_RL_ACCOUNT_FAIL_PER_15MIN` default `10`
  - `COOKIE_DOMAIN` for cross-subdomain cookies (`.bonbon.com.vn`)

### Migrations

No schema change is required if sessions live in Redis only. If we want durable audit, add an append-only table:

```
migrations/000010_auth_audit_log.up.sql
migrations/000010_auth_audit_log.down.sql
```

(Optional — default to logger-only for Phase 1.)

## Frontend Architecture Mapping

### `apps/web` (unified Next.js)

- `src/lib/api.ts` — new `authenticatedFetch` wrapper:
  - On `401 token_expired`, call `POST /auth/refresh` once, retry original request, else bounce to login.
  - Drop `localStorage` for refresh; access token lives in a React context only (`AuthProvider`).
- `src/lib/auth.tsx` (garage and car-owner variants) — add `refresh()` method called from mount.
- Middleware for cross-subdomain cookie: set cookie domain `.bonbon.com.vn`.

### `apps/bonbon-go` (React Native Expo)

- Add dependency `expo-secure-store`.
- `src/lib/api.ts`:
  - On login: persist `{refresh, access_expires_at}` via `SecureStore.setItemAsync("bonbon_refresh", ...)`.
  - On app launch: if refresh present, call `/auth/refresh` before rendering the gated tree.
  - Memory-only access token.

### Integration test updates (`e2e/`)

Following the convention in `developer-guide.md#e2e-test-structure`:

- `e2e/garage-web/auth.integration.spec.ts` — add cases for refresh, logout, logout-all, password change revocation.
- `e2e/car-owner-web/auth.integration.spec.ts` — add OTP attempt-cap case.
- `e2e/bonbon-go/auth.integration.spec.ts` — add session-persistence-after-restart case.
- Mock specs (`*.spec.ts`) — update to stub the new 15-min expiry + refresh call.

## Rollout Sequence

| Phase | Gate | Contents | Safety |
|---|---|---|---|
| **0** | land on `auth-standardize` branch | Per-env boot check on `JWT_SECRET`, structured audit log skeleton, CSPRNG OTP, IP rate limit on `/auth/*`. No client-visible changes. | Feature-flagged; can disable limits by config |
| **1** | behind flag `AUTH_REFRESH_V2` | Session table in Redis, `/auth/refresh`, access TTL 15m, cookie on web, secure-store on mobile. Old flow still works when flag off. | Dual-write sessions during rollout |
| **2** | default on | Logout-all, password-change revocation, reset flow, per-phone and per-account limits. | Retain 24h legacy tokens as fallback for 1 week |
| **3** | cleanup | Remove legacy 24h JWT path, drop the feature flag. | Post-soak |
| **later** | separate spec | Remove `"555555"`, TOTP, RS256 + JWKS, admin role, new-device notifications. | Explicit product sign-off |

## Test Plan

- **Unit (high coverage)**
  - `domain/session_test.go` — validation, expiry math.
  - `domain/auth_test.go` — password policy, reset payload validation.
  - `pkg/crypto` — OTP generator is uniform over `[000000, 999999]`.
- **Use case (medium)**
  - `usecase/auth/session_service_test.go` — rotation happy path, reuse detection revokes chain, password change deletes all sessions.
  - `usecase/auth/service_test.go` — OTP attempt cap, reset confirm clears sessions.
- **Middleware**
  - Table-driven test for each rate-limit bucket under concurrent hits.
  - `AuthRequired` rejects when session row is missing even if JWT is valid.
- **Integration E2E** (per `developer-guide.md` — `./run.sh test:integration`)
  - Garage-web: login → wait 16 min → call protected endpoint → expect silent refresh succeeds.
  - Car-owner mobile: login → kill app → relaunch → land on home (secure-store persisted).
  - Brute force: 11 wrong passwords → 429 + audit event.
  - Logout-all: two sessions → logout-all → both refreshes 401.
- **Security-review checkpoint** before Phase 2 default-on.

## Observability

Every auth event emits one structured log line:

```
evt=auth.login.success account_id=... role=garage_owner ip=... ua=... session_id=... rid=...
evt=auth.login.failure reason=bad_password phone=+84... ip=... rid=...
evt=auth.otp.request phone=+84... ip=... rid=...
evt=auth.otp.verify.success account_id=... phone=+84... session_id=... rid=...
evt=auth.otp.verify.failure reason=mismatch|expired|attempts_exceeded phone=+84... rid=...
evt=auth.refresh.success account_id=... session_id=... rid=...
evt=auth.refresh.reused account_id=... session_id=... rid=...   # triggers chain revoke
evt=auth.logout account_id=... session_id=... rid=...
evt=auth.logout_all account_id=... count=N rid=...
evt=auth.password.change account_id=... rid=...
evt=auth.reset.request phone=+84... ip=... rid=...
evt=auth.reset.confirm.success account_id=... rid=...
evt=authz.denied account_id=... role=... route=... reason=role|tenant rid=...
evt=auth.ratelimit.ip  ip=... route=... rid=...
evt=auth.ratelimit.phone phone=+84... route=... rid=...
```

Dashboards (future work, not in this spec) can count `evt=auth.login.failure` by IP to feed a WAF block list.

## Success Criteria

1. All 12 acceptance scenarios in `spec.md` pass in integration E2E.
2. `./run.sh test:all` green.
3. No regression in existing flows (login, order create by employee, etc.).
4. Production cannot boot with a missing/default `JWT_SECRET`.
5. Audit events are visible in `./run.sh logs` for every auth-touching request.
6. `"555555"` dev bypass remains functional and is explicitly flagged for removal in a follow-up ticket.
