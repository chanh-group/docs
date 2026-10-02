# LEMON CHAT — AUTH: Đăng ký · Đăng nhập · Quên mật khẩu · Đổi mật khẩu

> Luồng xác thực chi tiết của LEMON CHAT, state **Redis-first** (OTP, session, refresh rotation).
> Chuẩn body success/error dùng chung: [RESPONSES.md](./RESPONSES.md) · Tổng quan hệ thống: [LEMON_CHAT.md](./LEMON_CHAT.md)
>
> Mọi endpoint dưới prefix `/api/auth` (trừ `change-password` đặt tại `/api/users/me/password` theo
> chuẩn REST "đổi mật khẩu của chính mình"). Body/response theo envelope [RESPONSES.md](./RESPONSES.md);
> dưới đây chỉ mô tả `data` (hoặc `message`) riêng của từng endpoint.

---

## Mục lục

1. [Tổng quan state Redis](#1-tổng-quan-state-redis)
2. [Token model](#2-token-model)
3. [Luồng 1 — Đăng ký](#3-luồng-1--đăng-ký)
4. [Luồng 2 — Đăng nhập](#4-luồng-2--đăng-nhập)
5. [Luồng 3 — Refresh token (xoay vòng)](#5-luồng-3--refresh-token-xoay-vòng)
6. [Luồng 4 — Đăng xuất](#6-luồng-4--đăng-xuất)
7. [Luồng 5 — Quên mật khẩu](#7-luồng-5--quên-mật-khẩu)
8. [Luồng 6 — Đổi mật khẩu](#8-luồng-6--đổi-mật-khẩu)
9. [Bảng endpoint tổng hợp](#9-bảng-endpoint-tổng-hợp)
10. [Rate limit & anti-abuse](#10-rate-limit--anti-abuse)
11. [Bảng lỗi auth](#11-bảng-lỗi-auth)
12. [Checklist bắt buộc](#12-checklist-bắt-buộc)

---

## 1. Tổng quan state Redis

Auth **không lưu token/OTP trong DB** — mọi state tạm sống ở Redis:

| Key | Value | TTL | Ghi chú |
|---|---|---|---|
| `otp:signup:{email}` | `{ code, attempts }` | **5 phút** | OTP đăng ký, namespace riêng |
| `otp:reset:{email}` | `{ code, attempts }` | **5 phút** | OTP quên mật khẩu, namespace riêng |
| `refreshToken:{jti}` | `userId` | 7 ngày | 1 refresh token = 1 session |
| `sessions:{userId}` | JSON `[jti, …]` — max **5** | 7 ngày (KEEPTTL khi rewrite) | Danh sách phiên của user |
| `user:{id}` | UserDTO JSON | 60s | Profile cache (best-effort) |
| `ratelimit:{action}:ip:{ip}` | counter | theo action | fail-open |
| `ratelimit:{action}:email:{email}` | counter | 1 giờ | anti inbox-bomb |

**Nguyên tắc lỗi:**

| Đường | Redis lỗi | Hành vi |
|---|---|---|
| OTP store/check, refresh rotate, revoke session | **fail-closed 503** `SERVICE_UNAVAILABLE` | Không được báo "thành công" giả khi state chưa ghi xong |
| Profile cache, rate-limit | best-effort / fail-open | Không chặn user vì sự cố cache |

**Hằng số:**

| Hằng số | Giá trị |
|---|---|
| `OTP_EXPIRATION` | 300s (5 phút) |
| `OTP_LENGTH` | 6 chữ số (`CSPRNG`, `100000..1000000`) |
| `MAX_OTP_ATTEMPTS` | 5 — sai quá → **hủy mã**, bắt xin lại |
| `ACCESS_TOKEN_EXPIRATION` | 15 phút |
| `REFRESH_TOKEN_EXPIRATION` | 7 ngày |
| `MAX_SESSIONS_PER_USER` | 5 — login thiết bị thứ 6 **kick phiên cũ nhất** |

**Lua scripts** (atomic 1 round-trip) — xem [LEMON_CHAT.md §6.1](./LEMON_CHAT.md):
`OTP_CHECK` (so khớp + đếm attempts KEEPTTL), `CREATE_SESSION` (append + evict cũ nhất),
`ROTATE_REFRESH` (old→new nguyên tử), `REVOKE_SESSION` (DEL + rewrite KEEPTTL), `RATE_INCR(_MIXED)`.

---

## 2. Token model

```jsonc
// Access token — HS256, 15 phút
{ "sub": "<userId UUID>", "type": "access", "iat": …, "exp": …,
  "iss": "lemon-chat", "aud": "lemon-chat-client" }

// Refresh token — HS256, 7 ngày, thêm jti
{ "sub": "<userId UUID>", "type": "refresh", "jti": "<UUIDv7>", "iat": …, "exp": …,
  "iss": "lemon-chat", "aud": "lemon-chat-client" }
```

| Token | Vận chuyển | Ghi chú |
|---|---|---|
| Access | Body JSON (`data.accessToken`) | Client giữ **trong memory** (web) / secure storage (mobile) — không localStorage, không URL |
| Refresh | Cookie HttpOnly `refreshCookie` (web) / secure storage (mobile) | `Path=/`, `SameSite=Strict` (mặc định), `Secure` theo env; **không** đặt trong body |

- Validate JWT: pin `HS256` (anti algorithm-confusion), `leeway=30s`, bắt buộc `exp/iss/aud`.
- **"Ký trước, xoay sau":** ký cặp mới → rồi mới rotate Redis. Rotate lỗi (503) thì phiên cũ
  còn nguyên, client retry được — token đã ký mà chưa lưu session thì **không** trả client.
- **Replay → revoke family:** `ROTATE_REFRESH` trả `Stale` (token đã rotate/thu hồi mà còn dùng)
  → `revoke_family` xóa mọi `refreshToken:{jti}` + `sessions:{sub}` → `401 INVALID_SESSION`.
- Đổi/reset mật khẩu → revoke **toàn bộ** session (xem §7, §8).
- Validate combo cookie lúc boot: `SameSite=None` **bắt buộc** kèm `Secure` — sai thì exit (fail-fast).

---

## 3. Luồng 1 — Đăng ký

```
Client                       Server                            Redis            SMTP
  │ POST /auth/request-otp     │                                │                │
  │ { email }                  │── find_by_email ── DB          │                │
  │                            │   đã tồn tại → 409             │                │
  │                            │── generate OTP ───────────────►│ otp:signup:{email} (5')
  │                            │── publish mail.otp ───────────────────────────►│ (async)
  │◄── 200 "Đã gửi OTP" ──────│                                │                │
  │                            │                                │                │
  │ POST /auth/signup          │                                │                │
  │ { email, fullName,         │── OTP_CHECK ──────────────────►│ match?         │
  │   username, password, otp }│   sai/hết hạn → 400            │                │
  │                            │   sai 5 lần → DEL → 400        │                │
  │                            │── hash argon2id → INSERT users─┤ DB             │
  │                            │── DEL otp:signup:{email} ─────►│                │
  │◄── 201 user ───────────────│                                │                │
```

### 3.1 `POST /api/auth/request-otp`

Gửi OTP đăng ký. **Chưa tạo user.**

Request:

```jsonc
{ "email": "lemon@gmail.com" }
```

Response `200` — `messageOnly`:

```jsonc
{ "success": true, "message": "Đã gửi mã OTP đến email của bạn" }
```

Server-side (theo thứ tự):

1. Rate-limit: 3/IP/60s + 3/email/giờ → vượt → `429 TOO_MANY_REQUESTS`.
2. `find_by_email` — đã tồn tại → `409 USER_ALREADY_EXISTS`.
3. `generate_otp()` (CSPRNG 6 chữ số) → `SET otp:signup:{email} = { code, attempts: 0 } EX 300`
   (ghi Redis **fail-closed** — Redis lỗi → 503, không gửi mail giả).
4. Publish NATS `mail.otp` (async fire-and-forget — mail fail **không** fail request).

> Email lowercase khi build key: `otp:signup:lemon@gmail.com`.

### 3.2 `POST /api/auth/verify-otp` *(tùy chọn — FE kiểm tra sớm)*

Đối chiếu OTP **không tiêu thụ** (cho FE verify trước khi bấm Đăng ký), vẫn tăng `attempts` nếu sai.

Request:

```jsonc
{ "email": "lemon@gmail.com", "otp": "123456", "purpose": "signup" }   // purpose: "signup" | "reset"
```

Response `200` — `withMessage(true, "OTP hợp lệ")`:

```jsonc
{ "success": true, "message": "OTP hợp lệ", "data": true }
```

- Rate-limit: 10/IP/60s.
- `OTP_CHECK` Lua: đúng → `Match` (không xóa); sai → `attempts+1` **KEEPTTL** (không kéo dài cửa sổ
  brute-force); quá `MAX_OTP_ATTEMPTS` → DEL key, trả `Locked` → `400 INVALID_OTP`.
- Sai/hết hạn/locked → `400 INVALID_OTP` (không phân biệt lý do — không leak).

### 3.3 `POST /api/auth/signup`

Tạo tài khoản. OTP được **tiêu thụ** tại đây.

Request:

```jsonc
{
  "email": "lemon@gmail.com",
  "fullName": "Nguyễn Văn Lemon",     // 2–100 ký tự
  "username": "lemon_van",            // ^[a-zA-Z0-9_]{3,30}$ → lowercase khi ghi
  "password": "MatKhau#123",          // ≥ 8, có chữ + số
  "otp": "123456"
}
```

Response `201` — `created(user, "Đăng ký thành công")`:

```jsonc
{
  "success": true,
  "message": "Đăng ký thành công",
  "data": {
    "id": "0193a2b1-…",
    "email": "lemon@gmail.com",
    "username": "lemon_van",
    "fullName": "Nguyễn Văn Lemon",
    "avatarUrl": null,
    "emailVerifiedAt": "2026-10-02T03:12:00Z",
    "createdAt": "2026-10-02T03:12:00Z",
    "updatedAt": "2026-10-02T03:12:00Z"
  }
}
```

Server-side:

1. Validate body (class-validator): email, `fullName` 2–100, `username` regex
   `^[a-zA-Z0-9_]{3,30}$`, password ≥ 8 có chữ+số, otp đúng 6 số.
2. `OTP_CHECK otp:signup:{email}` với otp gửi kèm → sai/hết hạn/locked → `400 INVALID_OTP`.
   Đúng → **consume** (DEL key) — OTP dùng 1 lần.
3. `hash_password` argon2id (OWASP defaults, output 32B) — **async**, không block runtime.
4. `INSERT users` (`id` UUIDv7 app-sinh, `email_verified_at = now()`, `is_active = true`):
   - unique violation email → `409 USER_ALREADY_EXISTS`
   - unique violation username → `409 USERNAME_TAKEN`
5. Trả user (DTO camelCase, **không** bao giờ lộ `passwordHash`).
6. Client chuyển sang đăng nhập (hoặc FE tự gọi login) — signup **không** trả token.

> **Biến thể (không mặc định):** nếu muốn user tồn tại sớm, insert với `email_verified_at NULL`
> rồi chặn login đến khi verify (`403 EMAIL_NOT_VERIFIED`). Mặc định: chỉ tạo user khi OTP đúng
> — tránh user nửa vời trong DB.

---

## 4. Luồng 2 — Đăng nhập

```
Client                       Server                              Redis
  │ POST /auth/login           │                                  │
  │ { identifier, password }   │── find_by_identifier ── DB       │
  │                            │   (email OR username)            │
  │                            │── verify argon2id                │
  │                            │── create_session ───────────────►│ refreshToken:{jti}
  │                            │                                  │ sessions:{userId}
  │◄── 200 + accessToken ──────│   (Set-Cookie: refreshCookie)    │
```

### 4.1 `POST /api/auth/login`

Đăng nhập bằng **email hoặc username**.

Request:

```jsonc
{ "identifier": "lemon_van", "password": "MatKhau#123" }
// identifier: "lemon_van" (username) hoặc "lemon@gmail.com" (email)
```

Response `200` — `withMessage({ accessToken }, "Đăng nhập thành công")` + **Set-Cookie** `refreshCookie`:

```jsonc
{
  "success": true,
  "message": "Đăng nhập thành công",
  "data": { "accessToken": "eyJhbGciOi…" }
}
```

Server-side:

1. Rate-limit: 10/IP/60s + 5/identifier/15 phút (chống brute-force mật khẩu) → `429`.
2. **Phân loại identifier:** chứa `@` → tra theo email (`LOWER(email)`), ngược lại → theo
   username (đã lowercase). 1 câu query: `WHERE (email = $1 OR username = $1) AND deleted_at IS NULL`.
3. Không tìm thấy **hoặc** sai mật khẩu **hoặc** tài khoản OAuth không có password →
   **luôn** `400 INVALID_CREDENTIALS` (không phân biệt — anti-enumeration).
4. Chưa `email_verified_at` → `403 EMAIL_NOT_VERIFIED` (cho phép quay lại request-otp).
   Tài khoản `is_active = false` → `403 FORBIDDEN`.
5. `create_session(userId)`:
   - `jti = UUIDv7`
   - ký access (15') + refresh (7', kèm jti)
   - `CREATE_SESSION` Lua: SET `refreshToken:{jti}` + append `sessions:{userId}`; nếu > 5 phiên
     → evict **phiên cũ nhất** (DEL `refreshToken:{evicted}`)
   - **fail-closed:** Redis ghi lỗi → 503, không trả token đã ký
6. Set cookie refresh (`HttpOnly`, `SameSite`/`Secure` theo env) — mobile bỏ qua cookie,
   client tự lưu refresh vào secure storage.

---

## 5. Luồng 3 — Refresh token (xoay vòng)

### 5.1 `POST /api/auth/refresh`

Request: **không có body** — đọc refresh token từ cookie `refreshCookie` (web) hoặc
`Authorization: Bearer <refresh>` (mobile — header riêng, không lẫn access).

Response `200` — `withMessage({ accessToken }, "Phiên đã được làm mới")` + Set-Cookie mới:

```jsonc
{
  "success": true,
  "message": "Phiên đã được làm mới",
  "data": { "accessToken": "eyJhbGciOi…" }
}
```

Server-side ("ký trước, xoay sau"):

1. Thiếu token / verify JWT fail → `401 INVALID_SESSION`.
2. Ký cặp **mới** trước (`jti` mới).
3. `ROTATE_REFRESH` Lua (nguyên tử):
   - GET `refreshToken:{oldJti}` → đúng subject → DEL old → SET new (EX 7d) → rewrite `sessions`
   - old key missing / subject lệch → trả **`Stale`** (khả năng replay)
4. `Stale` → `revoke_family(sub)`: DEL mọi `refreshToken:{jti}` + `sessions:{sub}` → `401 INVALID_SESSION`.
   Token đánh cắp **không** dùng lại được, kể cả sau khi victim đã rotate.
5. Redis lỗi → `503 SERVICE_UNAVAILABLE` (**không** bao giờ báo là "đăng xuất").

> Client (web) chỉ có **1 luồng refresh** tại 1 thời điểm (single-flight) — tránh 2 request
> cùng mang 1 refresh cũ → 1 cái nhận `Stale` → revoke family oan.

---

## 6. Luồng 4 — Đăng xuất

### 6.1 `POST /api/auth/logout`

Request: không body — lấy refresh từ cookie (kèm access claims nếu có để xóa profile cache).

Response `200` — `messageOnly("Đã đăng xuất")`:

```jsonc
{ "success": true, "message": "Đã đăng xuất" }
```

Server-side:

1. Verify refresh → `REVOKE_SESSION` Lua: DEL `refreshToken:{jti}` + rewrite `sessions`
   (**KEEPTTL** — không reset TTL cả list). **Idempotent:** token lạ/hết hạn → vẫn 200.
2. Xóa cache `user:{id}` (best-effort).
3. Xóa cookie `refreshCookie` (cùng name/path/Secure để browser match đúng entry).
4. Redis lỗi trên revoke → 503 fail-closed (không "đăng xuất" giả khi session còn sống).

---

## 7. Luồng 5 — Quên mật khẩu

```
Client                        Server                           Redis           SMTP
  │ POST /auth/forgot-password/otp                            │                │
  │ { email }                  │── find_by_email ── DB        │                │
  │                            │   không thấy → vẫn 200 (*)   │                │
  │                            │── SET otp:reset:{email} ────►│ (5')           │
  │                            │── publish mail.otp ─────────────────────────►│
  │◄── 200 "Đã gửi OTP" ──────│                                │                │
  │                            │                                │                │
  │ POST /auth/forgot-password/reset                          │                │
  │ { email, otp, newPassword }│── OTP_CHECK otp:reset ───────►│ match?         │
  │                            │── hash + UPDATE password_hash│ DB             │
  │                            │── revoke ALL sessions ───────►│                │
  │◄── 200 "Đã đặt lại mật khẩu"                              │                │
```

### 7.1 `POST /api/auth/forgot-password/otp`

Request:

```jsonc
{ "email": "lemon@gmail.com" }
```

Response `200` — `messageOnly("Nếu email tồn tại, chúng tôi đã gửi mã OTP")` (**luôn 200**):

```jsonc
{ "success": true, "message": "Nếu email tồn tại, chúng tôi đã gửi mã OTP" }
```

Server-side:

1. Rate-limit: 3/IP/60s + 3/email/giờ → `429`.
2. `find_by_email` — **không thấy → vẫn `Ok`** (anti-enumeration: không lộ email nào đã đăng ký).
3. Thấy user: `generate_otp` → `SET otp:reset:{email} = { code, attempts: 0 } EX 300` (fail-closed)
   → publish `mail.otp`.
4. Message trả về **giống hệt** nhau cả khi email không tồn tại (kể cả timing — vẫn chạy đủ các bước).

### 7.2 `POST /api/auth/forgot-password/reset`

Request:

```jsonc
{ "email": "lemon@gmail.com", "otp": "123456", "newPassword": "MatKhauMoi#123" }
```

Response `200` — `messageOnly("Đã đặt lại mật khẩu")`:

```jsonc
{ "success": true, "message": "Đã đặt lại mật khẩu" }
```

Server-side:

1. Rate-limit: 10/IP/60s.
2. `OTP_CHECK otp:reset:{email}` với otp gửi kèm → sai/hết hạn/locked → `400 INVALID_OTP`;
   đúng → **consume** (DEL key, dùng 1 lần).
3. `find_by_email` → không thấy → `404 USER_NOT_FOUND`.
4. `hash_password(newPassword)` → `UPDATE users SET password_hash`.
5. **Revoke toàn bộ session** (fail-closed — không được báo thành công khi session cũ còn sống):
   - Đọc `sessions:{userId}` → `[jti, …]`
   - `DELETE refreshToken:{jti1} … refreshToken:{jtiN} sessions:{userId} user:{userId} otp:reset:{email}`
6. Trả 200 — client về trang đăng nhập.

---

## 8. Luồng 6 — Đổi mật khẩu

### 8.1 `POST /api/users/me/password` *(cần access token)*

Header: `Authorization: Bearer <accessToken>`.

Request:

```jsonc
{ "oldPassword": "MatKhau#123", "newPassword": "MatKhauMoi#123" }
```

Response `200` — `messageOnly("Đã đổi mật khẩu")`:

```jsonc
{ "success": true, "message": "Đã đổi mật khẩu" }
```

Server-side:

1. `AuthUser` từ middleware (`401 UNAUTHORIZED` nếu thiếu/sai access token).
2. Rate-limit: 5/user/10 phút + 10/IP/10 phút → `429`.
3. `find_by_id` → `404 USER_NOT_FOUND` (user đã bị xóa mềm giữa chừng).
4. `verify_password(oldPassword, password_hash)` → sai → `400 INVALID_CREDENTIALS`
   (dùng chung code với đăng nhập — không xác nhận mật khẩu cũ có "đúng user" hay không).
5. `hash_password(newPassword)` → `UPDATE password_hash`.
6. **Revoke toàn bộ session** — fail-closed như §7.2 (xóa mọi `refreshToken:{jti}` +
   `sessions:{userId}` + cache `user:{userId}`). Client sẽ bị đăng xuất **mọi thiết bị**,
   kể cả thiết bị đang đổi — bắt buộc login lại với mật khẩu mới.
7. Trả 200.

> **Tại sao revoke-all:** đổi mật khẩu là hành động "khôi phục quyền sở hữu" — nếu thiết bị bị
> đánh cắp vẫn còn session cũ thì việc đổi mật khẩu vô nghĩa. Không có "giữ phiên hiện tại".

---

## 9. Bảng endpoint tổng hợp

| Method | Endpoint | Auth | Body (JSON) | Response `data` | Rate limit |
|---|---|---|---|---|---|
| `POST` | `/api/auth/request-otp` | — | `{ email }` | — (messageOnly) | 3/IP/60s + 3/email/h |
| `POST` | `/api/auth/verify-otp` | — | `{ email, otp, purpose }` | `true` | 10/IP/60s |
| `POST` | `/api/auth/signup` | — | `{ email, fullName, username, password, otp }` | `UserResponse` (201) | 5/IP/h |
| `POST` | `/api/auth/login` | — | `{ identifier, password }` | `{ accessToken }` + cookie | 10/IP/60s + 5/id/15' |
| `POST` | `/api/auth/refresh` | cookie refresh | — | `{ accessToken }` + cookie mới | 30/IP/60s |
| `POST` | `/api/auth/logout` | cookie refresh | — | — (messageOnly) | 30/IP/60s |
| `POST` | `/api/auth/forgot-password/otp` | — | `{ email }` | — (messageOnly) | 3/IP/60s + 3/email/h |
| `POST` | `/api/auth/forgot-password/reset` | — | `{ email, otp, newPassword }` | — (messageOnly) | 10/IP/60s |
| `POST` | `/api/users/me/password` | Bearer access | `{ oldPassword, newPassword }` | — (messageOnly) | 5/user/10' + 10/IP/10' |

---

## 10. Rate limit & anti-abuse

| Chống | Cơ chế |
|---|---|
| Inbox-bomb (spam OTP vào 1 email) | 3 OTP/giờ/email — key `ratelimit:request-otp:email:{email}` |
| Brute-force mật khẩu | 5 lần/15 phút/identifier — đếm theo identifier để không đổi IP là reset |
| Brute-force OTP | `MAX_OTP_ATTEMPTS = 5` atomic trong Lua, **KEEPTTL** khi sai — không kéo dài cửa sổ |
| Dò email tồn tại | signup trả 409 khi trùng (chấp nhận leak 1 bit ở signup) nhưng **forgot-password luôn 200**; login luôn `INVALID_CREDENTIALS` |
| Replay refresh token | `ROTATE_REFRESH` Stale → revoke family |
| Đánh cắp session cũ | Đổi/reset mật khẩu → revoke-all |
| Session overflow (nhồi thiết bị) | Max 5 phiên/user, kick cũ nhất |
| Brute-force qua verify-otp | 10/IP/60s + vẫn tính attempts vào OTP key |

---

## 11. Bảng lỗi auth

Chi tiết catalogue: [RESPONSES.md §5.1](./RESPONSES.md). Dùng cho auth:

| Tình huống | Status | `code` |
|---|---|---|
| Thiếu/sai access token | 401 | `UNAUTHORIZED` |
| Refresh thiếu/hết hạn/đã thu hồi/replay | 401 | `INVALID_SESSION` |
| Sai identifier hoặc mật khẩu | 400 | `INVALID_CREDENTIALS` |
| Mật khẩu cũ sai (đổi mật khẩu) | 400 | `INVALID_CREDENTIALS` |
| OTP sai / hết hạn / bị hủy (5 lần) | 400 | `INVALID_OTP` |
| Email đã đăng ký | 409 | `USER_ALREADY_EXISTS` |
| Username đã tồn tại | 409 | `USERNAME_TAKEN` |
| Chưa xác thực email | 403 | `EMAIL_NOT_VERIFIED` |
| Tài khoản bị khóa (`is_active=false`) | 403 | `FORBIDDEN` |
| User không tồn tại (reset khi email lạ đã qua OTP — nhánh hiếm) | 404 | `USER_NOT_FOUND` |
| Body sai format/rule | 422 | `VALIDATION_ERROR` |
| Vượt rate-limit | 429 | `TOO_MANY_REQUESTS` |
| Redis lỗi trên path OTP/session | 503 | `SERVICE_UNAVAILABLE` |

---

## 12. Checklist bắt buộc

Trước khi ship auth, verify đủ:

- [ ] OTP 2 namespace **độc lập** (`otp:signup:` / `otp:reset:`) — OTP signup không dùng được cho reset
- [ ] OTP sai 5 lần → key bị **DEL** (phải xin lại, không dò tiếp)
- [ ] OTP đúng → **consume** (không dùng lại được lần 2)
- [ ] `ROTATE_REFRESH` atomic Lua; replay → **revoke family** (có test)
- [ ] Ký trước xoay sau — Redis lỗi = 503, phiên cũ còn nguyên
- [ ] Login luôn `INVALID_CREDENTIALS` cho mọi nhánh sai (không leak user tồn tại)
- [ ] Forgot-password luôn 200 (kể cả email lạ)
- [ ] Đổi/reset mật khẩu → **revoke-all** sessions, fail-closed
- [ ] Max 5 session, kick cũ nhất
- [ ] Cookie: `HttpOnly` + combo `SameSite`/`Secure` validate lúc boot
- [ ] Password hash argon2id, không log password/OTP/token
- [ ] Username `^[a-z0-9_]{3,30}$` enforce cả validate tầng API **và** CHECK constraint DB
- [ ] Rate limit đủ bảng §10
- [ ] Toàn bộ response theo envelope [RESPONSES.md](./RESPONSES.md)
