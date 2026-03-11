# GLStudy – Project Worklog

> Single source of truth for project vision, sprint plan, and active task tracking.
> Mark: `[ ]` todo · `[/]` in progress · `[x]` done · `[-]` blocked

---

## 1. Vision & Feature Roadmap

| Aspect | Description |
|---|---|
| **Target audience** | Vietnamese learners of English (beginner → intermediate) |
| **Core value** | Learn English through real video content with Vietnamese bilingual subtitles |
| **Differentiator** | Shadowing-first approach, bilingual subtitles, grammar + testing in one platform |
| **Long-term vision** | Full AI-integrated language platform |

### Phase 1 – MVP *(current)*

| Feature | Priority |
|---|---|
| Sign up / Log in / Log out — email + password & SSO (Google, GitHub) | Must |
| View & edit user profile | Must |
| Browse & watch learning videos (YouTube embedded) | Must |
| Bilingual subtitles (EN + VI) on video player | Must |
| Track "videos watched" per user | Must |
| Admin: add & manage video entries (YouTube URL + subtitles, no file upload) | Must |
| Responsive, mobile-friendly UI | Must |

### Phase 2 – Enhanced Learning

Grammar lessons & exercises · Online exam room · Video loop & speed controls · Bookmarks · Streaks & achievements · RBAC (moderator role) · CI/CD pipeline

### Phase 3 – AI & Scale

AI transcripts · AI conversation assistant · Adaptive exam room · Push notifications · Admin analytics dashboard · Kubernetes

---

## 2. Sprint Plan

MVP = **5 sprints × 2 weeks ≈ 10 weeks total**

| Sprint | Focus | Status |
|---|---|---|
| Sprint 1 | Project Setup & Infrastructure | 🔄 In Progress |
| Sprint 2 | Authentication System | ⬜ Not started |
| Sprint 3 | Video Management (Admin) | ⬜ Not started |
| Sprint 4 | Video Player & Bilingual Subtitles | ⬜ Not started |
| Sprint 5 | Dashboard, Statistics & Polish | ⬜ Not started |

**Current sprint**: Sprint 1 – Project Setup & Infrastructure

---

## 3. Work Items

### 🔐 gl-auth-api

> Repo: `/home/longdq/workspace/glstudy/gl-auth-api`
> Docs: [`docs/gl-auth-api/`](./gl-auth-api/README.md)

**Sprint 1 – Setup & Config**
- [x] Spring Boot 3.4.x project scaffold (Java 21)
- [x] PostgreSQL + Redis configuration
- [x] JWT service (jjwt 0.12.x) — access + refresh token
- [x] Spring Security config — JWT filter, custom entry points
- [x] Local authentication provider
- [x] OAuth2 providers — Google, GitHub
- [x] Springdoc OpenAPI / Swagger UI
- [x] GlobalExceptionHandler + ApiException
- [x] MapStruct + Lombok configuration

**Sprint 2 – Auth Entities & Endpoints**
- [x] `users` table — nanoid PK, username, email, password, SSO fields
- [x] `roles` + `permissions` tables — RBAC model
- [x] `user_roles` + `role_permissions` junction tables
- [x] `user_auth_providers` table — multi-provider SSO support
- [ ] Flyway migration scripts — verify all migrations exist and are ordered correctly
- [ ] Seed data migration — default roles (`ROLE_LEARNER`, `ROLE_ADMIN`)
- [x] `POST /auth/login` — unified local + SSO login
- [x] `POST /users/signup` — register
- [x] `POST /auth/refresh-token` — refresh access token
- [x] `POST /auth/logout` — revoke token
- [x] `GET /users/me` — get current user profile
- [x] `POST /users/check-email` — check email availability
- [x] `POST /users/check-username` — check username availability
- [x] `POST /users/set-password` — set password for SSO-only users
- [x] `GET /auth/confirm-link` — link OAuth provider to existing account
- [x] Inactive account enforcement (FR-019) — 401 on login, user_status Redis check (FR-020)

**Sprint 2 – Tests**
- [x] Unit tests: TokenServiceImpl (user_status check)
- [x] Unit tests: UserService (checkEmail, checkUsername, setPassword)
- [x] Unit tests: AuthenticationService (inactive account)
- [x] Controller tests: UserController (check-email, check-username, set-password)
- [x] Controller tests: AuthenticationController (inactive account login)
- [x] Integration tests: login → logout → blacklisted token → 401
- [ ] Unit tests: LocalAuthenticationProvider
- [ ] Unit tests: OAuth2AuthenticationProvider
- [ ] Integration tests: Google/GitHub SSO flow (mock OAuth server)

---

### 📹 gl-video-api

> Repo: `/home/longdq/workspace/glstudy/gl-video`
> Docs: [`docs/gl-video/`](./gl-video/README.md)

**Sprint 1 – Setup & Config**
- [ ] Spring Boot 3.4.x project scaffold (Java 21)
- [ ] PostgreSQL + Redis configuration
- [ ] No Spring Security — receives `X-User-Id` / `X-User-Role` from BFF
- [ ] Springdoc OpenAPI / Swagger UI
- [ ] GlobalExceptionHandler + ApiException (copy from gl-auth-api)
- [ ] MapStruct + Lombok

**Sprint 3 – Video Entities & Endpoints**
- [ ] `videos` table — `embed_url`, `youtube_video_id`, `embed_source`, metadata
- [ ] `subtitles` table — seq, start/end time, content, language
- [ ] Flyway migration scripts
- [ ] YouTube URL parser service (extract video ID, auto-populate thumbnail)
- [ ] `GET /v1/videos` — paginated list with filters + Redis cache
- [ ] `GET /v1/videos/:id` — detail with subtitles grouped by language
- [ ] `POST /v1/admin/videos` — create video entry (YouTube URL + subtitles)
- [ ] `PUT /v1/admin/videos/:id` — update video
- [ ] `DELETE /v1/admin/videos/:id` — delete
- [ ] Seed data — 5–10 sample YouTube videos with bilingual subtitles

**Sprint 4 – Watch Progress**
- [ ] `video_watch_history` table — upsert on re-watch, completed flag
- [ ] `user_stats` table — denormalized counters, streak tracking
- [ ] `POST /v1/videos/:id/progress` — upsert watch history + update stats on completion

**Tests**
- [ ] Unit tests: YouTube URL parser (various formats, invalid URLs)
- [ ] Unit tests: watch progress / completion logic
- [ ] Unit tests: streak calculation (edge cases: midnight, timezone)
- [ ] Integration tests: video CRUD flow
- [ ] Integration tests: watch progress → completion → stats update

---

### 🖥️ gl-frontend

> Repo: `/home/longdq/workspace/glstudy/glstudy-frontend`
> Docs: [`docs/frontend/`](./frontend/README.md)

**Sprint 1 – Setup & Config**
- [ ] Next.js 14 (App Router) + TypeScript scaffold
- [ ] Tailwind CSS + Radix UI setup
- [ ] Zustand store setup (auth-store, video-store)
- [ ] React Query setup (server state)
- [ ] Axios API client with interceptors (attach JWT, handle 401 refresh)
- [ ] ESLint + Prettier + Husky
- [ ] Frontend design system (Tailwind config, tokens, base components)
- [ ] Landing page (`/`)

**Sprint 2 – Auth Pages**
- [x] Login page + form validation (`/login`)
- [x] Register page + explicit username field, availability checks (`/login` register tab)
- [x] SSO buttons — Google, GitHub (OAuth2 redirect handlers wired)
- [x] OAuth2 callback BFF route (`/api/auth/oauth2/[provider]/callback`)
- [x] SSO register page — username selection for new SSO users (`/auth/sso-register`)
- [x] Confirm-link page + BFF route (`/auth/confirm-link`)
- [x] AuthGuard + protected routes + `?from=` URL capture
- [x] Inactive account error handling (`?error=account_inactive`)
- [x] BFF routes: check-email, check-username, set-password
- [x] Profile page — linked providers + set-password for SSO-only users (`/profile`)
- [ ] Navbar with auth state (user menu, login/logout)

**Sprint 3 – Video Catalog**
- [ ] Video catalog page with filters, search, pagination (`/videos`)
- [ ] Admin layout + sidebar
- [ ] Admin video create form — YouTube URL + subtitle editor (`/admin/videos/new`)
- [ ] Admin video management table — CRUD with status badges (`/admin/videos`)

**Sprint 4 – Video Player**
- [ ] Video player component — YouTube IFrame API + custom overlay (`/videos/:id`)
- [ ] Bilingual subtitle display + language toggle (EN / VI / Both / Off)
- [ ] Click-to-seek on subtitle lines
- [ ] Auto-save watch progress (30s interval via `useWatchProgress` hook)

**Sprint 5 – Dashboard & Polish**
- [ ] Dashboard page — stats cards, streak, recent + continue watching (`/dashboard`)
- [ ] Admin dashboard — user count, video count, recent activity (`/admin`)
- [ ] Admin user management table (`/admin/users`)
- [ ] Dark mode toggle (system preference + override)
- [ ] Loading skeletons for all pages
- [ ] Toast notifications (success/error feedback)
- [ ] SEO: meta tags + OG tags (SSR-friendly)
- [ ] Performance: Lighthouse audit + fixes

**Tests**
- [ ] Login form validation
- [ ] AuthGuard redirect test
- [ ] Video create form validation
- [ ] Dashboard render test
- [ ] Subtitle sync accuracy test
- [ ] Progress auto-save test (mock timer)
- [ ] E2E: register → browse → watch → dashboard

---

### 🏗️ Infrastructure

**Docker Compose**
- [x] `docker-compose.yml` — PostgreSQL, Redis
- [ ] `Dockerfile` for gl-auth-api
- [ ] `Dockerfile` for gl-video-api
- [ ] `Dockerfile` for gl-frontend
- [ ] Docker Compose service entries for all app containers
- [ ] Nginx config — reverse proxy + SSL (`infra/nginx/nginx.conf`)

---

## 4. Docs Consistency Checklist

> Run before closing any PR that makes structural changes to a service.

- [ ] Entity added/modified → `docs/{service}/schema.md` updated
- [ ] API endpoint added/modified → `docs/{service}/api-specification.md` updated
- [ ] New dependency added → `docs/architecture.md` tech stack table updated
- [ ] New env variable → `docs/{service}/README.md` updated
- [ ] Service-to-service communication changed → architecture diagram in `docs/architecture.md` updated

---

## 5. Risk Register

| Risk | Impact | Probability | Mitigation |
|---|---|---|---|
| YouTube API unavailability | Medium | Low | Graceful fallback message; plan self-hosted video for Phase 2 |
| Subtitle sync accuracy | Medium | Medium | Use precise timestamps (ms), manual QA |
| JWT token security | High | Low | httpOnly cookies, short expiry, Redis blacklist |
| Scope creep | High | High | Strict MVP feature gate, defer Phase 2 items |
| Solo developer bottleneck | Medium | High | Detailed docs, modular code for easy resumption |

---

## 6. Definition of Done (per Sprint)

- [ ] All tasks completed
- [ ] Unit test coverage ≥ 70% for new code
- [ ] No critical or high-severity bugs
- [ ] API documentation up to date in Swagger
- [ ] Responsive on mobile, tablet, and desktop
- [ ] Performance: page load < 2.5s, API response < 300ms (p95)
