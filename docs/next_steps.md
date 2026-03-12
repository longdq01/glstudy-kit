# GLStudy — Recommended Next Steps

> Based on full review of all 13 documentation files across `gl-auth-api`, `gl-video`, `glstudy-frontend`, system architecture, schema, API specs, design system, and the WORKLOG.

---

## Current State Summary

| Area | Status |
|---|---|
| **gl-auth-api** | Sprint 1–2 done (auth endpoints, SSO, JWT, RBAC). Missing: Flyway migrations, seed data, 3 test suites |
| **gl-video-api** | Not started at all |
| **gl-frontend** | Sprint 1 setup ✅, Sprint 2 partially done (login/register UI, navbar, profile, AuthGuard exist — but SSO wiring, BFF proxy routes for check-email/username/set-password, confirm-link, and SSO-register are missing) |

---

## Recommended Priority Order

### 🔴 Priority 1 — Complete Frontend Sprint 2 Auth (no backend changes)

These items are partially built or missing on the frontend, but the **backend already supports them**:

| Item | What to do |
|---|---|
| **Register: add `username` field** | Add explicit `username` field to register form, call `POST /api/users/check-username` for availability |
| **BFF routes: check-email, check-username, set-password** | Create 3 new API routes that proxy to `gl-auth-api` endpoints (`POST /v1/users/check-email`, `/v1/users/check-username`, `/v1/users/set-password`) |
| **SSO buttons wiring** | Wire Google/GitHub buttons to redirect to OAuth consent URLs (per SSO flow in `architecture.md`) |
| **OAuth2 callback BFF route** | Create `/api/auth/oauth2/[provider]/callback` to receive authorization code → forward to `POST /v1/auth/login` with `{ provider, authorizationCode }` |
| **SSO register page** | `/auth/sso-register` — shown when `needRegister=true` from SSO login; collects `username` → calls `POST /v1/users/signup` with one-time `token` |
| **Confirm-link page** | `/auth/confirm-link` — shown when `confirmCode` returned; calls `GET /v1/auth/confirm-link?confirmCode=...` |
| **AuthGuard: `?from=` capture** | Preserve original URL in `?from=` param when redirecting to `/login`, then redirect back after successful login |
| **Inactive account handling** | Check for `account_inactive` error from login response → show appropriate error (per `?error=account_inactive` pattern) |
| **Profile: linked providers** | Display linked OAuth providers from `user.providers[]` array. Add "set password" form for SSO-only users (password is `null`) |

> **Why first?** All backend APIs already exist. This completes a full, functional auth system before building the video service.

---

### 🟡 Priority 2 — gl-auth-api Cleanup

These are small tasks to close out the auth service properly:

| Item | Effort | Notes |
|---|---|---|
| **Flyway migration scripts** | Low | Verify all migrations exist and are ordered (`V1__create_users.sql`, etc.) — or create them if using JPA schema auto-generation |
| **Seed data migration** | Low | `V*__seed_roles.sql` — insert `ROLE_LEARNER`, `ROLE_ADMIN` |
| **Unit test: LocalAuthenticationProvider** | Medium | Test credential validation, inactive account rejection |
| **Unit test: OAuth2AuthenticationProvider** | Medium | Test code exchange, user upsert, confirm-code flow |
| **Integration test: SSO mock flow** | Medium | Mock OAuth server, test full SSO login → register → confirm-link |

---

### 🟢 Priority 3 — Build gl-video-api (Sprint 1 + Sprint 3)

This is the **biggest blocker** for all remaining frontend work. Build in this order:

#### Sprint 1 — Scaffold
1. Spring Boot 3.4.x project scaffold (Java 21)
2. PostgreSQL `glvideo` schema + Redis config
3. **No Spring Security** — just read `X-User-Id` / `X-User-Role` headers from BFF
4. Copy GlobalExceptionHandler + ApiException from `gl-auth-api`
5. Springdoc OpenAPI + Swagger UI
6. MapStruct + Lombok

#### Sprint 3 — Video Entities & Endpoints
1. Create Flyway migrations per [schema.md](file:///d:/Long/workspace/GLStudy/glstudy-kit/docs/gl-video/schema.md):
   - `V1__create_videos_table.sql`
   - `V2__create_subtitles_table.sql`
   - `V3__create_video_watch_history_table.sql`
   - `V4__create_user_stats_table.sql`
   - `V5__seed_sample_data.sql` (5–10 sample YouTube videos with EN+VI subtitles)
2. YouTube URL parser service (extract video ID from various URL formats, auto-populate thumbnail)
3. Public endpoints: `GET /v1/videos` (paginated + filters + Redis cache), `GET /v1/videos/:id` (detail with subtitles)
4. Admin endpoints: `POST /v1/admin/videos`, `PUT /v1/admin/videos/:id`, `DELETE /v1/admin/videos/:id`
5. Watch progress: `POST /v1/videos/:id/progress` + auto-complete at 90%
6. Tests: YouTube URL parser, video CRUD, watch progress

---

### 🔵 Priority 4 — Frontend Sprint 3 (Video Catalog + Admin)

Once `gl-video-api` is running:

| Item | Description |
|---|---|
| **BFF video proxy routes** | Already exist (`/api/videos`, `/api/videos/[id]`, `/api/videos/[id]/progress`) — but need to wire them to `gl-video-api` at `BACKEND_VIDEO_URL` |
| **Video catalog page** | `/videos` with filters (difficulty, category), search, pagination — per API spec |
| **Admin layout + sidebar** | Admin route group with role-based access |
| **Admin video create** | YouTube URL + subtitle editor form at `/admin/videos/new` |
| **Admin video management** | CRUD table at `/admin/videos` |

---

### ⚪ Priority 5 — Sprint 4 & 5 (Video Player + Polish)

Sprint 4: Video player with YouTube IFrame API, bilingual subtitle overlay, click-to-seek, watch progress auto-save.

Sprint 5: Dashboard stats, admin dashboard, dark mode, skeletons, toasts, SEO, performance audit.

---

## Suggested Immediate Action

> [!IMPORTANT]
> **Start with Priority 1** — complete the frontend Sprint 2 auth features. All backend APIs are ready. This gives you a fully working auth system (local + SSO) before building the video service.

Alternatively, if you prefer backend work first:
- **Start with Priority 3** — scaffold `gl-video-api` to unblock all video-related frontend work in parallel.

**Which priority would you like to start with?**
