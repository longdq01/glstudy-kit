# glstudy-frontend – Architecture

## 1. Pages & Routes

| Route | Page | Auth | Description |
|---|---|---|---|
| `/` | Landing | No | Hero section, features, CTA to sign up |
| `/login` | Login | No | Email + password form + SSO buttons |
| `/register` | Register | No | Registration form + validation |
| `/dashboard` | Dashboard | Yes | Stats cards, recent videos, streak |
| `/videos` | Video Catalog | Yes | Grid with filters, search, pagination |
| `/videos/:id` | Video Player | Yes | Player + bilingual subtitles + progress tracking |
| `/profile` | Profile | Yes | Edit display name, avatar, change password |
| `/admin` | Admin Dashboard | Admin | User/video counts, basic analytics |
| `/admin/videos` | Video Management | Admin | CRUD table of all videos |
| `/admin/videos/new` | Add Video | Admin | YouTube URL + subtitle data form |
| `/admin/users` | User Management | Admin | List users, view details |

## 2. Source Directory Layout

```
src/
├── app/                           # Next.js App Router
│   ├── layout.tsx                 # Root layout (fonts, providers)
│   ├── page.tsx                   # Landing page
│   ├── (auth)/                    # Auth group (no sidebar)
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── (main)/                    # Main group (with sidebar/nav)
│   │   ├── layout.tsx
│   │   ├── dashboard/page.tsx
│   │   ├── videos/
│   │   │   ├── page.tsx           # Video catalog
│   │   │   └── [id]/page.tsx      # Video player
│   │   └── profile/page.tsx
│   ├── admin/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── videos/
│   │   │   ├── page.tsx
│   │   │   └── new/page.tsx
│   │   └── users/page.tsx
│   └── api/                       # BFF API routes
│       ├── auth/
│       │   ├── login/route.ts
│       │   ├── register/route.ts
│       │   ├── refresh/route.ts
│       │   └── logout/route.ts
│       ├── users/me/route.ts
│       └── videos/
│           ├── route.ts
│           └── [id]/
│               ├── route.ts
│               └── progress/route.ts
├── components/
│   ├── ui/                        # Base UI (Button, Input, Card, Badge, Modal, Skeleton, ...)
│   ├── layout/                    # Navbar, Sidebar, Footer, MobileNav
│   ├── auth/                      # LoginForm, RegisterForm, AuthGuard
│   ├── video/                     # VideoPlayer, SubtitleDisplay, VideoCard, VideoGrid, VideoFilters, ProgressBar
│   ├── dashboard/                 # StatsCard, StreakDisplay, RecentVideos, WelcomeBanner
│   └── admin/                     # VideoUploadForm, VideoTable, UserTable, AdminStatsCards
├── hooks/
│   ├── useAuth.ts                 # Auth state & actions
│   ├── useVideos.ts               # Video list fetching
│   ├── useVideoPlayer.ts          # Player state
│   ├── useSubtitles.ts            # Subtitle sync logic
│   ├── useWatchProgress.ts        # Auto-save progress
│   └── useDebounce.ts
├── lib/
│   ├── api-client.ts              # Axios wrapper with JWT interceptors
│   ├── auth.ts                    # Token helpers
│   ├── constants.ts
│   ├── formatters.ts              # Date, time, number formatters
│   └── validators.ts              # Zod schemas
├── stores/
│   ├── auth-store.ts              # Zustand — user, isAuthenticated
│   └── video-store.ts             # Zustand — player state (currentTime, subtitleMode, volume)
└── styles/
    └── globals.css
```

## 3. VideoPlayer Component

```
VideoPlayer
├── PlayerControls
│   ├── Play / Pause
│   ├── Volume
│   ├── Fullscreen
│   └── SubtitleToggle  (EN | VI | Both | Off)
├── SubtitleDisplay (EN)
├── SubtitleDisplay (VI)
└── ProgressTracker
    ├── Auto-save every 30s → POST /api/videos/:id/progress
    └── Mark completed at ≥ 90% of duration
```

**Subtitle overlay layout:**

```
┌─────────────────────────────────────────┐
│                                         │
│              Video Player               │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  Hi, can I get a coffee?          │  │  ← English
│  │  Xin chào, cho tôi một ly cà phê │  │  ← Vietnamese
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

- Subtitles sync to playback time via YouTube IFrame API `onStateChange`
- Click a subtitle line to seek to that timestamp
- Font size adjustable for accessibility

## 4. State Management

| Store | Library | What it holds | Persisted? |
|---|---|---|---|
| `auth-store` | Zustand | `user`, `isAuthenticated`, `isLoading` | Memory; rehydrated from cookie on SSR |
| `video-store` | Zustand | `isPlaying`, `currentTime`, `subtitleMode`, `volume` | No (local to player session) |
| Server data | React Query | Video list, video detail, user stats — cached + background refresh | React Query cache |

## 5. Responsive Breakpoints

| Breakpoint | Width | Layout |
|---|---|---|
| Mobile | < 640px | Single column, bottom nav |
| Tablet | 640–1024px | Two columns, collapsible sidebar |
| Desktop | > 1024px | Three columns, persistent sidebar |
