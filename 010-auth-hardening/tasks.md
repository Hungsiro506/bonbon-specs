# 010 — Auth Hardening: Task Streams

Streams are grouped so they can be picked up by separate engineers or dispatched across PRs. Each task cites the file or directory per `developer-guide.md` conventions.

Legend: `[ ]` not started · `[~]` in progress · `[x]` done

## Stream A — Foundations (no client impact)

- [ ] **A01** `config/config.go`: add `JWT_ACCESS_TTL`, `JWT_REFRESH_TTL`, `AUTH_RL_IP_PER_MIN`, `AUTH_RL_PHONE_PER_15MIN`, `AUTH_RL_ACCOUNT_FAIL_PER_15MIN`, `COOKIE_DOMAIN`, `AUTH_REFRESH_V2` flag. Update `backend/.env.example` and `docs/quickstart.md` env table.
- [ ] **A02** `config/config.go`: boot-time guard — panic if `APP_ENV=production` and `JWT_SECRET` is empty, < 32 bytes, or equals the dev default.
- [ ] **A03** `backend/internal/usecase/auth/service.go:198`: swap `math/rand.Intn` → `crypto/rand.Int`. Add unit test in `pkg/crypto` (or dedicated `pkg/otp`) proving uniform distribution over `[000000, 999999]`.
- [ ] **A04** `backend/pkg/auth/jwt.go`: add optional `kid` claim; no algorithm change.
- [ ] **A05** Domain: sharpen `ValidatePassword` in `internal/domain/auth.go` to ≥ 10 chars and a top-N common-password reject list (embed as `pkg/passwords/common.txt` via `go:embed`).
- [ ] **A06** `internal/domain/errors.go`: add `ErrRateLimited`, `ErrRefreshReused`, `ErrResetOTPInvalid`, `ErrSecretMisconfigured`.

## Stream B — Rate limiting

- [ ] **B01** `internal/repo/contracts.go`: add `RateLimitRepo` interface (sliding window).
- [ ] **B02** `internal/repo/redis/ratelimit/repo.go`: Redis sliding-window implementation using Lua for atomicity.
- [ ] **B03** `internal/handler/http/middleware/authratelimit.go`: `AuthIPLimiter`, `AuthPhoneLimiter`, `AuthAccountLimiter` returning `fiber.Handler`. Emits `auth.ratelimit.*` audit events.
- [ ] **B04** `internal/handler/http/v1/router.go`: wire IP limiter on `/auth/*` group; phone limiter on each mutating auth endpoint. Re-apply role guards on `products` (writes), `employees`, `showcase` — re-land the work reverted on `feat/authorization` per memory.
- [ ] **B05** `internal/app/app.go`: construct `RateLimitRepo`, pass to `SetupRouter`.
- [ ] **B06** Tests: table-driven for each limiter + router-level integration test proving 429 and `Retry-After` header.

## Stream C — Session ledger + refresh tokens

- [ ] **C01** `internal/domain/session.go`: `Session` struct, validators, TTL helpers.
- [ ] **C02** `internal/repo/contracts.go`: add `SessionRepo`.
- [ ] **C03** `internal/repo/redis/session/repo.go`: implementation with Lua-backed `Rotate` and `DeleteAllForAccount`. Keys: `session:{id}`, `sessions:{account_id}`, `session:used:{id}`.
- [ ] **C04** `internal/usecase/auth/session_service.go`: new use case orchestrating create / rotate / revoke. Reuse detection: if `session:used:{id}` exists, delete entire set for account and return `ErrRefreshReused`.
- [ ] **C05** `internal/usecase/auth/service.go`: every login path creates a `Session` row; JWT includes `session_id` claim.
- [ ] **C06** `internal/handler/http/middleware/auth.go`: `AuthRequired` re-reads `session:{id}` and rejects if missing (restore reverted behavior).
- [ ] **C07** `internal/handler/http/v1/auth_handler.go`: add `POST /auth/refresh`, `POST /auth/logout/all`.
- [ ] **C08** `internal/handler/http/v1/cookie.go`: `setRefreshCookie`/`clearRefreshCookie` helpers (`HttpOnly; Secure; SameSite=Strict; Path=/api/v1/auth`, domain from config).
- [ ] **C09** `internal/handler/http/v1/router.go`: register new endpoints, guard `/auth/refresh` with IP limiter but not `AuthRequired`.
- [ ] **C10** Tests: rotation happy path, reuse detection revokes chain, session repo under concurrent rotate.

## Stream D — Password change & reset

- [ ] **D01** `internal/usecase/auth/service.go`: `ChangePassword(accountID, old, new)` — validate, bcrypt, update, revoke all sessions, emit `auth.password.change`.
- [ ] **D02** `internal/usecase/auth/service.go`: `RequestReset(phone)` reuses OTP send path, scope-tagged `reset:{phone}` in Redis to prevent cross-use with login OTP.
- [ ] **D03** `internal/usecase/auth/service.go`: `ConfirmReset(phone, otp, newPassword)` — verify, update password, revoke all sessions for that account, emit `auth.reset.confirm.success`.
- [ ] **D04** `internal/handler/http/v1/auth_handler.go`: `POST /auth/password/change`, `POST /auth/reset/request`, `POST /auth/reset/confirm`.
- [ ] **D05** `internal/handler/http/v1/router.go`: wire routes; password change behind `AuthRequired`; reset behind phone limiter.
- [ ] **D06** Tests: change revokes sessions, reset revokes sessions, cross-use of login OTP against reset endpoint is rejected.

## Stream E — OTP attempt cap

- [ ] **E01** `internal/usecase/auth/service.go:206-255`: on each wrong verify increment `otp:attempts:{phone}` (TTL-bound to OTP TTL). On 5th wrong attempt, delete `otp:{phone}` and return `ErrResetOTPInvalid`.
- [ ] **E02** Keep the `"555555"` bypass untouched — add a single comment "dev-bypass; tracked for removal in 010-follow-up".
- [ ] **E03** Tests: 6th wrong attempt forces re-request; bypass path still verifies in dev env.

## Stream F — Audit logging

- [ ] **F01** `internal/usecase/auth/audit.go`: thin emitter `Audit(ctx, evt, kv...)` that wraps the existing logger with a fixed set of fields (`evt`, `rid`, `ip`, `ua`).
- [ ] **F02** Add request-ID middleware (if not present) or ensure `rid` flows from `X-Request-Id` / generated UUID into `ctx` → logs.
- [ ] **F03** Emit events in: each login path, OTP request/verify, refresh, logout, logout-all, password change, reset flows, authz denied, rate limit triggers.
- [ ] **F04** `authorization_test.go` assertions on audit events (log capture helper).

## Stream G — Frontend: unified web (`apps/web`)

- [ ] **G01** `apps/web/src/lib/api.ts`: `authenticatedFetch` with 401-retry-refresh; drop `localStorage.getItem("bonbon_token")` everywhere.
- [ ] **G02** `apps/web/src/lib/auth.tsx` (garage + car-owner variants): refresh on mount; store access token in React context only.
- [ ] **G03** Ensure all outgoing requests include `credentials: "include"` for the `/auth/refresh` cookie path.
- [ ] **G04** Update error handling to treat 429 with a toast + countdown.
- [ ] **G05** Mock specs in `e2e/garage-web/*.spec.ts` and `e2e/car-owner-web/*.spec.ts` updated for new access-token semantics.

## Stream H — Frontend: mobile (`apps/bonbon-go`)

- [ ] **H01** Add `expo-secure-store` dependency.
- [ ] **H02** `apps/bonbon-go/src/lib/api.ts`: persist refresh via `SecureStore`; memory-only access token; silent refresh on app launch.
- [ ] **H03** Logout clears SecureStore.
- [ ] **H04** Mock specs updated in `e2e/bonbon-go/*.spec.ts`.

## Stream I — Integration E2E (`e2e/*/auth.integration.spec.ts`)

- [ ] **I01** `e2e/garage-web/auth.integration.spec.ts`: login + wait + silent refresh; brute-force 11 wrong passwords → 429; logout-all across two contexts.
- [ ] **I02** `e2e/car-owner-web/auth.integration.spec.ts`: 5 wrong OTPs → forced re-request; password-reset flow end-to-end.
- [ ] **I03** `e2e/bonbon-go/auth.integration.spec.ts`: login → relaunch → still signed in.
- [ ] **I04** `e2e/fixtures/seed-data.ts`: no changes to existing seed creds (garage `0901000001/owner123`, employee `0901000002/emp123`, car owner `0912000001`).

## Stream J — Rollout & cutover

- [ ] **J01** Feature flag `AUTH_REFRESH_V2` gates the new /refresh endpoint and cookie issuance. Legacy 24h JWT path retained while flag off.
- [ ] **J02** Dual-write sessions during migration: legacy logins also write a session row so rollback is safe.
- [ ] **J03** Staging canary for 48h with `AUTH_REFRESH_V2=true`; watch `auth.refresh.reused` rate (expected ~0).
- [ ] **J04** Flip production flag after canary clean.
- [ ] **J05** One-week soak; remove legacy 24h path and flag (Stream K).

## Stream K — Post-soak cleanup

- [ ] **K01** Delete legacy 24h JWT code path and flag.
- [ ] **K02** Open follow-up ticket "remove `"555555"` OTP bypass" with a link back to `spec.md` non-goal.
- [ ] **K03** Update `docs/auth-design.md` status from Draft → Implemented for Phases 0–2.

## Order of execution (dependency notes)

- A → B → C are the backend critical path. Stream A has no dependencies. B depends on A. C depends on A.
- D depends on C (needs session revoke primitives).
- E is independent of everything (OTP attempt cap), can ship alongside A.
- F can start after A and plug into every other stream.
- G and H block on C07 landing (the `/auth/refresh` endpoint).
- I blocks on J01 (flag wiring) + corresponding frontend streams.
- K blocks on J05 soak completion.
