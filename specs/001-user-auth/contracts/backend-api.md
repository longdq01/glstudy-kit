# Backend API Contracts: User Authentication System

**Service**: `gl-auth-api`
**Base URL**: `http://localhost:8080` (or `AUTH_API_URL` env var)
**Branch**: `001-user-auth`

All endpoints use the standard response envelope:
```json
{
  "success": true | false,
  "data": { ... } | null,
  "error": null | { "code": "...", "message": "..." },
  "timestamp": "ISO-8601"
}
```

> **Note**: The existing endpoints (`POST /v1/auth/login`, `POST /v1/auth/refresh-token`, `POST /v1/auth/logout`, `POST /v1/users/signup`, `GET /v1/users/me`, `GET /v1/auth/confirm-link`) are already built and documented in `docs/gl-auth-api/api-specification.md`. This file documents only the **new** endpoints to be implemented in this feature.

---

## New Endpoints

### POST `/v1/users/check-email`

Check whether an email address is already registered.

**Authentication**: None required
**Controller**: `UserController`
**Service method**: `checkEmail(CheckEmailRequest)`

**Request:**
```json
{ "email": "user@example.com" }
```

**Validation:**
- `email`: not blank, valid RFC 5322 format, max 255 chars

**Response (200):**
```json
{
  "success": true,
  "data": { "exists": true }
}
```

**Error responses:**
- `400 INVALID_ARGUMENT` — email blank or invalid format

---

### POST `/v1/users/check-username`

Check whether a username is already taken.

**Authentication**: None required
**Controller**: `UserController`
**Service method**: `checkUsername(CheckUsernameRequest)`

**Request:**
```json
{ "username": "johndoe" }
```

**Validation:**
- `username`: not blank, 3–50 chars, alphanumeric + underscores only

**Response (200):**
```json
{
  "success": true,
  "data": { "exists": false }
}
```

**Error responses:**
- `400 INVALID_ARGUMENT` — username blank or invalid format

---

### POST `/v1/users/set-password`

Set a local password for an SSO-only user (user whose `password` field is currently null).

**Authentication**: Required — `Authorization: Bearer <accessToken>`
**Controller**: `UserController`
**Service method**: `setPassword(SetPasswordRequest, String userId)`

**Request:**
```json
{
  "userId": "abc123",
  "password": "NewP@ss1"
}
```

**Validation:**
- `userId`: not blank
- `password`: not blank, min 8 chars, ≥1 uppercase, ≥1 digit, ≥1 special char from `@$!%*?&`
- Caller's token `userId` MUST match `request.userId` (cannot set another user's password)
- User MUST NOT already have a local password (idempotent guard)

**Response (200):**
```json
{ "success": true, "data": null }
```

**Error responses:**
- `400 BAD_REQUEST` — password does not meet strength requirements
- `400 EXISTED` — local password already set for this user
- `401 UNAUTHORIZED` — invalid or missing access token
- `403 FORBIDDEN` — token userId does not match request userId
- `404 NOT_FOUND` — user not found

---

## Modified Endpoints

### POST `/v1/users/signup` — Username field now required

The existing signup endpoint must be updated so that `username` is an explicit required field for local registration. Previously the `username` was derived from `email` in the frontend BFF.

**Change**: `username` field moves from optional/derived to `@NotBlank` required in `SignupRequest`.

**Local signup request (updated):**
```json
{
  "username": "johndoe",
  "email": "user@example.com",
  "fullName": "John Doe",
  "password": "SecureP@ss1",
  "social": false
}
```

**SSO registration completion (no change):**
```json
{
  "username": "johndoe",
  "fullName": "John Doe",
  "token": "<one-time token from login response>",
  "social": true
}
```

---

## Inactive Account Error

For `POST /v1/auth/login` — new error scenario (FR-019):

| Condition | HTTP | Error Code | Message |
|---|---|---|---|
| `user.status == 0` (inactive) | `401` | `UNAUTHORIZED` | Account is inactive. Contact support. |

For every `getTokenInfo()` call — `user_status:{userId}` Redis check (FR-020):

| Redis value | Behaviour |
|---|---|
| `"0"` | Throw `UNAUTHORIZED` immediately |
| `"1"` or key absent | Continue normally |
