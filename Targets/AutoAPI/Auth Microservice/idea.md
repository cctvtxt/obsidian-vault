![[Pasted image 20260416203337.png]]


че-то вроде требований к сервису аутентификации

Auth Microservice 

stateless-оркестратор токенов, задача: аутентификация, регистрация, генерация, отзыв и валидация токенов,
stateless авторизация с JWT + OAuth 2.1 с OIDC для IdP гугла и гитхаба. Ротация рефреш токенов. Функция генерация anti-csrf токенов и их проверки, но не факт что понадобятся,
graceful shutdown, graceful degradation,
в jwt в claims ничего не будем нестандартного добавлять как будто: у всех пользователей одна роль,
контракт взаимодействия с gRPC Register, Login, Refresh, Logout, LogoutAll, GetMe, VerifyEmail, ResetPassword, ResendVerification, ForgotPassword, ChangePassword,
API Gateway делаем 
POST   v1/auth/register
POST   v1/auth/login
POST   v1/auth/refresh
POST   v1/auth/logout
POST   v1/auth/logout-all
GET    v1/auth/me
POST   v1/auth/verify-email
POST   v1/auth/resend-verification
POST   v1/auth/forgot-password
POST   v1/auth/reset-password
POST   v1/auth/change-password,
health api: реализовать стандартное grpc.health.v1,
бд: PostgreSQL с нарушением всех этих ваших DATABASE PER SERVICE, как будто в нее отдельно должен ходить сервис подписок и добавлять привязки к аккаунтам мб не знаю,
кеш: Redis нужен для blacklista токенов




1. ## СТРУКТУРА ТЗ МИКРОСЕРВИСА АВТОРИЗАЦИИ

2. Назначение и границы ответственности

- Централизованная выдача, валидация и отзыв токенов доступа
- Управление ролями, разрешениями, атрибутами пользователей и ресурсами
- Внешняя оценка политик доступа (RBAC / ABAC / ReBAC)
- Аудит событий доступа и изменений политик
- **Не входит:** бизнес-логика доменных сервисов, UI логина/регистрации (отдаётся на фронтенд или отдельный сервис Identity UI)

1. Функциональные требования

|Категория|Требования| |---|---| |**Протоколы**|OAuth 2.1 / OIDC Core 1.0. Поддержка `authorization_code + PKCE`, `client_credentials`, `refresh_token`. Implicit flow запрещён.| |**Токены**|JWT (RFC 7519) для access token. Поддержка introspection (RFC 7662) и token revocation. Ротация signing keys без downtime.| |**Авторизация**|Поддержка RBAC и/или ABAC. Вынос политик в движок (OPA, Casbin, Cedar). Claims в токене минимальны, детали подгружаются по необходимости.| |**Управление доступом**|REST/gRPC API для CRUD ролей, групп, политик, маппинга `user ↔ resource ↔ action`. Поддержка версионирования политик.| |**Сессии**|Stateless access tokens. Хранение refresh token/sessions в защищённом хранилище с TTL, поддержкой принудительного отзыва.| |**События**|Webhook / Kafka / CloudEvents при: входе, выходе, смене роли, отзыве токена, превышении лимитов.|

1. Нефункциональные требования

|Параметр|Требование| |---|---| |**Доступность**|≥ 99.9% (SLA). Active-Active репликация, graceful degradation при падении зависимостей.| |**Производительность**|Валидация токена ≤ 50ms (p95). Горизонтальное масштабирование без stateful зависимостей в hot path.| |**Хранение**|PostgreSQL/MySQL для политик и аудитов. Redis/Memcached для кэша JWK, blacklist токенов, rate limits.| |**Мультиарендность**|(опционально) `tenant_id` во всех запросах и токенах. Изоляция данных на уровне БД или схемы.| |**Совместимость**|Kubernetes-native, readiness/liveness probes, graceful shutdown, config via env/ConfigMap/Secrets.|

1. Требования безопасности

- TLS 1.3 везде (in transit), шифрование чувствительных данных at rest (AES-256-GCM или аналог)
- Обязательный PKCE для публичных клиентов
- Rate limiting + CAPTCHA/2FA после N неудачных попыток
- Защита от replay, token fixation, privilege escalation
- Ротация секретов и ключей подписи каждые 30-90 дней (автоматизировано)
- Соответствие 152-ФЗ / GDPR / PCI-DSS (при обработке платежей)
- Immutable audit log (WORM или append-only таблица)

1. Интеграционные требования

- OpenAPI 3.1 + AsyncAPI для событий
- Поддержка API Gateway (Kong, Traefik, Apigee, NGINX) через JWT validation plugin
- SDK на Go/Java/Python/Node.js для валидации токенов и работы с политиками
- Совместимость с Service Mesh (Istio/Linkerd) через JWT request authentication

1. Эксплуатация и наблюдаемость

- Метрики (Prometheus): `auth_token_validation_duration_seconds`, `auth_policy_evaluation_total`, `active_sessions`, `error_rate`, `key_rotation_status`
- Логи: структурированные (JSON), correlation ID, без PII в plaintext
- Tracing: OpenTelemetry, сквозной trace_id через все сервисы
- Healthchecks: `/healthz`, `/readyz`, `/debug/vars`
- Автоматические бэкапы политик и аудитов, план восстановления (RPO ≤ 15min, RTO ≤ 1h)

## Итоговая карта безопасности

|Угроза|Контрмера|
|---|---|
|Brute force login|Rate limit (Redis sliding window) 5/15 мин по email|
|Credential stuffing|Captcha на Register + Login|
|JWT кража|Короткий TTL (15 мин) + jti blocklist в Redis|
|Refresh token кража|Rotation + replay detection → полный отзыв сессии|
|CSRF|HMAC double-submit в gRPC metadata|
|OAuth state fixation|UUID v4, single-use, TTL 10 мин в Redis|
|PKCE bypass|SHA256 challenge верифицируется сервером|
|Timing attacks|`crypto.timingSafeEqual`, всегда хэшируем пароль|
|IdP token exposure|AES-256-GCM шифрование перед записью в БД|
|Container escape|`read_only`, `cap_drop: ALL`, `no-new-privileges`, UID 1001|
|Межсервисный MITM|mTLS (mutual TLS) между всеми gRPC-клиентами|
|Secrets в ENV|Docker secrets → `/run/secrets/`, не в env переменных|

```ts
@Module({ imports: [ ConfigModule.forRoot({ isGlobal: true, validate: validateEnv }), PrismaModule, RedisModule, AuthModule, OAuthModule, TokenModule, CaptchaModule, HealthModule, AuditModule, ], }) export class AppModule {}
```

{  
"dependencies": {  
"@nestjs/common": "^10",  
"@nestjs/core": "^10",  
"@nestjs/microservices": "^10",  
"@nestjs/config": "^3",  
"@nestjs/terminus": "^10",  
"@grpc/grpc-js": "^1.10",  
"@grpc/proto-loader": "^0.7",  
"prisma": "^5",  
"@prisma/client": "^5",  
"ioredis": "^5",  
"argon2": "^0.31",  
"bcrypt": "^5",  
"jsonwebtoken": "^9",  
"jose": "^5",  
"zod": "^3",  
"pino": "^8",  
"rxjs": "^7"  
},  
"devDependencies": {  
"@types/jsonwebtoken": "^9",  
"@types/bcrypt": "^5",  
"typescript": "^5",  
"jest": "^29",  
"@nestjs/testing": "^10"  
}  
}

```
auth-service/
├── proto/
│   └── auth.proto                  # gRPC контракт (единственный публичный интерфейс)
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── src/
│   ├── main.ts                     # Bootstrap: только gRPC, без HTTP
│   ├── app.module.ts
│   │
│   ├── modules/
│   │   ├── auth/                   # Регистрация/логин по email
│   │   │   ├── auth.module.ts
│   │   │   ├── auth.grpc.controller.ts   # GrpcMethod handlers
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.repository.ts
│   │   │   └── dto/
│   │   │       ├── register.dto.ts
│   │   │       └── login.dto.ts
│   │   │
│   │   ├── oauth/                  # OAuth 2.1 + OIDC
│   │   │   ├── oauth.module.ts
│   │   │   ├── oauth.grpc.controller.ts
│   │   │   ├── oauth.service.ts
│   │   │   ├── providers/
│   │   │   │   ├── github.provider.ts
│   │   │   │   └── google.provider.ts
│   │   │   └── pkce.util.ts        # code_verifier / code_challenge
│   │   │
│   │   ├── token/                  # JWT + Refresh + Anti-CSRF
│   │   │   ├── token.module.ts
│   │   │   ├── token.service.ts    # sign, verify, rotate, revoke
│   │   │   ├── csrf.service.ts     # HMAC-SHA256 double-submit cookie
│   │   │   └── token.repository.ts # Redis: jti blocklist
│   │   │
│   │   ├── captcha/
│   │   │   ├── captcha.module.ts
│   │   │   └── captcha.service.ts  # hCaptcha / Cloudflare Turnstile
│   │   │
│   │   ├── health/
│   │   │   ├── health.module.ts
│   │   │   └── health.grpc.controller.ts  # liveness + readiness
│   │   │
│   │   └── audit/
│   │       ├── audit.module.ts
│   │       └── audit.service.ts    # запись security-событий в БД
│   │
│   ├── common/
│   │   ├── guards/
│   │   │   ├── jwt.guard.ts
│   │   │   └── csrf.guard.ts
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts    # Pino + correlation-id
│   │   │   └── error-transform.interceptor.ts
│   │   ├── filters/
│   │   │   └── grpc-exception.filter.ts
│   │   ├── decorators/
│   │   │   └── current-user.decorator.ts
│   │   └── pipes/
│   │       └── zod-validation.pipe.ts
│   │
│   ├── config/
│   │   ├── app.config.ts
│   │   ├── jwt.config.ts
│   │   └── oauth.config.ts
│   │
│   └── infrastructure/
│       ├── prisma/
│       │   ├── prisma.module.ts
│       │   └── prisma.service.ts
│       └── redis/
│           ├── redis.module.ts
│           └── redis.service.ts
│
├── test/
│   ├── unit/
│   └── e2e/
├── Dockerfile
├── docker-compose.yaml
└── .env.example

```

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id             String    @id @default(uuid()) @db.Uuid
  email          String    @unique
  passwordHash   String?   // null для OAuth-only пользователей
  emailVerified  Boolean   @default(false)
  isActive       Boolean   @default(true)
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt

  sessions       Session[]
  oauthProviders OAuthProvider[]
  auditLogs      AuditLog[]

  @@index([email])
  @@map("users")
}

model Session {
  id           String    @id @default(uuid()) @db.Uuid
  userId       String    @db.Uuid
  jti          String    @unique  // JWT ID для blocklist
  refreshToken String    // bcrypt-хэш refresh токена
  userAgent    String?
  ipAddress    String?
  expiresAt    DateTime
  revokedAt    DateTime?
  createdAt    DateTime  @default(now())

  user         User      @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([jti])
  @@map("sessions")
}

model OAuthProvider {
  id           String   @id @default(uuid()) @db.Uuid
  userId       String   @db.Uuid
  provider     Provider // GITHUB | GOOGLE
  providerId   String   // внешний ID пользователя у IdP
  accessToken  String?  // зашифрован AES-256-GCM
  refreshToken String?  // зашифрован AES-256-GCM
  idToken      String?
  scope        String?
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerId])
  @@index([userId])
  @@map("oauth_providers")
}

model AuditLog {
  id         String    @id @default(uuid()) @db.Uuid
  userId     String?   @db.Uuid
  event      AuditEvent
  ipAddress  String?
  userAgent  String?
  meta       Json?
  createdAt  DateTime  @default(now())

  user       User?     @relation(fields: [userId], references: [id], onDelete: SetNull)

  @@index([userId])
  @@index([event])
  @@index([createdAt])
  @@map("audit_logs")
}

enum Provider {
  GITHUB
  GOOGLE
}

enum AuditEvent {
  REGISTER
  LOGIN_SUCCESS
  LOGIN_FAILED
  LOGOUT
  TOKEN_REFRESHED
  TOKEN_REVOKED
  OAUTH_LINKED
  OAUTH_UNLINKED
  PASSWORD_CHANGED
  EMAIL_VERIFIED
  CAPTCHA_FAILED
}
```

```proto
syntax = "proto3";
package auth;

service AuthService {
  // Email регистрация
  rpc Register (RegisterRequest)  returns (AuthResponse);
  rpc VerifyEmail (VerifyEmailRequest) returns (StatusResponse);
  rpc Login    (LoginRequest)     returns (AuthResponse);
  rpc Logout   (LogoutRequest)    returns (StatusResponse);

  // OAuth 2.1 PKCE flow
  rpc GetOAuthUrl  (OAuthUrlRequest)   returns (OAuthUrlResponse);
  rpc OAuthCallback(OAuthCallbackRequest) returns (AuthResponse);

  // Токены
  rpc RefreshToken (RefreshRequest)   returns (AuthResponse);
  rpc ValidateToken(ValidateRequest)  returns (ValidateResponse);
  rpc RevokeToken  (RevokeRequest)    returns (StatusResponse);

  // Anti-CSRF
  rpc GetCsrfToken (CsrfRequest)      returns (CsrfResponse);

  // Health
  rpc Check (HealthCheckRequest)      returns (HealthCheckResponse);
}

message AuthResponse {
  string access_token  = 1;
  string refresh_token = 2;
  string csrf_token    = 3;  // double-submit CSRF
  int64  expires_in    = 4;
  string token_type    = 5;  // "Bearer"
}

message ValidateResponse {
  bool    valid   = 1;
  string  user_id = 2;
  string  email   = 3;
  repeated string roles = 4;
}

message OAuthUrlRequest {
  string provider      = 1;  // "github" | "google"
  string redirect_uri  = 2;
  string state         = 3;  // opaque, хранится в Redis 10 мин
  string code_challenge        = 4;  // PKCE SHA256
  string code_challenge_method = 5;  // "S256"
}
```

## Ключевые решения безопасности

**JWT (RS256 — асимметричный)**. Приватный ключ только внутри auth-сервиса; все остальные сервисы проверяют подпись публичным ключом. Access token живёт 15 минут, refresh token — 30 дней с ротацией при каждом использовании. `jti` каждого выданного токена хранится в Redis-blocklist до истечения его TTL — это даёт мгновенный отзыв без обращения к БД.

**Anti-CSRF (Double Submit Cookie + HMAC)**. При выдаче токенов генерируется `csrf_token = HMAC-SHA256(session_id, secret)`. Клиент хранит его в памяти и передаёт в gRPC-метаданных как `x-csrf-token`; `CsrfGuard` верифицирует подпись. Это защищает gRPC-over-WebSocket и HTTP/2 endpoints.

**OAuth 2.1 + PKCE**. `code_verifier` (43–128 байт случайной энтропии) и `code_challenge = BASE64URL(SHA256(verifier))` генерируются на клиенте. `state` (UUID v4) хранится в Redis 10 минут и уничтожается после первого использования. Получаемые от IdP access/refresh токены шифруются AES-256-GCM перед записью в БД.

**Хранение паролей**: `argon2id` с `memoryCost=65536`, `timeCost=3`, `parallelism=4`. Никогда не хранится сам пароль.

**Rate Limiting**: на уровне gRPC-interceptor, ключ — IP + метод. Хранится в Redis с sliding window. Для `/Login` и `/Register` — жёстче всего.

```yaml
# docker-compose.yaml — фрагмент, относящийся к auth-микросервису

version: "3.9"

networks:
  auth-net:
    driver: bridge
    internal: true      # изолирована от публичной сети
  observability-net:
    driver: bridge

volumes:
  postgres-data:
  redis-data:

services:

  # ─────────────────────────────────────────────
  #  Auth Microservice
  # ─────────────────────────────────────────────
  auth-service:
    build:
      context: ./auth-service
      dockerfile: Dockerfile
      target: production
    image: auth-service:latest
    restart: unless-stopped
    networks:
      - auth-net
      - observability-net
    expose:
      - "50051"        # gRPC — только внутри auth-net, наружу не торчит
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      NODE_ENV: production
      GRPC_PORT: 50051

      # Database
      DATABASE_URL: postgresql://auth_user:${POSTGRES_PASSWORD}@postgres:5432/auth_db?schema=public&sslmode=disable&connection_limit=20

      # Redis
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0

      # JWT (RS256) — ключи монтируются как secrets
      JWT_PRIVATE_KEY_PATH: /run/secrets/jwt_private_key
      JWT_PUBLIC_KEY_PATH:  /run/secrets/jwt_public_key
      JWT_ACCESS_TTL:       900          # 15 min
      JWT_REFRESH_TTL:      2592000      # 30 days

      # Anti-CSRF
      CSRF_SECRET: ${CSRF_SECRET}

      # OAuth — GitHub
      GITHUB_CLIENT_ID:     ${GITHUB_CLIENT_ID}
      GITHUB_CLIENT_SECRET: ${GITHUB_CLIENT_SECRET}

      # OAuth — Google
      GOOGLE_CLIENT_ID:     ${GOOGLE_CLIENT_ID}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}

      # Captcha (Cloudflare Turnstile)
      CAPTCHA_SECRET_KEY: ${CAPTCHA_SECRET_KEY}

      # Encryption key for OAuth tokens at rest (AES-256-GCM)
      OAUTH_TOKEN_ENCRYPT_KEY: ${OAUTH_TOKEN_ENCRYPT_KEY}

      # Logging
      LOG_LEVEL: info

      # gRPC mTLS
      GRPC_TLS_CERT_PATH: /run/secrets/grpc_cert
      GRPC_TLS_KEY_PATH:  /run/secrets/grpc_key
      GRPC_TLS_CA_PATH:   /run/secrets/grpc_ca

    secrets:
      - jwt_private_key
      - jwt_public_key
      - grpc_cert
      - grpc_key
      - grpc_ca

    healthcheck:
      test: ["CMD", "grpc_health_probe", "-addr=:50051", "-tls",
             "-tls-ca-cert=/run/secrets/grpc_ca"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 20s

    deploy:
      resources:
        limits:
          cpus: "1"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    user: "1001:1001"

  # ─────────────────────────────────────────────
  #  PostgreSQL 16
  # ─────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    networks:
      - auth-net
    expose:
      - "5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./auth-service/prisma/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    environment:
      POSTGRES_DB:       auth_db
      POSTGRES_USER:     auth_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U auth_user -d auth_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 512M
    read_only: true
    tmpfs:
      - /tmp
      - /var/run/postgresql
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

  # ─────────────────────────────────────────────
  #  Redis 7 (AOF persistence)
  # ─────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - auth-net
    expose:
      - "6379"
    volumes:
      - redis-data:/data
      - ./auth-service/config/redis.conf:/usr/local/etc/redis/redis.conf:ro
    command: redis-server /usr/local/etc/redis/redis.conf --requirepass ${REDIS_PASSWORD}
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 128M
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

# Docker secrets (файлы монтируются из FS хоста или Swarm)
secrets:
  jwt_private_key:
    file: ./secrets/jwt_private.pem
  jwt_public_key:
    file: ./secrets/jwt_public.pem
  grpc_cert:
    file: ./secrets/grpc.crt
  grpc_key:
    file: ./secrets/grpc.key
  grpc_ca:
    file: ./secrets/ca.crt
```

```Dockerfile
# ─── Stage 1: deps ───────────────────────────────────────────────
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --ignore-scripts

# ─── Stage 2: builder ────────────────────────────────────────────
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate
RUN npm run build

# ─── Stage 3: production ─────────────────────────────────────────
FROM node:22-alpine AS production
WORKDIR /app

# Копируем только нужное — без исходников и devDependencies
COPY --from=builder /app/dist       ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/prisma     ./prisma
COPY package.json ./

# grpc_health_probe для healthcheck
COPY --from=ghcr.io/grpc-ecosystem/grpc-health-probe:v0.4.28 \
     /ko-app/grpc-health-probe /usr/local/bin/grpc_health_probe

RUN addgroup -S appgroup && adduser -S appuser -G appgroup -u 1001
USER 1001

ENTRYPOINT ["node", "dist/main.js"]
```

## Ключевые паттерны реализации

`**main.ts**` **— только gRPC, никакого HTTP:**

typescript

```typescript
async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.GRPC,
      options: {
        url: `0.0.0.0:${process.env.GRPC_PORT}`,
        package: 'auth',
        protoPath: join(__dirname, '..', 'proto', 'auth.proto'),
        credentials: ServerCredentials.createSsl(
          readFileSync(process.env.GRPC_TLS_CA_PATH),
          [{ cert_chain: readFileSync(process.env.GRPC_TLS_CERT_PATH),
             private_key: readFileSync(process.env.GRPC_TLS_KEY_PATH) }],
          true, // require client cert — mTLS
        ),
      },
    },
  );
  app.useGlobalInterceptors(new LoggingInterceptor());
  app.useGlobalFilters(new GrpcExceptionFilter());
  app.useGlobalPipes(new ZodValidationPipe());
  await app.listen();
}
```

**Refresh token rotation (защита от кражи):**

1. При использовании refresh token — старый `jti` добавляется в Redis-blocklist.
2. Выдаётся новая пара `access + refresh` с новым `jti`.
3. Если тот же refresh-token использован повторно (обнаружение replay) — вся сессия немедленно отзывается и пишется `AuditEvent.TOKEN_REVOKED`.

**PKCE OAuth flow через gRPC:** клиент вызывает `GetOAuthUrl` → получает URL с `code_challenge` → редиректит пользователя → IdP возвращает `code` на callback URL → клиент вызывает `OAuthCallback` с `code + code_verifier` → сервис проверяет challenge, обменивает код на токены IdP, создаёт/обновляет пользователя → возвращает JWT.

**Health check** реализован через `@GrpcMethod('AuthService', 'Check')` и дополнительно через `TerminusModule` (проверка соединения с PostgreSQL и Redis ping).

++++++++++++++

### Impress — не нужен

Impress — это полноценный application server с dependency injection, concurrency control, execution timeout, auto routing, pluggable auth module и persistent sessions. [SlideShare](https://www.slideshare.net/slideshow/fwdays-2021-metarhia-technology-stack-for-nodejs/249059695) Это всё то, что ты уже строишь сам на Fastify. Использовать Impress вместе с Fastify — это два конкурирующих сервера в одном проекте. Смысла нет.

### metacom — не нужен

Metacom — это top-level application protocol для RPC поверх WebSocket, требует frame-based transport. [GitHub](https://github.com/metarhia/Contracts/blob/master/doc/Metacom.md) Он заменяет REST/HTTP, а не дополняет его. Если API уже на REST + Fastify — metacom тут лишний.

### Что реально полезно из Metarhia

|Библиотека|Что даёт|Смысл брать|
|---|---|---|
|`metautil`|Semaphore, Pool, Collector, утилиты|Да — для concurrency|
|`metaschema`|Runtime type validation|Да — валидация схем|
|`metalog`|Структурированные логи с буферизацией|Да — лучше pino для highload|
|`metasql`|Query builder + DDL из схемы|Условно — если без ORM|
|`metatests`|Простой test runner|Опционально|

Честно: публичных репозиториев с одновременно Fastify + Metarhia-библиотеками + OAuth GitHub практически нет. Зато есть отличные репозитории, покрывающие разные аспекты по отдельности — и из них можно собрать цельную картину. Вот подборка с объяснением что именно стоит изучить в каждом.

---

## Репозитории по категориям

### Архитектура и clean code — эталонные примеры

**[marcoturi/fastify-boilerplate](https://github.com/marcoturi/fastify-boilerplate)** Fastify 5 + TypeScript + Clean Architecture + CQRS + DDD + vertical slice. Функциональный подход — composition и factory functions вместо классов, framework-agnostic core (бизнес-логика не зависит от Fastify), модули можно вытащить в отдельный сервис [GitHub](https://github.com/marcoturi/fastify-boilerplate). Лучший стартовый шаблон для изучения современной архитектуры на Fastify.

Что изучить: структуру плагинов, как делается DI через декораторы, CQRS-разделение команд и запросов.

---

**[Marlinsk/nodejs-authentication-backend](https://github.com/Marlinsk/nodejs-authentication-backend)** Production-ready auth бэкенд с Clean Architecture, DDD, TDD. JWT + access/refresh tokens, 2FA через TOTP, password recovery, structured logging. 95+ тестов, adapter pattern для БД — можно переключать SQL/NoSQL/in-memory [GitHub](https://github.com/Marlinsk/nodejs-authentication-backend). Один из немногих публичных проектов где auth сделан по всем правилам архитектуры.

Что изучить: структуру domain/application/infrastructure слоёв, как организованы value objects (Email, Password), как абстрагированы провайдеры (token, hash, email).

---

**[luizomf/clean-architecture-api-boilerplate](https://github.com/luizomf/clean-architecture-api-boilerplate)** TypeScript + Express + Knex, чистая архитектура. Данные входят через routes → controllers → use cases → domain, и выходят обратно. Отдельные интерфейсы для каждого слоя — репозитории, middleware, security — всё через ports (interfaces) [GitHub](https://github.com/luizomf/clean-architecture-api-boilerplate). Показательно именно тем, что Framework (Express) легко заменяется — та же идея применима для Fastify.

---

### TypeScript DDD — для изучения паттернов

**[CodelyTV/typescript-ddd-skeleton](https://github.com/CodelyTV/typescript-ddd-skeleton)** Эталонный TypeScript DDD скелет от испанской школы Codely TV. Hexagonal Architecture, value objects, domain events, CQRS. Нет конкретно auth — зато лучший пример как строить доменный слой.

**[CodelyTV/typescript-ddd-example](https://github.com/CodelyTV/typescript-ddd-example)** Полный пример с MongoDB + PostgreSQL, RabbitMQ, тестами. Смотреть как организованы bounded contexts и как избежать утечки инфраструктурных деталей в домен.

---

### Fastify-специфичное

**[revell29/fastify-clean-architecture](https://github.com/revell29/fastify-clean-architecture)** Fastify + TypeScript + DDD + Clean Architecture. Структура: config → database (pg-promise) → repositories → webserver (server, routes, plugins) → interfaces (controllers, routes, serializers) [GitHub](https://github.com/revell29/fastify-clean-architecture). Сериализаторы как отдельный слой — хорошая практика для преобразования HTTP-объектов во внутренние.

**[nooreldeenmansour/fastify-jwt-auth-example](https://github.com/nooreldeenmansour/fastify-jwt-auth-example)** Использует только экосистему Fastify: fastify-autoreload, fastify-bcrypt, fastify-jwt, fastify-cors, fastify-env. Демонстрирует hooks, decorators, request/response lifecycle [GitHub](https://github.com/nooreldeenmansour/fastify-jwt-auth-example). Хорошо для понимания идиоматического Fastify — как правильно использовать плагинную систему.

---

### Тестирование Node.js сервисов

**[goldbergyoni/javascript-testing-best-practices](https://github.com/goldbergyoni/javascript-testing-best-practices)** Не auth-специфично, но это лучший гайд по тестированию Node.js сервисов с реальными примерами — component tests, integration tests, best practices для работы с БД в тестах. Содержит примеры на Fastify.

---

## Что конкретно смотреть в каждом

```
marcoturi/fastify-boilerplate
  └─ src/modules/*          ← vertical slice структура
  └─ src/shared/            ← CQRS buses, базовые интерфейсы

Marlinsk/nodejs-authentication-backend
  └─ src/domain/user/       ← entities, value objects, repo interfaces
  └─ src/application/       ← use cases без HTTP-зависимостей
  └─ src/infrastructure/    ← конкретные реализации

CodelyTV/typescript-ddd-skeleton
  └─ src/contexts/          ← bounded contexts
  └─ tests/                 ← structure тестов по слоям
```

---

## Честный вывод о состоянии экосистемы

Готовых репозиториев с именно Fastify + Metarhia + OAuth GitHub в одном флаконе нет — такой стек специфичен и практически не встречается в открытом коде. Это означает, что придётся собирать сервис самостоятельно, беря лучшие идеи из разных источников:

- архитектуру слоёв — из `Marlinsk/nodejs-authentication-backend` и `CodelyTV/typescript-ddd-skeleton`
- Fastify-специфику — из `marcoturi/fastify-boilerplate` и `nooreldeenmansour/fastify-jwt-auth-example`
- concurrency через `metautil.Semaphore`, логирование через `metalog`, валидацию через `metaschema` — добавить поверх

Это нормальная ситуация для нишевых стеков: открытый код даёт архитектурные паттерны, а конкретные библиотеки интегрируются самостоятельно.