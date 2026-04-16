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