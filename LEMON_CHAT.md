# LEMON CHAT — Thiết kế Database & Hạ tầng (Infra)

> Ứng dụng chat riêng: nhắn tin realtime, voice/video call 1:1, gửi file, trạng thái online/offline,
> story 24h, note 24h. Đăng ký/đăng nhập bằng email + mật khẩu (có OTP qua SMTP), **username**
> (không ký tự đặc biệt) và đăng nhập bằng username. Quản lý token, cache, OTP theo triết lý
> **Redis-first**: OTP atomic Lua, refresh rotation, fail-closed vs best-effort cache.
>
> Trạng thái tài liệu: **DESIGN** — chưa scaffold code. Stack API: **NestJS**.
>
> Tài liệu liên quan: [RESPONSES.md](./RESPONSES.md) (chuẩn body success/error dùng chung) ·
> [AUTH.md](./AUTH.md) (luồng đăng ký/đăng nhập/quên mật khẩu/đổi mật khẩu chi tiết) ·
> [SETUP.md](./SETUP.md) (Bun toolchain, config, logging, error tracing).

---

## Mục lục

1. [Phạm vi sản phẩm](#1-phạm-vi-sản-phẩm)
2. [Stack kỹ thuật](#2-stack-kỹ-thuật)
3. [Kiến trúc tổng thể](#3-kiến-trúc-tổng-thể)
4. [Quy ước schema](#4-quy-ước-schema)
5. [Database schema (PostgreSQL)](#5-database-schema-postgresql)
6. [Redis data model (Redis-first)](#6-redis-data-model-redis-first)
7. [Auth: email + OTP + username](#7-auth-email--otp--username) — chi tiết [AUTH.md](./AUTH.md)
8. [Hệ thống bạn bè](#8-hệ-thống-bạn-bè)
9. [Nhắn tin realtime](#9-nhắn-tin-realtime)
10. [Presence (online/offline)](#10-presence-onlineoffline)
11. [File & media](#11-file--media)
12. [Voice/Video call 1:1 (WebRTC)](#12-voicevideo-call-11-webrtc)
13. [Story 24h & Note 24h](#13-story-24h--note-24h)
14. [Tầng infra (NestJS modules)](#14-tầng-infra-nestjs-modules)
15. [Rate limit & security](#15-rate-limit--security)
16. [Background jobs](#16-background-jobs)
17. [Observability, shutdown, deploy](#17-observability-shutdown-deploy)
18. [Testing](#18-testing)
19. [Roadmap](#19-roadmap)
20. [Quyết định đã chốt](#20-quyết-đã-chốt)

Chuẩn response dùng chung: [RESPONSES.md](./RESPONSES.md) · Setup (Bun/config/logging/trace): [SETUP.md](./SETUP.md)

---

## 1. Phạm vi sản phẩm

### 1.1 Tính năng

| # | Tính năng | Ghi chú |
|---|-----------|---------|
| 1 | Đăng ký bằng **email + mật khẩu + OTP SMTP** | OTP xác thực email trước khi kích hoạt |
| 2 | Đăng ký nhập **họ tên đầy đủ + username** | Username không ký tự đặc biệt (vd không `@`) |
| 3 | Đăng nhập bằng **email hoặc username** + mật khẩu | Ô đăng nhập dùng chung `identifier` |
| 4 | (Tùy chọn) Đăng nhập Google OAuth | Link-by-email khi email đã tồn tại |
| 5 | Quản lý token Redis-first | Access 15' + refresh 7 ngày, xoay vòng, max 5 thiết bị |
| 6 | Nhắn tin realtime (text, ảnh, video, audio, file) | WebSocket, lịch sử phân trang cursor |
| 7 | Voice/Video call **1:1** | WebRTC, signaling qua WS, coturn (TURN) |
| 8 | Gửi file/media | Upload presigned S3/MinIO |
| 9 | Trạng thái online/offline + last seen | Redis presence, heartbeat |
| 10 | Bạn bè: tìm theo username, gửi/chấp nhận/kết bạn, block | Nguồn cho presence + story feed |
| 11 | Story 24h (ảnh/video/text) | Tự hết hạn sau 24h, có danh sách người xem |
| 12 | Note 24h (trạng thái text ngắn) | Mỗi user 1 note active, hết hạn sau 24h |

### 1.2 Ngoài phạm vi MVP (phase sau)

- Group chat (schema đã mở sẵn), push notification FCM/APNs, E2E encryption (không làm — TLS đủ),
  "xóa tin phía mình" (MVP chỉ thu hồi với mọi người), seen từng người trong nhóm.

---

## 2. Stack kỹ thuật

| Tầng | Lựa chọn | Lý do |
|---|---|---|
| API | **NestJS 11** (Fastify adapter) | Module DI hợp phân tầng infra |
| ORM | **Drizzle ORM** (`drizzle-orm` + `drizzle-kit`, driver `pg`) | Schema TS-first, SQL-first migration, type-safe, nhẹ |
| DB | **PostgreSQL 16** (`citext`, UUIDv7) | Source of truth |
| Cache/state | **Redis 7** (ioredis) | Token, OTP, presence, unread, rate-limit — **Redis-first** |
| Realtime | **Socket.IO** + `@socket.io/redis-adapter` | Room per conversation/user, fan-out đa node |
| Object storage | **S3 API** (MinIO local / S3 prod) | Presigned upload trực tiếp từ client |
| Mail | Nodemailer (SMTP, **I/O-bound** — `await` nhả event loop) + Handlebars | Gửi OTP, template multipart |
| Mail queue | **`p-queue`** in-process (bounded, FIFO) | Tương đương `tokio::mpsc` — cùng event loop, không worker/NATS/microservice (xem §16) |
| Call | **WebRTC 1:1** + coturn (STUN/TURN) | Signaling qua WS |
| Mật khẩu | **argon2id** | OWASP defaults, output 32B |
| JWT | HS256 qua **`jose`**, access 15' / refresh 7 ngày | Claims `{sub, type, jti}`; zero-dep, pin `algorithms: ['HS256']` |
| Log/metric | `nestjs-pino` + `/metrics` Prometheus | request-id, latency histogram — setup [SETUP.md §3](./SETUP.md) |
| Validate | `class-validator` + `zod` cho env config | Fail-fast boot — [SETUP.md §2](./SETUP.md) |
| Toolchain | **Bun** (install/run/test — không npm) | `bun test` thay Jest — [SETUP.md §1](./SETUP.md) |
| Test | `bun test` + Testcontainers (pg/redis/minio) | Integration thật |

---

## 3. Kiến trúc tổng thể

```
Client (web / mobile)
   │  HTTPS (REST)              │  WSS (Socket.IO)
   ▼                           ▼
┌──────────────────────────────────────────────┐
│                  NestJS API                  │
│  Auth · Users · Friends · Conversations      │
│  Messages · Files · Stories · Notes          │
│  Presence · Calls (signaling) · Health       │
└───┬──────────┬───────────┬──────────┬─────────┘
    │          │           │          │
    ▼          ▼           ▼          ▼
 Postgres    Redis      S3/MinIO   MailQueue (p-queue, in-process) ──► SMTP (Gmail)
    │          │
    │          ├── token / session / OTP (fail-closed)
    │          ├── presence / typing / unread / cache (best-effort)
    │          └── rate-limit (fail-open)
    └── source of truth
                 + coturn (STUN/TURN cho WebRTC)
```

**Triết lý Redis-first**:

| Chính sách | Áp dụng | Hành vi khi Redis lỗi |
|---|---|---|
| **Fail-closed** (503) | refresh session, OTP, verify, đăng ký, trạng thái ringing | Từ chối request — không rơi vào DB |
| **Best-effort + DB fallback** | profile cache, story feed, unread, presence | Đọc DB / coi như offline, không fail request |
| **Fail-open** | rate-limit | Cho qua + `warn` (không chặn user vì sự cố Redis) |

Mọi thao tác nhiều key **atomic bằng Lua script** (1 round-trip): rotate refresh, tạo session,
check OTP, tăng counter unread, heartbeat presence.

---

## 4. Quy ước schema

- **PK**: UUIDv7 app-generated (monotonic theo thời gian → cursor pagination `ORDER BY id`
  không cần cột thứ tự phụ).
- **Timestamps**: `TIMESTAMPTZ NOT NULL DEFAULT now()` cho `created_at` / `updated_at`;
  trigger `set_updated_at()` BEFORE UPDATE (1 hàm dùng chung mọi bảng).
- **Soft delete**: `deleted_at TIMESTAMPTZ` (users, conversations, messages).
- **Email**: kiểu `CITEXT` UNIQUE, chuẩn hóa `LOWER()` khi ghi từ app.
- **Enum DB ↔ enum app** map 1:1 (Drizzle `pgEnum`).
- **Index**: chỉ index đúng query path; partial index `WHERE deleted_at IS NULL` / `WHERE left_at IS NULL`.
- **FK**: `ON DELETE RESTRICT` cho aggregate root, `CASCADE` cho bảng con (attachments, views).

---

## 5. Database schema (PostgreSQL)

### 5.1 Extensions & enums

```sql
CREATE EXTENSION IF NOT EXISTS citext;

CREATE TYPE conversation_kind AS ENUM ('DIRECT', 'GROUP');          -- MVP: chỉ DIRECT
CREATE TYPE participant_role  AS ENUM ('MEMBER', 'ADMIN', 'OWNER');
CREATE TYPE message_kind      AS ENUM ('TEXT', 'IMAGE', 'VIDEO', 'AUDIO', 'FILE', 'SYSTEM');
CREATE TYPE attachment_kind   AS ENUM ('IMAGE', 'VIDEO', 'AUDIO', 'FILE');
CREATE TYPE call_media        AS ENUM ('VOICE', 'VIDEO');
CREATE TYPE call_status       AS ENUM ('INITIATED', 'RINGING', 'ACCEPTED', 'REJECTED',
                                       'MISSED', 'CANCELLED', 'COMPLETED', 'FAILED');
CREATE TYPE story_kind        AS ENUM ('IMAGE', 'VIDEO', 'TEXT');
CREATE TYPE friendship_status AS ENUM ('PENDING', 'ACCEPTED', 'BLOCKED');
```

```sql
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### 5.2 `users`

Đăng ký bắt buộc: **email, mật khẩu, họ tên đầy đủ, username**.

```sql
CREATE TABLE users (
  id                UUID PRIMARY KEY,
  email             CITEXT NOT NULL,
  password_hash     TEXT,                          -- null nếu tài khoản chỉ OAuth
  google_id         TEXT UNIQUE,
  username          TEXT NOT NULL,                 -- chuẩn hóa lowercase khi ghi
  full_name         TEXT NOT NULL,                 -- họ tên đầy đủ
  avatar_url        TEXT,
  bio               TEXT,
  phone             TEXT UNIQUE,
  email_verified_at TIMESTAMPTZ,                   -- set khi OTP thành công
  last_seen_at      TIMESTAMPTZ,                   -- "last seen" khi offline
  is_active         BOOLEAN NOT NULL DEFAULT TRUE,
  deleted_at        TIMESTAMPTZ,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT users_username_format_chk
    CHECK (username ~ '^[a-z0-9_]{3,30}$'),
  CONSTRAINT users_full_name_chk
    CHECK (char_length(btrim(full_name)) BETWEEN 2 AND 100)
);

CREATE UNIQUE INDEX users_email_uq    ON users (email);
CREATE UNIQUE INDEX users_username_uq ON users (username);
CREATE INDEX        users_last_seen   ON users (last_seen_at);
CREATE INDEX        users_active_idx  ON users (id) WHERE deleted_at IS NULL;

CREATE TRIGGER users_set_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

**Quy tắc username**

| Quy tắc | Giá trị |
|---|---|
| Bộ ký tự | `a-z`, `0-9`, `_` — **không ký tự đặc biệt** (không `@ . -` , không emoji, không khoảng trắng) |
| Độ dài | 3–30 |
| Case | Không phân biệt hoa/thường → lưu **lowercase** duy nhất |
| Regex | `^[a-z0-9_]{3,30}$` |
| Regex (app, khi nhập) | `^[a-zA-Z0-9_]{3,30}$` rồi lowercase trước khi ghi |
| Không đổi | Username immutable sau khi tạo (tránh broken deep-link/mention) |
| Tìm bạn | `GET /users/search?username=` — exact-match hoặc prefix (limit 20) |

### 5.3 `friendships`

Quan hệ bạn bè **có hướng khi pending, vô hướng khi accepted** — chuẩn hóa bằng
`least_id / greatest_id` để tránh A→B và B→A là 2 row.

```sql
CREATE TABLE friendships (
  id           UUID PRIMARY KEY,
  requester_id UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  addressee_id UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  user_low     UUID NOT NULL,      -- LEAST(requester_id, addressee_id)
  user_high    UUID NOT NULL,      -- GREATEST(requester_id, addressee_id)
  status       friendship_status NOT NULL DEFAULT 'PENDING',
  responded_at TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT friendships_no_self_chk CHECK (requester_id <> addressee_id),
  CONSTRAINT friendships_pair_chk    CHECK (user_low < user_high),
  CONSTRAINT friendships_pair_uq     UNIQUE (user_low, user_high)
);

CREATE INDEX friendships_addressee_idx ON friendships (addressee_id, status);
CREATE INDEX friendships_requester_idx ON friendships (requester_id, status);
CREATE INDEX friendships_low_idx      ON friendships (user_low)  WHERE status = 'ACCEPTED';
CREATE INDEX friendships_high_idx     ON friendships (user_high) WHERE status = 'ACCEPTED';

CREATE TRIGGER friendships_set_updated_at
  BEFORE UPDATE ON friendships
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

Index `user_low/user_high WHERE ACCEPTED` phục vụ query "danh sách bạn bè" của 1 user
(2 OR-điều kiện trên cặp) — list bạn bè nhỏ (vài trăm), cache Redis best-effort.

**Luồng bạn bè** (xem [§8](#8-hệ-thống-bạn-bè)).

### 5.4 `conversations`

Một bảng cho DIRECT (MVP) và GROUP (sau này) — anti-hội-thoại-trùng bằng `direct_key`.

```sql
CREATE TABLE conversations (
  id               UUID PRIMARY KEY,
  kind             conversation_kind NOT NULL DEFAULT 'DIRECT',
  direct_key       TEXT,                            -- kind=DIRECT: '{minUserId}:{maxUserId}'
  title            TEXT,                            -- GROUP sau
  avatar_url       TEXT,
  last_message_id  UUID,
  last_message_at  TIMESTAMPTZ,                     -- sort inbox, cập nhật cùng tx khi gửi tin
  last_message_preview TEXT,                        -- snippet hiển thị inbox
  deleted_at       TIMESTAMPTZ,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT conversations_direct_chk
    CHECK (kind <> 'DIRECT' OR direct_key IS NOT NULL)
);

CREATE UNIQUE INDEX conversations_direct_key_uq ON conversations (direct_key)
  WHERE direct_key IS NOT NULL;
CREATE INDEX conversations_last_message_idx ON conversations (last_message_at DESC);

CREATE TRIGGER conversations_set_updated_at
  BEFORE UPDATE ON conversations
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### 5.5 `conversation_participants`

Kiêm luôn read-state ("đã đọc đến đâu") + mute.

```sql
CREATE TABLE conversation_participants (
  conversation_id      UUID NOT NULL REFERENCES conversations (id) ON DELETE CASCADE,
  user_id              UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  role                 participant_role NOT NULL DEFAULT 'MEMBER',
  last_read_message_id UUID,
  last_read_at         TIMESTAMPTZ,
  muted_until          TIMESTAMPTZ,
  joined_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
  left_at              TIMESTAMPTZ,                 -- "rời nhóm/khối ẩn" — không xóa row
  PRIMARY KEY (conversation_id, user_id)
);

CREATE INDEX participants_user_idx
  ON conversation_participants (user_id) WHERE left_at IS NULL;
```

### 5.6 `messages` & `message_attachments`

```sql
CREATE TABLE messages (
  id              UUID PRIMARY KEY,
  conversation_id UUID NOT NULL REFERENCES conversations (id) ON DELETE CASCADE,
  sender_id       UUID NOT NULL REFERENCES users (id),
  kind            message_kind NOT NULL DEFAULT 'TEXT',
  body            TEXT,                            -- null khi chỉ có attachment
  reply_to_id     UUID REFERENCES messages (id),
  client_msg_id   TEXT,                            -- idempotency phía client (chống gửi trùng)
  edited_at       TIMESTAMPTZ,
  deleted_at      TIMESTAMPTZ,                     -- "thu hồi" với mọi người
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT messages_content_chk
    CHECK (body IS NOT NULL OR kind IN ('IMAGE', 'VIDEO', 'AUDIO', 'FILE')),
  CONSTRAINT messages_body_len_chk
    CHECK (body IS NULL OR char_length(body) <= 4000)
);

CREATE UNIQUE INDEX messages_client_msg_uq
  ON messages (conversation_id, sender_id, client_msg_id)
  WHERE client_msg_id IS NOT NULL;
CREATE INDEX messages_timeline_idx
  ON messages (conversation_id, id DESC) WHERE deleted_at IS NULL;
CREATE INDEX messages_reply_idx ON messages (reply_to_id);
```

```sql
CREATE TABLE message_attachments (
  id         UUID PRIMARY KEY,
  message_id UUID NOT NULL REFERENCES messages (id) ON DELETE CASCADE,
  kind       attachment_kind NOT NULL,
  bucket     TEXT NOT NULL,
  object_key TEXT NOT NULL,        -- 'chat/{convId}/{msgId}/{uuid}.{ext}'
  mime_type  TEXT NOT NULL,
  size_bytes BIGINT NOT NULL,
  width      INT,
  height     INT,
  duration_ms INT,                 -- audio/video
  checksum   TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX message_attachments_msg_idx ON message_attachments (message_id);
```

### 5.7 `calls`

Ghi row **lúc bắt đầu** (giữ được missed-call kể cả khi server restart), update trạng thái sau.

```sql
CREATE TABLE calls (
  id              UUID PRIMARY KEY,
  conversation_id UUID NOT NULL REFERENCES conversations (id) ON DELETE CASCADE,
  caller_id       UUID NOT NULL REFERENCES users (id),
  callee_id       UUID NOT NULL REFERENCES users (id),
  media           call_media NOT NULL,
  status          call_status NOT NULL DEFAULT 'INITIATED',
  started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  answered_at     TIMESTAMPTZ,
  ended_at        TIMESTAMPTZ,
  end_reason      TEXT,
  duration_secs   INT GENERATED ALWAYS AS
                    (CASE WHEN answered_at IS NULL OR ended_at IS NULL THEN 0
                          ELSE GREATEST(0, FLOOR(EXTRACT(EPOCH FROM (ended_at - answered_at))))::INT
                     END) STORED,
  CONSTRAINT calls_no_self_chk CHECK (caller_id <> callee_id)
);

CREATE INDEX calls_conv_idx    ON calls (conversation_id, started_at DESC);
CREATE INDEX calls_callee_idx  ON calls (callee_id, status, started_at DESC);
```

### 5.8 `stories`, `story_views`, `notes` (ephemeral 24h)

```sql
CREATE TABLE stories (
  id         UUID PRIMARY KEY,
  user_id    UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  kind       story_kind NOT NULL,
  bucket     TEXT,
  object_key TEXT,
  mime_type  TEXT,
  width      INT,
  height     INT,
  duration_ms INT,
  caption    TEXT,
  background TEXT,                          -- story TEXT: màu nền/emoji
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at TIMESTAMPTZ NOT NULL,          -- created_at + 24h (ghi sẵn lúc insert)

  CONSTRAINT stories_media_chk
    CHECK (kind = 'TEXT' OR object_key IS NOT NULL),
  CONSTRAINT stories_expiry_chk
    CHECK (expires_at = created_at + INTERVAL '24 hours')
);

CREATE INDEX stories_user_idx   ON stories (user_id, created_at DESC);
CREATE INDEX stories_expiry_idx ON stories (expires_at);

CREATE TABLE story_views (
  story_id  UUID NOT NULL REFERENCES stories (id) ON DELETE CASCADE,
  user_id   UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  viewed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (story_id, user_id)
);

CREATE INDEX story_views_story_idx ON story_views (story_id);

CREATE TABLE notes (
  id         UUID PRIMARY KEY,
  user_id    UUID NOT NULL REFERENCES users (id) ON DELETE CASCADE,
  content    TEXT NOT NULL CHECK (char_length(content) BETWEEN 1 AND 200),
  background TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX notes_user_idx    ON notes (user_id, expires_at DESC);
CREATE INDEX notes_expiry_idx  ON notes (expires_at);
```

> **Lưu ý "mỗi user 1 note active":** không ràng buộc ở DB (partial unique theo thời gian khó
> maintain) — app **upsert**: trước khi insert note mới thì `DELETE FROM notes WHERE user_id = $1`
> (hoặc update in-place). Note cũ được thay, không cần lịch sử.

**Chiến lược expiry 24h**

- `expires_at` ghi sẵn lúc insert (CHECK ràng buộc đúng +24h).
- Job `purge-ephemeral` mỗi **5 phút**: `DELETE FROM stories/notes WHERE expires_at < now()
  RETURNING object_key` → xóa batch object S3 tương ứng.
- Feed query `WHERE expires_at > now()`; cache Redis TTL 60s (best-effort).

### 5.9 Bảng KHÔNG có trong DB

OTP, refresh token, session list, presence, typing, trạng thái call ringing, unread counter,
rate-limit counters — **toàn bộ Redis** ([§6](#6-redis-data-model-redis-first)).

---

## 6. Redis data model (Redis-first)

| Key | Value | TTL | Chính sách |
|---|---|---|---|
| `refreshToken:{jti}` | `userId` | 7 ngày | fail-closed |
| `sessions:{userId}` | JSON `[jti, …]`, max **5** thiết bị | 7 ngày (KEEPTTL khi rewrite) | fail-closed |
| `otp:{email}` | `{code, attempts}` — OTP đăng ký | **2 phút** | fail-closed |
| `forgot_otp:{email}` | `{code, attempts}` — OTP quên mật khẩu | **2 phút** | fail-closed |
| `ratelimit:{action}:ip:{ip}` | counter | per-action | fail-open |
| `ratelimit:{action}:email:{email}` | counter | 1 giờ | fail-open |
| `ratelimit:{action}:user:{userId}` | counter | per-action | fail-open |
| `user:{id}` | UserDTO JSON | 60s | best-effort |
| `userByName:{username}` | `userId` | 60s | best-effort |
| `friendList:{userId}` | JSON `[userId, …]` | 300s | best-effort |
| `presence:{userId}` | `{state, lastSeen, conns}` | 90s (heartbeat 30s) | best-effort |
| `presence:online` | ZSET userId → heartbeatTs | — | best-effort |
| `unread:{userId}:{convId}` | counter | 7 ngày | rebuild từ DB khi miss |
| `typing:{convId}:{userId}` | `1` | 3s | best-effort |
| `call:{callId}` | hash `{status, caller, callee, offer?}` | 15 phút | fail-closed (ringing) |
| `stories:friends:{userId}` | JSON `[storyId, …]` | 60s | best-effort |
| `push:device:{userId}` | JSON device tokens (phase 2) | — | — |

### 6.1 Lua scripts (atomic, 1 round-trip)

Đặt trong `src/infra/redis/scripts/*.lua`, load 1 lần vào `EVALSHA` (cache SHA):

| Script | Việc làm | Ghi chú |
|---|---|---|
| `ROTATE_REFRESH` | GET old → check subject → DEL old → SET new EX → rewrite `sessions` | old missing/subject lệch → trả `STALE` |
| `CREATE_SESSION` | append jti vào `sessions`, evict **session cũ nhất** khi > 5, DEL `refreshToken:{evicted}` | max 5 thiết bị |
| `REVOKE_SESSION` | DEL `refreshToken:{jti}` + rewrite `sessions` **KEEPTTL** | đăng xuất 1 thiết bị, không reset TTL cả list |
| `OTP_CHECK` | so khớp code; sai → `attempts+1` **KEEPTTL**; quá 5 → DEL, trả `LOCKED` | không kéo dài cửa sổ brute-force |
| `RATE_INCR` | `INCR` + `EXPIRE` on first hit | chống leak key khi crash |
| `RATE_INCR_MIXED` | 2 key (user + IP) trong 1 script, cả 2 phải pass | dùng cho OTP |
| `UNREAD_INCR` | `INCR` unread cho N participant (trừ sender) | gọi sau khi commit message |
| `UNREAD_RESET` | `DEL unread:{user}:{conv}` | khi đọc tin |
| `PRESENCE_HEARTBEAT` | `SETEX presence:{uid}` + `ZADD presence:online` | 1 round-trip |
| `PRESENCE_OFFLINE` | DEL presence + `ZREM presence:online` + trả về `lastSeen` | chỉ khi conns = 0 |

**Hành vi lỗi (không được lẫn):**

- `RotationOutcome::Stale` (replay token đã rotate) → **revoke cả family** của user → 401.
- `CacheError::Redis` trên đường session/OTP → **503** (không bao giờ báo nhầm là "đăng xuất").
- Cache đọc (profile/feed) lỗi → miss + DB fallback.

---

## 7. Auth: email + OTP + username

> **Chi tiết body, step, lỗi, rate-limit: [AUTH.md](./AUTH.md).** Phần dưới chỉ là bản rút gọn.

### 7.1 Luồng đăng ký (2 bước)

1. `POST /auth/request-otp` `{ email }` — check email chưa tồn tại (`409 USER_ALREADY_EXISTS`),
   sinh OTP 6 số, lưu `otp:{email}` TTL **2 phút** (fail-closed), enqueue mail vào
   `MailQueue` (p-queue in-process, async). Rate: 3/IP/60s + 3/email/giờ (anti inbox-bomb).
2. `POST /auth/signup` `{ email, fullName, username, password, otp }` — validate (username
   `^[a-zA-Z0-9_]{3,30}$` lowercase, fullName 2–100, password ≥ 6 ký tự) → `OTP_CHECK` +
   **consume** → hash argon2id → INSERT user (`email_verified_at = now()`) → `201` + user DTO.
   Trùng email → `409 USER_ALREADY_EXISTS`; trùng username → `409 USERNAME_TAKEN`.

`POST /auth/verify-otp` `{ email, otp, purpose }` là bước **tùy chọn** để FE kiểm tra OTP sớm
(không tiêu thụ) trước khi submit signup/reset.

### 7.2 Gửi lại OTP `POST /auth/request-otp`

Gọi lại endpoint trên: sinh code mới, overwrite key + TTL mới. Chưa có tài khoản → resend
bình thường; quên mật khẩu → `POST /auth/forgot-password/otp` (luôn `200`, anti-enumeration).

### 7.3 Đăng nhập `POST /auth/login`

```jsonc
{ "identifier": "lemon_van OR lemon@gmail.com", "password": "MatKhau#123" }
```

1. `identifier` chứa `@` → coi là email, ngược lại → username (đều lowercase/`LOWER(email)`).
2. Tìm user (`email = $1 OR username = $1`), verify argon2id. Sai thông tin → luôn
   `400 INVALID_CREDENTIALS` (không phân biệt sai gì — anti-enumeration). Chưa verify →
   `403 EMAIL_NOT_VERIFIED`.
3. `create_session`: sinh `jti` (UUIDv7) → ký access+refresh → `CREATE_SESSION` Redis.
   Ký **trước** lưu **sau** — token đã ký mà chưa lưu thì **không** trả về client (fail-closed 503).
4. Trả `data.accessToken` trong body + set refresh cookie HttpOnly (`refreshCookie`).
   Mobile: client tự lưu refresh vào secure storage (không qua cookie).

### 7.4 Quên mật khẩu `POST /auth/forgot-password/otp` → `POST /auth/forgot-password/reset`

1. `forgot-password/otp` `{ email }` — luôn `200` kể cả email lạ (anti-enumeration); nếu có user
   → lưu `forgot_otp:{email}` TTL 2 phút + enqueue mail vào `MailQueue`. Rate: 3/IP/60s + 3/email/giờ.
2. `forgot-password/reset` `{ email, otp, newPassword }` — `OTP_CHECK` + consume → hash →
   UPDATE `password_hash` → **revoke toàn bộ session** (fail-closed) → `200`.

### 7.5 Refresh `POST /auth/refresh` — rotate

1. Verify refresh JWT (pin `HS256`, `type=refresh`, có `jti`).
2. Ký cặp token mới **trước** ("ký trước, xoay sau" — sự cố giữa chừng thì phiên cũ còn nguyên).
3. `ROTATE_REFRESH` atomic. `Stale` (token đã bị rotate/revoke — khả năng replay) →
   `revoke_family` (DEL mọi `refreshToken:{jti}` + `sessions:{sub}`) → `401 INVALID_SESSION`.
4. Redis lỗi → `503`, **không** báo là đăng xuất.

### 7.6 Đăng xuất / đổi mật khẩu

- `POST /auth/logout`: `REVOKE_SESSION` theo `jti` hiện tại (idempotent).
- `POST /api/users/me/password` `{ oldPassword, newPassword }` (cần access token): verify mật
  khẩu cũ → UPDATE hash → revoke **toàn bộ** session + xóa cache `user:{id}` (fail-closed).

### 7.7 Claims JWT

```jsonc
// access (15 phút)
{ "sub": "<userId uuid>", "type": "access", "exp": …, "iat": …, "iss": "lemon-chat", "aud": "lemon-chat-client" }
// refresh (7 ngày) — thêm
{ "jti": "<uuid v7>" }
```

Lưu trữ Redis (không DB): `refreshToken:{jti}` → userId, `sessions:{userId}` → `[jti, …]`.
**Max 5 phiên/thiết bị cùng lúc** — login thiết bị thứ 6 kick phiên cũ nhất.

### 7.8 Google OAuth (tùy chọn, phase 2)

PKCE S256 + state cookie 10'. Callback: link-by-email (nếu email đã có) hoặc tạo user
(`google_id`, `password_hash NULL`, `email_verified_at = now()`). **Bắt buộc** code→token exchange
đúng chuẩn (`oauth2.googleapis.com/token`) — không dùng trực tiếp authorization code làm access token.

---

## 8. Hệ thống bạn bè

### 8.1 API

| Method | Path | Ý nghĩa |
|---|---|---|
| `POST` | `/friends/requests` | Gửi lời mời `{ username }` (không nhận userId — chống kết bạn bừa) |
| `GET` | `/friends/requests?direction=incoming\|outgoing` | Danh sách lời mời |
| `POST` | `/friends/requests/:id/accept` | Chấp nhận |
| `POST` | `/friends/requests/:id/reject` | Từ chối |
| `DELETE` | `/friends/requests/:id` | Hủy lời mời (người gửi) |
| `GET` | `/friends` | Danh sách bạn bè (kèm presence best-effort) |
| `DELETE` | `/friends/:userId` | Hủy kết bạn |
| `POST` | `/friends/block/:userId` | Block (status → `BLOCKED`, mọi hướng) |
| `DELETE` | `/friends/block/:userId` | Unblock (xóa row → về "người lạ") |
| `GET` | `/users/search?username=` | Tìm user theo username (prefix, limit 20) |

### 8.2 Quy tắc nghiệp vụ

1. **Gửi lời mời:** tạo row `PENDING` với `requester = mình`. Nếu đã có row:
   - `PENDING` ngược chiều (họ mời mình) → **tự động ACCEPTED** (hiếm gặp nhưng tiện).
   - `ACCEPTED` → `409 ALREADY_FRIENDS`.
   - `BLOCKED` (bất kỳ chiều nào) → `403` (không leak ai block ai — trả chung `403 NOT_AVAILABLE`).
2. **Chỉ addressee** được accept/reject; chỉ requester được hủy lời mời.
3. **Block:** ghi đè status cũ → `BLOCKED`, `requester = người block`. Hai bên **không** thấy nhau
   trong search/story/presence; block cũng **ẩn conversation DIRECT** (filter khi query inbox).
4. **Hủy kết bạn:** DELETE row. Conversation DIRECT **giữ nguyên** lịch sử (không cascade xóa) —
   có thể kết bạn lại sau.
5. **Kết bạn ⇒ auto conversation:** khi friendship → `ACCEPTED`, đảm bảo tồn tại conversation
   `DIRECT` với `direct_key = '{minId}:{maxId}'` (upsert `ON CONFLICT DO NOTHING`) — 2 người
   thành bạn là có sẵn ô chat.
6. **Visibility:** story/note/presence chỉ hiển thị với **bạn bè ACCEPTED** (ngoại lệ: 1:1 trong
   conversation đang mở thì thấy presence của nhau kể cả chưa kết bạn — quy tắc "đang trò chuyện").

### 8.3 Cache bạn bè

- `friendList:{userId}` TTL 300s (best-effort) — invalidate khi accept/unfriend/block.
- Query bạn bè chính vẫn đi DB (nguồn sự thật), cache chỉ dùng cho presence fan-out & story feed.

---

## 9. Nhắn tin realtime

### 9.1 REST

| Method | Path | Ý nghĩa |
|---|---|---|
| `GET` | `/conversations` | Inbox: sort `last_message_at DESC`, kèm `unreadCount`, preview, presence đối phương |
| `POST` | `/conversations/direct` | Tạo/lấy conversation DIRECT `{ userId }` |
| `GET` | `/conversations/:id/messages?cursor=&limit=50` | Timeline `id DESC`, cursor = `id` cuối |
| `GET` | `/conversations/:id/messages?after=<id>` | Sync khi reconnect (tin bị lỡ) |
| `POST` | `/conversations/:id/messages` | Gửi tin (idempotent qua `clientMsgId`) |
| `PATCH` | `/messages/:id` | Sửa tin (chỉ TEXT, chỉ chủ tin, có `edited_at`) |
| `DELETE` | `/messages/:id` | Thu hồi (soft `deleted_at`, mọi người) |
| `POST` | `/conversations/:id/read` | Đánh dấu đã đọc `{ lastReadMessageId }` |

**Gửi tin** (`POST .../messages`):

1. Rate-limit `send_message:user` (vd 30/phút) — fail-open.
2. Verify sender ∈ participants (còn `left_at IS NULL`).
3. **Tx Postgres**: insert `messages` (+ attachments metadata nếu có) → update
   `conversations.last_message_*` (id/at/preview). `clientMsgId` unique → trả lại message cũ nếu
   duplicate (idempotent).
4. Sau commit: `UNREAD_INCR` (Redis, best-effort) + emit WS `message:new` vào room `conv:{id}`.

### 9.2 Socket.IO

**Handshake:** access token (header `Authorization` hoặc `auth.token`) → verify → join
`user:{userId}` + join mọi `conv:{id}` mà user còn tham gia. Token hết hạn → `disconnect`
`reason=auth` để client refresh rồi reconnect.

| Event (client → server) | Event (server → client) | Ghi chú |
|---|---|---|
| — | `message:new` | payload MessageDTO |
| — | `message:updated` / `message:revoked` | sửa/thu hồi |
| `typing:start` / `typing:stop` | `typing` | TTL Redis 3s, không ghi DB |
| `read:update` | `read:updated` | sync `lastReadMessageId` |
| `presence:ping` (30s) | `presence:update` | fan-out tới bạn bè |
| `call:invite/accept/reject/ice/end` | cùng tên (tới peer) | signaling WebRTC, [§12](#12-voicevideo-call-11-webrtc) |
| `sync:missed` `{ afterId }` | `sync:missed:result` | hoặc client dùng REST `?after=` |

Đa node: `@socket.io/redis-adapter` (pub/sub `request`/`response` của Socket.IO) — emit là đủ,
không cần biết socket ở node nào.

### 9.3 Unread counter

- Gửi tin → `INCR unread:{userId}:{convId}` với mọi participant trừ sender (best-effort).
- Đọc (`POST /read` hoặc mở conversation trên client) → `UNREAD_RESET` + update
  `last_read_message_id/at` trong DB.
- Counter miss (Redis flush) → **rebuild từ DB**:
  `SELECT COUNT(*) FROM messages WHERE conversation_id=$1 AND id > last_read_message_id
   AND deleted_at IS NULL AND sender_id <> $me`.

### 9.4 Đồng bộ khi reconnect

Client gửi `after = <id tin cuối đã nhận>` → trả tối đa 200 tin thiếu + unread mới nhất +
trạng thái presence của bạn bè. Không dùng "offset page" — cursor `id` (UUIDv7) ổn định khi
có tin mới chèn vào.

---

## 10. Presence (online/offline)

- **Online** = có ≥ 1 WS connection còn sống + heartbeat `presence:ping` mỗi 30s
  (client) — server `PRESENCE_HEARTBEAT` (SETEX TTL 90s + ZADD `presence:online`).
- **Grace period 60s:** khi WS disconnect mà `conns` về 0 → **không** set offline ngay
  (tránh nhấp nháy khi chuyển mạng/đổi Wi-Fi) — hẹn 60s sau mới `PRESENCE_OFFLINE`
  (`setTimeout` trong process, lưu timer ref để **cancel khi reconnect** — tương đương delayed job).
- **Offline** → ghi `users.last_seen_at` (bất đồng bộ, best-effort) → client hiển thị
  "lần cuối {từ đó}".
- Query presence cho 1 danh sách userId: `MGET presence:{ids}` (list bạn bè nhỏ, không scan key).
- Fan-out `presence:update` chỉ gửi tới **bạn bè** (không phát tán toàn server).
- Multi-node: presence sống trong Redis (mọi node thấy chung); timer grace là `setTimeout`
  local — node chết thì Redis TTL tự rơi (tự offline), không cần durable job.

---

## 11. File & media

### 11.1 Luồng upload (presigned — media KHÔNG đi qua API)

1. Client `POST /files/presign` `{ kind, contentType, sizeBytes, checksum? }`.
2. Server validate (giới hạn theo kind — [§11.2](#112-giới-hạn)), cấp **presigned PUT/POST** với
   điều kiện `content-length-range` + `content-type` bắt buộc, `object_key` server sinh
   (`chat/{convId}/{uuid}.{ext}` hoặc `stories/{userId}/{uuid}.{ext}`) — client **không** tự
   đặt key (tránh ghi đè/giải đoán path).
3. Client PUT thẳng lên S3/MinIO → `POST .../messages` đính kèm `objectKey` + metadata
   (width/height/duration/checksum) → server verify object tồn tại (`HEAD`) + checksum khớp
   → tạo `message_attachments` trong cùng tx với message.

Download: URL presigned GET TTL 15' (hoặc CDN). Không public-read object.

### 11.2 Giới hạn (hằng số config)

| Kind | Tối đa | MIME cho phép |
|---|---|---|
| IMAGE | 10 MB | jpeg, png, webp, gif |
| VIDEO | 50 MB | mp4, webm |
| AUDIO | 15 MB | m4a, mp3, ogg, opus (voice note) |
| FILE | 50 MB | mọi loại (trừ exe/script — blacklist) |
| Story IMAGE/VIDEO | như trên | như trên |

Checksum `SHA-256` khuyến khích (chống upload rác trùng lặp). Giai đoạn sau: quét malware
async (ClamAV job) trước khi object "chín".

---

## 12. Voice/Video call 1:1 (WebRTC)

### 12.1 Tín hiệu (signaling) qua Socket.IO

```
caller                     server                      callee
  │  call:invite              │                           │
  │  (callId, media, convId)  │── insert calls INITIATED  │
  │                           │── SET call:{callId}       │
  │                           │── call:invite ───────────►│ (ringing)
  │                           │                           │
  │  call:accept (sdpAnswer) ◄│──── call:accept (sdpAnswer)  ← callee tạo answer
  │  (caller tạo offer trước) │                           │
  │  call:ice ◄──────────────►│────── call:ice ─────────►│  (trickle ICE 2 chiều)
  │                           │                           │
  │  call:end                 │── update calls, DEL redis │
```

| Bước | Chi tiết |
|---|---|
| `call:invite` | Tạo row `calls` (`INITIATED`→`RINGING`) + Redis `call:{id}` TTL 15'. Emit tới room `user:{callee}`. **Callee offline/busy** → trả `call:busy` → status `MISSED`. |
| Ringing timeout 30s | Không accept → status `MISSED` (`setTimeout` 30s + timer ref, cancel khi accept/end — idempotent nếu đã kết thúc trước). |
| `call:accept` | Update `ACCEPTED`, `answered_at=now()`. SDP offer/answer đi qua WS (payload chỉ là JSON SDP — không qua REST). |
| `call:ice` | Trickle ICE candidate 2 chiều. |
| `call:reject` / `call:cancel` | `REJECTED` (callee từ chối) / `CANCELLED` (caller hủy). |
| `call:end` | `COMPLETED`, `ended_at`, `end_reason`; DEL `call:{id}`. |
| Busy | Mỗi user 1 call active tại 1 thời điểm — key `call:active:{userId}` TTL bằng call. |

ICE server: STUN công cộng + **coturn**. Cấp credential tạm thời (TURN REST):

```
username = "{expiryEpoch}:{userId}"          // hạn 24h
credential = base64(HMAC-SHA1(TURN_SECRET, username))
```

`GET /calls/turn-credentials` (cần auth) trả danh sách `iceServers` — **không** hardcode
secret ra client.

### 12.2 Media path

- P2P qua UDP; fallback relay TURN khi NAT khó.
- Codec: Opus (audio), VP8/VP9/AV1 (video) theo capability client đàm phán.
- **Không** ghi âm/quay màn hình ở server (không SFU ở MVP) — call không đi qua NestJS,
  server chỉ làm signaling.

---

## 13. Story 24h & Note 24h

### 13.1 Story

| Method | Path | Ý nghĩa |
|---|---|---|
| `POST` | `/stories` | Đăng story `{ kind, objectKey?, caption?, background? }` |
| `GET` | `/stories/feed` | Story của **bạn bè** còn hạn + của mình, nhóm theo user |
| `GET` | `/stories/:id` | Xem 1 story |
| `POST` | `/stories/:id/view` | Đánh dấu đã xem (upsert `story_views`) |
| `GET` | `/stories/:id/views` | Danh sách người xem (chỉ chủ story) |
| `DELETE` | `/stories/:id` | Xóa story + object S3 (chỉ chủ) |

- Hết hạn sau **24h** (`expires_at`), tự purge ([§5.8](#58-stories-story_views-notes-ephemeral-24h)).
- Feed cache `stories:friends:{userId}` TTL 60s (best-effort), invalidate khi bạn đăng story.
- Đã xem: chỉ chủ story thấy danh sách viewer; viewer khác **không** thấy ai đã xem (giống Zalo).

### 13.2 Note (trạng thái 24h)

| Method | Path | Ý nghĩa |
|---|---|---|
| `PUT` | `/notes` | **Upsert** note của mình `{ content, background? }` (đè note cũ) |
| `DELETE` | `/notes/me` | Gỡ note sớm |
| `GET` | `/notes/feed` | Note còn hạn của bạn bè (mỗi người 1 note) |

- Một note active / user (app upsert). Hết hạn 24h → hiển thị "chưa có note" (không cần xóa tay).
- Note khác story: không media, không viewer list, không lịch sử.

---

## 14. Tầng infra (NestJS modules)

```
src/
├── infra/
│   ├── config/           # typed config + Zod validate, fail-fast boot (SETUP.md §2)
│   ├── database/         # DrizzleService (pg.Pool), drizzle-kit migration runner
│   │   └── schema/       # pgTable/pgEnum TS-first (users, friendships, conversations, …)
│   ├── redis/            # ioredis: 1 conn lệnh + N conn pub/sub
│   │   └── scripts/      # *.lua → EVALSHA (cache SHA)
│   ├── storage/          # S3Client (aws-sdk v3), presigner, key builder
│   ├── mailer/           # MailQueue (p-queue) + Nodemailer + Handlebars — consumer in-process
│   ├── scheduler/        # @nestjs/schedule: cron purge 5', setTimeout delayed (presence-grace, call-timeout)
│   ├── realtime/         # Socket.IO gateway + redis-adapter + handshake auth
│   └── observability/    # nestjs-pino, request-id, /metrics, /health (SETUP.md §3–§4)
├── modules/
│   ├── auth/             # request-otp, verify-otp, signup, login (email|username), refresh, logout
│   ├── users/            # profile, search username, đổi avatar/bio
│   ├── friends/          # requests, accept/reject, block, list
│   ├── conversations/    # inbox, direct, read-state
│   ├── messages/         # CRUD tin, attachments
│   ├── files/            # presign, verify object
│   ├── stories/  notes/
│   ├── presence/  calls/
└── main.ts
```

**Quy ước infra:**

| Quy ước | Nội dung |
|---|---|
| Config | `fromEnv()` fail-fast khi boot (Zod schema — [SETUP.md §2](./SETUP.md)): `JWT_SECRET` ≥ 32 bytes bắt buộc; `SameSite=None` phải kèm `Secure`; production thiếu SMTP → **exit 1** (dev cho phép log-only) |
| Redis | **1 connection manager** dùng chung cache + rate-limit; connection **riêng** cho subscriber (pub/sub không được chặn connection lệnh) |
| Drizzle | Schema TS (`pgTable`) = entity; repository interface tách khỏi Drizzle impl (đổi ORM không sập domain); query phức tạp dùng `db.execute()` với tagged template `sql` |
| Mail queue | `p-queue` in-process (bounded 128, `concurrency: 1` = FIFO) — **tương đương `tokio::mpsc`**: `enqueue()` = `send()`, queue worker = consumer nhận từng mail. Backpressure: `queue.onSizeLessThan(128)` trước khi push (đúng semantics mpsc "send.await khi đầy"); tràn → drop + `warn` (mail là best-effort, không fail request). Gửi fail → retry 2 lần backoff 1s/5s rồi bỏ qua — **không có durable queue** (đúng lựa chọn: không microservice, không NATS/BullMQ) |
| DTO ≠ entity | API DTO camelCase, không lộ `password_hash` / `deleted_at` |
| Error | `BusinessError` (4xx, code ổn định i18n) vs `SystemError` (5xx; cache outage → **503**) — envelope theo [RESPONSES.md](./RESPONSES.md) |
| Mail | Service gọi `mailQueue.enqueue(...)` rồi quên (fire-and-forget) — SMTP chậm không chặn request |
| Idempotency | Gửi tin dùng `clientMsgId` unique per (conv, sender) |
| Upload | Presigned S3 — media không đi qua API ([§11](#11-file--media)) |
| TURN | Credential HMAC tạm thời, secret chỉ ở server ([§12](#12-voicevideo-call-11-webrtc)) |
| Shutdown | Dừng nhận kết nối mới → đóng WS (`server:shutdown`) → drain MailQueue (`onEmpty`) → close Drizzle pool/Redis |
| Dev env | Docker Compose: `postgres · redis · minio · coturn · api` (1 process duy nhất) |

**Drizzle schema ↔ DB:** schema định nghĩa TS-first (`drizzle-kit generate` sinh SQL migration).
CHECK/trigger/partial index: Drizzle diễn đạt được index `.where()` / unique; CHECK constraint +
trigger `set_updated_at()` viết trong file migration SQL custom (`drizzle-kit` giữ nguyên SQL khi
generate) — **SQL-first nên không bị giới hạn** như ORM schema-only. Apply bằng
`drizzle-kit migrate` lúc boot (hoặc CI trước deploy).

---

## 15. Rate limit & security

### 15.1 Rate limit (Redis, fail-open)

| Action | Giới hạn |
|---|---|
| `request-otp` IP | 3 / 60s |
| `request-otp` email | 3 / giờ (anti inbox-bomb) |
| `verify-otp` IP | 10 / 60s |
| `login` IP | 5 / 60s |
| `change-password` user + IP | 3/user/60s + 10/IP/60s |
| `forgot-password-otp` IP + email | 3/IP/60s + 3/email/giờ |
| `signup` IP | 5 / giờ |
| `send-message` user | 30 / phút |
| `presign` user | 60 / giờ |
| `call:invite` user | 10 / phút |
| `friend-request` user | 20 / ngày |

### 15.2 Security checklist

- argon2id cho mật khẩu (OWASP defaults, output 32B).
- JWT pin `HS256` (anti algorithm-confusion), `leeway=30s`, bắt buộc `exp/iss/aud`.
- Refresh token: HttpOnly cookie (`SameSite=Strict` mặc định) trên web; mobile lưu secure storage.
- Không log OTP/mật khẩu/token; mask email khi log (redact tự động — [SETUP.md §3](./SETUP.md)).
- CORS whitelist `FRONTEND_URL` (comma-separated); `credentials: true`.
- Security headers: `nosniff`, `X-Frame-Options: DENY`, HSTS, CSP `default-src 'self'`.
- Body limit 1 MB (metadata JSON) — media đi S3.
- Anti-enumeration: forgot-password luôn `200`; login/signup không tiết lộ "email hay username sai".
- S3 bucket: không public; presigned GET TTL ngắn; key do server sinh.

---

## 16. Background jobs (in-process — không NATS, không worker riêng)

Không microservice: mọi job chạy **trong API process** bằng `@nestjs/schedule` + `p-queue` + `setTimeout`.
Đánh đổi: job pending trong RAM bị mất khi process restart — chấp nhận được vì mỗi job đều
tự chạy lại theo lịch hoặc có TTL tự rơi (bảng dưới).

| Job | Cơ chế | Tần suất / trigger | Việc làm |
|---|---|---|---|
| `purge-stories-notes` | `@Cron('*/5 * * * *')` + Redis lock `SET NX` 5s (leader = node nào giữ được lock) | mỗi 5 phút | `DELETE … WHERE expires_at < now() RETURNING object_key` → xóa batch S3 |
| Mail (OTP…) | **`p-queue`** bounded 128, `concurrency: 1` (FIFO) | on-demand khi signup/reset | Gửi mail qua SMTP; fail → retry 2 lần (1s/5s) rồi bỏ |
| `presence-grace` | `setTimeout` 60s, **lưu timer ref** | sau disconnect | Nếu `conns=0` → `PRESENCE_OFFLINE` + ghi `last_seen_at`; reconnect → `clearTimeout` |
| `call-ringing-timeout` | `setTimeout` 30s, lưu timer ref | sau invite | Nếu `call:{id}` còn `RINGING` → `MISSED`; accept/end → `clearTimeout` |
| `push` (phase 2) | cùng `p-queue` (subject khác) | on-demand | Thông báo tin/call khi offline |

**`p-queue` không có worker riêng — chạy trên cùng event loop, và điều đó là ĐÚNG:**

| Câu hỏi | Trả lời |
|---|---|
| `p-queue` có thread/worker riêng không? | **Không** — chỉ là promise queue trên cùng event loop. Consumer = 1 async task `await smtp.send()` |
| Vậy có block request không? | **Không** — SMTP là **I/O-bound** (chờ network), `await` nhả event loop ngay. Giống hệt `tokio::mpsc`: consumer cũng chỉ là 1 async task trên cùng tokio runtime, **không** có OS thread riêng cho mail |
| "NestJS chậm" thì sao? | Chậm là overhead per-request (DI/middleware/serialize) trên **CPU path**. Mail nằm ngoài CPU path: enqueue xong là response đi trước, SMTP chờ song song. Thêm 1 promise queue không cộng thêm độ trễ request |
| Khi nào mới cần worker thật? | **CPU-bound**: probe media (ffprobe), transcode, encrypt file lớn → `worker_threads` (Bun hỗ trợ sẵn) hoặc `piscina` — tương đương `spawn_blocking`/`rayon`. Mail/OTP/purge/setTimeout **không** thuộc loại này |
| Backlog đầy thì sao? | Bounded 128 + `onSizeLessThan` = backpressure kiểu `send().await`; tràn drop + `warn` — request vẫn không chờ SMTP |

**Tại sao đủ dùng (không cần durable queue):**

| Job | Mất khi restart thì sao |
|---|---|
| Mail | OTP chỉ sống 2 phút — user thấy không có mail → bấm gửi lại (rate-limit vẫn chặn spam) |
| purge | Chạy lại sau ≤ 5 phút — story quá hạn thêm vài phút, không ai thấy (`expires_at > now()` filter sẵn) |
| presence-grace | Redis TTL 90s tự rơi → user coi như offline; `last_seen_at` ghi trễ 1 nhịp, chấp nhận được |
| call-timeout | Call kẹt `RINGING` trong DB → boot reconcile: `UPDATE calls SET status='MISSED' WHERE status='RINGING' AND started_at < now() - interval '2 min'` |

---

## 17. Observability, shutdown, deploy

### 17.1 Observability

Chi tiết setup log + error trace: [SETUP.md §3–§4](./SETUP.md).

- **Log**: `nestjs-pino` JSON, mọi request có `x-request-id` (genReqId + echo); WS events log ở `debug`;
  log level: SystemError→`error`, BusinessError→`warn`, milestone→`info`.
- **Metric** (`/metrics` Prometheus): `http_request_duration_seconds` (histogram),
  `ws_connections`, `ws_events_total{event}`, `redis_op_duration_seconds{op}`,
  `mail_queue_size` (độ đầy p-queue), `mail_sent_total{result}`,
  `messages_sent_total`, `calls_total{status}`,
  `otp_locked_total`, `refresh_stale_total` (cảnh báo replay).
- **Health**: `/health/live` (process sống), `/health/ready` (Postgres `SELECT 1`, Redis `PING`,
  S3 `HEAD bucket` — fail → 503, load balancer ngừng đưa traffic).
- **Tracing**: `requestId` xuyên HTTP → service → MailQueue job (kèm `requestId` trong payload) →
  log khi gửi xong/fail; SystemError wrap `cause` chain (không nuốt nguyên nhân gốc).

### 17.2 Graceful shutdown

1. SIGTERM → ngừng accept kết nối mới (HTTP + WS handshake).
2. Gửi `server:shutdown` cho WS clients (client tự reconnect).
3. Đợi in-flight request ≤ 10s → drain MailQueue (`queue.onEmpty()`, timeout 5s) → hủy mọi
   `setTimeout` đang giữ ref → close Drizzle pool → đóng Redis.
4. Process exit 0; quá timeout → exit 1.

### 17.3 Deploy & data

| Thành phần | Ghi chú |
|---|---|
| Compose (dev) | `postgres:16, redis:7, minio, coturn/coturn, api` |
| Prod | Container `api` duy nhất; Postgres managed (RDS/Cloud SQL) hoặc self-host + PITR |
| Redis | Prod bật **AOF everysec** — mất Redis = mất phiên đăng nhập/OTP (chấp nhận được, login lại); cache rebuild được |
| Mail queue | In-process (RAM) — restart giữa chừng thì mất mail chờ gửi; user bấm gửi lại OTP (rate-limit vẫn chặn spam) |
| Backup | `pg_dump` mỗi ngày + WAL archiving; S3 versioning/lifecycle |
| Scale ngang | API stateless + `socket.io-redis-adapter` → thêm node thoải mái; presence/unread ở Redis chung |
| Sticky session | **Không cần** (adapter lo fan-out) |

---

## 18. Testing

| Tầng | Tool | Trọng tâm |
|---|---|---|
| Unit | `bun test` (`bun:test`) | service thuần: username validate, direct_key, preview, state machine call |
| Integration | Testcontainers (pg + redis + minio) chạy dưới `bun test` | OTP atomic lock, refresh rotate/replay → revoke family, max 5 session, unread rebuild, purge 24h, friendship transitions, MailQueue FIFO + drain |
| E2E | `supertest` + `socket.io-client` | đăng ký → verify OTP → login → kết bạn → gửi tin 2 chiều → presence → call signaling → story hết hạn |
| Load (sau) | k6 / autocannon | WS fan-out, gửi tin, presence heartbeat |

Coverage cao cho: nhánh lỗi Redis (fail-closed vs fail-open), replay token, OTP sai 5 lần,
đăng nhập bằng username/email, block/unfriend visibility.

---

## 19. Roadmap

| Phase | Nội dung | Exit criteria |
|---|---|---|
| **P0 — Nền** | Scaffold NestJS (Bun toolchain), config Zod fail-fast, Drizzle+schema §5, Redis infra §6 (Lua scripts), MailQueue `p-queue` + `@nestjs/schedule`, Docker Compose | migrate xanh, `/health/ready` pass, `bun test` unit Lua wrapper |
| **P1 — Auth** | Register (fullName+username+email), OTP, login identifier, refresh rotation, logout, đổi mật khẩu | e2e auth xanh; replay → family revoke có test |
| **P2 — Bạn bè** | Search username, request/accept/reject/unfriend/block, auto DIRECT conversation | transition matrix test (mọi cặp status) |
| **P3 — Chat** | Conversations, gửi/nhận text, timeline cursor, read/unread, WS realtime, typing | 2 client đổi tin realtime, reconnect sync đúng |
| **P4 — Media** | Presign S3, attachments (image/video/audio/file), giới hạn MIME/size | upload→đính kèm→đối phương tải được |
| **P5 — Presence + Call** | Heartbeat, grace 60s, last seen; WebRTC signaling, TURN creds, call history | call 1:1 2 thiết bị, missed-call 30s |
| **P6 — Story/Note** | CRUD story/note, feed bạn bè, views, purge 24h | job purge xóa DB+S3; viewer list đúng |
| **P7 — Cứng** | Rate limit đủ bảng, metrics, graceful shutdown, backup, hardening | checklist §15.2 pass, 0 warning lint |

---

## 20. Quyết định đã chốt

| # | Quyết định |
|---|---|
| 1 | Framework API: **NestJS**; ORM **Drizzle** (`drizzle-kit` migrate); Redis **ioredis**; mail queue **`p-queue`** in-process; toolchain **Bun** (`bun test` làm test runner) |
| 2 | **Redis-first**: token, session, OTP, presence, unread, typing, ringing → Redis; DB là source of truth cho message/friend/call log |
| 3 | Có **hệ thống bạn bè** (request/accept/reject/block/unfriend) — nguồn visibility cho presence/story/note |
| 4 | Đăng ký gồm **email + mật khẩu + fullName + username**; username `^[a-z0-9_]{3,30}$`, **không ký tự đặc biệt**, unique, immutable |
| 5 | Đăng nhập bằng **email hoặc username** (`identifier`) |
| 6 | OTP xác thực email qua **SMTP** (async qua MailQueue `p-queue` in-process), TTL 2', max 5 lần sai, rate 3 email/giờ |
| 7 | Access JWT 15' + refresh 7 ngày **rotate**, max 5 thiết bị, replay → revoke family |
| 8 | Call **1:1 WebRTC**, signaling qua Socket.IO, coturn TURN, ring timeout 30s → MISSED |
| 9 | File/media: **presigned S3** (MinIO local), server chỉ giữ metadata |
| 10 | Story/Note **24h** hết hạn (`expires_at` + job purge 5 phút, xóa kèm object S3) |
| 11 | **Không E2E** — TLS đủ; chưa group chat, push, "xóa phía mình" (schema mở sẵn cho tương lai) |
| 12 | **Không microservice**: job chạy in-process — mail qua `p-queue` (bounded FIFO ≈ `tokio::mpsc`, **cùng event loop**, SMTP I/O-bound nên không block request), cron `@nestjs/schedule`, delayed job `setTimeout` + timer ref. Không NATS/BullMQ/worker riêng; chỉ `worker_threads` cho CPU-bound (probe media…) |
| 13 | Tài liệu này là thiết kế **LEMON CHAT** — đặt tại `docs_lemon_chat/LEMON_CHAT.md` |
