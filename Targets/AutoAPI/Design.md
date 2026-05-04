
# account-vault-db



```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Account {
  id              String       @id @default(uuid())
  createdAt       DateTime     @default(now())
  updatedAt       DateTime     @updatedAt
  email           String?      @unique
  name     String?
  status          AccountStatus @default(ACTIVE)
  isEmailVerified Boolean      @default(false)
  isPhoneVerified Boolean      @default(false)
  password        Password?    @relation(fields: [passwordId], references: [id])
  passwordId      String?
  
  
  oauthBindings   OAuthBinding[]
  apiKeys         ApiKey[]
  sessions        Session[]
  totpFactors     TotpFactor[]


  deletedAt       DateTime?
  
  @@account
}

model IdentityProvider {
  id               String    @id @default(cuid())
  name             String    @unique   // slug
  displayName      String
  authorizationUrl String
  tokenUrl         String
  clientId         String
  clientSecret     String    // encrypted at rest
  defaultScopes    String
  isActive         Boolean   @default(true)
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  accounts         Account[]
}

model Password {
  id            String   @id @default(uuid())
  hash          String
  algorithm     String
  salt          String?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  account       Account?
}

model OAuthBinding {
  id             String   @id @default(uuid())
  provider       String
  providerUserId String
  accessToken    String?  // encrypted
  refreshToken   String?  // encrypted
  scope          String?
  createdAt      DateTime @default(now())
  account        Account  @relation(fields: [accountId], references: [id])
  accountId      String
  @@unique([provider, providerUserId])
}

model ApiKey {
  id         String   @id @default(uuid())
  name       String?
  keyHash    String   // store hash of the key for verification
  scopes     String[] // scopes granted
  lastUsedAt DateTime?
  createdAt  DateTime @default(now())
  revoked    Boolean  @default(false)
  account    Account  @relation(fields: [accountId], references: [id])
  accountId  String
}


model TotpFactor {
  id           String   @id @default(uuid())
  secret       String   // encrypted
  label        String?
  createdAt    DateTime @default(now())
  confirmed    Boolean  @default(false)
  account      Account  @relation(fields: [accountId], references: [id])
  accountId    String
}

model PasswordResetToken {
  id         String   @id @default(uuid())
  tokenHash  String
  expiresAt  DateTime
  consumed   Boolean  @default(false)
  createdAt  DateTime @default(now())
  account    Account  @relation(fields: [accountId], references: [id])
  accountId  String
}
```

































# account-vault

```protobuf
service AccountVault {
  // CRUD аккаунта
  rpc CreateAccount(CreateAccountRequest) returns (Account);
  rpc GetAccountById(GetAccountByIdRequest) returns (Account);
  rpc GetAccountByEmail(GetAccountByEmailRequest) returns (Account);
  rpc GetAccountByUsername(GetAccountByUsernameRequest) returns (Account);
  rpc UpdateAccount(UpdateAccountRequest) returns (Account);
  rpc SoftDeleteAccount(SoftDeleteAccountRequest) returns (Empty);
  rpc HardDeleteAccount(HardDeleteAccountRequest) returns (Empty);
  // Пароль (только хэширование/проверка)
  rpc SetPassword(SetPasswordRequest) returns (Empty);
  rpc VerifyPassword(VerifyPasswordRequest) returns (PasswordValid);
  rpc HasPassword(HasPasswordRequest) returns (BoolResponse);
  // API Keys для LLM
  rpc CreateApiKey(CreateApiKeyRequest) returns (ApiKey);
  rpc ListApiKeys(ListApiKeysRequest) returns (ApiKeysList);
  rpc RevokeApiKey(RevokeApiKeyRequest) returns (Empty);
  rpc GetApiKeyByHash(GetApiKeyByHashRequest) returns (ApiKey); // для auth.middleware
  // Social связки
  rpc LinkSocialAccount(LinkSocialRequest) returns (Empty);
  rpc GetAccountBySocial(GetAccountBySocialRequest) returns (Account);
  rpc UnlinkSocialAccount(UnlinkSocialRequest) returns (Empty);
  
  // User preferences (дефолтные модель, температура, etc)
  rpc GetPreferences(GetPreferencesRequest) returns (Preferences);
  rpc UpdatePreferences(UpdatePreferencesRequest) returns (Preferences);
  
  // Админ/внутренние
  rpc ListAccounts(ListAccountsRequest) returns (AccountsList);
  rpc BanAccount(BanAccountRequest) returns (Empty);
  rpc UnbanAccount(UnbanAccountRequest) returns (Empty);
}
```

## Область ответственности — account-vault


- Хранение и управление учетными записями пользователей (профили, email/phone, api ключи, хеши паролей, ключи API, recovery-токены, OAuth-учётные данные.
- Управление методами аутентификации (пароль, внешние провайдеры, 2FA привязки).
- Управление сессиями и их метаданными (device info, last active, revoke).
- Управление привилегиями/ролями и привязками пользователям.
- Поддержка жизненного цикла аккаунта: регистрация, подтверждение, деактивация, удаление, восстановление.
- Взаимодействие с другими сервисами: выдача/валидация данных для token, проверка риска (risk), отправка уведомлений (notification), логирование в audit.
- Шифрование и безопасное хранение чувствительных полей; ротация ключей хранения.
- Версионирование и миграции схемы аккаунтов.
- Упрощённый API для оркестрации (atomic операции: create-account-with-secrets, update-credentials, revoke-all-sessions).



# grpc

- CreateAccount(credentials)
- GetAccount(identifier or email) 
- RestoreAccount(identifier or email)
- BlockAccount(identifier or email)
- DeleteAccount(identifier or email)
- UpdatePassword(identifier or email, password)
- UpdateEmail(identifier or email, email)
- UpdateUserName(identifier or email, username)

- SaveApiKey
- GetApiKeys
- Delete........


- ? RotateSecrets
- GetApiKey
- - RevokeApiKey
- - GenerateApiKey

- LinkOAuthProvider
- UnlinkOAuthProvider


- VerifyAccountEmail



- UpdatePassword
- ValidateCredentials

- Attach2FA
- Verify2FA
- Detach2FA

- HealthCheck (grpc.health.v1.Health/Check)





# =account-vault — account aggregate, профиль, статус, soft-delete/disable
main.ts
app.module.ts
app.controller.ts
# \api
account_vault.proto
# \prisma
schema.prisma
seed.ts # cиды (тестовые данные)
# \interceptors\
logging.interceptor.ts — логирует входящие запросы
timeout.interceptor.ts — автоматически прерывает запрос
# \pipes\
validation.pipe.ts — проверяет, что данные из тела запроса или query-параметров соответствуют DTO-схемам
# \dto\
pagination.dto.ts — задаёт типовые поля для пагинации: page, limit, offset, sortBy, order. Все эндпоинты, которые возвращают списки, используют его 
# \utils\
hash.utils.ts — функции для хеширования паролей (bcrypt), генерации
# \accounts
# \secrets
# \credentials
# - SessionsModule — сессии, устройства, refresh-token linkage, revoke.

# \migration
- Гарантирует синхронизацию Prisma schema и реальной БД во всех средах (dev/stage/prod).
- Позволяет безопасно и отслеживаемо вносить изменения в структуру (версионирование, откат, ревью).
- Облегчает командную разработку: у каждого разработчика единый процесс обновления схемы.
- Упрощает CI/CD: миграции применяются в деплое, уменьшая риск несоответствий.
- Seed автоматизирует создание системных данных (админ, роли) и тестовых наборов — важно для быстрых развёртываний и тестирования.

- Реализовать MigrationModule как минимум с поддержкой: применение миграций в CI/CD, отдельный seed для системных записей, безопасный режим включения в прод (ручной либо с проверками). Это даёт баланс безопасности и автоматизации.


\crypto — hashing, secure token hashing, compare functions.
# \health — db readiness/liveness + grpc.health.v1.




  

credential-module — password hash, password verify, password change, password policy.

  

identity-module — email identity, normalized email, uniqueness.

— email verification token/code lifecycle. forgot/reset password tokens and flows.

  

federation-module — link/unlink external IdP identities (google, github subject/provider).

  

account — GetMe, lookup by email/id/provider-subject.



repository-module — PostgreSQL repos, transactions, optimistic locking.

  



  


  














auth должен быть входной точкой для всех пользовательских auth-flow: register, login, oauth callback, forgot/reset password, logout, me. Он не должен сам хранить пароли, refresh token state и email-коды как source of truth; его задача — оркестровать сценарий, вызывать account-vault, token, notification, risk, captcha, audit, и отдавать клиенту согласованный API/gRPC контракт

  

\auth

  

auth регистрация вход

oauth внешний провайдер

verification проверки

notification отправка сообщений

email

token обработка jwt

capcha

risk rate-limit triggers, brute-force policy

health

audit

  

/account-vault

  

/token

  

/notification

  

verification

  

---

  

# Для auth я бы сделал такие Nest-модули:

  

auth-api-module — REST controllers v1/auth/\*, DTO, validation, mapping ошибок.
auth-grpc-module — gRPC controllers и protobuf adapter layer.

auth-application-module — use cases: register, login, refresh, logout, get-me, forgot-password.

  

oauth-module — Google/GitHub OIDC flow, state/nonce orchestration, callback handling.

  

session-orchestration-module — координация login/logout/refresh сценариев между account-vault и token.

  

captcha-module — verify captcha provider response, feature flag.

  

risk-module — device/ip/email heuristics, velocity checks, suspicious login scoring.

  

audit-module — публикация audit events.

  

health-module — liveness/readiness + grpc.health.v1.

  

config-module — env schema, secrets references, provider config.

  

client-module — gRPC/HTTP clients к account-vault, token, notification.
  

# token — отдельный security-state сервис для всего, что связано с выпуском и жизненным циклом токенов. Сюда хорошо ложатся:

  

выпуск access token JWT;

  

выпуск refresh token;

  

refresh token rotation;

  

revoke / logout / logout-all;

  

blacklist / denylist;

  

token family / session lineage;

  

JWT verify / JWKS / key rotation;

  

при необходимости introspection для внутренних клиентов.

  

Модули внутри token

token-issue-module — issue access/refresh pair.

  

refresh-module — refresh rotation, one-time refresh semantics, reuse detection.

  

revoke-module — logout, logout-all, revoke by session/user/token family.

  

blacklist-module — Redis denylist для jti/session identifiers.

  

jwt-module — signing/verifying access JWT.

  

jwks-module — public keys endpoint / key discovery для internal consumers.

  

key-management-module — active key set, rotation schedule, key versioning (kid).

  

session-module — logical session/device record, token family tree.

  

introspection-module — если нужен opaque-like internal verification endpoint.

  

health-module — Redis/key-store health + grpc.health.v1.

  

# Что вынести в notification

  

notification должен быть тупым инфраструктурным сервисом доставки сообщений, а не местом бизнес-логики аутентификации. Он принимает команду вроде “send verification email” или “send password reset email”, рендерит шаблон, отправляет и возвращает delivery result/event.

  

Туда логично вынести:

  

email templates;

  

localization, если нужна;

  

SMTP/provider integration;

  

retry/dead letter;

  

rate-limits на доставку;

  

webhook handling от email provider;

  

tracking статуса отправки.

  

Модули внутри notification

notification-api-module — gRPC/queue consumers для команд отправки.

  

email-module — SMTP/provider adapters.

  

template-module — шаблоны verification/reset/login-link.

  

delivery-module — retry, idempotency, scheduling.

  

preferences-module — если позже появятся user notification preferences.

  

provider-webhook-module — bounce, complaint, delivered, opened.

  

health-module — provider connectivity + grpc.health.v1.

  

# Где оставить captcha, risk, audit, health

  

Эти части лучше мыслить не как самостоятельные большие микросервисы на старте, а как внутренние модули auth, пока нет явной причины выносить их отдельно. Аутентификация — их основной потребитель, и преждевременный вынос увеличит hop count и связанность без большого выигрыша.

  

captcha

captcha как модуль в auth:

  

verify provider token;

  

feature flag on/off;

  

включать только для register, forgot-password, suspicious login.

  

Отдельный сервис нужен только если captcha используется многими публичными сервисами платформы.

  

risk

risk как модуль в auth:

  

IP reputation hooks;

  

velocity per email/ip/device;

  

geo/device anomaly;

  

решение: allow, challenge, deny, step-up.

  

Выносить в сервис стоит, когда risk станет общей anti-abuse платформой для auth, payments, signup, support portals





















---------












## auth-orchestrator

`app` — корневой композиционный модуль, собирает все feature-модули, конфиг, interceptors, filters и DI wiring.[](https://docs.nestjs.com/security/authentication)

`grpc` — транспортный слой gRPC: protobuf handlers, mapping request/response DTO, adapters между transport и application use cases.[](https://docs.nestjs.com/security/authentication)

`commands` — write-side use cases, то есть команды, которые меняют состояние: register, login, refresh, logout, verify-email, reset-password и так далее.  
`queries` — read-side use cases, которые ничего не меняют и возвращают данные, например `GetMe` или получение auth/session state.

`register` — сценарий регистрации: создание аккаунта, первичная проверка входных данных, запуск email verification flow.  
`login` — сценарий входа: проверка credential flow, вызов risk/captcha при необходимости, запуск выдачи токенов.

`refresh` — сценарий обновления access token через refresh token, orchestration между проверкой refresh token и выпуском новой пары.  
`logout` — завершение текущей сессии или текущего refresh-chain.

`logout-all` — инвалидирует все активные сессии пользователя, обычно через token family revocation или user-level revocation marker.[](https://oneuptime.com/blog/post/2026-02-02-jwt-revocation/view)  
`me` — возвращает текущий профиль/identity snapshot для уже аутентифицированного пользователя.

`verify-email` — подтверждение email по verification token или code.  
`resend-verification` — повторная отправка письма подтверждения.

`forgot-password` — старт password recovery flow, генерация reset intent и запуск письма.  
`reset-password` — завершение recovery flow, установка нового пароля по reset token.

`change-password` — смена пароля из уже аутентифицированной сессии, с проверкой текущего пароля и последующей инвалидцией нужных сессий.  
`oauth-google` — обработка логина/регистрации через Google, обмен code на identity claims, account linking.

`oauth-github` — то же самое для GitHub.  
`anti-abuse` — orchestration anti-bot/anti-fraud мер: когда вызывать captcha, когда спрашивать risk score, когда тормозить flow.

`events` — публикация доменных и интеграционных событий наружу: login succeeded, password changed, verification requested и подобное.  
`health` — health/readiness/liveness слой для сервиса и реализация `grpc.health.v1`.[](https://docs.nestjs.com/security/authentication)



## token-service

`jwt` — выпуск и базовая проверка JWT access token: claims assembly, signing, standard claims validation.[](https://habr.com/ru/companies/ozontech/articles/976950/)  
`jwks` — публикация набора публичных ключей для верификации подписи JWT другими сервисами и gateway.[](https://habr.com/ru/companies/ozontech/articles/976950/)

`signing-key-rotation` — lifecycle signing keys: генерация, активация, rollover, retire old keys, совместимость во время ротации.  
`refresh-tokens` — выпуск, хранение метаданных и первичная обработка refresh token.

`token-families` — управление цепочками refresh rotation, где новый refresh наследует family/lineage старого.  
`revocation` — отзыв отдельных токенов, family-level отзыв, user-level logout-all markers.[](https://oneuptime.com/blog/post/2026-02-02-jwt-revocation/view)

`blacklist` — логика blacklist для уже выданных access token или jti-based revocation checks.[](https://oneuptime.com/blog/post/2026-02-02-jwt-revocation/view)  
`session-index` — индекс активных сессий пользователя: device/session listing, lookup для logout/logout-all.

`reuse-detection` — обнаружение повторного использования refresh token после ротации, признак возможной кражи токена. Это один из ключевых модулей secure refresh rotation.  
`persistence` — постоянное хранилище token state, families, revocation records.

`redis-cache` — быстрый слой для blacklist, short-lived revocation markers, hot lookup и ускорение проверок.[](https://oneuptime.com/blog/post/2026-02-02-jwt-revocation/view)  
`health` — readiness/liveness и health checks для Redis/DB/crypto dependencies.







token — отдельный security-state сервис для всего, что связано с выпуском и жизненным циклом токенов

=token
\verification
\rotation
\introspection
\opaque
- gen refresh
- generate-api-key
\jwt
- genereate access 
\jwks -  public keys endpoint / key discovery
\rotation 
\blacklist

\health





















## notification-service

`email` — прикладной модуль отправки email-команд: send verification, send reset, send login alert и подобное.  
`templates` — шаблоны писем, подстановка переменных, локализация, versioning шаблонов.

`providers` — адаптеры к SMTP, SES, Mailgun, SendGrid и другим провайдерам.  
`outbox` — прием и обработка сообщений из outbox/event queue для надежной доставки.

`retry` — политики ретраев, backoff, dead-letter routing, защита от дублированной отправки.  
`delivery-events` — фиксация статусов доставки: queued, sent, delivered, bounced, complained.

`health` — состояние провайдеров, очередей и readiness сервиса.

## risk-service

`rules` — набор правил принятия risk-решений: deny/challenge/allow/step-up.  
`signals` — сбор и нормализация входных сигналов: IP, device, geo, failed attempts, token reuse events.

`ip-analysis` — анализ IP-адреса: reputation, ASN, proxy/VPN/Tor heuristics, suspicious ranges.  
`device-analysis` — оценка device fingerprint или device trust signals.

`velocity-detection` — детекция аномальной скорости действий: brute force bursts, impossible travel, rapid refresh misuse.  
`decision-api` — внешний модуль, который возвращает итоговое risk decision другим сервисам.

`health` — readiness зависимостей и доступность risk engine.

## audit-service

`ingest` — прием audit/security событий от других сервисов.  
`security-events` — доменная модель событий безопасности: login_failed, refresh_reused, password_changed, email_verified и так далее.

`storage` — append-only или специализированное хранилище audit trail.  
`search` — выборка и поиск по audit-логам для расследований и админских сценариев.

`retention` — сроки хранения, архивирование, cleanup policies, иногда legal hold.  
`health` — доступность ingest/storage/search зависимостей.











_______
Зона ответственности gateway - валидация входа, CORS, cookies, заголовки, публичный формат ошибок и агрегация нескольких внутренних вызовов в один внешний ответ

Зона ответственности не включает балансировку нагрузки, балансировку рационально реализовывать в k8s с балансировщиком envoy, e.t.c. Выбор свободных подов для gRPC-запросов за счет 


=# redis
\balcklist







=gateway
\guards
- check access
\interceptors
- trace
\main.service.ts
- error-mapping
- graceful shutdown
- graceful degradation

\configuration - читает внешний конфиг с привязкой эндпоинтов и функций микросервисов для обработки данных
\authorization
\mapping - преобразование request по proto
\grpc 
\http - внешний http сервер
- tls
\cache

\audit
- rate-limit
- IP reputation hooks;
- timeout
- deadlines
- retry
- csrf - скорее всего не нужно, встрою в стандартный flow передачи данных или буду хранить токены 
\health
\docs
\monitoring


остались вопросы с производжительностью и потоками


при истечении возвращает ошибку
клиент отправляет то же но с рефрешем
токен сервис делает ротацию
мб ему все равно понадобится jwks








Решение 1. Не делать JWKS и валидацию подписи JWT в каждом сервисе, а реализовать авторизацию в gateway-сервисе 

Решение 2. Не делать в балансировщиком, чтобы реализовать создание нескольких экземпляров гетвея. «Sidecar-прокси» Envoy


`serialization` — нормализация внешних ответов, public DTO shaping, redaction чувствительных полей.  
`docs` — OpenAPI/Swagger или иной внешний контракт REST API.


`app` — корневой модуль gateway, который собирает HTTP API, gRPC clients, filters, guards, interceptors и общую конфигурацию

`rest` — внешний REST слой: контроллеры, DTO, request validation, versioning (`/v1/auth/*`), mapping HTTP semantics в internal application calls. gRPC-to-REST трансляция может делаться и на уровне Envoy transcoding, но если тебе нужен явный контроль над DTO, ошибками и BFF-логикой, этот слой разумно держать в самом gateway.

`grpc-clients` — клиенты для вызова внутренних gRPC сервисов: `auth-orchestrator`, `account-vault`, `token-service`, `risk-service` и других. Его задача — инкапсулировать transport adapters, , retries, metadata propagation и service discovery.

`auth-http` — модуль внешних auth endpoints: `register`, `login`, `refresh`, `logout`, `logout-all`, `me`, `verify-email`, `forgot-password` и остальное. Он не реализует бизнес-логику сам, а только правильно прокладывает HTTP flow до нужных внутренних use cases.

`request-context` — формирует и прокидывает correlation id, request id, user agent, source IP, forwarded headers и прочий request-scoped context дальше во внутренние gRPC вызовы. Это особенно важно, потому что Envoy стоит снаружи и часть сетевого контекста должна корректно пережить hop до backend-сервисов.




`guards` — HTTP guards для проверки access token, session state, scopes или простых policy checks перед доступом к endpoint.  
`interceptors` — cross-cutting поведение: logging, timeout, metrics, response shaping, tracing.
























=account-vault
\dto
\main.service.ts
- grpcohohohohohho





\account\profile - account aggregate, профиль, статус, soft-delete/disable.

\credentials
- password hash, verify, change, password policy

  

identity-module — email identity, normalized email, uniqueness.

  

verification-module — email verification token/code lifecycle.

  

\recovery — forgot/reset password tokens and flows.

  

federation-module — link/unlink external IdP identities (google, github subject/provider).

  

account-query-module — GetMe, lookup by email/id/provider-subject.

  

/repository — PostgreSQL repos, transactions, optimistic locking.
/crypto-module — hashing, secure token hashing, compare functions.

  

\health




`users` — базовая модель пользователя и операции над ней: create/find/update/disable/delete-soft.  
`emails` — работа с email identities: primary email, normalized email, uniqueness, смена email, secondary emails если появятся.

`credentials` — учет credential records пользователя: password credential, external identity links, credential metadata.  
`password-policy` — правила пароля: длина, сложность, reuse policy, запрет слабых или утекших паролей.

`password-hashing` — отдельный слой для hash/verify/re-hash логики, например Argon2id/bcrypt abstraction и миграция алгоритмов.  
`verification` — состояние подтверждения email и связанные инварианты: verified/unverified/pending, timestamps, versioning.

`account-locks` — блокировки и ограничения на уровне аккаунта: locked, disabled, cooldown, temporary suspension.  
`persistence` — репозитории, ORM/SQL mapping, транзакции, доступ к PostgreSQL.

`migrations` — схема БД и миграции для identity-данных.  
`health` — состояние БД, readiness и стандартный health endpoint.[](https://docs.nestjs.com/security/authentication)





















задача — оркестровать сценарий, вызывать account-vault, token, notification, risk, captcha, audit, и отдавать клиенту согласованный API/gRPC контракт

=auth
\oauth
\orcestration
- registration login/logout/refresh
- login
- signup
- verification
- 2fa
- forgot-password
- suspicious login
\risk
- rate-limit
- brute-force policy
\capcha
\health 
- liveness/readiness
- grpc.health.v1.
\audit




# Что вынести в notification

  

notification должен быть тупым инфраструктурным сервисом доставки сообщений, а не местом бизнес-логики аутентификации. Он принимает команду вроде “send verification email” или “send password reset email”, рендерит шаблон, отправляет и возвращает delivery result/event.

  

Туда логично вынести:


email templates;

=notification
\template — шаблоны verification/reset/login-link
\notification-api-module — gRPC/queue consumers для команд отправки.
\ рассылка websocket с очередью 

\email
- SMTP
- integration
\tracking

retry/dead letter;

rate-limits на доставку;


webhook handling от email provider;

  

\tracking

delivery-module — retry, idempotency, scheduling.

preferences-module — если позже появятся user notification preferences.


\provider-webhook — bounce, complaint, delivered, opened
\health provider connectivity + grpc.health.v1.

  






  
