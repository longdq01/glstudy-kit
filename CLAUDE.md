# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Repository Role

`glstudy-kit` is the **documentation hub** for the GLStudy platform. It does not contain application code. The three service repos live alongside it:

| Repo | Path | Purpose |
|---|---|---|
| `gl-auth-api` | `../gl-auth-api/` | Spring Boot auth microservice (JWT, OAuth2) |
| `gl-video` | `../gl-video/` | Spring Boot video management microservice |
| `glstudy-frontend` | `../glstudy-frontend/` | Next.js 14 frontend |

---

## Key Documentation Files

**Always read `docs/AGENTS.md` before making structural changes.** It defines mandatory rules for keeping docs in sync when code changes.

| File | What it governs |
|---|---|
| `docs/AGENTS.md` | AI instructions, mandatory update rules, backend conventions |
| `docs/WORKLOG.md` | Vision, feature roadmap, sprint plan, active task tracking — check before starting work |
| `docs/architecture.md` | System diagram, tech stack |

Service-specific docs (schema, API spec, dev setup) live in `docs/{service_name}/`:

| Service folder | Files |
|---|---|
| `docs/gl-auth-api/` | `README.md`, `schema.md`, `api-specification.md` |
| `docs/gl-video/` | `README.md`, `schema.md`, `api-specification.md` |
| `docs/frontend/` | `README.md`, `design-system.md` |

---

## Architecture Overview

```
Browser
  └→ Next.js 14 (BFF + UI, port 3000)
       ├→ gl-auth-api  (Spring Boot, port 8080) — only service that issues/validates JWTs
       └→ gl-video     (Spring Boot, port 8082) — trusts BFF; receives userId via X-User-Id header
```

- Non-auth services have **no Spring Security** — authentication is done by the BFF before proxying
- Each service has its own PostgreSQL schema; schemas are never shared
- Redis is used by auth for token storage and by video for caching

---

## Mandatory Documentation Update Rules (from AGENTS.md)

| Trigger | Files to update |
|---|---|
| JPA `@Entity` added/modified | `docs/{service}/schema.md` |
| REST endpoint added/modified | `docs/{service}/api-specification.md` |
| New Maven dependency | `docs/architecture.md` § Tech Stack |
| New `application.yml` / `.env` key | `docs/{service}/README.md` |
| New service-to-service call | `docs/architecture.md` architecture diagram |

---

## Backend Conventions (Spring Boot services)

**Package structure** (base: `{base_package}.{service_name}/`):
```
config/  controller/  service/  repository/
model/
  entity/  dto/request/  dto/response/  mapper/
constant/  exception/
```

**Naming**:
- Entity: `PascalCase` → table: `snake_case` plural
- Service interface: `{Resource}Service`, impl: `{Resource}ServiceImpl`
- DTOs: `{Action}{Resource}Request` / `{Resource}Response`

**Error handling** — every service uses:
- `ApiException` (custom RuntimeException with `CodeResponse`)
- `GlobalExceptionHandler` (`@RestControllerAdvice`)
- `CodeResponse` enum with Vietnamese messages: `throw ApiException.ErrNotFound().message("...").build();`

**Standard response envelope**:
```json
{ "success": true, "data": {...}, "error": null, "timestamp": "ISO-8601" }
```
Reference: `BaseResponse` in `gl-auth-api`.

**Primary keys**: Users use `String` nanoid (`com.aventrix.jnanoid`); other entities use `UUID` or `BIGINT`.

## Active Technologies
- Java 21 (backend), TypeScript / Node.js 20 (frontend) (001-user-auth)
- PostgreSQL 15 (`glauth` schema), Redis 7 (session/token store) (001-user-auth)

## Recent Changes
- 001-user-auth: Added Java 21 (backend), TypeScript / Node.js 20 (frontend)
