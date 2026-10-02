# LEMON CHAT — SETUP: Bun · Config · Logging · Error Tracing

> Toolchain **Bun** (thay npm/node), cấu hình env fail-fast, logging chuẩn JSON, cách trace
> system error. Chi tiết contract response: [RESPONSES.md](./RESPONSES.md) · Auth: [AUTH.md](./AUTH.md) ·
> Tổng quan: [LEMON_CHAT.md](./LEMON_CHAT.md)

---

## Mục lục

1. [Toolchain Bun (thay npm)](#1-toolchain-bun-thay-npm)
2. [Config — env fail-fast](#2-config--env-fail-fast)
3. [Logging — setup backend log](#3-logging--setup-backend-log)
4. [System error tracing](#4-system-error-tracing)
5. [Checklist](#5-checklist)

---

## 1. Toolchain Bun (thay npm)

Dùng **Bun** cho toàn bộ: package manager, runtime, test runner, script runner.

| npm/node cũ | Bun | Việc |
|---|---|---|
| `npm install` | `bun install` | Cài dependency (đọc `bun.lock`) |
| `npm install <pkg>` / `npm i -S <pkg>` | `bun add <pkg>` | Thêm dependency |
| `npm install -D <pkg>` | `bun add -d <pkg>` | Thêm devDependency |
| `npx <bin>` | `bunx <bin>` | Chạy binary 1 lần (`bunx drizzle-kit generate`) |
| `npm run <script>` | `bun run <script>` | Chạy script trong `package.json` |
| `npm test` / `jest` | `bun test` | Test runner (`bun:test`, API quen thuộc kiểu Jest) |
| `node dist/main.js` | `bun run dist/main.js` (hoặc `bun src/main.ts` khi dev) | Chạy app |
| `npm ci` (CI) | `bun install --frozen-lockfile` | Cài theo lockfile, không sửa lock |
| `npm outdated` | `bun outdated` | Kiểm tra version cũ |

### 1.1 `package.json` scripts mẫu

```jsonc
{
  "scripts": {
    "dev": "bun --watch src/main.ts",
    "build": "bun build ./src/main.ts --outdir dist --target node",
    "start": "bun run dist/main.js",
    "test": "bun test",
    "test:watch": "bun test --watch",
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "drizzle:generate": "bunx drizzle-kit generate",
    "drizzle:migrate": "bunx drizzle-kit migrate"
  }
}
```

### 1.2 Lưu ý Bun + NestJS

| Chủ điểm | Ghi chú |
|---|---|
| Runtime | Bun chạy được NestJS với cả Express lẫn Fastify adapter — mặc định giữ Express adapter cho đỡ lệch package (`cookie-parser`…); Fastify thì dùng `@fastify/cookie` |
| Event loop & worker | Mail queue (`p-queue`) chạy **cùng event loop** — ổn vì SMTP là I/O `await` (không block). Việc **CPU-bound** (probe media/transcode/encrypt lớn) mới tách `worker_threads` (Bun native) hoặc `piscina` — đừng tách thread cho I/O, thừa complexity không lợi |
| Native module | `argon2` có prebuilt binary — `bun install` là xong; nếu rơi vào env lạ thì `bun rebuild argon2` |
| Test | `bun test` (import `{ describe, it, expect }` từ `bun:test`) — API quen thuộc, **không** cần Jest config. Testcontainers/supertest/socket.io-client chạy bình thường dưới `bun test` |
| ESM/CJS | Bun nuốt được cả hai — `jose` v6 ESM-only không còn là vấn đề với Bun runtime (chỉ còn vấn đề khi build bằng `tsc` ra CJS cho Node thuần; đã bọc `JwtService` wrapper như [RESPONSES.md §7.4](./RESPONSES.md)) |
| Lockfile | Commit `bun.lock`; CI luôn `bun install --frozen-lockfile` |
| Docker | Image `oven/bun:1-slim` multi-stage: stage 1 `bun install --frozen-lockfile` + `bun run build`, stage 2 copy dist + `bun install --production` |

---

## 2. Config — env fail-fast

**Nguyên tắc:** config validate **một lần lúc boot** — thiếu/sai env là **exit 1 ngay**, không để
runtime vỡ nửa chừng. Runtime chỉ đọc config đã validate (typed), không `process.env` rải rác.

### 2.1 Cấu trúc

```
src/infra/config/
├── env.schema.ts     # Zod schema cho toàn bộ env (single source of truth)
├── config.module.ts  # ConfigModule.forRoot + custom provider typed
├── constants.ts      # Hằng số KHÔNG phải env (TTL, rate-limit, giới hạn file…)
└── cookie.ts         # validate combo SameSite/Secure (fail-fast)
```

### 2.2 Env schema (Zod) — `env.schema.ts`

```ts
import { z } from 'zod';

export const envSchema = z
  .object({
    APP_ENV: z.enum(['development', 'test', 'production']).default('development'),
    HOST: z.string().default('0.0.0.0'),
    PORT: z.coerce.number().int().positive().default(3000),
    FRONTEND_URL: z.string().min(1), // comma-separated CORS origins

    // DB
    DATABASE_URL: z.string().url(),
    DB_MAX_CONNECTIONS: z.coerce.number().int().positive().default(20),
    DB_MIN_CONNECTIONS: z.coerce.number().int().nonnegative().default(5),

    // Redis
    REDIS_URL: z.string().url(),
    REDIS_CONNECTION_TIMEOUT: z.coerce.number().int().positive().default(5000),

    // JWT — bắt buộc ≥ 32 bytes, fail-closed
    JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 bytes'),
    JWT_ISSUER: z.string().default('lemon-chat'),
    JWT_AUDIENCE: z.string().default('lemon-chat-client'),

    // Cookie
    COOKIE_SECURE: z
      .enum(['true', 'false'])
      .default('false')
      .transform((v) => v === 'true'),
    COOKIE_SAMESITE: z.enum(['strict', 'lax', 'none']).default('strict'),

    // SMTP (production bắt buộc đủ, dev cho phép log-only)
    SMTP_HOST: z.string().optional(),
    SMTP_PORT: z.coerce.number().int().optional(),
    SMTP_USERNAME: z.string().optional(),
    SMTP_PASSWORD: z.string().optional(),
    SMTP_FROM: z.string().optional(),

    // S3
    S3_ENDPOINT: z.string().url(),
    S3_BUCKET: z.string().min(1),
    S3_ACCESS_KEY: z.string().min(1),
    S3_SECRET_KEY: z.string().min(1),

    // Log
    LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace']).default('info'),
  })
  .superRefine((env, ctx) => {
    // SameSite=None không có Secure → browser reject cookie → auth gãy âm thầm
    if (env.COOKIE_SAMESITE === 'none' && !env.COOKIE_SECURE) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        path: ['COOKIE_SAMESITE'],
        message: 'COOKIE_SAMESITE=none requires COOKIE_SECURE=true',
      });
    }
    // Production: thiếu SMTP là fail — không được ship mode log-only
    if (env.APP_ENV === 'production') {
      for (const key of ['SMTP_HOST', 'SMTP_PORT', 'SMTP_USERNAME', 'SMTP_PASSWORD', 'SMTP_FROM'] as const) {
        if (!env[key]) {
          ctx.addIssue({
            code: z.ZodIssueCode.custom,
            path: [key],
            message: `${key} is required in production`,
          });
        }
      }
    }
  });

export type EnvConfig = z.infer<typeof envSchema>;
```

### 2.3 Boot fail-fast — `config.module.ts`

```ts
import { Module } from '@nestjs/common';
import { ConfigModule as NestConfigModule, ConfigService } from '@nestjs/config';
import { envSchema, type EnvConfig } from './env.schema';

@Module({
  imports: [
    NestConfigModule.forRoot({
      isGlobal: true,
      // Validate ngay lúc boot — ZodError → throw → main.ts catch → exit(1)
      validate: (raw) => {
        const parsed = envSchema.safeParse(raw);
        if (!parsed.success) {
          // In đủ field lỗi để fix env trong 1 lần, không đoán mò
          console.error('❌ Invalid environment variables:', parsed.error.flatten().fieldErrors);
          process.exit(1);
        }
        return parsed.data;
      },
    }),
  ],
  providers: [
    {
      provide: 'APP_ENV_TYPED',
      // ConfigService#get đã trả đúng EnvConfig sau validate
      useFactory: (cs: ConfigService<EnvConfig, true>) => cs,
      inject: [ConfigService],
    },
  ],
  exports: ['APP_ENV_TYPED'],
})
export class AppConfigModule {}
```

**Quy ước config:**

| Quy ước | Nội dung |
|---|---|
| 1 schema | Mọi env nằm trong `env_schema` — không `process.env.X` ở service/controller |
| Typed | Inject `ConfigService<EnvConfig, true>` — `config.get('JWT_SECRET', { infer: true })` có autocomplete |
| Constants ≠ env | TTL, rate-limit, `MAX_OTP_ATTEMPTS`, giới hạn file… vào `constants.ts` (SCREAMING_SNAKE) — đổi code là đổi hành vi, không giấu trong env |
| Secrets | Dev: `.env` (gitignore). Prod: secret manager / env runtime của orchestrator — **không** commit `.env`, không log giá trị secret |
| Cookie combo | Validate `SameSite=none` + `Secure` lúc boot (§2.2) — sai là exit, không để auth gãy âm thầm |
| Multi-env | `.env.development` / `.env.test` / `.env.production`; `APP_ENV` quyết định rule chặt hơn (SMTP bắt buộc ở production) |

### 2.4 Hằng số — `constants.ts` (trích, đồng bộ [AUTH.md §1](./AUTH.md))

```ts
export const OTP_EXPIRATION = 120;              // 2 phút
export const MAX_OTP_ATTEMPTS = 5;
export const ACCESS_TOKEN_EXPIRATION = 15 * 60; // 15 phút
export const REFRESH_TOKEN_EXPIRATION = 7 * 24 * 60 * 60;
export const MAX_SESSIONS_PER_USER = 5;
export const CACHE_EXPIRATION = 60;
export const RATE_LIMIT_REQUEST_OTP_IP_MAX = 3;
export const RATE_LIMIT_REQUEST_OTP_EMAIL_MAX = 3;
export const RATE_LIMIT_VERIFY_OTP_IP_MAX = 10;
export const RATE_LIMIT_SIGNIN_IP_MAX = 5;
export const RATE_LIMIT_CHANGE_PASSWORD_USER_MAX = 3;
export const RATE_LIMIT_CHANGE_PASSWORD_IP_MAX = 10;
export const RATE_LIMIT_EMAIL_WINDOW = 3600; // 1 giờ
```

---

## 3. Logging — setup backend log

Dùng **`nestjs-pino`** (wrap `pino`): JSON log, request-id tự động, không `console.log` ở
production code.

### 3.1 Wiring — `observability/logger.module.ts`

```ts
import { LoggerModule } from 'nestjs-pino';
import { randomUUID } from 'node:crypto';

@Module({
  imports: [
    LoggerModule.forRoot({
      pinoHttp: {
        level: process.env['LOG_LEVEL'] ?? 'info',
        // JSON ở mọi env khi prod; pretty khi dev cho dễ đọc
        transport:
          process.env['APP_ENV'] === 'development'
            ? { target: 'pino-pretty', options: { singleLine: true } }
            : undefined,
        // Request-id: nhận từ header nếu có (LB/FE gửi), không thì sinh UUID
        genReqId: (req, res) => {
          const incoming = req.headers['x-request-id'];
          const id = typeof incoming === 'string' && incoming ? incoming : randomUUID();
          res.setHeader('x-request-id', id); // echo về client → đối chiếu ticket lỗi
          return id;
        },
        // Log shape gọn, không log query/body thô
        serializers: {
          req: (req) => ({ method: req.method, url: req.url, id: req.id }),
          res: (res) => ({ statusCode: res.statusCode }),
        },
        // AUTO-REDACT — không bao giờ để secret lọt log
        redact: {
          paths: [
            'req.headers.authorization',
            'req.headers.cookie',
            'req.headers["set-cookie"]',
            'req.body.password',
            'req.body.oldPassword',
            'req.body.newPassword',
            'req.body.otp',
            '*.password',
            '*.passwordHash',
            '*.otp',
            '*.accessToken',
            '*.refreshToken',
            '*.secret',
          ],
          censor: '***',
        },
        // Request thành công ồn ào thì tắt; lỗi thì phải thấy
        autoLogging: {
          ignore: (req) => req.url === '/health/live' || req.url === '/metrics',
        },
      },
    }),
  ],
})
export class ObservabilityModule {}
```

### 3.2 Quy tắc log level (bắt buộc thống nhất)

| Level | Dùng cho | Ví dụ |
|---|---|---|
| `fatal` | Process sắp chết (không recover được) | Boot config sai đã exit, mất invariant nội bộ |
| `error` | **SystemError** — bug/dependency chết, cần nhìn | DB query fail, Redis 503, unhandled |
| `warn` | **BusinessError** — hành vi bất thường nhưng hệ thống sống | Login sai, OTP sai, replay refresh (`refresh_stale`), rate-limit dính |
| `info` | Milestone nghiệp vụ quan trọng | `user.signed_up`, `user.signed_in`, `password.changed`, boot xong |
| `debug` | Chi tiết triển khai | WS event từng cái, MailQueue enqueue/sent, cache hit/miss |
| `trace` | Gần raw (ít dùng) | Timing từng bước I/O khi profile |

**Cấm:** `console.log`/`println` trong production code — luôn inject `Logger` từ NestJS
(`private readonly logger = new Logger(AuthService.name)`) hoặc `logger` của pino context.

### 3.3 Log shape chuẩn

Mọi dòng log JSON có **requestId** (child logger) để nối chuỗi:

```jsonc
{
  "level": 30,
  "time": 1759470000000,
  "requestId": "b3f1…",
  "context": "AuthService",
  "msg": "user.signed_in",
  "userId": "0193a2b1-…",
  "method": "POST",
  "url": "/api/auth/login",
  "responseTime": 42
}
```

| Quy ước | Nội dung |
|---|---|
| `msg` | Event name dạng `domain.action` (`user.signed_up`, `otp.locked`) — **không** viết câu tùy tiện, để group/aggregation được |
| Context | Tên class service/gateway/filter — NestJS `Logger(Class.name)` tự gắn |
| requestId | 1 request = 1 id xuyên suốt HTTP + service + MailQueue job (kèm `requestId` trong job payload) |
| WS | Gateway log `debug` từng event; `info` chỉ milestone (`call.ended`); mọi dòng kèm `socketId` + `userId` |
| MailQueue job | Kèm `requestId` + `jobId` trong payload — log khi enqueue/sent/fail **có requestId cũ** → truy vết từ user report → 1 chuỗi HTTP → queue → SMTP |
| Mask | Email mask `l***n@gmail.com`; **không** log password/OTP/token/secret bao giờ (đã `redact` tự động, nhưng cũng đừng tự tay `logger.info({ otp })`) |

---

## 4. System error tracing

### 4.1 Phân loại lỗi → log level → trace

```
Exception thrown trong handler/service
        │
        ▼
  ErrorFilter (global)  ←── [RESPONSES.md §7.2]
        │
        ├─ BusinessError ──► logger.warn({ code, msg, userId })     // không stack (không phải bug)
        │                    response 4xx envelope
        │
        ├─ SystemError  ──► logger.error({ code, err, stack, cause })
        │                    response 5xx envelope (503 nếu Cache)
        │
        └─ unknown Error ─► logger.error({ err, stack })  // wrap thành INTERNAL_SERVER_ERROR
                             response 500 envelope
```

### 4.2 ErrorFilter mở rộng — log đủ context để trace

```ts
@Catch()
export class ErrorFilter implements ExceptionFilter {
  constructor(private readonly logger: LoggerService) {}

  catch(err: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const req = ctx.getRequest();
    const appErr = toAppError(err); // BusinessError | SystemError (RESPONSES.md §7.2)
    const status = appErr.status();
    const code = appErr.code();

    // Payload chung để TRUY VẾT: 1 dòng log = 1 lỗi, có đủ neo để tìm
    const trace = {
      code,
      requestId: req.id,
      userId: req.user?.id,
      method: req.method,
      url: req.url,
      // Với SystemError: giữ message + stack + cause chain (Nguyên nhân gốc)
      ...(appErr.isSystem() ? { err: serializeError(err) } : {}),
    };

    if (appErr.isSystem()) this.logger.error(trace, appErr.message);
    else this.logger.warn(trace, appErr.message); // Business 4xx — không phải bug

    ctx.getResponse().status(status).json({ success: false, code, message: translate(code, req) });
  }
}

/** Giữ message + stack + cause (Error.cause) — mất cause là mất dấu gốc rễ. */
function serializeError(err: unknown) {
  if (!(err instanceof Error)) return { message: String(err) };
  return {
    name: err.name,
    message: err.message,
    stack: err.stack,
    cause: err.cause instanceof Error ? serializeError(err.cause) : err.cause,
  };
}
```

**`SystemError` luôn wrap `cause`** (đừng nuốt lỗi):

```ts
// ✅ giữ nguyên nhân gốc
export class RedisOpError extends Error {
  constructor(op: string, cause: Error) {
    super(`redis ${op} failed`, { cause });
    this.name = 'RedisOpError';
  }
}

// ❌ mất dấu — không bao giờ làm vậy
// throw new Error('something failed');
```

### 4.3 Trace xuyên service → MailQueue (in-process)

Không có service nào khác — mail queue chạy trong cùng process (p-queue), nên trace rất gọn:

| Bước | Cơ chế |
|---|---|
| HTTP request | `genReqId` (§3.1) — mọi log trong request là child logger với `requestId` |
| Service nội bộ | `logger.log`/`error` của NestJS context — tự kèm context class; truyền `requestId` qua service call khi khác boundary |
| Enqueue mail | Đóng `requestId` + `jobId` vào job payload trước khi `mailQueue.enqueue()` |
| MailQueue consumer | Trong cùng process — đọc `requestId` từ job payload → child logger → log sent/fail **có requestId cũ** → truy vết từ user report → 1 chuỗi HTTP → queue → SMTP |
| WS event | `socket.handshake.auth.requestId` hoặc sinh mới khi connect; log kèm `socketId` |
| Error không bắt được | `process.on('unhandledRejection'|'uncaughtException')` → `logger.fatal` + graceful shutdown (§[LEMON_CHAT.md 17.2](./LEMON_CHAT.md)) |

### 4.4 Lỗi dependency → hành vi + log (đồng bộ [RESPONSES.md §8](./RESPONSES.md))

| Dependency | Path | Hành vi client | Log |
|---|---|---|---|
| Redis | OTP/session/rotate (critical) | `503 SERVICE_UNAVAILABLE` | `error` + `err` (RedisError) — cảnh báo oncall |
| Redis | cache read (best-effort) | đi tiếp vào DB | `warn` `cache.fallback` — không spam (sample 1%) |
| Redis | rate-limit (fail-open) | cho qua request | `warn` `ratelimit.degraded` |
| Postgres | mọi query | `500 INTERNAL_SERVER_ERROR` | `error` + stack + cause |
| S3 | presign/upload verify | `500`/`PRESIGN_FAILED` | `error` + stack |
| SMTP | gửi mail (qua MailQueue) | không fail request (fire-and-forget) | `warn` `mail.send_failed`, retry 2 lần (1s/5s) trong queue |

### 4.5 Alert metric (từ log/metric — [LEMON_CHAT.md §17.1](./LEMON_CHAT.md))

| Tín hiệu | Ý nghĩa | Hành động |
|---|---|---|
| `refresh_stale_total` tăng | **có thể replay token** | Điều tra ngay (an ninh) |
| `otp_locked_total` tăng | brute-force OTP diện rộng | Xem IP nguồn, siết rate-limit |
| `5xx rate` tăng | bug hoặc dependency chết | Xem log `error` theo `requestId` cluster |
| `mail_queue_size` chạm trần (128) | SMTP nghẽn, mail drop | Xem log `mail.send_failed`, kiểm tra SMTP |
| `redis_op_duration_seconds` p99 cao | Redis nghẽn | Kiểm tra AOF/replica, slow log |

---

## 5. Checklist

- [ ] Toolchain Bun: `bun install` / `bun test` / `bunx` — không npm/npx còn sót trong scripts + CI + Dockerfile
- [ ] `bun.lock` commit; CI `bun install --frozen-lockfile`
- [ ] Env schema Zod validate **một lần** lúc boot — sai env là `exit 1`, có in field lỗi
- [ ] `JWT_SECRET` ≥ 32 bytes bắt buộc; `SameSite=none` bắt buộc `Secure=true`; production thiếu SMTP là fail boot
- [ ] Không `process.env` ngoài config module; không `console.log` trong production code
- [ ] Log JSON qua `nestjs-pino`; `genReqId` + echo `x-request-id`
- [ ] `redact` đủ password/otp/token/cookie/secret
- [ ] Log level đúng quy tắc: System→`error`, Business→`warn`, milestone→`info`, chi tiết→`debug`
- [ ] `msg` dạng `domain.action`; mọi dòng có `requestId` (kể cả MailQueue job — lấy từ job payload)
- [ ] `SystemError` wrap `cause` — không nuốt nguyên nhân gốc
- [ ] `ErrorFilter` log Business=`warn` (không stack), System=`error` (đủ stack + cause)
- [ ] Health: `/health/live` + `/health/ready` không auto-log (giảm nhiễu)
- [ ] `unhandledRejection`/`uncaughtException` → `logger.fatal` + graceful shutdown
