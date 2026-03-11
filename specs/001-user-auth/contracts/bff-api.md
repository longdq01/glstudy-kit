# BFF API Contracts: User Authentication System

**Service**: `glstudy-frontend` (Next.js App Router API Routes)
**Branch**: `001-user-auth`

The BFF (Backend For Frontend) layer manages cookies, proxies to `gl-auth-api`, and shields the browser from direct backend access. All tokens are stored in httpOnly cookies — never exposed to client-side JavaScript.

> **Existing BFF routes** (`/api/auth/login`, `/api/auth/register`, `/api/auth/logout`, `/api/auth/refresh`, `/api/users/me GET+PUT`) are already built. This file documents only the **new** BFF routes to be implemented in this feature.

---

## Cookie Contract

| Cookie Name | Value | Flags | Lifetime |
|---|---|---|---|
| `access_token` | JWT access token | `httpOnly`, `secure`, `sameSite=lax` | Session (access token TTL) |
| `refresh_token` | JWT refresh token | `httpOnly`, `secure`, `sameSite=lax` | 7 days |

---

## New BFF Routes

### GET `/api/auth/oauth2/[provider]/callback`

Handles the OAuth2 authorization code callback from Google or GitHub.

**Path param**: `provider` — `google` | `github`
**Query params**: `code` (string, required), `state` (string, optional)

**Flow:**
1. Receives authorization code from provider redirect
2. Calls `POST /v1/auth/login` on `gl-auth-api` with `{ provider, authorizationCode: code }`
3. Handles the three response scenarios:

| Scenario | `gl-auth-api` Response | BFF Action |
|---|---|---|
| Existing user, same provider | `needRegister=false`, no `token`, no `confirmCode` | Set `access_token` + `refresh_token` cookies → redirect `/dashboard` |
| New SSO user | `needRegister=true`, `token` present | Redirect `/auth/sso-register?token=<token>` (token goes in URL, not cookie) |
| Email already linked to different provider | `confirmCode` present | Redirect `/auth/confirm-link?confirmCode=<code>` |
| Provider unavailable / error | Error from `gl-auth-api` | Redirect `/login?error=sso_failed` |
| Inactive account | `UNAUTHORIZED` from `gl-auth-api` | Redirect `/login?error=account_inactive` |

**Response**: HTTP 302 redirect (never returns JSON directly to browser)

---

### GET `/api/auth/confirm-link`

Confirms linking a new OAuth provider to an existing account.

**Query param**: `confirmCode` (string, required)

**Flow:**
1. Calls `GET /v1/auth/confirm-link?confirmCode=<code>` on `gl-auth-api`
2. On success: sets `access_token` + `refresh_token` cookies → returns 200 with user info
3. On error: returns 400 or 401 with error detail

**Response (200):**
```json
{
  "user": {
    "id": "abc123",
    "username": "johndoe",
    "fullName": "John Doe",
    "email": "user@example.com",
    "roles": [],
    "providers": ["google", "local"]
  }
}
```

**Error responses:**
- `400` — confirmCode missing or invalid
- `401` — confirmCode expired

---

### POST `/api/users/check-email`

Proxies availability check to backend (no auth required).

**Request:**
```json
{ "email": "user@example.com" }
```

**Flow:** Forwards to `POST /v1/users/check-email` on `gl-auth-api`.

**Response (200):**
```json
{ "exists": true }
```

---

### POST `/api/users/check-username`

Proxies availability check to backend (no auth required).

**Request:**
```json
{ "username": "johndoe" }
```

**Flow:** Forwards to `POST /v1/users/check-username` on `gl-auth-api`.

**Response (200):**
```json
{ "exists": false }
```

---

### POST `/api/users/set-password`

Allows an SSO-only user to set a local password. Requires active session.

**Authentication**: Reads `access_token` cookie (httpOnly — not visible to client JS)

**Request:**
```json
{ "password": "NewP@ss1" }
```

**Note**: `userId` is read from the `access_token` cookie on the BFF side. The client does NOT send `userId` — the BFF extracts it from the validated token and injects it into the upstream `POST /v1/users/set-password` request body.

**Flow:**
1. Read `access_token` from cookies; if missing → 401
2. Forward `POST /v1/users/set-password` with `{ userId, password }` using `Authorization: Bearer <access_token>` header
3. Return success or forward error

**Response (200):**
```json
{ "success": true }
```

**Error responses:**
- `400` — password does not meet strength requirements
- `400` — password already set
- `401` — no active session

---

## Modified BFF Routes

### POST `/api/auth/register`

Currently derives `username` from email. Must be updated to accept explicit `username` field.

**Updated request:**
```json
{
  "username": "johndoe",
  "email": "user@example.com",
  "fullName": "John Doe",
  "password": "SecureP@ss1"
}
```

**Flow (unchanged):** Forwards to `POST /v1/users/signup` on `gl-auth-api`, sets cookies, returns user info.

---

## Error Handling Conventions

All BFF routes return a consistent error shape for non-redirect responses:

```json
{
  "error": "DESCRIPTION",
  "message": "Human-readable message"
}
```

Inactive account (from 401 on token validation on any authenticated route):
- BFF clears cookies and returns `401` with `{ "error": "ACCOUNT_INACTIVE" }`
- Client 401 interceptor redirects to `/login?error=account_inactive`
