# gl-video-api – Service Overview

Video catalog, subtitle management, and watch progress tracking microservice for the GLStudy platform.

## Responsibility

- Video catalog management (CRUD, filtering, search)
- Bilingual subtitle storage and retrieval (EN + VI)
- Watch progress tracking per user
- Denormalized user learning stats (streaks, total watch time)

> **Authentication**: This service does NOT include Spring Security. All requests are authenticated by the Next.js BFF via `gl-auth-api` before being proxied here. The authenticated `userId` is passed in via the `X-User-Id` request header.

## Runtime

| Property | Value |
|---|---|
| Port | **8082** |
| Base path | `/v1` |
| PostgreSQL schema | `glvideo` |
| Redis database | `2` |
| Swagger UI | `http://localhost:8082/swagger-ui.html` |

## Video Delivery

Videos are embedded from YouTube (or other external sources) — no file hosting is needed in MVP.

- Admin provides a YouTube URL when creating a video entry
- Backend extracts the `youtubeVideoId` from the URL automatically
- Frontend renders a YouTube `<iframe>` embed with a custom subtitle overlay
- Bilingual subtitles (EN + VI) are stored in PostgreSQL as structured rows

## Related Docs

- [Schema](./schema.md) – Video database schema
- [API](./api-specification.md) – Endpoint reference
- [System Architecture](../architecture.md)
