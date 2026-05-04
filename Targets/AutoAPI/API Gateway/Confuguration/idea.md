# 🔍 Research: Глобальный конфиг для API Gateway — рационально ли это?



gateway может ходить в один стабильный upstream Envoy/Service, а Envoy уже будет делать discovery, health-check и L7 balancing для gRPC


конфигурация роутов, чтобы при добавлении новых и отключении старых, не нужно было трогать гетвей

здесь прописываем оркестрацию на grpc-функции микросервисов, иными словами для каждого запрашиваемого эндпоинта существует полноценная функция у специализированного микросервиса, возможность сделать пайплайн в гетвее с введением конфига пропадает

формат ответа определяется gRPC-функцией


```
[policy.role-based]
name="Role-Based"
target = "dns:///auth-orchestrator.auth-system.svc.cluster.local:50051"
rpc = "/auth.v1.AuthOrchestrator/validate-jwt"
source = "header.authorization", pass_through = true


[v1.user-login]
status = enabled
access = "free"
path = "api/v1/auth/register"
method = "POST"
target = "dns:///auth-orchestrator.auth-system.svc.cluster.local:50051"
rpc = "/auth.v1.AuthOrchestrator/Register"
timeout_ms = 5000
retry_attempts = 0

lb_policy = "round_robin"
rate_limit_profile = "login"
```





## Что улучшить

Я бы добавил еще такие поля, потому что без них gateway быстро упрется в production-проблемы:

- `strip_prefix` или `rewrite_path` — почти всегда нужен path rewrite между внешним REST URL и внутренним gRPC mapping.
    
- `grpc_metadata` / `forward_headers` — какие заголовки пробрасывать в metadata: `authorization`, `x-request-id`, `x-forwarded-for`, `traceparent`, tenant-id.
    
- `authn` и `authz_policy` отдельно от `access` — “free” слишком абстрактно; лучше `auth = none|jwt|mtls|internal` и `required_scopes = [...]`.
    
- `idempotency` — для понимания, можно ли делать retry вообще; login/register обычно не стоит ретраить вслепую.
    
- `circuit_breaker`, `outlier_detection`, `max_inflight`, `hedging` — это уже nearer к Envoy/HAProxy production policy.[](https://oneuptime.com/blog/post/2026-01-27-envoy-grpc-load-balancing/view)
    
- `health_check` / `grpc_health_service` — если gateway сам балансирует upstream’ы.
    
- `tls_mode` — `plaintext`, `tls`, `mtls`; в k8s это часто меняется между namespace’ами и окружениями.
    
- `request_validation` — schema/body limits, required fields, max payload.
    
- `observability` — `metrics_name`, `log_body`, `audit`, `trace_sampling`.
    
- `error_mapping` — как `grpc-status` и `grpc-message` переводятся в HTTP status/code для клиента.


## О чем ты забыл архитектурно

Главный пробел — не разделены **listener/route/upstream/policy**. У зрелых прокси это почти всегда отдельные сущности: входящий маршрут, кластер назначения, transport/security, resilience policy

| Слой        | Аналог                                                                                                                                        |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `route`     | Nginx `location`, HAProxy ACL/use_backend, Envoy route [oneuptime](https://oneuptime.com/blog/post/2026-01-27-envoy-grpc-load-balancing/view) |
| `upstream`  | Nginx upstream, HAProxy backend servers, Envoy cluster [oneuptime](https://oneuptime.com/blog/post/2026-01-27-envoy-grpc-load-balancing/view) |
| `transport` | TLS/mTLS, h2, keepalive, authority/SNI [oneuptime](https://oneuptime.com/blog/post/2026-01-27-envoy-grpc-load-balancing/view)                 |
| `policy`    | timeout, retry, rate limit, circuit breaker, auth [oneuptime](https://oneuptime.com/blog/post/2026-01-27-envoy-grpc-load-balancing/view)      |
| `mapping`   | HTTP JSON/body → protobuf request, headers → gRPC metadata                                                                                    |


















---

# 📄 Ваш конфиг в формате TOML

> ⚠️ **Важно**: TOML менее выразителен для глубоко вложенных структур. Я сохранил семантику вашего YAML, но структура стала более «плоской» и явной. Для чтения рекомендую использовать редактор с TOML-подсветкой.

```toml
# gateway.toml — Декларативная конфигурация API Gateway (NestJS)
# Версия спецификации: gateway.mycorp.io/v1alpha1

[metadata]
name = "public-api-gateway"
revision = "2026-04-19.002"

# =============================================================================
# ГЛОБАЛЬНЫЕ НАСТРОЙКИ СЕРВЕРА
# =============================================================================
[server]
publicBaseUrl = "https://api.example.com"
listen.port = 3000
listen.globalPrefix = "/api"
trustProxy = true

# =============================================================================
# SWAGGER / OPENAPI ДОКУМЕНТАЦИЯ
# =============================================================================
[docs]
enabled = true
route.uiPath = "/docs"
route.jsonPath = "/docs-json"
source = "gateway-runtime"

[docs.openapi]
title = "Public API Gateway"
description = "Unified gateway API for auth and platform endpoints"
version = "1.0.0"

[[docs.openapi.tags]]
name = "auth"
description = "Authentication endpoints"

[docs.openapi.securitySchemes.bearer]
type = "http"
scheme = "bearer"
bearerFormat = "JWT"

[docs.exposure]
includeRoutes = ["auth-login", "auth-me"]
hideInternalRoutes = true

[docs.ui]
persistAuthorization = true
displayRequestDuration = true
docExpansion = "list"
tryItOutEnabled = true

[docs.cache]
enabled = true
ttl = "60s"

# =============================================================================
# OBSERVABILITY
# =============================================================================
[observability]
requestIdHeader = "x-request-id"
correlationIdHeader = "x-correlation-id"

[observability.tracing]
enabled = true
propagate = ["x-request-id", "x-correlation-id", "traceparent", "tracestate"]

# =============================================================================
# БЕЗОПАСНОСТЬ: CORS И ХЕДЕРЫ
# =============================================================================
[security.cors]
enabled = true
allowOrigins = ["https://app.example.com"]
allowMethods = ["GET", "POST", "OPTIONS"]
allowHeaders = ["authorization", "content-type", "x-request-id", "x-correlation-id", "x-tenant-id"]
exposeHeaders = ["x-request-id"]
allowCredentials = true
maxAgeSeconds = 600

[security.headers]
forward = ["x-request-id", "x-correlation-id", "x-forwarded-for", "x-forwarded-proto", "x-real-ip", "user-agent", "x-tenant-id"]
drop = ["x-internal-debug", "x-service-token"]

# =============================================================================
# ДЕФОЛТЫ ДЛЯ UPSTREAM (gRPC)
# =============================================================================
[upstreamDefaults]
protocol = "grpc"
discovery = "kubernetes-dns"
connectTimeout = "2s"
requestTimeout = "5s"
idleTimeout = "30s"

[upstreamDefaults.retries]
enabled = true
retryableStatuses = ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
attempts = 1
perTryTimeout = "1500ms"

[upstreamDefaults.circuitBreaker]
enabled = true
maxConnections = 1000
maxPendingRequests = 500
maxRequests = 5000

[upstreamDefaults.outlierDetection]
enabled = true
consecutive5xx = 5
interval = "10s"
baseEjectionTime = "30s"
maxEjectionPercent = 50

# =============================================================================
# ОПИСАНИЕ ВНУТРЕННИХ СЕРВИСОВ (gRPC)
# =============================================================================
[[services]]
id = "auth-grpc"
kind = "grpc"

[services.kubernetes]
namespace = "auth"
serviceName = "auth-svc"
dns = "auth-svc.auth.svc.cluster.local"
port = 9090

[services.grpc]
authority = "auth-svc.auth.svc.cluster.local"
package = "auth.v1"

[services.healthCheck]
enabled = true
type = "grpc"
service = "grpc.health.v1.Health"

# =============================================================================
# ПРОВАЙДЕРЫ АУТЕНТИФИКАЦИИ
# =============================================================================
[[authProviders]]
id = "jwt-access"
type = "jwt"
fromHeader = "Authorization"
scheme = "Bearer"
jwksUri = "https://identity.example.com/.well-known/jwks.json"
cacheTtl = "10m"
clockSkew = "30s"
requiredClaims = ["sub", "exp"]

# =============================================================================
# МАРШРУТЫ (РОУТЫ)
# =============================================================================

# -----------------------------------------------------------------------------
# Роут: auth-login — POST /api/auth/login → gRPC AuthService.Login
# -----------------------------------------------------------------------------
[[routes]]
id = "auth-login"
enabled = true
type = "proxy"
match.path = "/api/auth/login"
match.method = "POST"

[routes.docs]
enabled = true
tag = "auth"
summary = "Login"
description = "Authenticates user and returns tokens"
requestBodySchema = "AuthLoginRequest"
# responseSchema задаётся отдельно в секции [schemas]

[routes.docs.responseSchema]
"200" = "AuthLoginResponse"
"400" = "ErrorResponse"
"401" = "ErrorResponse"
"429" = "ErrorResponse"

[routes.auth]
mode = "public"

[routes.validation]
contentTypes = ["application/json"]
maxBodyBytes = 32768

# Rate limit: массив правил
[[routes.rateLimit]]
key = "ip"
limit = 20
window = "1m"

[[routes.rateLimit]]
key = "ip+path"
limit = 100
window = "10m"

[routes.upstream]
serviceRef = "auth-grpc"
method = "auth.v1.AuthService/Login"
timeout = "3s"

[routes.upstream.retries]
enabled = false

[routes.upstream.requestMapping]
transport = "grpc"
body = "*"

[routes.upstream.requestMapping.headersToMetadata]
"x-request-id" = "x-request-id"
"x-correlation-id" = "x-correlation-id"
"x-tenant-id" = "x-tenant-id"
"user-agent" = "user-agent"
"x-forwarded-for" = "client-ip"

[routes.upstream.responseMapping.setHeaders]
"cache-control" = "no-store"

[routes.audit]
enabled = true
event = "auth.login.attempt"
redactFields = ["password", "otp"]

# -----------------------------------------------------------------------------
# Роут: auth-me — GET /api/auth/me → gRPC AuthService.GetMe
# -----------------------------------------------------------------------------
[[routes]]
id = "auth-me"
enabled = true
type = "proxy"
match.path = "/api/auth/me"
match.method = "GET"

[routes.docs]
enabled = true
tag = "auth"
summary = "Current user"
description = "Returns current authenticated user profile"
security = [["bearer", []]]  # OpenAPI security requirement

[routes.docs.responseSchema]
"200" = "AuthMeResponse"
"401" = "ErrorResponse"
"403" = "ErrorResponse"

[routes.auth]
mode = "jwt"
providerRef = "jwt-access"
requiredScopes = ["profile:read"]

[[routes.rateLimit]]
key = "subject"
limit = 120
window = "1m"

[[routes.rateLimit]]
key = "tenant"
limit = 3000
window = "1m"

[routes.upstream]
serviceRef = "auth-grpc"
method = "auth.v1.AuthService/GetMe"
timeout = "2s"

[routes.upstream.retries]
enabled = true
retryableStatuses = ["UNAVAILABLE", "DEADLINE_EXCEEDED"]
attempts = 1
perTryTimeout = "1500ms"

[routes.upstream.requestMapping]
transport = "grpc"

[routes.upstream.requestMapping.headersToMetadata]
"x-request-id" = "x-request-id"
"x-correlation-id" = "x-correlation-id"
"x-tenant-id" = "x-tenant-id"

[routes.upstream.requestMapping.claimsToMetadata]
"sub" = "x-auth-sub"
"tenant_id" = "x-auth-tenant-id"
"scope" = "x-auth-scope"
"roles" = "x-auth-roles"

[routes.upstream.responseMapping.setHeaders]
"cache-control" = "no-store, private"

[routes.audit]
enabled = true
event = "auth.me.read"

# -----------------------------------------------------------------------------
# Роут: docs-ui — локальный хендлер Swagger UI
# -----------------------------------------------------------------------------
[[routes]]
id = "docs-ui"
enabled = true
type = "local"
match.path = "/api/docs"
match.method = "GET"

[routes.handler]
kind = "swagger-ui"

[routes.auth]
mode = "public"

[[routes.rateLimit]]
key = "ip"
limit = 60
window = "1m"

# -----------------------------------------------------------------------------
# Роут: docs-json — локальный хендлер OpenAPI JSON
# -----------------------------------------------------------------------------
[[routes]]
id = "docs-json"
enabled = true
type = "local"
match.path = "/api/docs-json"
match.method = "GET"

[routes.handler]
kind = "swagger-json"

[routes.auth]
mode = "public"

[[routes.rateLimit]]
key = "ip"
limit = 120
window = "1m"

# =============================================================================
# СХЕМЫ OPENAPI (DTO для валидации и документации)
# =============================================================================
[schemas.AuthLoginRequest]
type = "object"
required = ["login", "password"]

[schemas.AuthLoginRequest.properties.login]
type = "string"
example = "user@example.com"

[schemas.AuthLoginRequest.properties.password]
type = "string"
format = "password"

[schemas.AuthLoginRequest.properties.otp]
type = "string"
nullable = true

[schemas.AuthLoginRequest.properties.deviceId]
type = "string"
nullable = true

[schemas.AuthLoginResponse]
type = "object"

[schemas.AuthLoginResponse.properties.accessToken]
type = "string"

[schemas.AuthLoginResponse.properties.refreshToken]
type = "string"

[schemas.AuthLoginResponse.properties.expiresIn]
type = "integer"

[schemas.AuthLoginResponse.properties.user]
"$ref" = "#/components/schemas/UserProfile"

[schemas.AuthMeResponse]
type = "object"

[schemas.AuthMeResponse.properties.user]
"$ref" = "#/components/schemas/UserProfile"

[schemas.UserProfile]
type = "object"

[schemas.UserProfile.properties.id]
type = "string"

[schemas.UserProfile.properties.email]
type = "string"

[schemas.UserProfile.properties.displayName]
type = "string"

[schemas.ErrorResponse]
type = "object"

[schemas.ErrorResponse.properties.code]
type = "string"

[schemas.ErrorResponse.properties.message]
type = "string"

[schemas.ErrorResponse.properties.requestId]
type = "string"
```

---

# 🔎 Анализ пунктов конфигурации роутов: что можно улучшить?

## 📋 Структура роута: разбор по блокам

```toml
[[routes]]
id = "..."              # ✅ Обязательно: уникальный идентификатор для аудита и логгирования
enabled = true          # ✅ Хорошо: позволяет отключать роуты без удаления из конфига
type = "proxy|local"    # ✅ Критично: разделяет проксирование и локальную логику
```

### 🔹 `match` — сопоставление запроса
```toml
[routes.match]
path = "/api/auth/login"   # ✅ Строгий путь предпочтительнее wildcard для безопасности
method = "POST"            # ✅ Явное указание метода предотвращает случайные совпадения
```
> 💡 **Рекомендация**: Добавить поддержку `pathPrefix` для группировки роутов (например, `/api/auth/*` → auth-grpc), но с осторожностью — может усложнить отладку.

### 🔹 `docs` — документация
```toml
[routes.docs]
enabled = true
tag = "auth"
summary = "..."
description = "..."
requestBodySchema = "AuthLoginRequest"
```
> ✅ **Правильно**: Связь со схемами через `$ref` обеспечивает консистентность валидации и документации.  
> ⚠️ **Риск**: Если схема в `schemas` изменится, но `docs.responseSchema` не обновится — Swagger будет некорректным.  
> 🔧 **Решение**: Валидировать в CI, что все `requestBodySchema` / `responseSchema` существуют в секции `[schemas]`.

### 🔹 `auth` — аутентификация
```toml
[routes.auth]
mode = "public|jwt"
providerRef = "jwt-access"      # ✅ Ссылка на провайдера — хорошая практика
requiredScopes = ["profile:read"] # ✅ Fine-grained authorization
```
> ✅ **Правильно**: Разделение `mode` и `providerRef` позволяет переиспользовать провайдеры.  
> 💡 **Улучшение**: Добавить `optionalScopes` для гибкой авторизации (например, `admin:write` даёт доп. поля в ответе).

### 🔹 `rateLimit` — лимитирование
```toml
[[routes.rateLimit]]
key = "ip"          # ✅ Хорошо для публичных эндпоинтов
limit = 20
window = "1m"

[[routes.rateLimit]]
key = "ip+path"     # ✅ Комбинированный ключ предотвращает обход
limit = 100
window = "10m"
```
> ⚠️ **Внимание**: В multi-pod окружении `key = "ip"` требует **распределённого хранилища** (Redis) для консистентного учёта.  
> 🔧 **Решение**: Добавить в конфиг `rateLimit.backend = "redis"` с настройками подключения.

### 🔹 `upstream` — проксирование в gRPC
```toml
[routes.upstream]
serviceRef = "auth-grpc"
method = "auth.v1.AuthService/Login"
timeout = "3s"

[routes.upstream.requestMapping]
transport = "grpc"
body = "*"  # ✅ Простой маппинг: всё тело → gRPC-сообщение
```
> ✅ **Правильно**: `headersToMetadata` и `claimsToMetadata` — стандартный паттерн для propagation контекста.  
> 💡 **Улучшение**: Добавить `responseMapping.body` для трансформации gRPC-ответа в HTTP (если нужна адаптация формата).

### 🔹 `audit` — аудит
```toml
[routes.audit]
enabled = true
event = "auth.login.attempt"
redactFields = ["password", "otp"]  # ✅ Критично для compliance
```
> ✅ **Отлично**: Явное указание `redactFields` предотвращает утечки.  
> 🔧 **Дополнение**: Добавить `logLevel = "info|debug"` и `async = true` для асинхронной записи, чтобы не блокировать ответ.

---

## 🚀 Что можно добавить для расширения функционала

| Пункт | Зачем | Пример в TOML |
|-------|-------|---------------|
| **`circuitBreaker` на уровне роута** | Переопределить дефолты для критичных эндпоинтов | `[routes.upstream.circuitBreaker]` с `enabled`, `threshold` |
| **`timeout` per-try** | Тонкая настройка retry-логики | `[routes.upstream.retries] perTryTimeout = "800ms"` |
| **`cache` для ответов** | Уменьшить нагрузку на бэкенд для read-операций | `[routes.cache] enabled = true, ttl = "5m", key = "user_id"` |
| **`transform` запроса/ответа** | Адаптация форматов без изменения бэкенда | `[routes.upstream.requestMapping.transform] body = "jq(.email | ascii_downcase)"` |
| **`healthCheck` на уровне роута** | Быстрое исключение проблемных маршрутов из балансировки | `[routes.healthCheck] path = "/health", interval = "10s"` |
| **`featureFlag`** | Постепенный релиз новых роутов | `[routes.featureFlag] key = "new-auth-flow", rolloutPercent = 10` |

---

## 🧩 Итоговая рекомендация по формату

```toml
# ✅ Используйте YAML, если:
# - Конфиг глубоко вложенный (как у вас)
# - Команда привыкла к YAML (Kubernetes, Helm)
# - Нужна поддержка комментариев (в TOML комментарии есть, но менее распространены)

# ✅ Используйте TOML, если:
# - Конфиг относительно плоский
# - Критична защита от ошибок отступов
# - Интеграция с инструментами, где TOML стандарт (Cargo, Python pyproject)

# 🎯 Для вашего кейса: 
# Я рекомендую **остаться на YAML**, но:
# 1. Добавить строгую валидацию по JSON Schema
# 2. Использовать `yamllint` с кастомными правилами
# 3. Документировать структуру конфига в README
```

Если вы всё же выберете TOML — используйте **модульную структуру**: разбейте конфиг на файлы по секциям (`server.toml`, `routes-auth.toml`, `schemas.toml`) и собирайте их в runtime. Это улучшит читаемость и упростит merge-конфликты в команде.

---

## 🛠 Чек-лист перед внедрением

- [ ] Конфиг валидируется по схеме в CI (`yamllint` / `toml-cli validate`)
- [ ] Все `serviceRef` и `providerRef` проверяются на существование
- [ ] Секреты вынесены из конфига (используются env vars или Vault)
- [ ] Rate limit использует Redis для распределённого учёта (если pod'ов > 1)
- [ ] Аудит пишется асинхронно, чтобы не блокировать ответ
- [ ] `/docs-json` кэшируется на старте приложения, а не генерируется на каждый запрос
- [ ] Конфиг имеет `revision` и логируется при старте для аудита изменений

Ваша архитектура уже на очень высоком уровне — декларативный конфиг с чётким разделением ответственности это сильный фундамент. Осталось лишь аккуратно реализовать загрузчик и валидатор. 🚀