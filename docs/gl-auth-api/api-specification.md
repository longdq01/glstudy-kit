# gl-auth-api – API Reference

**Base URL:** `http://localhost:8080`
**Swagger UI:** `http://localhost:8080/swagger-ui.html`

All responses follow the standard envelope:

```json
{
  "success": true | false,
  "data": { ... } | null,
  "error": null | { "code": "...", "message": "..." },
  "timestamp": "ISO-8601"
}
```

---

## Authentication (`/v1/auth`)

### POST `/v1/auth/login`

Unified login endpoint for both local and SSO flows.

**Local login request:**
```json
{
  "provider": "local",
  "principal": "user@example.com",
  "password": "SecureP@ss1"
}
```

**SSO login request (Google / GitHub):**
```json
{
  "provider": "google",
  "authorizationCode": "<code from OAuth2 callback>"
}
```

`provider` must be one of: `local`, `google`, `github`.

**Response (200):**
```json
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOi...",
    "refreshToken": "dGhpcyBpcyBh...",
    "needRegister": false,
    "userInfo": {
      "id": "abc123",
      "username": "johndoe",
      "fullName": "John Doe",
      "email": "user@example.com",
      "status": 1,
      "emailVerified": true,
      "roles": [],
      "providers": []
    },
    "token": null,
    "confirmCode": null
  }
}
```

**SSO-specific response scenarios:**

| Scenario | `needRegister` | `token` | `confirmCode` | Next step |
|---|---|---|---|---|
| Existing user, same provider | `false` | null | null | Logged in |
| New user (email not found) | `true` | one-time token | null | Call `POST /v1/users/signup` with `token` |
| Email exists under different provider | `false` | null | confirm code | Call `GET /v1/auth/confirm-link?confirmCode=...` |

**Error cases (both local and SSO):**

| Condition | HTTP | Code | Message |
|---|---|---|---|
| Wrong password / bad credentials | 403 | `Forbidden` | Không xác thực được thông tin người dùng |
| Account is inactive (FR-019) | 401 | `Unauthorized` | Account is inactive. Contact support. |

---

### GET `/v1/auth/confirm-link`

Confirm linking a new OAuth provider to an existing account.

**Query parameter:** `confirmCode` (string, required)

**Response (200):** Same shape as login — returns `accessToken` + `refreshToken` + `userInfo`.

---

### POST `/v1/auth/logout`

Revokes the current access token.

**Header:** `Authorization: Bearer <accessToken>`

**Response (200):**
```json
{ "success": true, "data": null }
```

---

### POST `/v1/auth/refresh-token`

Issues a new access token using a valid refresh token.

**Request:**
```json
{ "refreshToken": "dGhpcyBpcyBh..." }
```

**Response (200):** Same shape as login — returns new `accessToken` + `refreshToken` + `userInfo`.

---

## Users (`/v1/users`)

### POST `/v1/users/signup`

Register a new user. Used for both local signup and completing SSO registration.

**Local signup request:**
```json
{
  "username": "johndoe",
  "email": "user@example.com",
  "fullName": "John Doe",
  "password": "SecureP@ss1",
  "social": false
}
```

**SSO registration completion request** (after `needRegister=true` from login):
```json
{
  "username": "johndoe",
  "fullName": "John Doe",
  "token": "<one-time token from login response>",
  "social": true
}
```

**Response (200):** Same shape as login — returns `accessToken` + `refreshToken` + `userInfo`.

---

### POST `/v1/users/check-email`

Check whether an email address is already registered.

**Request:**
```json
{ "email": "user@example.com" }
```

**Response (200):**
```json
{
  "success": true,
  "data": { "exists": true }
}
```

---

### POST `/v1/users/check-username`

Check whether a username is already taken.

**Request:**
```json
{ "username": "johndoe" }
```

**Response (200):**
```json
{
  "success": true,
  "data": { "exists": false }
}
```

---

### GET `/v1/users/me`

Get the currently authenticated user's profile.

**Header:** `Authorization: Bearer <accessToken>` (required)

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "abc123",
    "username": "johndoe",
    "fullName": "John Doe",
    "email": "user@example.com",
    "phoneNumber": null,
    "status": 1,
    "emailVerified": true,
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-01-01T00:00:00Z",
    "lastLogin": "2026-03-04T10:00:00Z",
    "roles": [],
    "providers": []
  }
}
```

**Errors:** `401 Unauthorized` if token is missing or invalid.

---

### POST `/v1/users/set-password`

Set a password for an SSO-only user (who has no local password yet).

**Header:** `Authorization: Bearer <accessToken>` (required)

**Request:**
```json
{
  "userId": "abc123",
  "password": "NewP@ss1"
}
```

Password validation: min 8 chars, must include uppercase, lowercase, digit, and special character (`@$!%*?&`).

**Response (200):**
```json
{ "success": true, "data": null }
```

---

## Error Codes

| `CodeResponse` | HTTP | Status string | Message (VI) |
|---|---|---|---|
| `OK` | 200 | `OK` | Thành công |
| `INVALID_ARGUMENT` | 400 | `InvalidArgument` | Tham số không hợp lệ |
| `BAD_REQUEST` | 400 | `BadRequest` | Yêu cầu không hợp lệ |
| `EXISTED` | 400 | `Existed` | Đã tồn tại |
| `UNAUTHORIZED` | 401 | `Unauthorized` | Không xác thực được thông tin người dùng |
| `FORBIDDEN` | 403 | `Forbidden` | Bạn không có quyền truy cập tài nguyên |
| `NOT_FOUND` | 404 | `NotFound` | Không tìm thấy thông tin yêu cầu |
| `INTERNAL` | 500 | `InternalServerError` | Có lỗi xảy ra |

Throw via: `throw ApiException.ErrNotFound().message("Custom message").build();`
