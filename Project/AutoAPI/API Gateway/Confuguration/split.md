# 🧩 Модульная структура конфигурации API Gateway

Отличный вопрос! Разделение конфига на модули — это **лучшая практика** для командной разработки и масштабирования. Ниже — продуманная структура с разделением общих и специфических настроек.

---

## 📁 Предлагаемая структура файлов

```
config/
├── gateway.toml                 # 🔗 Главный файл: импорты + метаданные
│
├── 01-server.toml              # 🌐 Глобальные настройки сервера
├── 02-observability.toml       # 📊 Tracing, logging, metrics
├── 03-security.toml            # 🔐 CORS, headers, auth providers
├── 04-upstream-defaults.toml   # ⚙️ Дефолты для gRPC-вызовов
├── 05-docs.toml                # 📚 Swagger/OpenAPI настройки
│
├── services/                   # 🧭 Описание внутренних микросервисов
│   ├── auth-grpc.toml
│   ├── payments-grpc.toml
│   └── users-grpc.toml
│
├── routes/                     # 🛣️ Маршруты (по доменам или сервисам)
│   ├── auth.toml               # Роуты аутентификации
│   ├── payments.toml           # Роуты платежей
│   └── internal.toml           # Health, docs, admin
│
└── schemas/                    # 📋 OpenAPI схемы (DTO)
    ├── auth.toml
    ├── payments.toml
    └── common.toml
```

> 💡 **Префиксы `01-`, `02-`** в корневых файлах обеспечивают предсказуемый порядок загрузки при слиянии конфигов.

---

## 🔗 `gateway.toml` — главный файл-манифест

```toml
# config/gateway.toml
# Главный манифест: метаданные + список импортов

[metadata]
name = "public-api-gateway"
revision = "2026-04-19.002"
description = "Unified API Gateway for platform services"

# Порядок импортов важен: последующие файлы могут переопределять предыдущие
[imports]
# Базовые настройки (загружаются первыми)
core = [
  "01-server.toml",
  "02-observability.toml",
  "03-security.toml",
  "04-upstream-defaults.toml",
  "05-docs.toml",
]

# Сервисы: описания gRPC-бэкендов
services = [
  "services/auth-grpc.toml",
  "services/payments-grpc.toml",
  # "services/users-grpc.toml",  # закомментировано — не активно
]

# Роуты: внешние эндпоинты
routes = [
  "routes/auth.toml",
  "routes/payments.toml",
  "routes/internal.toml",
]

# Схемы: DTO для валидации и документации
schemas = [
  "schemas/common.toml",
  "schemas/auth.toml",
  "schemas/payments.toml",
]

# 🔧 Опционально: оверрайды для окружений (env-specific)
# overrides = "envs/production.toml"
```

> 🎯 **Принцип**: `gateway.toml` не содержит бизнес-логики — только оркестрация. Это упрощает code review и понимание "что загружается".

---

## 🌐 `01-server.toml` — настройки сервера

```toml
# config/01-server.toml

[server]
publicBaseUrl = "https://api.example.com"
listen.port = 3000
listen.globalPrefix = "/api"
trustProxy = true

# Опционально: настройки TLS для direct-режима (без Ingress)
# [server.tls]
# enabled = true
# certPath = "/certs/tls.crt"
# keyPath = "/certs/tls.key"
```

---

## 📊 `02-observability.toml` — наблюдаемость

```toml
# config/02-observability.toml

[observability]
requestIdHeader = "x-request-id"
correlationIdHeader = "x-correlation-id"
logLevel = "info"  # debug | info | warn | error

[observability.tracing]
enabled = true
propagate = ["x-request-id", "x-correlation-id", "traceparent", "tracestate"]
# sampler = "always"  # или "ratio:0.1" для прода

[observability.metrics]
enabled = true
prefix = "gateway"
# backend = "prometheus"  # или "otel"

[observability.audit]
enabled = true
output = "stdout"  # или "kafka", "file:/var/log/audit.json"
redactFields = ["password", "otp", "token", "secret"]
```

---

## 🔐 `03-security.toml` — безопасность

```toml
# config/03-security.toml

# -----------------------------------------------------------------------------
# CORS
# -----------------------------------------------------------------------------
[security.cors]
enabled = true
allowOrigins = ["https://app.example.com"]
allowMethods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
allowHeaders = [
  "authorization",
  "content-type",
  "x-request-id",
  "x-correlation-id",
  "x-tenant-id",
]
exposeHeaders = ["x-request-id", "x-correlation-id"]
allowCredentials = true
maxAgeSeconds = 600

# -----------------------------------------------------------------------------
# Хедеры: forward / drop / set
# -----------------------------------------------------------------------------
[security.headers]
forward = [
  "x-request-id",
  "x-correlation-id",
  "x-forwarded-for",
  "x-forwarded-proto",
  "x-real-ip",
  "user-agent",
  "x-tenant-id",
]
drop = ["x-internal-debug", "x-service-token", "x-forwarded-host"]

# Хедеры, которые добавляются ко ВСЕМ ответам
[security.headers.set]
x-content-type-options = "nosniff"
x-frame-options = "DENY"
strict-transport-security = "max-age=31536000; includeSubDomains"
content-security-policy = "default-src 'self'"

# -----------------------------------------------------------------------------
# Провайдеры аутентификации
# -----------------------------------------------------------------------------
[[security.authProviders]]
id = "jwt-access"
type = "jwt"
fromHeader = "Authorization"
scheme = "Bearer"
jwksUri = "https://identity.example.com/.well-known/jwks.json"
cacheTtl = "10m"
clockSkew = "30s"
requiredClaims = ["sub", "exp"]
issuer = "https://identity.example.com"  # опционально: проверка iss

[[security.authProviders]]
id = "api-key"
type = "apiKey"
fromHeader = "X-API-Key"
# secrets выносятся в env vars или Vault: ${API_KEY_STORE}
```

> 🔐 **Важно**: Секреты (JWKS private keys, API keys) **не храните в конфиге**. Используйте `${ENV_VAR}` синтаксис + загрузку через `process.env` или Vault.

---

## ⚙️ `04-upstream-defaults.toml` — дефолты для gRPC

```toml
# config/04-upstream-defaults.toml

[upstreamDefaults]
protocol = "grpc"
discovery = "kubernetes-dns"  # или "consul", "static"
connectTimeout = "2s"
requestTimeout = "5s"
idleTimeout = "30s"

[upstreamDefaults.retries]
enabled = true
retryableStatuses = ["UNAVAILABLE", "DEADLINE_EXCEEDED", "RESOURCE_EXHAUSTED"]
attempts = 1
perTryTimeout = "1500ms"
backoff = "exponential"  # linear | exponential | constant

[upstreamDefaults.circuitBreaker]
enabled = true
maxConnections = 1000
maxPendingRequests = 500
maxRequests = 5000
ejectionTime = "30s"

[upstreamDefaults.outlierDetection]
enabled = true
consecutive5xx = 5
interval = "10s"
baseEjectionTime = "30s"
maxEjectionPercent = 50

# Опционально: настройки mTLS для service mesh
# [upstreamDefaults.tls]
# mode = "ISTIO_MUTUAL"  # или "SIMPLE", "DISABLE"
# caCert = "/var/run/secrets/istio/root-cert.pem"
```

---

## 📚 `05-docs.toml` — Swagger/OpenAPI

```toml
# config/05-docs.toml

[docs]
enabled = true
route.uiPath = "/docs"
route.jsonPath = "/docs-json"
source = "gateway-runtime"  # генерация из конфига, не из кода

[docs.openapi]
title = "Public API Gateway"
description = "Unified gateway API for auth and platform endpoints"
version = "1.0.0"
termsOfService = "https://example.com/terms"

[[docs.openapi.tags]]
name = "auth"
description = "Authentication endpoints"

[[docs.openapi.tags]]
name = "payments"
description = "Payment processing endpoints"

[docs.openapi.securitySchemes.bearer]
type = "http"
scheme = "bearer"
bearerFormat = "JWT"

[docs.openapi.securitySchemes.apiKey]
type = "apiKey"
in = "header"
name = "X-API-Key"

[docs.exposure]
includeRoutes = ["auth-login", "auth-me", "payments-create"]
hideInternalRoutes = true

[docs.ui]
persistAuthorization = true
displayRequestDuration = true
docExpansion = "list"
tryItOutEnabled = true
defaultModelsExpandDepth = 1

[docs.cache]
enabled = true
ttl = "60s"  # кэш сгенерированного OpenAPI-документа
```

---

## 🧭 `services/*.toml` — описания микросервисов

### `services/auth-grpc.toml`
```toml
# config/services/auth-grpc.toml

[id = "auth-grpc"]  # ⚠️ TOML не поддерживает ключ в [[services]], поэтому используем плоскую структуру
kind = "grpc"
enabled = true

[services.auth-grpc.kubernetes]
namespace = "auth"
serviceName = "auth-svc"
dns = "auth-svc.auth.svc.cluster.local"
port = 9090

[services.auth-grpc.grpc]
authority = "auth-svc.auth.svc.cluster.local"
package = "auth.v1"
# protoFile = "proto/auth/v1/auth.proto"  # опционально: для генерации клиентов

[services.auth-grpc.healthCheck]
enabled = true
type = "grpc"
service = "grpc.health.v1.Health"
endpoint = "/grpc.health.v1.Health/Check"
interval = "10s"
timeout = "3s"
```

### `services/payments-grpc.toml`
```toml
# config/services/payments-grpc.toml

[services.payments-grpc]
kind = "grpc"
enabled = true

[services.payments-grpc.kubernetes]
namespace = "payments"
serviceName = "payments-svc"
dns = "payments-svc.payments.svc.cluster.local"
port = 9090

[services.payments-grpc.grpc]
authority = "payments-svc.payments.svc.cluster.local"
package = "payments.v1"

[services.payments-grpc.healthCheck]
enabled = true
type = "grpc"
service = "grpc.health.v1.Health"
```

> 💡 **Паттерн**: `[services.<id>....]` позволяет легко добавлять новые сервисы без изменения структуры файла.

---

## 🛣️ `routes/*.toml` — маршруты (по доменам)

### `routes/auth.toml`
```toml
# config/routes/auth.toml

# -----------------------------------------------------------------------------
# POST /api/auth/login → AuthService.Login
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
description = "Authenticates user and returns access/refresh tokens"
requestBodySchema = "AuthLoginRequest"

[routes.docs.responseSchema]
"200" = "AuthLoginResponse"
"400" = "ErrorResponse"
"401" = "ErrorResponse"
"429" = "ErrorResponse"

[routes.auth]
mode = "public"  # не требует аутентификации

[routes.validation]
contentTypes = ["application/json"]
maxBodyBytes = 32768

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
enabled = false  # login не ретраим — лучше сразу вернуть ошибку

[routes.upstream.requestMapping]
transport = "grpc"
body = "*"

[routes.upstream.requestMapping.headersToMetadata]
"x-request-id" = "x-request-id"
"x-correlation-id" = "x-correlation-id"
"x-tenant-id" = "x-tenant-id"
"user-agent" = "user-agent"

[routes.upstream.responseMapping.setHeaders]
"cache-control" = "no-store"
"pragma" = "no-cache"

[routes.audit]
enabled = true
event = "auth.login.attempt"
redactFields = ["password", "otp"]
logLevel = "info"


# -----------------------------------------------------------------------------
# GET /api/auth/me → AuthService.GetMe
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
description = "Returns profile of authenticated user"
security = [["bearer", []]]

[routes.docs.responseSchema]
"200" = "AuthMeResponse"
"401" = "ErrorResponse"
"403" = "ErrorResponse"

[routes.auth]
mode = "jwt"
providerRef = "jwt-access"
requiredScopes = ["profile:read"]

[[routes.rateLimit]]
key = "subject"  # по user ID из JWT claims
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

[routes.upstream.requestMapping.headersToMetadata]
"x-request-id" = "x-request-id"
"x-correlation-id" = "x-correlation-id"
"x-tenant-id" = "x-tenant-id"

[routes.upstream.requestMapping.claimsToMetadata]
"sub" = "x-auth-sub"
"tenant_id" = "x-auth-tenant-id"
"scope" = "x-auth-scope"

[routes.upstream.responseMapping.setHeaders]
"cache-control" = "no-store, private"

[routes.audit]
enabled = true
event = "auth.me.read"
logLevel = "debug"  # менее критично, чем login
```

### `routes/internal.toml` — локальные роуты (docs, health)
```toml
# config/routes/internal.toml

# -----------------------------------------------------------------------------
# Swagger UI
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
# OpenAPI JSON
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

# -----------------------------------------------------------------------------
# Health check для Kubernetes
# -----------------------------------------------------------------------------
[[routes]]
id = "health"
enabled = true
type = "local"
match.path = "/health"
match.method = "GET"

[routes.handler]
kind = "health-check"
checks = ["config-loaded", "grpc-clients-connected"]

[routes.auth]
mode = "public"  # health должен быть публичным для kubelet
```

---

## 📋 `schemas/*.toml` — OpenAPI схемы (DTO)

### `schemas/common.toml`
```toml
# config/schemas/common.toml

[schemas.ErrorResponse]
type = "object"
required = ["code", "message"]

[schemas.ErrorResponse.properties.code]
type = "string"
example = "AUTH_INVALID_CREDENTIALS"

[schemas.ErrorResponse.properties.message]
type = "string"
example = "Invalid login or password"

[schemas.ErrorResponse.properties.requestId]
type = "string"
description = "Request ID for support tracing"
```

### `schemas/auth.toml`
```toml
# config/schemas/auth.toml

[schemas.AuthLoginRequest]
type = "object"
required = ["login", "password"]

[schemas.AuthLoginRequest.properties.login]
type = "string"
format = "email"
example = "user@example.com"
minLength = 5
maxLength = 255

[schemas.AuthLoginRequest.properties.password]
type = "string"
format = "password"  # не отображается в UI
minLength = 8
maxLength = 128

[schemas.AuthLoginRequest.properties.otp]
type = "string"
nullable = true
pattern = "^[0-9]{6}$"
description = "One-time password for 2FA"

[schemas.AuthLoginRequest.properties.deviceId]
type = "string"
nullable = true
format = "uuid"
description = "Device identifier for session binding"


[schemas.AuthLoginResponse]
type = "object"
required = ["accessToken", "refreshToken", "expiresIn"]

[schemas.AuthLoginResponse.properties.accessToken]
type = "string"
description = "JWT access token"

[schemas.AuthLoginResponse.properties.refreshToken]
type = "string"
description = "Refresh token for token rotation"

[schemas.AuthLoginResponse.properties.expiresIn]
type = "integer"
description = "Token TTL in seconds"
example = 3600

[schemas.AuthLoginResponse.properties.user]
"$ref" = "#/components/schemas/UserProfile"


[schemas.AuthMeResponse]
type = "object"
required = ["user"]

[schemas.AuthMeResponse.properties.user]
"$ref" = "#/components/schemas/UserProfile"


[schemas.UserProfile]
type = "object"
required = ["id", "email"]

[schemas.UserProfile.properties.id]
type = "string"
format = "uuid"

[schemas.UserProfile.properties.email]
type = "string"
format = "email"

[schemas.UserProfile.properties.displayName]
type = "string"
nullable = true

[schemas.UserProfile.properties.roles]
type = "array"
items = { type = "string" }
nullable = true
```

---

## 🔧 Как загружать модульные конфиги в NestJS

```typescript
// config/loader.ts
import * as fs from 'fs';
import * as path from 'path';
import * as toml from 'toml';
import { merge } from 'lodash';

interface GatewayConfig {
  metadata: { name: string; revision: string };
  imports: { core: string[]; services: string[]; routes: string[]; schemas: string[] };
  // ... остальные секции
}

export async function loadModularConfig(basePath: string): Promise<GatewayConfig> {
  // 1. Загружаем манифест
  const manifestPath = path.join(basePath, 'gateway.toml');
  const manifest = toml.parse(fs.readFileSync(manifestPath, 'utf8'));
  
  let mergedConfig: any = { metadata: manifest.metadata };

  // 2. Загружаем секции в порядке из манифеста
  const loadFiles = (fileList: string[]) => {
    for (const file of fileList) {
      const fullPath = path.join(basePath, file);
      if (fs.existsSync(fullPath)) {
        const content = toml.parse(fs.readFileSync(fullPath, 'utf8'));
        mergedConfig = merge(mergedConfig, content);
      }
    }
  };

  loadFiles(manifest.imports.core);
  loadFiles(manifest.imports.services);
  loadFiles(manifest.imports.routes);
  loadFiles(manifest.imports.schemas);

  // 3. Валидация по JSON Schema
  await validateConfig(mergedConfig);

  // 4. Подстановка env vars (${VAR_NAME} → process.env.VAR_NAME)
  return substituteEnvVars(mergedConfig);
}

function substituteEnvVars(obj: any): any {
  // Рекурсивная замена ${VAR} на process.env.VAR
  // Можно использовать dotenv-expand или кастомную реализацию
  // ...
  return obj;
}
```

> 💡 **Совет**: Добавьте `config:validate` скрипт в `package.json`, который запускает валидацию конфига без запуска приложения — это ускорит CI.

---

## ✅ Преимущества модульной структуры

| Преимущество | Описание |
|-------------|----------|
| **🧑‍💻 Параллельная работа** | Команда аутентификации правит `routes/auth.toml`, команда платежей — `routes/payments.toml` без конфликтов |
| **🔍 Локальная валидация** | Можно валидировать `routes/auth.toml` изолированно перед мёржем |
| **♻️ Переиспользование** | `03-security.toml` и `04-upstream-defaults.toml` общие для всех роутов — меньше дублирования |
| **🧪 Тестирование** | Легко замокать отдельный сервис: `services/auth-grpc.toml` → `services/auth-grpc.mock.toml` |
| **📦 Оверрайды по окружениям** | Добавить `envs/staging.toml` с изменёнными таймаутами, не трогая основной конфиг |
| **🔐 Безопасность** | Секреты вынесены в env vars; конфиг можно коммитить в репо без риска утечки |

---

## 🚀 Дополнительные улучшения

### 1. Поддержка оверрайдов по окружениям
```toml
# config/envs/production.toml
[server]
publicBaseUrl = "https://api.prod.example.com"

[upstreamDefaults.retries]
attempts = 3  # больше ретраев в продакшене

[docs.cache]
ttl = "300s"  # дольше кэш в продакшене
```

Загружается последним в `loadModularConfig()` и переопределяет базовые настройки.

### 2. Валидация кросс-ссылок
```typescript
// config/validator.ts
export function validateConfig(config: any) {
  // Проверка: все serviceRef в роутах существуют в services
  for (const route of config.routes || []) {
    if (route.upstream?.serviceRef) {
      const serviceExists = config.services?.some(
        (s: any) => s[route.upstream.serviceRef] || s.id === route.upstream.serviceRef
      );
      if (!serviceExists) {
        throw new Error(`Route ${route.id} references unknown service: ${route.upstream.serviceRef}`);
      }
    }
    // Аналогично для providerRef, requestBodySchema, etc.
  }
}
```

### 3. Генерация документации по структурам
```bash
# scripts/generate-config-docs.ts
# Автоматически генерирует README с описанием всех полей конфига
# на основе JSDoc-комментариев или JSON Schema
```

---

## 🎯 Итоговая рекомендация

```toml
# ✅ Оставьте YAML, если:
# - Команда уже использует YAML в Kubernetes
# - Нужны комментарии в середине сложных структур
# - Глубокая вложенность (TOML становится многословным)

# ✅ Перейдите на TOML, если:
# - Хотите строгую типизацию и защиту от ошибок отступов
# - Конфиг будет редактироваться не-разработчиками (DevOps, PM)
# - Интеграция с инструментами, где TOML стандарт

# 🎯 Гибридный подход (рекомендую):
# - Основные конфиги в YAML (для читаемости)
# - Роуты и схемы — в отдельных YAML-файлах по доменам
# - Валидация через общую JSON Schema
# - Загрузка через модульный loader с поддержкой импортов
```

Если вы выберете модульную структуру — **начните с разделения `routes/` и `services/`**, это даст максимальный выигрыш в командной работе уже на старте.

Нужна помощь с:
1. Генерацией **JSON Schema** для валидации модульных конфигов?
2. Написанием **NestJS ConfigModule** с поддержкой импортов?
3. Примером **GitHub Actions workflow** для валидации конфига при PR?

Готов помочь с любым из пунктов! 🚀