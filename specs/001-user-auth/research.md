# Research: User Authentication System

**Phase**: 0 — Pre-design research
**Date**: 2026-03-11
**Branch**: `001-user-auth`

---

## 1. Backend State Audit (gl-auth-api)

### Base Package
- **Decision**: `vn.glvideo.gl_auth_api`
- All Java source files and Spring component scans use this package root.

### Database Migration Strategy
- **Finding**: No Flyway exists. Schema is managed by Hibernate `ddl-auto: update`.
- **Implication**: The spec assumption that "Flyway migrations must be written" does not apply to the current codebase. Schema changes are applied automatically via JPA entities. If Flyway adoption is desired, it is a separate infrastructure task, not part of this feature.
- **Decision for this feature**: Continue with Hibernate DDL auto; no Flyway files to write.

### Access Token Lifetime Discrepancy
- **Spec says**: 15 minutes
- **Implementation has**: 1 hour (`TokenServiceImpl`)
- **Docs say** (`docs/gl-auth-api/README.md`): 15 minutes
- **Decision**: Implementation value (1 hour) is authoritative since the backend is already built and deployed. The spec's 15-minute statement originated from planning docs, not code. Document this as a known configuration drift; adjusting TTL is a single config change that can be done independently.
- **Alternatives considered**: Enforce 15 min now — rejected because it requires coordinating token refresh behavior on the frontend, out of scope for this feature's completion.

### Redis Token Storage Pattern
- **Decision**: Allowlist + blacklist dual pattern.
  - Refresh tokens stored as: `refresh_token:{jti}` → serialized UserDTO JSON, TTL 7 days
  - Revoked tokens stored as: `blacklist:token:{jti}` → marker entry, TTL matching original token
  - Temporary register token: `token:register:{identifier}`
  - Confirm-link token: `token:confirm_link:{identifier}`
- **Rationale**: Allowlist enables fast lookup of valid refresh tokens. Blacklist enables immediate access-token revocation without waiting for natural expiry.
- **FR-020 implication** (inactive account — invalidate all sessions): The current pattern stores one refresh token per `jti`, not per `userId`. To invalidate all sessions for a deactivated user, the service must either (a) maintain a per-user set of active `jti` values, or (b) add a `user_status` check on every token validation call. **Decision**: Add `user_status` check on every `getTokenInfo()` call — simpler and avoids additional Redis data structure. Status is read from the cached UserDTO in Redis, which is stored at token issuance time.
- **Alternatives considered**: Per-user Redis set (`user_sessions:{userId}` → set of `jti`) — rejected for MVP due to added complexity.

### Existing Endpoints (already built)
| Endpoint | Status |
|---|---|
| `POST /v1/auth/login` (local + SSO) | ✅ Built |
| `POST /v1/users/signup` | ✅ Built |
| `POST /v1/auth/refresh-token` | ✅ Built |
| `POST /v1/auth/logout` | ✅ Built |
| `GET /v1/users/me` | ✅ Built |
| `GET /v1/auth/confirm-link` | ✅ Built |
| `POST /v1/users/check-email` | ❌ Missing |
| `POST /v1/users/check-username` | ❌ Missing |
| `POST /v1/users/set-password` | ❌ Missing |

### Tests
- **Finding**: No unit or integration tests exist (confirmed by WORKLOG — all test items unchecked).
- **Decision**: Write all three test layers as part of this feature: controller (MockMvc), service (Mockito), integration (Spring Boot test slice).

---

## 2. Frontend State Audit (glstudy-frontend)

### What Already Exists
The frontend is substantially more complete than the WORKLOG indicates:

| Component | Status | Notes |
|---|---|---|
| Login page (`/login`) | ✅ Built | Tabs: Login + Register, Zod validation, password strength |
| Register tab | ✅ Built | On same page as login (`/app/(auth)/login/page.tsx`) |
| BFF: `/api/auth/login` | ✅ Built | Sets `access_token` + `refresh_token` httpOnly cookies |
| BFF: `/api/auth/register` | ✅ Built | Derives username from email currently |
| BFF: `/api/auth/logout` | ✅ Built | Clears cookies |
| BFF: `/api/auth/refresh` | ✅ Built | Uses `refresh_token` cookie |
| BFF: `/api/users/me` (GET + PUT) | ✅ Built | Profile fetch and update |
| Zustand auth store | ✅ Built | `user`, `isAuthenticated`, `isLoading` |
| `useAuth` hook | ✅ Built | Fetches user on mount |
| AuthGuard | ✅ Built | Redirects unauthenticated users to `/login` |
| Axios with 401 interceptor + auto-refresh | ✅ Built | Silent token renewal |
| SSO buttons UI | ✅ Built | Google + GitHub buttons on login page |
| Profile page (`/profile`) | ✅ Partial | Exists; needs linked providers + set-password form |

### What Needs to Be Built
| Component | Status | Notes |
|---|---|---|
| SSO OAuth2 actual handlers | ❌ Missing | Buttons exist; click handlers not wired |
| OAuth2 callback BFF route | ❌ Missing | `/api/auth/oauth2/[provider]/callback` |
| Confirm-link flow | ❌ Missing | Modal + BFF route |
| Username selection (new SSO user) | ❌ Missing | Step after OAuth2 for new users |
| BFF: check-email route | ❌ Missing | `/api/users/check-email` |
| BFF: check-username route | ❌ Missing | `/api/users/check-username` |
| BFF: set-password route | ❌ Missing | `/api/users/set-password` |
| Set-password form on profile | ❌ Missing | For SSO-only users |
| Linked providers on profile | ❌ Missing | Show which providers are linked |
| Inactive account error handling | ❌ Missing | Show specific error on login |

### Configuration Discrepancy
- `AUTH_API_URL` defaults to `http://localhost:8081` in BFF routes.
- `docs/gl-auth-api/README.md` says port 8080.
- **Decision**: Use whatever port `gl-auth-api` actually runs on. The BFF reads from env var `AUTH_API_URL`; the default just needs to match the actual service port. Document the correct port in quickstart.

### Register Username
- Current register BFF derives username from email (e.g., `user@gmail.com` → `user`).
- Spec requires username to be explicitly chosen by the user (FR-001, acceptance scenario 3).
- **Decision**: Update the register form to include an explicit username field and update the BFF route accordingly.

---

## 3. OAuth2 SSO Flow Design

### Chosen Pattern: Frontend-Initiated Code Exchange
- **Decision**: Keep the existing pattern — the frontend redirects to the OAuth2 provider, receives the authorization code in the callback URL, then sends `POST /v1/auth/login` with `{ provider, authorizationCode }` to the backend.
- **Rationale**: This pattern is already partially implemented (backend `POST /v1/auth/login` handles SSO). The alternative (Spring Security's built-in OAuth2 client login) would require significant backend restructuring.
- **Alternatives considered**: Spring Security OAuth2 client login flow — rejected because the backend already has a custom code-exchange implementation.

### Frontend OAuth2 Flow Steps
1. User clicks "Sign in with Google" → frontend redirects to Google consent URL with `client_id`, `redirect_uri`, `scope`, `state`
2. Google redirects to `/auth/google/callback?code=xxx&state=xxx`
3. Next.js BFF route handles the callback: calls `POST /v1/auth/login { provider: "google", authorizationCode: code }`
4. Backend returns one of three scenarios: logged in, `needRegister=true`, or `confirmCode` present
5. BFF handles each scenario: set cookies → redirect to dashboard, or redirect to username-selection page, or show confirm-link modal

---

## 4. Session Renewal Pattern

### Chosen Pattern: Axios Interceptor (already built)
- **Decision**: Keep existing Axios 401 interceptor in `api-client.ts`. On 401, it calls `POST /api/auth/refresh`, then retries the original request.
- **Rationale**: Already implemented and working. No changes needed.

---

## 5. Inactive Account Enforcement (FR-019, FR-020)

- **Decision**: Add user status check in `TokenServiceImpl.getTokenInfo()`. The UserDTO cached in Redis at token issuance time includes a `status` field. On every token validation, check `user.status == ACTIVE`. If not, throw `ApiException.ErrUnauthorized()`.
- **FR-020** (invalidate all sessions on deactivation): Since tokens store UserDTO in Redis by `jti`, a simple approach is to add a per-user status key: `user_status:{userId}` → status value, updated when admin changes status. `getTokenInfo()` checks this key (fast Redis read) on every validation.
- **Alternatives considered**: Per-user session set in Redis — rejected for MVP complexity.
