# gl-auth-api – Service Overview

Authentication and user management microservice for the GLStudy platform.

## Responsibility

- Local authentication (email + password via bcrypt)
- SSO authentication (Google OAuth2, GitHub OAuth2)
- JWT issuance and validation (access token + refresh token)
- User registration, profile, and password management
- RBAC: roles and permissions management

> **This is the only service that handles authentication.** Other services (`gl-video-api`, etc.) do NOT include Spring Security. They receive authenticated requests from the Next.js BFF, which passes `userId` via an `X-User-Id` header after validating through `gl-auth-api`.

## Runtime

| Property | Value |
|---|---|
| Port | **8080** |
| Base path | `/v1` |
| PostgreSQL schema | `glauth` |
| Redis database | `1` |
| Swagger UI | `http://localhost:8080/swagger-ui.html` |

## Token Strategy

| Token | Expiry | Storage |
|---|---|---|
| Access token | 15 minutes | Client (httpOnly cookie set by BFF) |
| Refresh token | 7 days | Redis |

Logout revokes the access token in Redis. Refresh token rotation is handled by `POST /v1/auth/refresh-token`.

## SSO Flow Summary

1. Frontend redirects user to Google / GitHub OAuth2 consent screen
2. Provider returns an authorization code to the frontend callback URL (`http://localhost:3000/auth/{provider}/callback`)
3. Frontend sends `POST /v1/auth/login` with `provider` + `authorizationCode`
4. Service exchanges code for provider access token → fetches user profile
5. If the email is already registered with a different provider, a `confirmCode` is returned — the user must call `GET /v1/auth/confirm-link?confirmCode=...` to link accounts
6. If the user is new to the platform, `needRegister=true` is returned along with a one-time `token` — the frontend sends `POST /v1/users/signup` with that token to complete registration

## Related Docs

- [Schema](./schema.md) – Auth database schema
- [API](./api.md) – Endpoint reference
- [System Architecture](../02-architecture.md)
- [Project Overview](../01-project-overview.md)
