# Implementation Plan: User Authentication System

**Branch**: `001-user-auth` | **Date**: 2026-03-11 | **Spec**: [spec.md](spec.md)

## Summary

Implement a complete user authentication system for GLStudy covering email/password registration and login, Google + GitHub SSO via OAuth2 authorization-code exchange, session persistence with JWT (access + refresh tokens), transparent token renewal, profile viewing with linked providers, and admin-controlled account deactivation with immediate session invalidation.

The backend (`gl-auth-api`) is largely built; three endpoints are missing (`check-email`, `check-username`, `set-password`) and `user_status` Redis check for FR-020 is unimplemented. The frontend has login/register UI and BFF auth routes; SSO callback handlers, username-selection flow, confirm-link flow, and the three new BFF proxy routes are missing.

## Technical Context

**Language/Version**: Java 21 (backend), TypeScript / Node.js 20 (frontend)
**Primary Dependencies**:
- Backend: Spring Boot 3.4.x, Spring Data JPA, Spring Security, jjwt 0.12.x, MapStruct 1.6+, jnanoid, Springdoc OpenAPI 2.x
- Frontend: Next.js 14 (App Router), React 18, Zustand, React Query, React Hook Form, Zod, Axios
**Storage**: PostgreSQL 15 (`glauth` schema), Redis 7 (session/token store)
**Testing**: JUnit 5 + Mockito + MockMvc (backend), Jest + React Testing Library (frontend)
**Target Platform**: Linux server / Docker; web browser (Chrome, Firefox, Safari)
**Project Type**: Web application — microservice backend + Next.js BFF + browser frontend
**Performance Goals**: Login p95 < 300 ms; availability check < 1 s; LCP < 2.5 s (per constitution)
**Constraints**: httpOnly cookies mandatory (no localStorage); no Flyway (Hibernate DDL auto); coverage ≥70%
**Scale/Scope**: MVP — single-region, ~hundreds of concurrent users

## Constitution Check

*Re-evaluated post-design. All gates pass.*

| Principle | Gate | Status |
|---|---|---|
| I. Microservice Independence | `gl-auth-api` owns `glauth` schema exclusively; no cross-service DB access | PASS |
| II. Security-First Auth | Tokens in httpOnly cookies; only `gl-auth-api` issues/validates JWTs; Bean Validation on all DTOs | PASS |
| III. Unified Conventions | New endpoints follow `{Resource}Controller → {Resource}Service → {Resource}Repository` naming; `BaseResponse` envelope used | PASS |
| IV. Documentation Consistency | `docs/gl-auth-api/api-specification.md` and `docs/gl-auth-api/schema.md` updated as part of done | REQUIRED IN PR |
| V. Test Coverage | Three-layer tests (MockMvc, Mockito, integration) planned; coverage ≥70% required | REQUIRED |

**Note on Flyway**: Constitution §V mandates Flyway for schema changes. Research confirmed Flyway is not currently used — Hibernate `ddl-auto: update` is in place. Schema changes for this feature (new `user_auth_providers` index, `user_roles`, `role_permissions` junction tables) are applied automatically via JPA. This is a known deviation documented in `research.md`. A separate infrastructure task to adopt Flyway is recommended for Sprint 3.

## Project Structure

### Documentation (this feature)

```
specs/001-user-auth/
├── plan.md                    # This file
├── spec.md                    # Feature specification
├── research.md                # Phase 0 research findings
├── data-model.md              # Entity definitions and Redis keys
├── quickstart.md              # Local dev setup guide
├── contracts/
│   ├── backend-api.md         # New backend endpoint contracts
│   └── bff-api.md             # New BFF route contracts
├── checklists/
│   └── requirements.md        # Spec quality checklist (passed)
└── tasks.md                   # Phase 2 output (created by /speckit.tasks)
```

### Source Code

```
gl-auth-api/                              ← Spring Boot 3.4, port 8081
└── src/main/java/vn/glvideo/gl_auth_api/
    ├── config/                           ← SecurityConfig, RedisConfig, CorsConfig
    ├── controller/
    │   ├── AuthController.java           ← /v1/auth/* (existing)
    │   └── UserController.java           ← /v1/users/* (add check-email, check-username, set-password)
    ├── service/
    │   ├── AuthService.java + Impl       ← login, logout, refresh, confirm-link (existing)
    │   ├── TokenService.java + Impl      ← JWT issue/validate + user_status check (add FR-020)
    │   └── UserService.java + Impl       ← signup, me, (add check-email, check-username, set-password)
    ├── repository/
    │   ├── UserRepository.java           ← existsByEmail, existsByUsername queries needed
    │   └── UserAuthProviderRepository.java
    ├── model/
    │   ├── entity/
    │   │   ├── User.java                 ← existing
    │   │   ├── Role.java                 ← existing
    │   │   ├── Permission.java           ← existing
    │   │   └── UserAuthProvider.java     ← existing
    │   └── dto/
    │       ├── request/
    │       │   ├── CheckEmailRequest.java     ← NEW
    │       │   ├── CheckUsernameRequest.java  ← NEW
    │       │   └── SetPasswordRequest.java    ← NEW
    │       └── response/
    │           └── AvailabilityResponse.java  ← NEW
    └── src/test/java/vn/glvideo/gl_auth_api/
        ├── controller/                   ← MockMvc tests (NEW — all test layers)
        ├── service/                      ← Mockito tests (NEW)
        └── integration/                  ← Spring Boot test slice (NEW)

glstudy-frontend/                         ← Next.js 14 App Router, port 3000
└── src/
    ├── app/
    │   ├── api/
    │   │   ├── auth/
    │   │   │   ├── login/route.ts        ← existing
    │   │   │   ├── register/route.ts     ← UPDATE: accept explicit username field
    │   │   │   ├── logout/route.ts       ← existing
    │   │   │   ├── refresh/route.ts      ← existing
    │   │   │   ├── confirm-link/route.ts ← NEW
    │   │   │   └── oauth2/
    │   │   │       └── [provider]/
    │   │   │           └── callback/route.ts  ← NEW
    │   │   └── users/
    │   │       ├── me/route.ts           ← existing
    │   │       ├── check-email/route.ts  ← NEW
    │   │       ├── check-username/route.ts ← NEW
    │   │       └── set-password/route.ts ← NEW
    │   └── (auth)/
    │       ├── login/page.tsx            ← existing; add SSO click handlers
    │       ├── sso-register/page.tsx     ← NEW: username selection for new SSO users
    │       └── confirm-link/page.tsx     ← NEW: confirm linking providers
    ├── components/
    │   └── profile/
    │       ├── LinkedProviders.tsx       ← NEW: show linked providers
    │       └── SetPasswordForm.tsx       ← NEW: password form for SSO-only users
    └── store/
        └── authStore.ts                 ← existing; no changes needed
```

**Structure Decision**: Web application (Option 2) with two independently-developed services sharing no code. Backend work is additive (3 endpoints + 1 service method). Frontend work is additive (5 BFF routes + 2 pages + 2 components + 1 form update).

## Complexity Tracking

> No constitution violations requiring justification.

Flyway deviation is a pre-existing project-wide condition, not introduced by this feature. Tracked as technical debt in research.md.
