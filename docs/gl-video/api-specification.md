# gl-video-api – API Reference

**Base URL:** `http://localhost:8082`
**Swagger UI:** `http://localhost:8082/swagger-ui.html`

All responses follow the standard envelope:

```json
{
  "success": true | false,
  "data": { ... } | null,
  "error": null | { "code": "...", "message": "..." },
  "timestamp": "ISO-8601"
}
```

> **Auth**: No JWT validation in this service. The BFF authenticates the user via `gl-auth-api` and passes `userId` as `X-User-Id` header. Admin-only endpoints check the `X-User-Role` header.

---

## Videos (`/v1/videos`)

### GET `/v1/videos`

List videos with pagination and optional filters.

**Query parameters:**

| Param | Type | Default | Description |
|---|---|---|---|
| `page` | int | `0` | Page number (0-indexed) |
| `size` | int | `12` | Items per page (max 50) |
| `difficulty` | string | — | `BEGINNER`, `INTERMEDIATE`, `ADVANCED` |
| `category` | string | — | Filter by category |
| `search` | string | — | Full-text search in title / description |
| `sort` | string | `createdAt,desc` | Sort field and direction |

**Response (200):**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "video-uuid",
        "title": "Daily Conversations at a Café",
        "thumbnailUrl": "https://img.youtube.com/vi/dQw4w9WgXcQ/hqdefault.jpg",
        "difficultyLevel": "BEGINNER",
        "category": "conversation",
        "durationSeconds": 180,
        "createdAt": "2026-02-20T10:00:00Z"
      }
    ],
    "pagination": {
      "page": 0,
      "size": 12,
      "totalElements": 45,
      "totalPages": 4,
      "hasNext": true
    }
  }
}
```

---

### GET `/v1/videos/:videoId`

Get full video detail including subtitles and the requesting user's watch progress.

**Header:** `X-User-Id: <userId>` (optional — omit for unauthenticated browsing; watchProgress will be null)

**Response (200):**
```json
{
  "success": true,
  "data": {
    "id": "video-uuid",
    "title": "Daily Conversations at a Café",
    "description": "Learn how to order coffee...",
    "embedUrl": "https://www.youtube.com/embed/dQw4w9WgXcQ",
    "youtubeVideoId": "dQw4w9WgXcQ",
    "embedSource": "YOUTUBE",
    "thumbnailUrl": "https://img.youtube.com/vi/dQw4w9WgXcQ/hqdefault.jpg",
    "difficultyLevel": "BEGINNER",
    "category": "conversation",
    "durationSeconds": 180,
    "subtitles": {
      "en": [
        { "seq": 1, "start": 0.000, "end": 2.500, "text": "Hi, can I get a coffee?" }
      ],
      "vi": [
        { "seq": 1, "start": 0.000, "end": 2.500, "text": "Xin chào, cho tôi một ly cà phê được không?" }
      ]
    },
    "watchProgress": {
      "lastPosition": 45.200,
      "completed": false,
      "watchDurationSeconds": 50
    },
    "createdAt": "2026-02-20T10:00:00Z"
  }
}
```

> `watchProgress` is `null` if `X-User-Id` is absent or the user hasn't started watching.

---

### POST `/v1/videos/:videoId/progress`

Update the authenticated user's watch progress. Auto-marks `completed = true` when `currentPosition >= 90%` of `durationSeconds`.

**Header:** `X-User-Id: <userId>` (required)

**Request:**
```json
{
  "currentPosition": 120.500,
  "watchDurationSeconds": 125
}
```

**Response (200):**
```json
{
  "success": true,
  "data": {
    "lastPosition": 120.500,
    "watchDurationSeconds": 125,
    "completed": false,
    "totalVideosWatched": 12
  }
}
```

---

## Admin Videos (`/v1/admin/videos`)

All admin endpoints require `X-User-Role: ADMIN` header set by the BFF.

### POST `/v1/admin/videos`

Create a new video entry.

**Request:**
```json
{
  "title": "Daily Conversations at a Café",
  "description": "Learn how to order coffee and small talk in English",
  "embedUrl": "https://www.youtube.com/watch?v=dQw4w9WgXcQ",
  "difficultyLevel": "BEGINNER",
  "category": "conversation",
  "durationSeconds": 180,
  "subtitlesEn": [
    { "seq": 1, "start": 0.000, "end": 2.500, "text": "Hi, can I get a coffee?" }
  ],
  "subtitlesVi": [
    { "seq": 1, "start": 0.000, "end": 2.500, "text": "Xin chào, cho tôi một ly cà phê được không?" }
  ]
}
```

- `youtubeVideoId` is extracted from `embedUrl` automatically
- `thumbnailUrl` falls back to the YouTube OG image if not provided
- `subtitlesEn` / `subtitlesVi` are optional; can be added later

**Response (201):** Same structure as `GET /v1/videos/:videoId`.

---

### PUT `/v1/admin/videos/:videoId`

Update an existing video entry.

**Request:** Same fields as POST (all optional — partial update).

**Response (200):** Same structure as `GET /v1/videos/:videoId`.

---

### DELETE `/v1/admin/videos/:videoId`

Delete a video and all associated subtitles and watch history.

**Response (200):**
```json
{ "success": true, "data": null }
```

---

## Error Codes

| `CodeResponse` | HTTP | Status string | Message (VI) |
|---|---|---|---|
| `OK` | 200 | `OK` | Thành công |
| `INVALID_ARGUMENT` | 400 | `InvalidArgument` | Tham số không hợp lệ |
| `BAD_REQUEST` | 400 | `BadRequest` | Yêu cầu không hợp lệ |
| `NOT_FOUND` | 404 | `NotFound` | Không tìm thấy thông tin yêu cầu |
| `FORBIDDEN` | 403 | `Forbidden` | Bạn không có quyền truy cập tài nguyên |
| `INTERNAL` | 500 | `InternalServerError` | Có lỗi xảy ra |
