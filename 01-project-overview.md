# GLStudy – Project Overview

> **An English-learning platform designed for Vietnamese learners**, combining video shadowing, bilingual subtitles, grammar lessons, online testing, and a progressive feature roadmap toward AI-powered language mastery.

---

## 1. Vision & Goals

| Aspect | Description |
|---|---|
| **Target audience** | Vietnamese learners of English (beginner → intermediate) |
| **Core value prop** | Learn English through real video content with Vietnamese bilingual subtitles |
| **Differentiator** | Shadowing-first approach, bilingual subtitles, grammar + testing in one platform |
| **Long-term vision** | Full AI-integrated language platform (conversation, grammar, testing, analytics) |

## 2. Feature Roadmap

### Phase 1 – MVP ✅ *(Target: this release)*

| # | Feature | Priority |
|---|---|---|
| F1 | Sign up / Log in / Log out – **email + password & SSO (Google, GitHub)** | **Must** |
| F2 | View & edit user profile | **Must** |
| F3 | Browse & watch learning videos (YouTube embedded) | **Must** |
| F4 | Bilingual subtitles (EN + VI) on video player | **Must** |
| F5 | Track "videos watched" per user | **Must** |
| F6 | Admin: add & manage video entries (YouTube URL + subtitles, no file upload) | **Must** |
| F7 | Responsive, mobile-friendly UI | **Must** |

### Phase 2 – Enhanced Learning 📚

| # | Feature |
|---|---|
| F8 | Role-based access control (learner, admin, moderator) |
| F9 | Grammar lessons & interactive exercises |
| F10 | Online test/exam room with basic auto-grading |
| F11 | Video controls: loop segments, playback speed |
| F12 | Bookmark / favorite videos |
| F13 | Learning streaks & achievements |

### Phase 3 – AI & Scale 🤖

| # | Feature |
|---|---|
| F14 | AI-generated transcripts for videos |
| F15 | AI conversation partner / virtual assistant |
| F16 | Advanced exam room (adaptive difficulty, analytics) |
| F17 | Push notifications (new content, streak reminders) |
| F18 | Analytics dashboard for admins |

## 3. Success Metrics (MVP)

| Metric | Target |
|---|---|
| User registration → first video watched | > 60% conversion |
| Average session duration | > 5 minutes |
| Page load time (LCP) | < 2.5 seconds |
| API response time (p95) | < 300ms |
| Unit test coverage | > 70% |

## 4. Constraints & Assumptions

- **MVP uses 2 microservices**: `gl-auth-api` (authentication & user management) + `gl-video-api` (video & subtitle management). Each service owns its own database schema.
- **Phase 1 uses embedded video only** – videos are linked from YouTube (no file upload or hosting). Self-hosted video delivery deferred to Phase 2+.
- Subtitles are stored as structured DB rows (not relying on YouTube auto-captions) to support bilingual toggle and future AI features.
- The platform initially supports **web only**; mobile apps are out of scope for MVP.
- Content is curated by admins (no user-generated content in MVP).
- SSO is supported from day one (Google + GitHub via OAuth2), alongside local email/password auth.

---

*Next: [02-architecture.md](./02-architecture.md)*
