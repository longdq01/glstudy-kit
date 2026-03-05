# GLStudy – Work Log

> Track work-in-progress across all services. Update this file as you start/complete tasks.
> Mark: `[ ]` todo · `[/]` in progress · `[x]` done · `[-]` blocked

---

## 🔄 Active Sprint

> Update the sprint name when you start a new one.

**Current sprint**: Sprint 1 – Project Setup & Infrastructure

---

## 🗂️ Work Items by Service

### 🔐 gl-auth-api

> Service path: `/home/longdq/workspace/glstudy/gl-auth-api`
> Service docs: [`docs/gl-auth/`](./gl-auth/README.md)

#### Setup & Config
- [x] Spring Boot 3.4.x project scaffold (Java 21)
- [x] PostgreSQL + Redis configuration
- [x] JWT service (jjwt 0.12.x) — access + refresh token
- [x] Spring Security config — JWT filter, custom entry points
- [x] Local authentication provider
- [x] OAuth2 providers — Google, GitHub
- [x] Springdoc OpenAPI / Swagger UI
- [x] GlobalExceptionHandler + ApiException
- [x] MapStruct + Lombok configuration

#### Entities & Schema
- [x] `users` table — nanoid PK, username, email, password, SSO fields
- [x] `roles` + `permissions` tables — RBAC model
- [x] `user_roles` + `role_permissions` junction tables
- [x] `user_auth_providers` table — multi-provider SSO support
- [ ] Flyway migration scripts — verify all migrations exist and are ordered correctly
- [ ] Seed data migration — default roles (ROLE_LEARNER, ROLE_ADMIN)

#### API Endpoints
- [x] `POST /auth/login` — local login
- [x] `POST /auth/register` — sign up
- [x] `POST /auth/oauth2` — SSO token exchange (Google, GitHub)
- [x] `POST /auth/refresh` — refresh access token
- [x] `GET /users/me` — get current user profile
- [x] `PUT /users/me` — update profile
- [ ] `GET /users/check-email` — check email availability
- [ ] `GET /users/check-username` — check username availability

#### Tests
- [ ] Unit tests: JWT service
- [ ] Unit tests: LocalAuthenticationProvider
- [ ] Unit tests: OAuth2AuthenticationProvider
- [ ] Integration tests: register → login → refresh → logout flow
- [ ] Integration tests: Google/GitHub SSO flow (mock OAuth server)

---

### 📹 gl-video-api

> Service path: `/home/longdq/workspace/glstudy/gl-video`
> Service docs: [`docs/gl-video/`](./gl-video/README.md)

#### Setup & Config
- [ ] Spring Boot 3.4.x project scaffold (Java 21)
- [ ] PostgreSQL + Redis configuration
- [ ] JWT validation filter (verify tokens issued by gl-auth-api)
- [ ] Springdoc OpenAPI / Swagger UI
- [ ] GlobalExceptionHandler + standard error format
- [ ] MapStruct + Lombok

#### Entities & Schema
- [ ] `videos` table — embed_url, youtube_video_id, embed_source, metadata
- [ ] `subtitles` table — seq, start/end time, content, language
- [ ] `video_watch_history` table — upsert on re-watch, completed flag
- [ ] `user_stats` table — denormalized counters, streak tracking
- [ ] Flyway migration scripts
- [ ] Seed data — 5–10 sample YouTube videos with bilingual subtitles

#### API Endpoints
- [ ] `GET /videos` — paginated list with filters + Redis cache
- [ ] `GET /videos/:id` — detail with subtitles grouped by language
- [ ] `POST /admin/videos` — create video entry (YouTube URL + subtitles)
- [ ] `PUT /admin/videos/:id` — update video
- [ ] `DELETE /admin/videos/:id` — soft delete
- [ ] `POST /videos/:id/progress` — upsert watch history + update stats

#### Tests
- [ ] Unit tests: YouTube URL parser service
- [ ] Unit tests: watch progress / completion logic
- [ ] Unit tests: streak calculation (edge cases: midnight, timezone)
- [ ] Integration tests: video CRUD flow
- [ ] Integration tests: watch progress → completion → stats update

---

### 🖥️ gl-frontend

> Service repo: `glstudy/gl-frontend` *(not created yet)*
> Service docs: [`docs/gl-frontend/`](./{service_name}/README.md)

#### Setup & Config
- [ ] Next.js 14 (App Router) + TypeScript scaffold
- [ ] Tailwind CSS + Radix UI setup
- [ ] Zustand store setup (auth-store, video-store)
- [ ] React Query setup (server state)
- [ ] Axios/fetch API client with interceptors (attach JWT, handle 401 refresh)
- [ ] ESLint + Prettier + Husky

#### Pages & Features
- [ ] Landing page (`/`)
- [ ] Login page + form validation (`/login`)
- [ ] Register page + form validation (`/register`)
- [ ] SSO buttons — Google, GitHub (OAuth2 redirect)
- [ ] AuthGuard + protected routes
- [ ] Dashboard page (`/dashboard`)
- [ ] Video catalog page with filters (`/videos`)
- [ ] Video player + bilingual subtitle overlay (`/videos/:id`)
- [ ] Profile page (`/profile`)
- [ ] Admin: video management table (`/admin/videos`)
- [ ] Admin: add video form (`/admin/videos/new`)
- [ ] Admin: user management (`/admin/users`)

#### Tests
- [ ] Login form validation
- [ ] AuthGuard redirect test
- [ ] Video create form validation
- [ ] Dashboard render test

---

## 🏗️ Infrastructure (infra/)

- [x] `docker-compose.yml` — PostgreSQL, Redis
- [ ] Nginx config (`infra/nginx/nginx.conf`)
- [ ] `Dockerfile` for gl-auth-api
- [ ] `Dockerfile` for gl-video-api
- [ ] `Dockerfile` for gl-frontend
- [ ] docker-compose service entries for all containers

---

## 📋 Architecture & Docs Consistency Checklist

> Run this checklist whenever you make a significant change to a service.

- [ ] Entity added/modified → update `docs/{service_name}/schema.md` and [03-database-schema.md](./03-database-schema.md)
- [ ] API endpoint added/modified → update `docs/{service_name}/api.md` and [04-api-specification.md](./04-api-specification.md)
- [ ] New dependency added → update [02-architecture.md](./02-architecture.md) tech stack table
- [ ] New env variable → update [00-developer-setup.md](./00-developer-setup.md)
- [ ] Service-to-service communication changed → update architecture diagram in [02-architecture.md](./02-architecture.md)

---

*Last updated: 2026-02-27*
