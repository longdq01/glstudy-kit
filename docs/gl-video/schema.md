# gl-video-api – Database Schema

PostgreSQL schema name: **`glvideo`**

## Tables

### `videos`

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `UUID` | PK, `gen_random_uuid()` | Primary key |
| `title` | `VARCHAR(255)` | NOT NULL | Video title |
| `description` | `TEXT` | NULLABLE | Video description |
| `youtube_video_id` | `VARCHAR(50)` | NULLABLE | Extracted YouTube ID (e.g. `dQw4w9WgXcQ`) — set server-side |
| `embed_source` | `VARCHAR(20)` | NOT NULL, DEFAULT `'YOUTUBE'` | `YOUTUBE`, `VIMEO`, `OTHER` |
| `embed_url` | `VARCHAR(500)` | NOT NULL | Full embed URL (e.g. `https://youtube.com/watch?v=...`) |
| `thumbnail_url` | `VARCHAR(500)` | NULLABLE | Custom thumbnail; falls back to YouTube OG image |
| `language` | `VARCHAR(10)` | NOT NULL, DEFAULT `'en'` | Primary language of the video |
| `difficulty_level` | `VARCHAR(20)` | NOT NULL | `BEGINNER`, `INTERMEDIATE`, `ADVANCED` |
| `duration_seconds` | `INTEGER` | NOT NULL | Video duration |
| `category` | `VARCHAR(100)` | NULLABLE | Content category |
| `status` | `VARCHAR(20)` | NOT NULL, DEFAULT `'DRAFT'` | `DRAFT`, `PUBLISHED`, `ARCHIVED` |
| `created_by` | `VARCHAR` | FK → users.id (cross-service ref) | Admin user ID who created the entry |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT `NOW()` | Creation time |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT `NOW()` | Last update time |

**Indexes:**
- `idx_videos_status` on `status`
- `idx_videos_difficulty` on `difficulty_level`
- `idx_videos_category` on `category`
- `idx_videos_created_at` on `created_at DESC`

> `youtube_video_id` is extracted server-side from `embed_url` on creation so the frontend can build embed URLs (`https://www.youtube.com/embed/{id}`) without re-parsing.

---

### `subtitles`

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `UUID` | PK | Primary key |
| `video_id` | `UUID` | FK → `videos.id`, NOT NULL | Parent video |
| `language` | `VARCHAR(10)` | NOT NULL | `en` or `vi` |
| `sequence_number` | `INTEGER` | NOT NULL | Order within the video for the given language |
| `start_time` | `DECIMAL(10,3)` | NOT NULL | Start time in seconds |
| `end_time` | `DECIMAL(10,3)` | NOT NULL | End time in seconds |
| `content` | `TEXT` | NOT NULL | Subtitle text |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT `NOW()` | Creation time |

**Indexes:**
- `idx_subtitles_video_lang` on `(video_id, language)`
- `idx_subtitles_video_seq` on `(video_id, language, sequence_number)` (unique)
- `idx_subtitles_time_range` on `(video_id, language, start_time, end_time)`

**Constraints:**
- `UNIQUE (video_id, language, sequence_number)`
- `CHECK (end_time > start_time)`

---

### `video_watch_history`

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `UUID` | PK | Primary key |
| `user_id` | `VARCHAR` | FK cross-service (users.id), NOT NULL | Watcher |
| `video_id` | `UUID` | FK → `videos.id`, NOT NULL | Watched video |
| `watch_duration_seconds` | `INTEGER` | NOT NULL, DEFAULT `0` | Total seconds watched |
| `last_position` | `DECIMAL(10,3)` | NOT NULL, DEFAULT `0` | Resume position in seconds |
| `completed` | `BOOLEAN` | NOT NULL, DEFAULT `FALSE` | `true` when `last_position >= 90%` of `duration_seconds` |
| `watched_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT `NOW()` | First watch timestamp |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT `NOW()` | Last progress update |

**Indexes:**
- `idx_watch_user_video` on `(user_id, video_id)` (unique)
- `idx_watch_user_completed` on `(user_id, completed)`
- `idx_watch_watched_at` on `watched_at DESC`

**Constraints:**
- `UNIQUE (user_id, video_id)` — one record per user per video; progress updates use UPSERT

---

### `user_stats`

Denormalized table to avoid expensive `COUNT(*)` queries on watch history. Updated via application logic whenever a video is marked completed.

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `UUID` | PK | Primary key |
| `user_id` | `VARCHAR` | FK cross-service, UNIQUE, NOT NULL | Stats owner |
| `total_videos_watched` | `INTEGER` | NOT NULL, DEFAULT `0` | Completed video count |
| `total_watch_time_seconds` | `INTEGER` | NOT NULL, DEFAULT `0` | Total time spent watching |
| `current_streak_days` | `INTEGER` | NOT NULL, DEFAULT `0` | Current consecutive daily learning streak |
| `longest_streak_days` | `INTEGER` | NOT NULL, DEFAULT `0` | All-time best streak |
| `last_active_date` | `DATE` | NULLABLE | Last day user watched a video |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT `NOW()` | Last update |

**Indexes:**
- `idx_user_stats_user_id` on `user_id` (unique)

---

## Flyway Migrations

```
db/migration/
├── V1__create_videos_table.sql
├── V2__create_subtitles_table.sql
├── V3__create_video_watch_history_table.sql
├── V4__create_user_stats_table.sql
└── V5__seed_sample_data.sql          (dev only)
```
