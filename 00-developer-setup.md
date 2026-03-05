# GLStudy – Developer Setup Guide

## Prerequisites

| Tool | Version | Install |
|---|---|---|
| Java JDK | 17+ | [Adoptium](https://adoptium.net/) |
| Node.js | 18+ | [Node.js](https://nodejs.org/) |
| Docker | 24+ | [Docker Desktop](https://docker.com/) |
| Git | 2.40+ | `apt install git` / `brew install git` |

## Quick Start

### 1. Clone & Setup

attach github url here 

### 2. Start Infrastructure

```bash
# Start PostgreSQL, Redis, MinIO
docker-compose up -d

# Verify containers are running
docker-compose ps
```

### 3. Start Backend

```bash
cd backend

# Copy environment config
cp src/main/resources/application-local.yml.example src/main/resources/application-local.yml

# Run with local profile
./mvnw spring-boot:run -Dspring-boot.run.profiles=local

# Backend: http://localhost:8080
# Swagger: http://localhost:8080/swagger-ui.html
```

### 4. Start Frontend

```bash
cd frontend

# Install dependencies
npm install

# Copy environment config
cp .env.example .env.local

# Run dev server
npm run dev

# Frontend: http://localhost:3000
```

## Docker Compose Services

| Service | Port | Credentials |
|---|---|---|
| PostgreSQL | 5432 | `postgres` / `postgres` / DB: `golden_event` |
| Redis | 6379 | no auth (dev only) |

## Environment Variables

### Backend (`application-local.yml`)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/golden_event
    username: postgres
    password: postgres
  redis:
    host: localhost
    port: 6379

app:
  jwt:
    secret: your-256-bit-secret-key-here
    access-token-expiry: 900        # 15 minutes
    refresh-token-expiry: 604800    # 7 days
```

### Frontend (`.env.local`)

```env
NEXT_PUBLIC_APP_NAME=GLStudy
NEXT_PUBLIC_API_URL=http://localhost:3000/api
BACKEND_API_URL=http://localhost:8080/api/v1
```

## Useful Commands

```bash
# Backend
./mvnw test                          # Run all tests
./mvnw test -pl auth                 # Run auth module tests
./mvnw flyway:migrate                # Run DB migrations
./mvnw flyway:clean                  # Reset DB (⚠️ destructive)

# Frontend
npm run dev                          # Dev server
npm run build                        # Production build
npm run test                         # Run Jest tests
npm run test:watch                   # Watch mode
npm run lint                         # ESLint check

# Docker
docker-compose up -d                 # Start infra
docker-compose down                  # Stop infra
docker-compose down -v               # Stop + delete volumes (⚠️)
docker-compose logs -f postgres      # Follow DB logs
```

## Project Documentation

| Document | Path | Description |
|---|---|---|
| Project Overview | [`docs/01-project-overview.md`](./01-project-overview.md) | Vision, roadmap, metrics |
| Architecture | [`docs/02-architecture.md`](./02-architecture.md) | System design, tech decisions |
| Database Schema | [`docs/03-database-schema.md`](./03-database-schema.md) | ER diagram, table definitions |
| API Specification | [`docs/04-api-specification.md`](./04-api-specification.md) | Endpoints, request/response |
| Frontend Architecture | [`docs/05-frontend-architecture.md`](./05-frontend-architecture.md) | Pages, components, design system |
| Implementation Roadmap | [`docs/06-implementation-roadmap.md`](./06-implementation-roadmap.md) | Sprint plan, task breakdown |
