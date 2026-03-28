## Частые домены backend

- **Identity & Access Management (IAM)** — пользователи, логин, роли, permissions, SSO, MFA, service accounts.
    
- **User Management** — профиль пользователя, настройки, предпочтения, статус аккаунта.[](https://www.codelit.io/blog/backend-architecture-patterns-guide)
    
- **Organization / Tenant Management** — компании, команды, workspace, multi-tenancy, memberships, invitations.
    
- **Authorization / Access Control** — RBAC, ABAC, policy engine, resource-level access.
    
- **Payment Management** — платежи, методы оплаты, инвойсы, рефанды, chargebacks, billing state.[](https://www.codelit.io/blog/backend-architecture-patterns-guide)
    
- **Billing & Invoicing** — тарифы, подписки, счета, usage metering, tax, proration.[](https://www.codelit.io/blog/backend-architecture-patterns-guide)
    
- **Order Management** — корзина, заказы, статусы, fulfillment, возвраты.[](https://www.codelit.io/blog/backend-architecture-patterns-guide)
    
- **Catalog / Product Management** — товары, планы, услуги, прайсинг, availability.[](https://www.codelit.io/blog/backend-architecture-patterns-guide)
    
- **Notification Management** — email, SMS, push, in-app notifications, templates, delivery tracking.
    
- **Content Management** — страницы, статьи, медиа, редакторский workflow, publishing.
    
- **Search** — индексирование, поиск, фильтры, relevance, autocomplete.
    
- **File / Document Management** — upload, storage, versioning, access to files.
    
- **Reporting & Analytics** — отчеты, dashboards, aggregations, export.
    
- **Audit & Compliance** — audit log, change history, traceability, retention.
    
- **Workflow / Orchestration** — бизнес-процессы, approvals, state machines, background jobs.
    
- **Support / Ticketing** — обращения, SLA, комментарии, assignment, escalation.
    
- **Messaging / Chat** — conversations, threads, presence, moderation.
    
- **Integration Management** — webhooks, third-party APIs, sync jobs, connectors.
    
- **Configuration / Feature Flags** — runtime settings, toggles, experiments.
    
- **Session / Token Management** — session lifecycle, refresh tokens, revocation.
    
- **Risk / Fraud / Abuse Prevention** — rate limits, anomaly detection, device/IP reputation.
    
- **Inventory Management** — остатки, reservations, stock movements.
    
- **Shipping / Delivery Management** — addresses, carriers, tracking, shipping labels.
    
- **Subscription Management** — планы, upgrades/downgrades, renewal, cancellation.
    
- **Localization / Translation** — языки, timezone, regional formatting.
    
- **Admin Console / Backoffice** — операционные инструменты для support и ops.
    

## Более универсальные названия

Если хочешь назвать области **шире и технологически нейтрально**, можно использовать такие термины:

|Вариант|Когда подходит|
|---|---|
|**IAM**|Когда важны логин, роли, доступы, политики [](https://github.com/kdeldycke/awesome-iam).|
|**Billing**|Когда есть подписки, счета и списания.|
|**Core domain**|Главная бизнес-логика продукта.|
|**Customer Management**|Если фокус на клиентах, а не на пользователях как техсущности.|
|**Operations**|Для админки, саппорта, ручных операций.|
|**Platform Services**|Для общих сервисов: auth, notifications, config, audit.|
|**Business Services**|Для прикладных доменов продукта.|
|**Shared Services**|Для повторно используемых сервисов между модулями.|
|**Backoffice**|Для внутренних инструментов команды.|
|**Domain Modules**|Если архитектура модульная, а не микросервисная.|

## Варианты нейминга по стилю

Если тебе нужны названия в **коде, документации или архитектурной схеме**, обычно используют такие стили:

- **Noun-based:** `users`, `payments`, `billing`, `notifications`, `orders`.
    
- **Business-oriented:** `access-management`, `revenue-management`, `customer-identity`.
    
- **Platform-oriented:** `iam-service`, `eventing`, `integration-hub`, `audit-log`.
    
- **DDD-style:** `identity`, `compliance`, `fulfillment`, `catalog`, `subscription`.
    

## Пример набора для SaaS

Для типичного SaaS можно разложить backend так:

- `identity`
    
- `access-management`
    
- `tenant-management`
    
- `subscription-management`
    
- `billing`
    
- `payments`
    
- `notifications`
    
- `audit-log`
    
- `feature-flags`
    
- `integrations`
    
- `reporting`
    
- `admin-backoffice`