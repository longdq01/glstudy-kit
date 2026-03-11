# Data Model: User Authentication System

**Phase**: 1 — Design
**Date**: 2026-03-11
**Branch**: `001-user-auth`

All entities belong to the **`glauth`** PostgreSQL schema, managed by Hibernate `ddl-auto: update`.

---

## Entities

### `users`

Managed by class: `vn.glvideo.gl_auth_api.model.entity.User`

| Field | Java Type | Column | Constraints | Notes |
|---|---|---|---|---|
| `id` | `String` | `id` | PK | Nanoid via `com.aventrix.jnanoid` |
| `username` | `String` | `username` | UNIQUE, NOT NULL | Max 50 chars |
| `fullName` | `String` | `full_name` | NULLABLE | Display name, max 100 chars |
| `email` | `String` | `email` | UNIQUE, NOT NULL, max 255 | Login email |
| `phoneNumber` | `String` | `phone_number` | NULLABLE | Optional |
| `status` | `Integer` | `status` | NULLABLE | `0` = inactive, `1` = active |
| `password` | `String` | `password` | NULLABLE | bcrypt-hashed; null for SSO-only users |
| `emailVerified` | `Boolean` | `email_verified` | NULLABLE | Not enforced in MVP |
| `createdAt` | `Timestamp` | `created_at` | NOT NULL | Registration time |
| `updatedAt` | `Timestamp` | `updated_at` | NOT NULL | Last profile update |
| `lastLogin` | `Timestamp` | `last_login` | NULLABLE | Updated on each login |

**State transitions**:
```
ACTIVE (status=1) ←→ INACTIVE (status=0)
  - Admin action only; no user self-deactivation in MVP
  - On → INACTIVE: all active sessions immediately invalidated via Redis user_status key
  - On → ACTIVE: user_status Redis key updated; user can log in again
```

**Validation rules**:
- `username`: 3–50 chars, alphanumeric + underscores, unique
- `email`: valid RFC 5322 format, max 255 chars, unique
- `fullName`: 2–100 chars
- `password` (when present): min 8 chars, ≥1 uppercase, ≥1 digit, ≥1 special char from `@$!%*?&`

---

### `roles`

Managed by class: `vn.glvideo.gl_auth_api.model.entity.Role`

| Field | Java Type | Column | Constraints | Notes |
|---|---|---|---|---|
| `id` | `Long` | `id` | PK, sequence | Auto-generated |
| `name` | `String` | `name` | NULLABLE | `ROLE_LEARNER`, `ROLE_ADMIN` |
| `description` | `String` | `description` | NULLABLE | Human-readable label |
| `status` | `Integer` | `status` | NULLABLE | Active/inactive flag |

**Seeded values** (via application startup or data.sql):
- `ROLE_LEARNER` — assigned to all self-registered users
- `ROLE_ADMIN` — assigned only via database seed or admin action

---

### `permissions`

Managed by class: `vn.glvideo.gl_auth_api.model.entity.Permission`

| Field | Java Type | Column | Constraints | Notes |
|---|---|---|---|---|
| `id` | `Long` | `id` | PK, sequence | Auto-generated |
| `code` | `Integer` | `code` | NULLABLE | Permission code |
| `parentCode` | `Integer` | `parent_code` | NULLABLE | Hierarchical parent |
| `description` | `String` | `description` | NULLABLE | Description |
| `contextPath` | `String` | `context_path` | NULLABLE | API path this permission covers |

---

### `user_roles` (junction)

| Column | Type | Constraints |
|---|---|---|
| `user_id` | `VARCHAR` | FK → `users.id` |
| `role_id` | `BIGINT` | FK → `roles.id` |

---

### `role_permissions` (junction)

| Column | Type | Constraints |
|---|---|---|
| `role_id` | `BIGINT` | FK → `roles.id` |
| `permissions_id` | `BIGINT` | FK → `permissions.id` |

---

### `user_auth_providers`

Managed by class: `vn.glvideo.gl_auth_api.model.entity.UserAuthProvider`

| Field | Java Type | Column | Constraints | Notes |
|---|---|---|---|---|
| `id` | `Integer` | `id` | PK, sequence | Auto-generated |
| `userId` | `String` | `user_id` | FK → `users.id`, NOT NULL | Owning user |
| `provider` | `String` | `provider` | NULLABLE | `local`, `google`, `github` |
| `providerUserId` | `String` | `provider_user_id` | NULLABLE | Provider's user identifier (sub/id) |
| `providerMetadata` | `String` | `provider_metadata` | NULLABLE | JSON blob of raw provider profile |
| `isPrimary` | `Boolean` | `is_primary` | NULLABLE | Whether this is the primary sign-in method |
| `linkedAt` | `Timestamp` | `linked_at` | NOT NULL | When the provider was linked |

**Indexes**:
- `idx_user_auth_providers_user_id` on `user_id`

**Constraints**:
- `UNIQUE (user_id, provider)` — one record per user per provider

**Provider values**: `local` | `google` | `github`

---

## Redis Keys (Session Layer)

| Key Pattern | Value | TTL | Purpose |
|---|---|---|---|
| `refresh_token:{jti}` | Serialized UserDTO JSON | 7 days | Allowlist of valid refresh tokens |
| `blacklist:token:{jti}` | marker string | Remaining token lifetime | Revoked access/refresh tokens |
| `token:register:{identifier}` | One-time token data | Short-lived | Temporary token for SSO new-user registration completion |
| `token:confirm_link:{identifier}` | Confirm-link data | Short-lived | Temporary token for linking SSO provider to existing account |
| `user_status:{userId}` | `"0"` or `"1"` | No TTL (permanent) | Fast inactive-account check on token validation (FR-020) |

**Session invalidation on account deactivation** (FR-020):
1. Admin sets user `status = 0` in `users` table
2. Service writes `user_status:{userId} = "0"` to Redis
3. On every subsequent `getTokenInfo()` call, Redis key is checked; if `"0"`, throw `UNAUTHORIZED`
4. Existing refresh tokens remain in Redis but are effectively blocked by the status check

---

## DTOs Required (New)

### Requests
- `CheckEmailRequest` — `{ email: String }`
- `CheckUsernameRequest` — `{ username: String }`
- `SetPasswordRequest` — `{ userId: String, password: String }`

### Responses
- `AvailabilityResponse` — `{ exists: Boolean }`

### Existing (no change needed)
- `LoginRequest` — already handles both local and SSO cases
- `SignupRequest` — update to require explicit `username` field
- `AuthResponse` — already includes `accessToken`, `refreshToken`, `userInfo`, `needRegister`, `token`, `confirmCode`
- `UserResponse` — already includes user profile fields; add `providers: List<String>` field
