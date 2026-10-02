# LEMON CHAT — Chuẩn Response Body (Success & Error)

> Chuẩn **dùng chung cho mọi API** của LEMON CHAT. Mọi endpoint REST đều trả về envelope
> thống nhất dưới đây — client không cần phân nhánh theo từng route.
>
> Tài liệu liên quan: [LEMON_CHAT.md](./LEMON_CHAT.md) · [AUTH.md](./AUTH.md) · [SETUP.md](./SETUP.md)

---

## 1. Nguyên tắc

| Quy tắc | Nội dung |
|---|---|
| Envelope | Mọi response (2xx lẫn 4xx/5xx) đều là JSON envelope — không bao giờ trả body trần |
| Field đặt tên | `camelCase` ở mọi nơi (JSON request lẫn response) |
| `message` | Chuỗi hiển thị cho người dùng, **đa ngôn ngữ** theo header `Accept-Language` (mặc định `vi`) |
| `code` (error) | Mã **ổn định** `SCREAMING_SNAKE_CASE` — không đổi theo ngôn ngữ, client match `code` để xử lý logic, **không** parse `message` |
| `data` | Chỉ xuất hiện khi có payload; bị bỏ qua (không serialize) khi `null`/không có |
| HTTP status | Vẫn đúng chuẩn (200/201/400/401/…); `success` ở body để client check 1 chỗ |
| Idempotent read | GET thành công luôn có `data` |

---

## 2. Success envelope

```jsonc
{
  "success": true,
  "message": "OK",
  "data": { /* payload — tùy endpoint, có thể không có */ }
}
```

| Trường | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `success` | `boolean` | luôn = `true` | |
| `message` | `string` | luôn có | Mặc định `"OK"`, hoặc thông điệp nghiệp vụ ("Đã gửi OTP", …) |
| `data` | `object` \| `array` | tùy endpoint | **Bỏ qua khi không có** (không serialize `"data": null`) |

### 2.1 Bốn dạng chuẩn

| Dạng | HTTP | Body | Dùng khi |
|---|---|---|---|
| `ok(data)` | `200` | `{ success: true, message: "OK", data }` | GET/PUT/PATCH thành công có payload |
| `withMessage(data, message)` | `200` | `{ success: true, message, data }` | Thành công muốn hiển thị thông điệp ("Đăng nhập thành công") |
| `created(data, message)` | `201` | `{ success: true, message, data }` | POST tạo resource (đăng ký, gửi tin, đăng story) |
| `messageOnly(message)` | `200` | `{ success: true, message }` — **không** `data` | Hành động không trả payload (gửi OTP, reset mật khẩu, logout) |

### 2.2 Ví dụ

```jsonc
// GET /api/users/me — 200
{
  "success": true,
  "message": "OK",
  "data": {
    "id": "0193a2b1-7c4d-7e8f-9a0b-1c2d3e4f5a6b",
    "email": "lemon@gmail.com",
    "username": "lemon_van",
    "fullName": "Nguyễn Văn Lemon",
    "avatarUrl": null,
    "emailVerifiedAt": "2026-10-02T03:12:00Z",
    "createdAt": "2026-10-02T03:10:00Z",
    "updatedAt": "2026-10-02T03:12:00Z"
  }
}

// POST /api/auth/request-otp — 200 (message only)
{
  "success": true,
  "message": "Đã gửi mã OTP đến email của bạn"
}

// POST /api/auth/signup — 201
{
  "success": true,
  "message": "Đăng ký thành công",
  "data": {
    "id": "0193a2b1-7c4d-7e8f-9a0b-1c2d3e4f5a6b",
    "email": "lemon@gmail.com",
    "username": "lemon_van",
    "fullName": "Nguyễn Văn Lemon"
  }
}

// Danh sách (GET /api/friends) — 200
{
  "success": true,
  "message": "OK",
  "data": [ { "id": "…", "username": "mint", "fullName": "Mint", "presence": "online" } ]
}
```

---

## 3. Error envelope

```jsonc
{
  "success": false,
  "code": "INVALID_OTP",
  "message": "Mã OTP không đúng hoặc đã hết hạn"
}
```

| Trường | Kiểu | Bắt buộc | Ghi chú |
|---|---|---|---|
| `success` | `boolean` | luôn = `false` | |
| `code` | `string` | luôn có | Mã ổn định (bảng §5) — client switch theo field này |
| `message` | `string` | luôn có | Nội dung theo `Accept-Language` |

**Không có `data` trong error body.** Chi tiết field sai (validate) nằm trong `message`
hoặc (tuỳ chọn mở rộng) thêm `"details": [{ field, message }]` — nếu mở rộng thì giữ
nguyên 3 field bắt buộc trên.

### 3.1 Ví dụ

```jsonc
// 401 — sai mật khẩu
{ "success": false, "code": "INVALID_CREDENTIALS", "message": "Thông tin đăng nhập không đúng" }

// 400 — OTP sai
{ "success": false, "code": "INVALID_OTP", "message": "Mã OTP không đúng hoặc đã hết hạn" }

// 422 — validate body
{ "success": false, "code": "VALIDATION_ERROR", "message": "username chỉ được chứa chữ thường, số và dấu gạch dưới" }

// 429 — rate limit
{ "success": false, "code": "TOO_MANY_REQUESTS", "message": "Bạn thao tác quá nhiều, vui lòng thử lại sau" }

// 503 — Redis sập trên path critical (auth/OTP)
{ "success": false, "code": "SERVICE_UNAVAILABLE", "message": "Dịch vụ tạm thời gián đoạn, vui lòng thử lại" }
```

---

## 4. HTTP status mapping

| Status | Khi nào | `code` tiêu biểu |
|---|---|---|
| `200 OK` | Thành công (`ok` / `withMessage` / `messageOnly`) | — |
| `201 Created` | Tạo resource (`created`) | — |
| `400 Bad Request` | Sai input nghiệp vụ, OTP sai, mật khẩu cũ sai | `INVALID_OTP`, `INVALID_CREDENTIALS`, `INVALID_OLD_PASSWORD`, `BAD_REQUEST` |
| `401 Unauthorized` | Chưa đăng nhập / token sai / phiên bị thu hồi | `UNAUTHORIZED`, `INVALID_SESSION` |
| `403 Forbidden` | Không có quyền (kể cả email chưa verify) | `FORBIDDEN`, `EMAIL_NOT_VERIFIED` |
| `404 Not Found` | Resource không tồn tại | `USER_NOT_FOUND`, `CONVERSATION_NOT_FOUND`, `MESSAGE_NOT_FOUND`, `NOT_FOUND` |
| `409 Conflict` | Trùng dữ liệu / trạng thái xung đột | `USER_ALREADY_EXISTS`, `USERNAME_TAKEN`, `ALREADY_FRIENDS`, `CONFLICT` |
| `422 Unprocessable Entity` | Body parse đúng nhưng fail rule validate | `VALIDATION_ERROR` |
| `429 Too Many Requests` | Vượt rate-limit | `TOO_MANY_REQUESTS` |
| `500 Internal Server Error` | Lỗi hệ thống (DB, bug, …) | `INTERNAL_SERVER_ERROR` |
| `503 Service Unavailable` | Dependency critical sập (Redis trên path auth/OTP) — **retry được** | `SERVICE_UNAVAILABLE` |

> **Phân biệt 500 vs 503:** 503 = sự cố tạm thời của dependency bắt buộc, client/LB **retry**;
> 500 = lỗi server, báo cáo — đừng retry mù quáng.

---

## 5. Catalogue `code` (ổn định, không đổi theo ngôn ngữ)

### 5.1 Auth & Users

| `code` | Status | Ý nghĩa |
|---|---|---|
| `UNAUTHORIZED` | 401 | Thiếu/sai access token |
| `FORBIDDEN` | 403 | Không có quyền truy cập |
| `INVALID_CREDENTIALS` | 400 | Sai identifier hoặc mật khẩu |
| `INVALID_OLD_PASSWORD` | 400 | Mật khẩu cũ không đúng (đổi mật khẩu) |
| `INVALID_OTP` | 400 | OTP sai / hết hạn / đã bị hủy (sai quá 5 lần) |
| `INVALID_SESSION` | 401 | Refresh token sai, đã bị thu hồi, hoặc replay |
| `EMAIL_NOT_VERIFIED` | 403 | Tài khoản chưa xác thực email |
| `USER_ALREADY_EXISTS` | 409 | Email đã được đăng ký |
| `USERNAME_TAKEN` | 409 | Username đã tồn tại |
| `USER_NOT_FOUND` | 404 | Không tìm thấy user |

### 5.2 Bạn bè

| `code` | Status | Ý nghĩa |
|---|---|---|
| `FRIEND_REQUEST_NOT_FOUND` | 404 | Lời mời kết bạn không tồn tại |
| `ALREADY_FRIENDS` | 409 | Đã là bạn bè |
| `FRIENDSHIP_BLOCKED` | 403 | Một trong hai phía đã block |
| `CANNOT_FRIEND_SELF` | 400 | Không thể tự kết bạn với mình |

### 5.3 Conversations & Messages

| `code` | Status | Ý nghĩa |
|---|---|---|
| `CONVERSATION_NOT_FOUND` | 404 | Không tìm thấy cuộc trò chuyện |
| `NOT_PARTICIPANT` | 403 | Không phải thành viên cuộc trò chuyện |
| `MESSAGE_NOT_FOUND` | 404 | Không tìm thấy tin nhắn |
| `MESSAGE_REVOKED` | 400 | Tin đã bị thu hồi (khi sửa/thu hồi lần nữa) |

### 5.4 Calls / Stories / Files

| `code` | Status | Ý nghĩa |
|---|---|---|
| `CALL_NOT_FOUND` | 404 | Không tìm thấy cuộc gọi |
| `CALL_BUSY` | 409 | Đối phương đang bận máy |
| `CALL_ENDED` | 409 | Cuộc gọi đã kết thúc |
| `STORY_NOT_FOUND` | 404 | Story không tồn tại hoặc đã hết hạn |
| `NOTE_TOO_LONG` | 400 | Note vượt 200 ký tự |
| `FILE_TOO_LARGE` | 400 | Vượt giới hạn dung lượng |
| `FILE_TYPE_NOT_ALLOWED` | 400 | MIME không trong whitelist |
| `PRESIGN_FAILED` | 500 | Không cấp được URL upload |

### 5.5 General

| `code` | Status | Ý nghĩa |
|---|---|---|
| `BAD_REQUEST` | 400 | Lỗi input chung |
| `NOT_FOUND` | 404 | Resource chung không tồn tại |
| `CONFLICT` | 409 | Xung đột dữ liệu chung |
| `VALIDATION_ERROR` | 422 | Validate body/path/query fail |
| `TOO_MANY_REQUESTS` | 429 | Vượt rate-limit |
| `INTERNAL_SERVER_ERROR` | 500 | Lỗi hệ thống |
| `SERVICE_UNAVAILABLE` | 503 | Dependency critical gián đoạn (Redis path auth) |

---

## 6. Đa ngôn ngữ (`message`)

- Client gửi `Accept-Language: vi` (mặc định) hoặc `en`.
- Server dịch `message` theo `code` + params (i18n resource file). `code` **luôn giữ nguyên**.
- `message` tự do (nội dung chi tiết từ validator) được trả nguyên văn, không dịch.

```
Accept-Language: en
→ { "success": false, "code": "INVALID_OTP", "message": "OTP code is invalid or expired" }
```

---

## 7. Áp dụng trong NestJS

### 7.1 Success — global interceptor

```ts
// success.interceptor.ts — bọc MỌI controller response
@Injectable()
export class SuccessInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(
      map((result) => {
        // Controller trả { data, message?, status? } từ helper ok()/withMessage()/created()/messageOnly()
        const { data, message = 'OK', status = 200 } = normalize(result);
        const body: Record<string, unknown> = { success: true, message };
        if (data !== undefined && data !== null) body.data = data; // bỏ qua khi không có
        return { status, body };
      }),
    );
  }
}
```

Helper cho controller (trả thuần, interceptor lo envelope):

```ts
export const ok = <T>(data: T) => ({ data });
export const withMessage = <T>(data: T, message: string) => ({ data, message });
export const created = <T>(data: T, message: string) => ({ data, message, status: 201 });
export const messageOnly = (message: string) => ({ message });
```

### 7.2 Error — global exception filter

```ts
// error.filter.ts — bắt mọi exception → envelope thống nhất + log đúng level
@Catch()
export class ErrorFilter implements ExceptionFilter {
  constructor(private readonly logger: LoggerService) {}

  catch(err: unknown, host: ArgumentsHost) {
    const lang = host.switchToHttp().getRequest().headers['accept-language'] ?? 'vi';
    const appErr = toAppError(err); // BusinessError | SystemError
    const status = appErr.status();
    const code = appErr.code();     // SCREAMING_SNAKE_CASE ổn định
    const message = translate(code, lang, appErr.params());
    // System → error (kèm stack + cause), Business → warn (không phải bug)
    if (appErr.isSystem()) this.logger.error({ code, err }, appErr.message);
    else this.logger.warn({ code }, appErr.message);
    host.switchToHttp().getResponse().status(status).json({ success: false, code, message });
  }
}
```

Phân loại lỗi (đồng bộ status mapping §4):

| Loại | Ví dụ | Status |
|---|---|---|
| `BusinessError` | `InvalidOtp`, `UserNotFound`, … | theo bảng §4/§5 |
| `SystemError::Cache` (Redis trên path critical) | rotate/OTP fail | **503** `SERVICE_UNAVAILABLE` |
| `SystemError::*` còn lại | DB, bug, … | **500** `INTERNAL_SERVER_ERROR` |
| Validation pipe fail | class-validator | **422** `VALIDATION_ERROR` |

Chi tiết error tracing (log shape, cause chain, requestId xuyên NATS): [SETUP.md §4](./SETUP.md).

### 7.3 Quy ước bất di bất dịch

1. Controller **không** tự build envelope — luôn qua helper + interceptor.
2. Service ném typed error (`BusinessError`/`SystemError`), **không** trả `{ success: false }` thủ công.
3. Thêm endpoint mới mà cần code lỗi mới → thêm vào catalogue §5 **trước**, rồi mới code.
4. `code` là contract với client — **đổi tên code = breaking change** (cần versioning).

### 7.4 Gói thư viện NestJS đề xuất (đều maintained, cộng đồng lớn)

| Việc | Package | Ghi chú |
|---|---|---|
| Validate body | `class-validator` + `class-transformer` + `ValidationPipe` | Tích hợp sẵn NestJS; map fail → `422 VALIDATION_ERROR` qua `exceptionFactory` |
| Validate bằng Zod (thay class-validator) | `nestjs-zod` + `zod` | Chọn 1 trong 2 — Zod mạnh hơn, share schema được với client |
| Envelope success/error | **tự viết** interceptor + filter (§7.1/§7.2) | Không có package chuẩn nào — 30 dòng là xong, đừng kéo dependency |
| JWT | **`jose`** | Zero-dependency, hiện đại (Auth.js, Cloudflare Workers cũng dùng); `SignJWT`/`jwtVerify` pin `algorithms: ['HS256']` chống algorithm-confusion; có sẵn JWKS nếu sau này nâng RS256/Google. **Bỏ `@nestjs/jwt`** (wrap `jsonwebtoken` 9, thêm `@types/jsonwebtoken`, ít lợi hơn) |
| JWT + NestJS CJS | `jose@4.15.9` (dual CJS/ESM) | `jose` v5/v6 **ESM-only** — NestJS build CommonJS mặc định sẽ không `require` được. Chọn 1: pin `jose@4`, hoặc build ESM (`module: NodeNext`), hoặc wrapper `JwtService` dùng `await import('jose')`. Khuyến nghị: wrapper `JwtService` + `jose` v6 — che được biên ESM/CJS, đổi lib sau này không sập domain |
| Guard Bearer | `AuthGuard` tự viết (~40 dòng) đọc `jose` verify qua `JwtService` wrapper | Self-written guard nhẹ hơn passport, dễ set `req.user` chuẩn `AuthUser` |
| Cookie refresh | `cookie-parser` + `@nestjs/platform-express` (`res.cookie(..., { httpOnly, sameSite, secure })`) | Fastify adapter: `@fastify/cookie` |
| argon2id | `argon2` | Prebuilt binary, native module chuẩn cho argon2id — **chốt rồi**, không thay bcrypt |
| Redis | `ioredis` (v6) | Đây chính là "package mạnh" của mảng Redis: Lua `defineCommand`, ConnectionManager auto-reconnect, pipeline. So với `node-redis` thì ioredis API Lua/script gọn hơn — giữ ioredis |
| Redis cho NestJS | `@nestjs-modules/ioredis` (hoặc custom `RedisModule` 20 dòng) | Custom provider được khuyến nghị — ít magic, inject được connection riêng cho pub/sub. Lua: `defineCommand('otpCheck', { numberOfKeys, lua })` — type-safe, giữ SHA 1 lần |
| Drizzle | `drizzle-orm` + `drizzle-kit` + `pg` | `drizzle(pool)` inject qua custom provider; migrate bằng `drizzle-kit migrate` |
| NATS JetStream | `nats` (official) | API `js.consumers.get(...)` / `msg.ack()` / `msg.nak(delay)` đầy đủ; hoặc `nestjs-nats-jetstream` nếu muốn DI sẵn |
| Mail | `@nestjs-modules/mailer` (Nodemailer + Handlebars adapter) | Hoặc Nodemailer thuần trong worker — đủ dùng nếu chỉ gửi OTP template |
| Socket.IO | `@nestjs/websockets` + `@nestjs/platform-socket.io` + `@socket.io/redis-adapter` | Official — gateway class y hệt controller |
| i18n message | `nestjs-i18n` | Resolver theo `Accept-Language`, resource file `vi.json`/`en.json` — khớp §6 |
| Health check | `@nestjs/terminus` | Gộp Postgres/Redis/NATS/S3 vào `/health/ready` |
| Toolchain | **Bun** (thay npm/node cho install + run + test) | Bảng đổi lệnh + scripts mẫu: [SETUP.md §1](./SETUP.md) |
| Test | **`bun test`** (`bun:test`) + `testcontainers` + `supertest` + `socket.io-client` | `bun add -d` để cài, `bun test` để chạy — chi tiết [SETUP.md §1](./SETUP.md), [LEMON_CHAT.md §18](./LEMON_CHAT.md) |
| Log | `nestjs-pino` | JSON log + request-id + redact — setup đầy đủ [SETUP.md §3](./SETUP.md) |
| Config | `@nestjs/config` + `zod` schema | Fail-fast boot khi env thiếu/sai — [SETUP.md §2](./SETUP.md) |

> **Contract là cross-language:** envelope, `code`, status map, flow auth trong docs là giao kèo
> với client — backend viết bằng ngôn ngữ/framework nào cũng phải trả về đúng y hệt. Client
> không phân biệt được công nghệ backend nếu implement đúng docs này.

---

## 8. Áp dụng trong NestJS — Redis dependency tradeoff

Hệ Redis-first (token/OTP/session **chỉ** sống ở Redis) đánh đổi: **Redis là hard dependency**
của auth path — Redis chết = không đăng nhập/refresh được (đúng ý đồ fail-closed). Giảm thiểu:

| Giải pháp | Ghi chú |
|---|---|
| AOF `everysec` + replica | Mất dữ liệu ≤ 1s; session mất thì user login lại — chấp nhận được |
| Sentinel / Cluster | Tự failover, hạn chế downtime single-node |
| `/health/ready` check Redis | LB ngừng đưa traffic vào node mất Redis — thay vì 503 lan |
| 503 đúng chuẩn `SERVICE_UNAVAILABLE` | Client retry có backoff, **không** hiểu nhầm là logout |
| Không có memory-fallback cho session | Cố ý: fallback đa node = rotate không atomic = lỗ hổng replay |

> Đây là **quyết định có chủ đích** (đã chốt): ưu tiên tính đúng đắn của rotate/OTP atomic hơn
> là "auth sống sót khi Redis chết". Nếu sau này cần auth availability cao hơn → Redis HA
> (chọn hạ tầng), **không** bịa fallback in-process.
