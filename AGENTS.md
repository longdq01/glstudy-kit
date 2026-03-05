# GLStudy – AI Agent Instructions

> This file instructs AI coding assistants (e.g. Antigravity, Copilot, Cursor) on the rules for
> maintaining **architecture and documentation consistency** across all GLStudy services.
> 
> **⚠️ Always read this file before making any structural changes to a service.**
>
> **❓ AI must ask clarifying questions for any unclear or ambiguous problem before proceeding.**

---

## 1. Project Overview

GLStudy is a multi-service project. Each service is an independent Git repository and Spring Boot application, but they share one **unified project documentation** in the `docs/` folder:

| Doc file | Purpose |
|---|---|
| [01-project-overview.md](./01-project-overview.md) | Vision, roadmap, MVP features |
| [02-architecture.md](./02-architecture.md) | **System architecture** — service interactions, tech stack, deployment |
| [03-database-schema.md](./03-database-schema.md) | **Unified DB schema** — all entities across all services |
| [04-api-specification.md](./04-api-specification.md) | **API contracts** — all public endpoints |
| [05-frontend-architecture.md](./05-frontend-architecture.md) | Frontend routing, components, state management |
| [06-implementation-roadmap.md](./06-implementation-roadmap.md) | Sprint plan and task breakdown |

Each service also has its **own docs** inside `docs/{service_name}/`:

| Service | Service docs path |
|---|---|
| `gl-auth` | `docs/gl-auth/` (README.md, schema.md, api.md) |
| `gl-video` | `docs/gl-video/` (README.md, schema.md, api.md) |
| `gl-frontend` | `docs/gl-frontend/` (README.md) |

> **AI Note**: These general docs (`00-*` through `06-*`, `AGENTS.md`, `WORKLOG.md`) are shared across **all service repos**. Each service repo contains the full `docs/` folder. The `{service_name}/` subfolder within `docs/` holds docs specific to that service. When referencing service-specific docs, always use `docs/{service_name}/` relative to the repo root.

---

## 2. Mandatory Update Rules

### Rule 1 — Entity / Schema Change
When you add, rename, or modify a JPA `@Entity` in any service:

1. **Update the service-specific schema** → `docs/{service_name}/schema.md`
   - Add/modify the table definition, columns, indexes, constraints
   - Update the ER diagram if relationships changed
2. **Update the project-level schema** → `docs/03-database-schema.md`
   - Update the ER diagram and corresponding table definition
   - Note which service owns the table

### Rule 2 — API Endpoint Change
When you add, modify, or remove a REST endpoint in any service:

1. **Update the service-specific API doc** → `docs/{service_name}/api.md`
   - Request / response schema, validation rules, error codes
2. **Update the project-level API spec** → `docs/04-api-specification.md`
   - Keep the endpoint summary table current

### Rule 3 — New Dependency
When you add a new library (Maven dependency) to a service:

1. **Update the tech stack table** → `docs/02-architecture.md` § 3. Technology Stack
2. If it introduces a new infrastructure component (e.g. a message broker, cache), update the **architecture diagram** in `docs/02-architecture.md` § 1

### Rule 4 — Service-to-Service Communication
When a service starts calling another service (REST, event, etc.):

1. **Update the architecture diagram** → `docs/02-architecture.md` § 1
2. Document the contract in both services' `api.md` docs

### Rule 5 — New Environment Variable
When you add a new config key to `application.yml` or `.env`:

1. **Update** → `docs/00-developer-setup.md` § Environment Variables
2. If it's service-specific, also note it in `docs/{service_name}/README.md`

---

## 3. Backend Service Conventions

All backend services (SpringBoot for now) must follow these conventions for consistency:

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

### Authentication \u0026 security across services
- `gl-auth-api` is the **only service** that handles authentication (JWT issuing + validation)
- Other services (e.g. `gl-video`) do **NOT** include Spring Security and do **NOT** validate JWTs — they are accessed only through the BFF (Next.js API Routes), which authenticates requests via `gl-auth-api` before proxying to downstream services
- Non-auth services trust that requests reaching them have already been authenticated by the gateway/BFF layer
- Non-auth services receive `userId` via a request header (e.g. `X-User-Id`) set by the BFF after authentication

### Exception & error handling structure
All services must use the same exception handling pattern for consistency:

```
exception/
├── ApiException.java          ← Custom RuntimeException with CodeResponse
├── ErrorResponse.java         ← Error response DTO (code, status, message, validation errors)
└── GlobalExceptionHandler.java ← @RestControllerAdvice — catches ApiException, validation, uncaught
```

**`CodeResponse`** enum (in `constant/`):
```java
// Each entry: (HTTP status code, status string, Vietnamese message)
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
// Fields: code, message, status, data
BaseResponse.success(data)   // → 200
BaseResponse.ok()            // → 200 with null data
```

> **AI Rule**: When adding a new service, copy `exception/`, `constant/CodeResponse`, and `model/dto/response/BaseResponse` from `gl-auth-api` as the reference implementation. Add service-specific error codes to `CodeResponse` as needed.

---

## 4. Documentation File Hierarchy

```
docs/
├── AGENTS.md                      ← YOU ARE HERE (AI reads this first)
├── WORKLOG.md                     ← Cross-service work checklist
├── 00-developer-setup.md          ← How to run locally
├── 01-project-overview.md         ← Vision, roadmap
├── 02-architecture.md             ← System-level architecture (AI updates this)
├── 03-database-schema.md          ← All schemas (AI updates this)
├── 04-api-specification.md        ← All APIs (AI updates this)
├── 05-frontend-architecture.md    ← Frontend structure
├── 06-implementation-roadmap.md   ← Sprint plan
└── {service_name}/               ← Service-specific docs (varies per repo)
    ├── README.md              ← Service overview
    ├── schema.md              ← Service DB schema (AI updates this)
    └── api.md                 ← Service API spec (AI updates this)
```

> **AI Note**: This `docs/` folder is duplicated across all service repos (`gl-auth`, `gl-video`, `gl-frontend`, etc.). The general docs (numbered files, `AGENTS.md`, `WORKLOG.md`) are identical everywhere. Only the `{service_name}/` subfolder differs per repo — it contains docs for the specific service you are currently working in.

---

## 5. When In Doubt

- **Assume the change affects the project docs** — it's better to update more than less
- **Never change a table name or entity field** without updating both `docs/{service_name}/schema.md` AND `docs/03-database-schema.md`
- **Check `docs/WORKLOG.md`** before starting work to see what's already in progress
