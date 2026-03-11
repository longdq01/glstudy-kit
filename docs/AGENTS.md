# GLStudy – AI Agent Instructions

> This file instructs AI coding assistants (e.g. Claude Code, Copilot, Cursor) on the rules for
> maintaining **architecture and documentation consistency** across all GLStudy services.
>
> **⚠️ Always read this file before making any structural changes to a service.**
>
> **❓ AI must ask clarifying questions for any unclear or ambiguous problem before proceeding.**
>
> **📜 This file is governed by `.specify/memory/constitution.md`.** The constitution is the
> supreme source of principles. This file implements those principles as actionable rules.
> When they conflict, the constitution wins. When docs structure changes, both must be updated.

---

## 1. Project Overview

GLStudy is a multi-service project. Each service is an independent Git repository, but they share unified **project-level docs** in the `docs/` folder of this repo:

| Doc file | Purpose |
|---|---|
| [WORKLOG.md](./WORKLOG.md) | Vision, feature roadmap, sprint plan, active task tracking |
| [architecture.md](./architecture.md) | **System architecture** — service interactions, tech stack, deployment |

Each service has its **own docs** inside `docs/{service_name}/`:

| Service | Service docs path | Contents |
|---|---|---|
| `gl-auth-api` | `docs/gl-auth-api/` | `README.md`, `schema.md`, `api-specification.md` |
| `gl-video` | `docs/gl-video/` | `README.md`, `schema.md`, `api-specification.md` |
| `frontend` | `docs/frontend/` | `README.md`, `design-system.md` |

> **AI Note**: Each service's database schema and API spec live **only** in that service's `docs/{service_name}/` subfolder. There is no unified schema or API file — each service owns its own data and contracts.

---

## 2. Mandatory Update Rules

### Rule 1 — Entity / Schema Change
When you add, rename, or modify a JPA `@Entity` in any service:

1. **Update** → `docs/{service_name}/schema.md`
   - Add/modify the table definition, columns, indexes, constraints

### Rule 2 — API Endpoint Change
When you add, modify, or remove a REST endpoint in any service:

1. **Update** → `docs/{service_name}/api-specification.md`
   - Request / response schema, validation rules, error codes

### Rule 3 — New Dependency
When you add a new library (Maven dependency) to a service:

1. **Update the tech stack table** → `docs/architecture.md` § 3. Technology Stack
2. If it introduces a new infrastructure component (e.g. a message broker, cache), update the **architecture diagram** in `docs/architecture.md` § 1

### Rule 4 — Service-to-Service Communication
When a service starts calling another service (REST, event, etc.):

1. **Update the architecture diagram** → `docs/architecture.md` § 1
2. Document the contract in both services' `api-specification.md`

### Rule 5 — New Environment Variable
When you add a new config key to `application.yml` or `.env`:

1. **Update** → `docs/{service_name}/README.md` § Dev Setup / Environment Variables

---

## 3. Backend Service Conventions

All backend services (Spring Boot) must follow these conventions for consistency:

### Package structure
```
{base_package}.{service_name}/
├── config/
│   ├── security/
│   ├── cache/
│   ├── datasource/
│   └── validation/
├── controller/
├── service/
├── repository/
├── model/
│   ├── entity/
│   ├── dto/
│   │   ├── request/
│   │   └── response/
│   └── mapper/
├── constant/
└── exception/
```

### Naming conventions
| Item | Convention | Example |
|---|---|---|
| Entity class | `PascalCase` | `UserAuthProvider` |
| Table name | `snake_case`, plural | `user_auth_providers` |
| Controller | `{Resource}Controller` | `VideoController` |
| Service interface | `{Resource}Service` | `VideoService` |
| Service impl | `{Resource}ServiceImpl` | `VideoServiceImpl` |
| Repository | `{Resource}Repository` | `VideoRepository` |
| Mapper | `{Resource}Mapper` | `VideoMapper` |
| DTO (request) | `{Action}{Resource}Request` | `CreateVideoRequest` |
| DTO (response) | `{Resource}Response` or `{Resource}DTO` | `VideoResponse` |

### Primary key style
- **Users**: `String` (nanoid via `com.aventrix.jnanoid`) — already used in `gl-auth-api`
- **Other entities**: `UUID` (PostgreSQL `gen_random_uuid()`) or sequence-based `BIGINT`

### Standard response envelope
All services return:
```json
{
  "success": true | false,
  "data": { ... } | null,
  "error": null | { "code": "...", "message": "..." },
  "timestamp": "ISO-8601"
}
```
See `BaseResponse` in `gl-auth-api` for the reference implementation.

### Authentication & security across services
- `gl-auth-api` is the **only service** that handles authentication (JWT issuing + validation)
- Other services do **NOT** include Spring Security — they are accessed only through the BFF (Next.js API Routes), which authenticates requests via `gl-auth-api` before proxying
- Non-auth services receive `userId` via `X-User-Id` header set by the BFF after authentication

### Exception & error handling structure
All services must use the same exception handling pattern:

```
exception/
├── ApiException.java           ← Custom RuntimeException with CodeResponse
├── ErrorResponse.java          ← Error response DTO
└── GlobalExceptionHandler.java ← @RestControllerAdvice
```

**`CodeResponse`** enum (in `constant/`):
```java
OK(200, "OK", "Thành công"),
NOT_FOUND(404, "NotFound", "Không tìm thấy thông tin yêu cầu"),
INVALID_ARGUMENT(400, "InvalidArgument", "Tham số không hợp lệ"),
INTERNAL(500, "InternalServerError", "Có lỗi xảy ra"),
FORBIDDEN(403, "Forbidden", "Bạn không có quyền truy cập tài nguyên"),
UNAUTHORIZED(401, "Unauthorized", "Không xác thực được thông tin người dùng"),
BAD_REQUEST(400, "BadRequest", "Yêu cầu không hợp lệ"),
EXISTED(400, "Existed", "Đã tồn tại")
```

**`ApiException`** usage: `throw ApiException.ErrNotFound().message("Video not found").build();`

**`BaseResponse`** (in `model/dto/response/`):
```java
BaseResponse.success(data)   // → 200 with data
BaseResponse.ok()            // → 200 with null data
```

> **AI Rule**: When adding a new service, copy `exception/`, `constant/CodeResponse`, and `model/dto/response/BaseResponse` from `gl-auth-api` as the reference implementation.

---

## 4. Documentation File Hierarchy

```
docs/
├── AGENTS.md                      ← YOU ARE HERE (AI reads this first)
├── WORKLOG.md                     ← Vision, roadmap, sprint plan, active task tracking
├── architecture.md             ← System-level architecture (AI updates this)
├── gl-auth-api/
│   ├── README.md                  ← Service overview & dev setup
│   ├── schema.md                  ← Auth DB schema (AI updates this)
│   └── api-specification.md       ← Auth API spec (AI updates this)
├── gl-video/
│   ├── README.md                  ← Service overview & dev setup
│   ├── schema.md                  ← Video DB schema (AI updates this)
│   └── api-specification.md       ← Video API spec (AI updates this)
└── frontend/
    ├── README.md                  ← Frontend overview & dev setup
    ├── architecture.md            ← Pages, components, state management
    └── design-system.md           ← Component library, design tokens
```

---

## 5. When In Doubt

- **Assume the change affects the service docs** — it's better to update more than less
- **Never change a table name or entity field** without updating `docs/{service_name}/schema.md`
- **Check `docs/WORKLOG.md`** before starting work to see what's already in progress
