# External 03 — Connector интерфейс и модуль

> **Что это:** новый модуль `common/modules/connectors/`. Внешний модуль, ломающий привязку media-service к конкретному транспорту (Redis/HTTP/Kafka).
> **Подчиняется:** `03-media-service-structure.md` → `modules/connectors/`

## Проблема

Media-service узнаёт о загрузке файла в MinIO через events. Раньше — HTTP webhook (привязка к HTTP, валидация секрета). Теперь — Redis Stream. Но media-service **не должен знать** через что пришли события.

Нужен интерфейс: media-service определяет ЧТО должен делать connector, а не КАК он это делает.

## Решение: интерфейс IConnector

### `interfaces/connector.interface.ts` — в common/modules/connectors/

```typescript
export const I_CONNECTOR = Symbol('I_CONNECTOR');

/**
 * Контракт для обработчика событий хранилища.
 * Определяет ЧТО делает connector, а не КАК получает события.
 * Реализация: Redis Stream, Kafka, SQS, HTTP webhook — интерфейс не меняется.
 */
export interface IConnector {
    /** Запуск прослушивания событий. Один раз при старте. */
    start(): Promise<void>;

    /** Остановка прослушивания. При shutdown. */
    stop(): Promise<void>;

    /** Обработка события загрузки файла. Для каждого полученного event. */
    handleStorageEvent(event: StorageEvent): Promise<void>;
}

/**
 * Унифицированный формат события от хранилища.
 * Не зависит от формата MinIO, AWS S3, Yandex S3.
 */
export interface StorageEvent {
    key: string;
    bucket: string;
    eventType: string;          // 's3:ObjectCreated:Put' | 's3:ObjectRemoved:Delete'
    size: number | null;        // bytes
    etag: string | null;
    timestamp: number;          // ms
}
```

### Токен

```typescript
// connectors.tokens.ts
export const CONNECTORS_OPTIONS = Symbol('CONNECTORS_OPTIONS');
```

### Опции модуля

```typescript
// connectors.interface.ts
export interface ConnectorsModuleOptions {
    stream: string;
    consumerGroup: string;
    consumerName: string;
    pollCount?: number;   // default 10
    pollBlockMs?: number; // default 5000
}
```

## Модуль connectors (Redis реализация)

### `connectors.module.ts` — DynamicModule

```typescript
@Module({})
export class ConnectorsModule {
    static forRoot(options: ConnectorsModuleOptions): DynamicModule {
        const provider = {
            provide: I_CONNECTOR,
            useFactory: (redis: Redis, s3: S3ClientService) =>
                new RedisConnectorService(redis, s3, options),
            inject: [REDIS_CLIENT, S3ClientService],
        };
        return {
            module: ConnectorsModule,
            imports: [],
            providers: [provider],
            exports: [I_CONNECTOR],
        };
    }
}
```

## Как интерфейс отделён от реализации

```
┌──────────────────────────────────────────────────────┐
│  IConnector + StorageEvent (interfaces/)             │
│  start / stop / handleStorageEvent(StorageEvent)     │
└────────────────────────┬─────────────────────────────┘
                         │ implements
                         ▼
┌──────────────────────────────────────────────────────┐
│  RedisConnectorService (redis/)                      │
│  XGROUP CREATE → XREADGROUP → handleStorageEvent     │
│  Узнаёт формат MinIO из типов, конвертирует          │
│  MinioEventFields → StorageEvent                     │
└──────────────────────────────────────────────────────┘
```

**Если завтра Kafka:** новый `KafkaConnectorService implements IConnector`, меняешь `forRoot` options. `IConnector` не трогаешь.

## Структура модуля

```
common/modules/connectors/
├── index.ts                              # barrel
├── connectors.module.ts                  # DynamicModule forRoot
├── connectors.tokens.ts                  # CONNECTORS_OPTIONS
├── connectors.interface.ts               # ConnectorsModuleOptions
├── interfaces/
│   ├── connector.interface.ts            # IConnector, StorageEvent
│   └── index.ts
└── redis/
    ├── redis-connector.service.ts        # RedisConnectorService implements IConnector
    ├── minio.types.ts                    # MinioEventFields (формат MinIO в Redis)
    └── index.ts
```

## Что вынести в common (shared ответственность)

Интерфейс `IConnector` и `StorageEvent` — **в common**, потому что:
- Connector может понадобиться другому сервису (не только media)
- На случай Kafka/SQS connector — общий контракт
- Media-service импортирует только интерфейс, не реализацию

**Реализацию** (Redis) media-service использует через `forRoot()`.

## Импорты (зависимости)

- `@nestjs/common` — Module, Injectable, Inject
- `ioredis` — REDIS_CLIENT (из common/modules/redis)
- `@aws-sdk/client-s3` — через S3ClientService

## Важно

- `RedisConnectorService` **не знает** про purpose profiles, mime-sniffer, BullMQ
- Он только конвертирует `MinioEventFields` → `StorageEvent`, вызывает `handleStorageEvent()`
- Бизнес-логика (sniff, tag, emit) — **в media-service**, не в common
