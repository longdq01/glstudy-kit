# Quickstart: User Authentication System

**Branch**: `001-user-auth`
**Date**: 2026-03-11

This guide covers how to run and test the authentication system locally.

---

## Prerequisites

- Docker + Docker Compose
- Java 21, Maven
- Node.js 20+, npm/yarn

---

## 1. Start Infrastructure

```bash
# From repo root — starts PostgreSQL and Redis
docker compose up -d

# Verify
docker ps  # should show postgres and redis containers running
```

**PostgreSQL** defaults: host `localhost:5432`, database `glstudy`, user `glstudy`, password `glstudy`
**Redis** defaults: host `localhost:6379`, no auth

---

## 2. Start `gl-auth-api`

```bash
cd gl-auth-api
mvn spring-boot:run
# or: mvn clean package -DskipTests && java -jar target/*.jar
```

Service starts on **port 8081** (verify against `application.yml` `server.port`).

Swagger UI: `http://localhost:8081/swagger-ui.html`

**Required environment variables** (set in your shell or `.env.local`):

| Variable | Description | Default |
|---|---|---|
| `SPRING_DATASOURCE_URL` | PostgreSQL JDBC URL | `jdbc:postgresql://localhost:5432/glstudy` |
| `SPRING_DATASOURCE_USERNAME` | DB username | `glstudy` |
| `SPRING_DATASOURCE_PASSWORD` | DB password | `glstudy` |
| `SPRING_REDIS_HOST` | Redis host | `localhost` |
| `SPRING_REDIS_PORT` | Redis port | `6379` |
| `JWT_SECRET` | HMAC secret for JWT signing | see `application.yml` |
| `GOOGLE_CLIENT_ID` | OAuth2 Google client ID | required for SSO |
| `GOOGLE_CLIENT_SECRET` | OAuth2 Google client secret | required for SSO |
| `GITHUB_CLIENT_ID` | OAuth2 GitHub client ID | required for SSO |
| `GITHUB_CLIENT_SECRET` | OAuth2 GitHub client secret | required for SSO |

Schema is managed automatically by Hibernate `ddl-auto: update` — no migration to run.

**Seed data**: Roles (`ROLE_LEARNER`, `ROLE_ADMIN`) are inserted via `data.sql` or application startup. Verify after start:
```sql
SELECT * FROM glauth.roles;
```

---

## 3. Start `glstudy-frontend`

```bash
cd glstudy-frontend
cp .env.example .env.local   # then edit values below
npm install
npm run dev
```

Frontend runs on **port 3000**.

**Required `.env.local` variables:**

| Variable | Value |
|---|---|
| `AUTH_API_URL` | `http://localhost:8081` |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Your Google OAuth2 app client ID |
| `NEXT_PUBLIC_GITHUB_CLIENT_ID` | Your GitHub OAuth2 app client ID |
| `GOOGLE_CLIENT_SECRET` | (server-side only, not prefixed) |
| `GITHUB_CLIENT_SECRET` | (server-side only, not prefixed) |
| `NEXTAUTH_SECRET` | Any random string ≥32 chars |
| `NEXT_PUBLIC_APP_URL` | `http://localhost:3000` |

**Note**: `AUTH_API_URL` was previously defaulted to `http://localhost:8080` in some BFF routes. The correct port is **8081** if `gl-auth-api` runs on 8081. Verify in `src/app/api/auth/login/route.ts`.

---

## 4. Configure OAuth2 Providers

### Google

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → APIs & Services → Credentials
2. Create OAuth 2.0 Client ID (Web application)
3. Add authorized redirect URI: `http://localhost:3000/auth/google/callback`
4. Copy Client ID and Client Secret to env vars

### GitHub

1. Go to GitHub → Settings → Developer Settings → OAuth Apps
2. Create new OAuth App
3. Homepage URL: `http://localhost:3000`
4. Authorization callback URL: `http://localhost:3000/auth/github/callback`
5. Copy Client ID and Client Secret to env vars

---

## 5. Verify the System

### Email Registration

1. Open `http://localhost:3000/login` → Register tab
2. Fill in username, display name, email, password
3. Submit → should redirect to `/dashboard` as logged-in user

### Email Login

1. Open `http://localhost:3000/login` → Login tab
2. Enter credentials → redirected to dashboard

### SSO Login

1. Click "Sign in with Google" or "Sign in with GitHub"
2. Complete OAuth2 consent
3. If new user: shown username-selection form
4. If existing account under different provider: shown confirm-link screen

### Session Persistence

1. Log in → close browser tab → reopen `http://localhost:3000/dashboard`
2. Should be logged in without re-entering credentials

### Availability Check

The registration form calls check-email and check-username as you type. Verify in browser DevTools → Network tab that requests go to `/api/users/check-email` and `/api/users/check-username`.

---

## 6. Run Tests

```bash
# Backend unit + integration tests
cd gl-auth-api
mvn test

# Backend test with coverage report (target/site/jacoco/index.html)
mvn test jacoco:report
```

Coverage threshold: **≥70%** (enforced by JaCoCo in `pom.xml`).

```bash
# Frontend tests
cd glstudy-frontend
npm test
```

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `401 UNAUTHORIZED` on every request | Redis not running, or JWT secret mismatch |
| Schema not created | Hibernate `ddl-auto` not set to `update`; check `application.yml` |
| SSO redirect fails | Redirect URI in OAuth2 app must match exactly (trailing slash matters) |
| `AUTH_API_URL` connection refused | Verify `gl-auth-api` port in `application.yml` matches `AUTH_API_URL` in frontend |
| Role not found on registration | Seed data not applied; check `data.sql` execution in startup logs |
