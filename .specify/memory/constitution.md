<!--
SYNC IMPACT REPORT
==================
Version change: 1.0.0 → 1.1.0 (doc structure update)

Changed in §IV (Documentation Consistency):
  - Removed references to deleted files: docs/03-database-schema.md, docs/04-api-specification.md,
    docs/00-developer-setup.md
  - Updated docs/{service}/api.md → docs/{service}/api-specification.md
  - Updated docs/02-architecture.md → docs/architecture.md
  - Added explicit docs/ file hierarchy for clarity
  - Added bidirectional sync rule between constitution and docs/

Changed in Technology Stack Constraints:
  - Updated ADR file reference: docs/02-architecture.md → docs/architecture.md

Deferred TODOs:
  - TODO(RATIFICATION_DATE): original project adoption date is unknown.
    Update manually once the team confirms the effective date.
-->

# GLStudy Constitution

## Core Principles

### I. Microservice Independence

Each backend service (`gl-auth-api`, `gl-video-api`, and any future service) MUST:

- Own its own PostgreSQL schema. Cross-service direct database access is forbidden.
- Communicate with peer services exclusively via REST contracts or event streams — no in-process
  calls or shared libraries that couple deployment cycles.
- Be independently deployable, buildable, and testable without changes to sibling services.
- Expose a consistent response envelope `{ success, data, error, timestamp }` so the BFF can
  aggregate responses uniformly.

**Rationale**: Hard service boundaries prevent the distributed-monolith anti-pattern, allow
independent scaling, and keep the blast radius of failures contained to a single service.

### II. Security-First Authentication

- `gl-auth-api` is the ONLY service authorized to issue or validate JWTs. All other services MUST
  NOT include Spring Security or perform any JWT validation.
- Access tokens MUST be stored in httpOnly secure cookies. Use of localStorage or sessionStorage
  for tokens is forbidden.
- Refresh tokens MUST be stored in Redis to enable fast validation and immediate revocation.
- The Next.js BFF MUST authenticate every inbound request via `gl-auth-api` before proxying to
  downstream services, and MUST inject `X-User-Id` into the downstream request header.
- All DTOs MUST use Bean Validation (Jakarta). Parameterized queries via JPA are mandatory;
  raw SQL string concatenation is forbidden.
- YouTube URLs MUST be validated against an allowlist format; CSP headers MUST restrict embed
  sources to YouTube domains.

**Rationale**: Centralizing authentication eliminates duplicated security logic, prevents XSS token
exposure, and enables token revocation across all sessions without service restarts.

### III. Unified Service Conventions

All backend services MUST conform to the canonical structure defined in `docs/AGENTS.md` § 3:

- **Package layout**: `config/`, `controller/`, `service/`, `repository/`, `model/`, `constant/`,
  `exception/` — no deviations without justification.
- **Response envelope**: `BaseResponse` (copied from `gl-auth-api`) for all endpoints.
- **Exception handling**: `ApiException` + `GlobalExceptionHandler` (copied from `gl-auth-api`).
- **Naming**: PascalCase entities; snake_case plural table names; `{Resource}Controller`,
  `{Resource}Service`, `{Resource}ServiceImpl`, `{Resource}Repository`, `{Resource}Mapper`.
- **Primary keys**: String (nanoid) for users; UUID or BIGINT sequence for all other entities.

New services MUST copy `exception/`, `constant/CodeResponse`, and `model/dto/response/BaseResponse`
from `gl-auth-api` as the reference implementation.

**Rationale**: Uniform conventions reduce cognitive overhead when context-switching between services
and enable consistent API documentation, error handling, and onboarding.

### IV. Documentation Consistency

Every structural change MUST update the corresponding documentation in the same commit or PR:

- JPA entity / schema change → `docs/{service_name}/schema.md`
- REST endpoint change → `docs/{service_name}/api-specification.md`
- New Maven/npm dependency → tech stack table in `docs/architecture.md`
- New infrastructure component → architecture diagram in `docs/architecture.md` § 1
- New environment variable → `docs/{service_name}/README.md`

**Documentation structure** (each service owns its own docs — no shared schema or API files):

```
docs/
├── AGENTS.md                        ← Runtime guidance for developers and AI agents
├── WORKLOG.md                       ← Vision, feature roadmap, sprint plan, active task tracking
├── architecture.md                  ← System architecture, tech stack, deployment (AI updates)
├── gl-auth-api/
│   ├── README.md                    ← Service overview & dev setup
│   ├── schema.md                    ← Auth DB schema (AI updates)
│   └── api-specification.md         ← Auth API contracts (AI updates)
├── gl-video/
│   ├── README.md
│   ├── schema.md                    ← Video DB schema (AI updates)
│   └── api-specification.md         ← Video API contracts (AI updates)
└── frontend/
    ├── README.md                    ← Frontend overview & dev setup
    ├── architecture.md              ← Pages, components, state management
    └── design-system.md
```

**Bidirectional sync rule**: This constitution and `docs/` are co-authoritative.
- When `docs/` structure changes (files renamed, added, deleted) → update file paths in this
  constitution's §IV and in `docs/AGENTS.md` in the same PR.
- When a constitution principle changes the way docs are organized → update `docs/AGENTS.md`
  to reflect the new rules in the same PR.
- The constitution governs **what** must be documented; `docs/AGENTS.md` governs **how** and
  **where** to document it. Both must remain consistent.

Developers and AI agents MUST read `docs/AGENTS.md` before making any structural change.
Documentation updates are part of the definition of done — PRs without them MUST NOT be merged.

**Rationale**: Documentation that lags code creates inconsistency and increases onboarding cost;
keeping docs in sync as part of done ensures the shared knowledge base remains reliable.

### V. Test Coverage & Quality

- Backend unit test coverage MUST remain above 70% across all services.
- Backend tests MUST be organized into three layers: controller (MockMvc), service (Mockito),
  and integration (Spring Boot test slice).
- Frontend tests MUST use Jest + React Testing Library.
- Database schema changes MUST use Flyway with sequential version numbers (`V1__`, `V2__`, …);
  manual schema mutations without a migration file are forbidden.
- No feature is considered complete until all acceptance scenarios from the feature spec pass.
- LCP MUST remain below 2.5 seconds; API p95 response time MUST remain below 300 ms.

**Rationale**: Automated tests and migration discipline are the primary safety nets for a
multi-service platform; enforcing thresholds prevents regressions and schema drift as the
codebase grows.

## Technology Stack Constraints

The following choices are fixed for MVP. Any substitution requires an Architecture Decision Record
(ADR) documented in `docs/architecture.md` before implementation begins.

| Layer | Mandated Technology |
|---|---|
| Backend services | Java 21 + Spring Boot 3.4.x |
| ORM / migrations | Spring Data JPA + Flyway |
| Database | PostgreSQL 15+ (one schema per service) |
| Cache / sessions | Redis 7+ |
| DTO mapping | MapStruct 1.6+ |
| Auth library | jjwt 0.12.x |
| API documentation | Springdoc OpenAPI 2.x |
| Frontend runtime | React 18 + Next.js 14 (App Router) |
| UI components | Tailwind CSS + Radix UI |
| State management | Zustand (client state) + React Query (server state) |
| Testing (BE) | JUnit 5 + Mockito |
| Testing (FE) | Jest + React Testing Library |
| Containerization | Docker Compose (local); Docker (production) |

Phase 2+ additions (RabbitMQ, Kafka, MongoDB, Kubernetes, GitHub Actions) MUST be staged and
documented before introduction; they MUST NOT be introduced in MVP sprints.

## Development Workflow

- Direct pushes to `main` are forbidden; all changes MUST go through a PR with at least one
  reviewer approval.
- `docs/WORKLOG.md` MUST be checked before starting new work to prevent duplicate effort.
- Local development MUST use Docker Compose (`docker-compose up -d`) for PostgreSQL and Redis.
- A feature is ready to merge only when: tests pass, coverage threshold is met, documentation
  is updated, and the spec's acceptance scenarios are verified.
- Complexity that deviates from this constitution MUST be justified with a rationale comment in
  the PR or an ADR entry. Unjustified deviations MUST be rejected in code review.

## Governance

This constitution supersedes all other project practices and guidelines. Any conflict between this
document and another guide resolves in favor of this constitution.

**Amendment procedure**:

1. Open a PR proposing the change with a clear rationale and impact assessment.
2. Increment `CONSTITUTION_VERSION` per semantic versioning:
   - MAJOR: principle removals, redefinitions, or backward-incompatible governance changes.
   - MINOR: new principles, new mandatory sections, or materially expanded guidance.
   - PATCH: clarifications, wording improvements, or typo fixes.
3. Set `LAST_AMENDED_DATE` to the merge date (ISO format YYYY-MM-DD).
4. Run the consistency propagation checklist (templates, agent files, `docs/AGENTS.md`).
5. Merge only after explicit approval from the project lead.

All PRs MUST verify compliance with the Core Principles before merging. Runtime development
guidance is maintained in `docs/AGENTS.md`.

**Version**: 1.1.0 | **Ratified**: TODO(RATIFICATION_DATE): confirm original adoption date with the team | **Last Amended**: 2026-03-11
