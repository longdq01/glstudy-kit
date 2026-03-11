# glstudy-frontend – Service Overview

Next.js 14 frontend and Backend-for-Frontend (BFF) for the GLStudy platform.

## Responsibility

- **UI**: React-based pages for learners and admins
- **BFF**: Next.js API Routes proxy requests to backend microservices, set httpOnly cookies, and pass `X-User-Id` / `X-User-Role` headers to downstream services
- **SSR/SSG**: Server-side rendering for landing and video catalog pages

## Runtime

| Property | Value |
|---|---|
| Port | **3000** |
| API prefix | `/api` (BFF routes) |
| Auth cookie | httpOnly, SameSite=Strict |

## Stack

| Layer | Library |
|---|---|
| Framework | Next.js 14 (App Router), React 18, TypeScript |
| Styling | Tailwind CSS, Radix UI |
| Server state | TanStack React Query |
| Client state | Zustand |
| Forms | React Hook Form + Zod |
| HTTP | Axios |

## Pages

| Route | Auth | Description |
|---|---|---|
| `/` | No | Landing page |
| `/login` | No | Email/password + SSO login |
| `/register` | No | Registration form |
| `/dashboard` | Yes | Stats, streaks, recent videos |
| `/videos` | Yes | Video catalog with filters and search |
| `/videos/:id` | Yes | Video player with bilingual subtitle overlay |
| `/profile` | Yes | Edit profile, change password |
| `/admin` | Admin | Admin dashboard |
| `/admin/videos` | Admin | Video CRUD management |
| `/admin/users` | Admin | User listing |

## Dev Setup

```bash
npm install
cp .env.example .env.local
npm run dev        # http://localhost:3000
npm run build      # production build
npm run lint       # ESLint
npm test           # Jest
```

**Required env vars (`.env.local`):**
```env
NEXT_PUBLIC_APP_NAME=GLStudy
NEXT_PUBLIC_API_URL=http://localhost:3000/api
BACKEND_AUTH_URL=http://localhost:8080
BACKEND_VIDEO_URL=http://localhost:8082
```

## Related Docs

- [Design System](./design-system.md) – Component library and design tokens
- [Frontend Architecture](./architecture.md) – Pages, components, state management
- [System Architecture](../architecture.md)
