# LEMON CHAT — DATABASE: Drizzle ORM · Schema · Relations · Migrations · Testing

> Chi tiết hiện thực tầng dữ liệu của LEMON CHAT: **Drizzle ORM** (driver `pg`), kiến trúc
> **Domain-Driven**, ánh xạ đầy đủ 10 bảng quan hệ, phá vỡ vòng lặp phụ thuộc (circular dependency),
> quản lý migration và bộ kiểm thử toàn diện.
>
> Tài liệu liên quan: [LEMON_CHAT.md](./LEMON_CHAT.md) (tổng quan thiết kế hệ thống) ·
> [SETUP.md](./SETUP.md) (Bun toolchain, config, logging, trace) ·
> [AUTH.md](./AUTH.md) (luồng xác thực Redis-first) ·
> [RESPONSES.md](./RESPONSES.md) (chuẩn envelope API) ·
> [STORIES_NOTES.md](./STORIES_NOTES.md) (kiến trúc Story & Note 24h).

---

## Mục lục

1. [Triết lý & Kiến trúc tầng dữ liệu](#1-triết-lý--kiến-trúc-tầng-dữ-liệu)
2. [Cấu trúc mã nguồn thư mục](#2-cấu-trúc-mã-nguồn-thư-mục)
3. [Đặc tả 10 Bảng dữ liệu theo từng Domain](#3-đặc-tả-10-bảng-dữ-liệu-theo-từng-domain)
4. [Ánh xạ Quan hệ (Drizzle Relations) & Nested Queries](#4-ánh-xạ-quan-hệ-drizzle-relations--nested-queries)
5. [Tích hợp NestJS & Repository Pattern](#5-tích-hợp-nestjs--repository-pattern)
6. [Quản lý Migrations & Drizzle Kit](#6-quản-lý-migrations--drizzle-kit)
7. [Chiến lược Kiểm thử toàn diện (Testing Strategy)](#7-chiến-lược-kiểm-thử-toàn-diện-testing-strategy)
8. [Checklist phát triển & Bảo trì](#8-checklist-phát-triển--bảo-trì)

---

## 1. Triết lý & Kiến trúc tầng dữ liệu

### 1.1 Lựa chọn công nghệ
* **ORM:** **Drizzle ORM** (`drizzle-orm` + `drizzle-kit`, driver `pg`).
* **NestJS Module:** `@nestjs/drizzle` (thư viện tích hợp chính thức của NestJS).
* **Database Driver:** `pg` (`node-postgres`) với connection pooling (`pg.Pool`).
* **Dialect:** PostgreSQL 16 (hỗ trợ `CITEXT`, UUIDv7, Stored Generated Columns, Partial Indexes).

### 1.2 Nguyên tắc thiết kế
1. **Domain-Driven Colocation:**
   Mỗi domain sở hữu bảng và logic lưu trữ riêng (`src/domain/<domain>/`). Không gom toàn bộ schema vào một file khổng lồ làm mất tính đóng gói.
2. **Decoupling Table Schema vs Relations (Anti-Circular Dependency):**
   Trong môi trường Node.js ESM (`module: nodenext`), việc hai bảng trỏ khóa ngoại lẫn nhau rất dễ gây lỗi runtime `ReferenceError: Cannot access table before initialization`.
   * **Giải pháp:** Tách mỗi domain thành 2 file:
     * `<domain>.schema.ts`: Định nghĩa cấu trúc vật lý bảng SQL (dùng callback lazy `() => otherTable.id`).
     * `<domain>.relations.ts`: Định nghĩa quan hệ logic Drizzle `relations()` phục vụ Relational Query API.
3. **Root Schema Aggregator:**
   File `src/infra/database/schema.ts` tập hợp và re-export toàn bộ bảng và quan hệ từ các domain, cung cấp điểm neo duy nhất cho `DrizzleModule` và `drizzle.config.ts`.
4. **Repository Pattern:**
   Tầng Domain Service chỉ tương tác qua abstract interface (ví dụ `UserRepository`). Mã cụ thể của Drizzle nằm trong `UserPgRepository` (`user.pg.ts`), bảo vệ core business logic khỏi bị phụ thuộc chặt chẽ vào ORM.

---

## 2. Cấu trúc mã nguồn thư mục

```text
lemon-backend/
├── drizzle/                               # Thư mục chứa các file SQL Migration
│   ├── meta/                              # Drizzle migration journal (_journal.json)
│   └── 0000_white_sinister_six.sql        # Migration khởi tạo 10 bảng, extension, triggers
├── drizzle.config.ts                      # Cấu hình Drizzle Kit CLI
└── src/
    ├── infra/
    │   └── database/
    │       ├── custom-types.ts            # Kiểu citext, helper timestamps (created_at, updated_at)
    │       ├── database.module.ts         # Module toàn cục DrizzleModule.forRootAsync
    │       └── schema.ts                  # Root aggregator re-export tất cả domain schemas & relations
    ├── domain/
    │   ├── shared.ts                      # Shared types (PaginatedResult, BaseTimestamp)
    │   ├── user/                          # Domain: Người dùng & Hồ sơ
    │   │   ├── user.schema.ts             # Bảng users + checks + indexes
    │   │   ├── user.relations.ts          # Quan hệ của users
    │   │   ├── user.entity.ts             # Domain Entity UserEntity
    │   │   ├── user.repository.ts         # Interface UserRepository
    │   │   └── user.pg.ts                 # Triển khai UserPgRepository (InjectDrizzle)
    │   ├── friend/                        # Domain: Bạn bè
    │   │   ├── friend.schema.ts           # Bảng friendships + enum friendship_status
    │   │   └── friend.relations.ts        # Quan hệ bạn bè (requester, addressee)
    │   ├── conversation/                  # Domain: Hội thoại & Thành viên
    │   │   ├── conversation.schema.ts     # Bảng conversations, conversation_participants
    │   │   └── conversation.relations.ts  # Quan hệ phòng chat, người tham gia
    │   ├── message/                       # Domain: Tin nhắn & Đính kèm
    │   │   ├── message.schema.ts          # Bảng messages, message_attachments
    │   │   └── message.relations.ts       # Quan hệ reply_to, replies, attachments
    │   ├── call/                          # Domain: Cuộc gọi WebRTC
    │   │   ├── call.schema.ts             # Bảng calls, enums call_media/call_status
    │   │   └── call.relations.ts          # Quan hệ caller, callee, conversation
    │   ├── story/                         # Domain: Story 24h
    │   │   ├── story.schema.ts            # Bảng stories, story_views
    │   │   └── story.relations.ts         # Quan hệ views, author
    │   └── note/                          # Domain: Note 24h
    │       ├── note.schema.ts             # Bảng notes
    │       └── note.relations.ts          # Quan hệ note -> user
```

---

## 3. Đặc tả 10 Bảng dữ liệu theo từng Domain

### 3.1 Domain User (`src/domain/user/`)
* **Bảng:** `users`
* **Custom Type:** `citext` (PostgreSQL extension không phân biệt chữ hoa/thường cho email).
* **Cột chính:**
  * `id` (`uuid`, PK) — UUIDv7 sinh từ application.
  * `email` (`citext`, NOT NULL, Unique Index).
  * `password_hash` (`text`, nullable khi dùng Google OAuth).
  * `google_id` (`text`, Unique).
  * `username` (`text`, NOT NULL, Unique Index) — chuẩn hóa lowercase.
  * `full_name` (`text`, NOT NULL).
  * `avatar_url`, `bio`, `phone` (`text`, Unique).
  * `email_verified_at`, `last_seen_at`, `deleted_at` (`timestamptz`).
  * `is_active` (`boolean`, default `true`).
  * `created_at`, `updated_at` (`timestamptz`, default `now()`).
* **Ràng buộc & Chỉ mục:**
  * CHECK `users_username_format_chk`: `username ~ '^[a-z0-9_]{3,30}$'`
  * CHECK `users_full_name_chk`: `char_length(btrim(full_name)) BETWEEN 2 AND 100`
  * Partial Index `users_active_idx`: `(id) WHERE deleted_at IS NULL`
  * Index `users_last_seen`: `(last_seen_at)`

### 3.2 Domain Friend (`src/domain/friend/`)
* **Bảng:** `friendships`
* **Enum:** `friendship_status` (`'PENDING' | 'ACCEPTED' | 'BLOCKED'`).
* **Cột chính:**
  * `id` (`uuid`, PK).
  * `requester_id` (`uuid`, FK $\rightarrow$ `users.id` CASCADE).
  * `addressee_id` (`uuid`, FK $\rightarrow$ `users.id` CASCADE).
  * `user_low` (`uuid`, NOT NULL) — ID nhỏ hơn giữa hai user.
  * `user_high` (`uuid`, NOT NULL) — ID lớn hơn giữa hai user.
  * `status` (`friendship_status`, default `'PENDING'`).
  * `responded_at` (`timestamptz`).
* **Ràng buộc & Chỉ mục:**
  * CHECK `friendships_no_self_chk`: `requester_id <> addressee_id`
  * CHECK `friendships_pair_chk`: `user_low < user_high`
  * UNIQUE `friendships_pair_uq`: `(user_low, user_high)` — đảm bảo duy nhất 1 hàng quan hệ giữa 2 người.
  * Partial Indexes: `(user_low) WHERE status = 'ACCEPTED'`, `(user_high) WHERE status = 'ACCEPTED'`
  * Indexes: `(addressee_id, status)`, `(requester_id, status)`

### 3.3 Domain Conversation (`src/domain/conversation/`)
* **Bảng 1:** `conversations`
  * **Enum:** `conversation_kind` (`'DIRECT' | 'GROUP'`).
  * `id` (`uuid`, PK), `kind` (default `'DIRECT'`).
  * `direct_key` (`text`) — Format: `'{minUserId}:{maxUserId}'` để chống tạo trùng phòng DIRECT.
  * `title`, `avatar_url`, `last_message_preview` (`text`).
  * `last_message_id` (`uuid`), `last_message_at` (`timestamptz`).
  * CHECK `conversations_direct_chk`: `kind <> 'DIRECT' OR direct_key IS NOT NULL`.
  * Partial Unique Index: `(direct_key) WHERE direct_key IS NOT NULL`.
  * Index: `(last_message_at DESC)`.
* **Bảng 2:** `conversation_participants`
  * **Enum:** `participant_role` (`'MEMBER' | 'ADMIN' | 'OWNER'`).
  * Composite PK: `PRIMARY KEY (conversation_id, user_id)`.
  * `conversation_id` (FK CASCADE), `user_id` (FK CASCADE).
  * `last_read_message_id` (`uuid`), `last_read_at` (`timestamptz`), `muted_until` (`timestamptz`).
  * `joined_at` (`timestamptz`), `left_at` (`timestamptz` — rời phòng, không xóa dòng).
  * Partial Index: `(user_id) WHERE left_at IS NULL`.

### 3.4 Domain Message (`src/domain/message/`)
* **Bảng 1:** `messages`
  * **Enum:** `message_kind` (`'TEXT' | 'IMAGE' | 'VIDEO' | 'AUDIO' | 'FILE' | 'SYSTEM'`).
  * `id` (`uuid`, PK), `conversation_id` (FK CASCADE), `sender_id` (FK RESTRICT).
  * `reply_to_id` (`uuid`, Self FK `messages.id`).
  * `client_msg_id` (`text`) — chống gửi trùng (idempotency).
  * CHECK `messages_content_chk`: `body IS NOT NULL OR kind IN ('IMAGE', 'VIDEO', 'AUDIO', 'FILE')`.
  * CHECK `messages_body_len_chk`: `body IS NULL OR char_length(body) <= 4000`.
  * Partial Unique Index: `(conversation_id, sender_id, client_msg_id) WHERE client_msg_id IS NOT NULL`.
  * Timeline Index: `(conversation_id, id DESC) WHERE deleted_at IS NULL`.
* **Bảng 2:** `message_attachments`
  * **Enum:** `attachment_kind` (`'IMAGE' | 'VIDEO' | 'AUDIO' | 'FILE'`).
  * `id` (`uuid`, PK), `message_id` (FK CASCADE).
  * `bucket`, `object_key`, `mime_type`, `checksum` (`text`).
  * `size_bytes` (`bigint`), `width`, `height`, `duration_ms` (`integer`).

### 3.5 Domain Call (`src/domain/call/`)
* **Bảng:** `calls`
* **Enums:** `call_media` (`'VOICE' | 'VIDEO'`), `call_status` (`'INITIATED' | 'RINGING' | 'ACCEPTED' | 'REJECTED' | 'MISSED' | 'CANCELLED' | 'COMPLETED' | 'FAILED'`).
* **Cột chính & Cột tính toán ảo:**
  * `id` (`uuid`, PK), `conversation_id` (FK CASCADE), `caller_id` (FK), `callee_id` (FK).
  * `media`, `status` (default `'INITIATED'`).
  * `started_at`, `answered_at`, `ended_at`, `end_reason`.
  * `duration_secs` (`integer`): **STORED GENERATED COLUMN**
    ```sql
    GENERATED ALWAYS AS (
      CASE WHEN answered_at IS NULL OR ended_at IS NULL THEN 0
           ELSE GREATEST(0, FLOOR(EXTRACT(EPOCH FROM (ended_at - answered_at))))::INT
      END
    ) STORED
    ```
* CHECK `calls_no_self_chk`: `caller_id <> callee_id`.
* Indexes: `(conversation_id, started_at DESC)`, `(callee_id, status, started_at DESC)`.

### 3.6 Domain Story & Note (`src/domain/story/` & `src/domain/note/`)
* **Bảng `stories` (24h Ephemeral):**
  * **Enum:** `story_kind` (`'IMAGE' | 'VIDEO' | 'TEXT'`).
  * `id` (`uuid`, PK), `user_id` (FK CASCADE), `caption`, `background`, media dimensions.
  * `expires_at` (`timestamptz`): CHECK `expires_at = created_at + INTERVAL '24 hours'`.
  * CHECK: `kind = 'TEXT' OR object_key IS NOT NULL`.
  * Index: `(user_id, created_at DESC)`, `(expires_at)`.
* **Bảng `story_views`:**
  * Composite PK: `PRIMARY KEY (story_id, user_id)`.
  * `story_id` (FK CASCADE), `user_id` (FK CASCADE), `viewed_at`.
* **Bảng `notes` (Trạng thái ngắn 24h):**
  * `id` (`uuid`, PK), `user_id` (FK CASCADE), `content` (CHECK 1-200 ký tự), `background`.
  * `expires_at` (`timestamptz`).
  * Index: `(user_id, expires_at DESC)`, `(expires_at)`.

---

## 4. Ánh xạ Quan hệ (Drizzle Relations) & Nested Queries

Drizzle cung cấp Relational Query API (`db.query.<table_name>`) cho phép nạp dữ liệu phân cấp mà không cần viết `LEFT JOIN` thủ công.

### 4.1 Danh mục quan hệ đã cấu hình

```mermaid
classDiagram
    users "1" --> "*" friendships : friendshipsInitiated / Received
    users "1" --> "*" conversationParticipants : joins
    users "1" --> "*" messages : sends
    users "1" --> "*" calls : callsInitiated / Received
    users "1" --> "*" stories : posts
    users "1" --> "*" notes : posts
    conversations "1" --> "*" conversationParticipants : participants
    conversations "1" --> "*" messages : messages
    conversations "1" --> "*" calls : calls
    messages "1" --> "*" messageAttachments : attachments
    messages "1" --> "*" messages : replyTo / replies
    stories "1" --> "*" storyViews : views
```

### 4.2 Ví dụ Truy vấn lồng sâu trong Service/Repository

#### Lấy chi tiết cuộc trò chuyện và lịch sử tin nhắn:
```typescript
const conversation = await db.query.conversations.findFirst({
  where: eq(conversations.id, convId),
  with: {
    participants: {
      with: {
        user: true, // Lấy profile từng thành viên
      },
    },
    messages: {
      limit: 20,
      orderBy: desc(messages.id),
      with: {
        sender: true,
        attachments: true,
        replyTo: true,
      },
    },
  },
});
```

#### Lấy Story feed của bạn bè kèm danh sách người đã xem:
```typescript
const storiesFeed = await db.query.stories.findMany({
  where: gt(stories.expiresAt, new Date()),
  with: {
    user: true,
    views: {
      with: {
        user: true,
      },
    },
  },
});
```

---

## 5. Tích hợp NestJS & Repository Pattern

### 5.1 Đăng ký DatabaseModule (`src/infra/database/database.module.ts`)
Module nạp toàn cục qua `@Global()`, cấu hình kết nối pool và tự động giải phóng tài nguyên khi ứng dụng shutdown:

```typescript
import { Global, Module } from '@nestjs/common';
import { DrizzleModule } from '@nestjs/drizzle';
import { drizzle, type NodePgDatabase } from 'drizzle-orm/node-postgres';
import * as schema from './schema.js';

export type DrizzleDb = NodePgDatabase<typeof schema>;

@Global()
@Module({
  imports: [
    DrizzleModule.forRootAsync({
      useFactory: () => ({
        drizzle,
        connection: {
          connectionString:
            process.env['DATABASE_URL'] ??
            'postgres://postgres:postgres@localhost:5432/lemon_chat',
        },
        schema,
        autoCloseConnection: true, // Tự động gọi pool.end() khi shutdown
      }),
    }),
  ],
  exports: [DrizzleModule],
})
export class DatabaseModule {}
```

### 5.2 Triển khai trong Domain Repository (`user.pg.ts`)
Inject client bằng decorator `@InjectDrizzle()`:

```typescript
import { Injectable } from '@nestjs/common';
import { InjectDrizzle } from '@nestjs/drizzle';
import { eq } from 'drizzle-orm';
import type { DrizzleDb } from '../../infra/database/database.module.js';
import type { UserEntity } from './user.entity.js';
import type { UserRepository } from './user.repository.js';
import { users } from './user.schema.js';

@Injectable()
export class UserPgRepository implements UserRepository {
  constructor(@InjectDrizzle() private readonly db: DrizzleDb) {}

  async findById(id: string): Promise<UserEntity | null> {
    const [result] = await this.db.select().from(users).where(eq(users.id, id));
    return result ?? null;
  }
}
```

---

## 6. Quản lý Migrations & Drizzle Kit

### 6.1 Cấu hình `drizzle.config.ts`
```typescript
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',
  schema: './src/infra/database/schema.ts',
  out: './drizzle',
  dbCredentials: {
    url: process.env['DATABASE_URL'] ?? 'postgres://postgres:postgres@localhost:5432/lemon_chat',
  },
  verbose: true,
  strict: true,
});
```

### 6.2 Các lệnh CLI
```bash
# 1. Sinh file SQL migration mới từ TypeScript schemas
bun run db:generate

# 2. Áp dụng migration vào PostgreSQL
bun run db:migrate

# 3. Mở UI Drizzle Studio quản lý dữ liệu trực quan
bun run db:studio
```

### 6.3 Xử lý Trigger & Extension trong PostgreSQL
File migration đầu tiên `drizzle/0000_white_sinister_six.sql` đã được tối ưu sẵn:
1. Kích hoạt extension: `CREATE EXTENSION IF NOT EXISTS citext;` (trước khi tạo bảng `users`).
2. Khởi tạo hàm trigger `set_updated_at()`:
   ```sql
   CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
   BEGIN
     NEW.updated_at = now();
     RETURN NEW;
   END;
   $$ LANGUAGE plpgsql;
   ```
3. Gắn trigger tự động cập nhật thời gian cho các bảng:
   ```sql
   CREATE TRIGGER users_set_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION set_updated_at();
   CREATE TRIGGER friendships_set_updated_at BEFORE UPDATE ON friendships FOR EACH ROW EXECUTE FUNCTION set_updated_at();
   CREATE TRIGGER conversations_set_updated_at BEFORE UPDATE ON conversations FOR EACH ROW EXECUTE FUNCTION set_updated_at();
   ```

---

## 7. Chiến lược Kiểm thử toàn diện (Testing Strategy)

Toàn bộ tầng dữ liệu được bảo vệ bằng hệ thống Unit & Integration test tự động (`bun run test`):

| Test Suite | File kiểm thử | Trách nhiệm kiểm tra |
|---|---|---|
| **Schema Integrity** | `test/unit/database/schema.spec.ts` | Khẳng định đủ 10 bảng, 8 enums, 10 relations được export từ `schema.ts`. |
| **Relational Queries** | `test/unit/database/relations.spec.ts` | Kiểm tra tính đúng đắn của Drizzle query engine với các truy vấn lồng sâu (nested relations). |
| **SQL Query Builder** | `test/unit/database/query-builder.spec.ts` | Đảm bảo `.toSQL()` sinh đúng cú pháp SQL, tham số bind (`$1`, `$2`), soft-delete và cursor pagination. |
| **NestJS DI Wiring** | `test/unit/database/di.spec.ts` | Kiểm thử NestJS `TestingModule`, tiêm phụ thuộc `@InjectDrizzle()` vào các repository thành công. |
| **Table Constraints** | `test/unit/database/constraints.spec.ts` | Kiểm tra composite PKs, generated column `duration_secs`, giá trị Enums chuẩn. |

---

## 8. Checklist phát triển & Bảo trì

- [ ] **Mọi bảng mới:** Đặt trong `src/domain/<domain>/<domain>.schema.ts`.
- [ ] **Mọi quan hệ mới:** Đặt trong `src/domain/<domain>/<domain>.relations.ts` (không import chéo schema file để tránh circular dependency).
- [ ] **Re-export bắt buộc:** Khai báo export bảng và relation tại `src/infra/database/schema.ts`.
- [ ] **Sau khi sửa schema:** Luôn chạy `bun run typecheck` $\rightarrow$ `bun run test` $\rightarrow$ `bun run db:generate`.
- [ ] **Quy ước tên cột:** Tên trường trong DB là `snake_case`, tên thuộc tính TypeScript là `camelCase`.
- [ ] **Khóa chính:** Mọi bảng aggregate root dùng UUIDv7 sinh từ application logic (không dùng serial/autoincrement).

