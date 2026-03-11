# GLStudy – System Architecture

## 1. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Client Layer                          │
│                  Browser  (React + Next.js)                  │
└────────────────────────────┬─────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────┐
│                     API Gateway / BFF                         │
│              Next.js API Routes  (port 3000)                  │
└──────────────────┬──────────────────────┬─────────────────────┘
                   │                      │
     ┌─────────────▼──────────┐  ┌────────▼───────────────────┐
     │      gl-auth-api       │  │       gl-video-api          │
     │   (Spring Boot 3.4)    │  │    (Spring Boot 3.4)        │
     │       port 8080        │  │        port 8082            │
     │                        │  │                             │
     │ Auth · Users · Roles   │  │ Videos · Subtitles          │
     │ Permissions            │  │ Watch Progress · Stats      │
     └───────────┬────────────┘  └──────────┬──────────────────┘
                 │                           │
     ┌───────────▼──────────┐   ┌───────────▼──────────────────┐
     │   PostgreSQL (auth)  │   │   PostgreSQL (video)          │
     │   Redis              │   │   Redis                       │
     └──────────────────────┘   └──────────────────────────────┘

External services (not proxied through BFF):
  Browser ──────────────────────────────────► YouTube iframe
  gl-auth-api ──────────────────────────────► Google / GitHub OAuth2
```

## 2. Architecture Decisions

### 2.1 Microservices from Day One

| Service | Repository | Responsibility | Port |
|---|---|---|---|
| **gl-auth-api** | `glstudy/gl-auth-api` | Auth (local + SSO), User management, Roles & Permissions | 8080 |
| **gl-video-api** | `glstudy/gl-video-api` | Video catalog, Subtitles, Watch progress, Stats | 8082 |

MVP starts with **2 independent Spring Boot microservices**, each with its own PostgreSQL schema. Services communicate only via REST (no shared DB). The BFF (Next.js API Routes) aggregates responses.

### 2.2 Why Next.js as BFF (Backend-for-Frontend)

- **SSR/SSG** for SEO-friendly pages (landing, video catalog)
- **API Routes** act as a thin proxy/aggregation layer between the React frontend and Java backend
- Avoids CORS complexity in production
- Sets httpOnly cookies and injects `X-User-Id` / `X-User-Role` headers before proxying to downstream services

### 2.3 Authentication Strategy (Local + SSO)

**Local login flow:**

```
1. Browser          → POST /api/auth/login          → Next.js BFF
2. Next.js BFF      → POST /v1/auth/login           → gl-auth-api
3. gl-auth-api      validates credentials (bcrypt) against PostgreSQL
4. gl-auth-api      generates JWT (access + refresh), stores refresh token in Redis
5. gl-auth-api      → { accessToken, refreshToken, userInfo }   → BFF
6. BFF              sets httpOnly cookies, returns 200 to Browser
```

**SSO flow (Google / GitHub):**

```
1. Browser          → redirected to OAuth2 consent screen (Google / GitHub)
2. Provider         → returns authorization code to frontend callback URL
3. Browser          → POST /api/auth/login { provider, authorizationCode } → BFF
4. BFF              → POST /v1/auth/login                                  → gl-auth-api
5. gl-auth-api      exchanges code for provider token, fetches user profile
6. gl-auth-api      upserts user + UserAuthProvider record in PostgreSQL
7. gl-auth-api      generates JWT, stores refresh token in Redis
8. gl-auth-api      → { accessToken, refreshToken, userInfo }              → BFF
9. BFF              sets httpOnly cookies, returns 200 to Browser
```

**SSO edge cases:**

| Scenario | `needRegister` | `confirmCode` | Next step |
|---|---|---|---|
| Existing user, same provider | `false` | null | Logged in |
| New user (email not found) | `true` | null | Frontend calls `POST /v1/users/signup` with one-time `token` |
| Email exists under different provider | `false` | confirm code | Frontend calls `GET /v1/auth/confirm-link?confirmCode=...` |

**Key security rules:**
- JWT access tokens: 15-minute expiry, stored in httpOnly cookies
- Refresh tokens: 7-day expiry, stored in Redis for fast validation and revocation
- `gl-auth-api` is the **only service** that handles authentication — other services do NOT include Spring Security
- Non-auth services receive `userId` via `X-User-Id` header set by the BFF after authentication

### 2.4 Video Delivery Strategy (YouTube Embed)

```
Admin enters YouTube URL
        │
        ▼
  gl-video-api  ──► PostgreSQL (video metadata + subtitles stored as rows)
        │
        ▼
   Next.js BFF  ──► Browser
                       │
                       ├──► YouTube <iframe> (video stream — direct from YouTube)
                       └──► Custom subtitle overlay (EN / VI toggle)
```

- No file hosting in MVP — videos are embedded from YouTube
- `youtubeVideoId` is extracted server-side from `embed_url` on creation
- Bilingual subtitles are structured PostgreSQL rows (not YouTube auto-captions), enabling future AI features
- In Phase 2+, self-hosted videos can be added without changing the data model

## 3. Technology Stack – MVP

| Layer | Technology | Rationale |
|---|---|---|
| **Frontend** | React 18 + Next.js 14 (App Router) | SSR, routing, API routes, image optimization |
| **UI Library** | Tailwind CSS + Radix UI | Rapid, accessible UI development |
| **State Mgmt** | Zustand + React Query | Client state + server state caching |
| **Video Embed** | YouTube IFrame API / oEmbed | No hosting cost, zero storage in MVP |
| **Backend** | Java 21 / Spring Boot 3.4.x | LTS Java, virtual threads, modern ecosystem |
| **ORM** | Spring Data JPA + Hibernate | Standard, migrations via Flyway |
| **Auth** | Spring Security + JWT (jjwt 0.12.x) | Industry standard |
| **SSO** | Google OAuth2 + GitHub OAuth2 | Already implemented in `gl-auth-api` |
| **Mapping** | MapStruct 1.6+ | Type-safe DTO mapping |
| **Database** | PostgreSQL 15+ | ACID, JSON support, full-text search |
| **Cache** | Redis 7+ | Token storage, rate limiting, response caching |
| **API Docs** | Springdoc OpenAPI 2.x | Auto-generated Swagger UI |
| **Testing** | JUnit 5 + Mockito (BE), Jest + RTL (FE) | Standard testing stack |

### Future Additions (Phase 2+)

| Technology | Purpose | Phase |
|---|---|---|
| MongoDB | AI-generated content, flexible schemas | Phase 3 |
| RabbitMQ | Async task processing (notifications) | Phase 2 |
| Kafka | Event streaming (analytics, audit logs) | Phase 3 |
| Node.js / Go | Notification & integration services | Phase 2 |
| Kubernetes | Container orchestration | Phase 3 |
| GitHub Actions | CI/CD pipeline | Phase 2 |

## 4. Backend Service Structure

Each backend service follows the same internal layout:

```
src/
├── main/
│   ├── java/{base_package}/<service-name>/
│   │   ├── <ServiceNameApplication>.java
│   │   ├── config/
│   │   │   ├── security/            # SecurityConfiguration, JWT filter, OAuth2 (auth only)
│   │   │   ├── cache/               # RedisConfig
│   │   │   ├── datasource/          # DataSourceConfig
│   │   │   └── validation/          # Custom validators
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   ├── model/
│   │   │   ├── entity/
│   │   │   ├── dto/request/
│   │   │   ├── dto/response/
│   │   │   └── mapper/              # MapStruct mappers
│   │   ├── constant/                # Enums, CodeResponse
│   │   └── exception/               # GlobalExceptionHandler, ApiException, ErrorResponse
│   └── resources/
│       ├── application.yml
│       ├── application-local.yml    # Local dev overrides (gitignored)
│       └── db/migration/            # Flyway SQL migrations (V1__, V2__, ...)
└── test/
    └── java/{base_package}/<service-name>/
        ├── controller/
        ├── service/
        └── integration/
```

## 5. Deployment Architecture (MVP – Local Server)

```
User (LAN / Internet)
        │
        ▼
┌───────────────────────────────────────────────────────────────────┐
│                  Docker Compose  (Local Server)                    │
│                                                                    │
│  ┌──────────┐    ┌──────────────┐  ┌─────────────────────────┐   │
│  │  Nginx   │──► │ gl-frontend  │  │      gl-auth-api         │   │
│  │ (proxy)  │    │  (Next.js)   │─►│   (Spring Boot :8080)    │   │
│  └──────────┘    └──────────────┘  └────────────┬────────────┘   │
│                                                   │               │
│  Routing:                          ┌──────────────▼────────────┐  │
│  /auth/* → gl-auth-api            │      gl-video-api          │  │
│  /api/videos/* → gl-video-api     │   (Spring Boot :8082)      │  │
│  /* → gl-frontend                 └──────────────┬────────────┘  │
│                                                   │               │
│                          ┌────────────────────────▼────────────┐  │
│                          │  PostgreSQL (2 schemas) │  Redis     │  │
│                          └─────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘

External:  gl-frontend  ──────────► YouTube (video embed)
           gl-auth-api  ──────────► Google / GitHub OAuth2
```

Two PostgreSQL schemas (`glauth` + `glvideo`) can run in the same PostgreSQL instance to keep infra simple while respecting service boundaries.

## 6. Security Considerations

| Area | MVP Implementation |
|---|---|
| **Authentication** | JWT with httpOnly cookies, bcrypt password hashing |
| **Authorization** | Role check (LEARNER / ADMIN) via Spring Security in gl-auth-api; header-based in gl-video-api |
| **Input validation** | Bean Validation (Jakarta) on all DTOs |
| **SQL injection** | Parameterized queries via JPA |
| **XSS** | React auto-escaping + CSP headers |
| **CSRF** | SameSite cookies + CSRF token for mutations |
| **Rate limiting** | Redis-based rate limiter on auth endpoints |
| **HTTPS** | Enforced in production via Nginx |
| **YouTube embed** | Allowlist YouTube domains in CSP; validate YouTube URL format on input |

---

*See also: [frontend/architecture.md](./frontend/architecture.md)*
