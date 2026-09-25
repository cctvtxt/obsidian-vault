# External 04 — Расширение common/modules/redis для Streams

> **Что это:** добавляет Stream-абстракцию поверх raw `REDIS_CLIENT` в `common/modules/redis/`.
> **Подчиняется:** `03-media-service-structure.md` → Redis Stream Consumer

## Проблема

Сейчас `common/modules/redis/` предоставляет только raw ioredis `REDIS_CLIENT`. Media-service для Redis Streams (MinIO events) будет делать raw вызовы:

```typescript
await this.redis.xgroup('CREATE', stream, group, '0', 'MKSTREAM');
await this.redis.xreadgroup('GROUP', group, consumer, 'COUNT', 10, 'BLOCK', 5000, 'STREAMS', stream, '>');
await this.redis.xack(stream, group, id);
```

Это хрупко: магические строки, повторяющиеся аргументы, тестируемость низкая.

## Решение: RedisStreamService

Изолируем Stream-логику в сервис. Media-service (и connector) не знает про XGROUP/XREAD, только про семантику.

### `redis-stream.service.ts` — новый файл

```typescript
@Injectable()
export class RedisStreamService {
    constructor(@Inject(REDIS_CLIENT) private readonly client: Redis) {}

    /**
     * Создать consumer group. MKSTREAM — создаст stream если его нет.
     * Idempotent: если группа уже есть — игнорируем (BUSYGROUP).
     */
    async ensureGroup(stream: string, group: string): Promise<void> {
        try {
            await this.client.xgroup('CREATE', stream, group, '0', 'MKSTREAM');
        } catch (error) {
            const err = error as { message?: string };
            if (err.message?.includes('BUSYGROUP')) return; // уже существует
            throw error;
        }
    }

    /**
     * Прочитать события из consumer group. Блокирующий.
     * Возвращает массив записей [{ id, fields }]. Пусто — null.
     */
    async readGroup(
        stream: string,
        group: string,
        consumer: string,
        opts?: { count?: number; blockMs?: number },
    ): Promise<StreamEntry[] | null> {
        const result = await this.client.xreadgroup(
            'GROUP', group, consumer,
            'COUNT', opts?.count ?? 10,
            'BLOCK', opts?.blockMs ?? 5000,
            'STREAMS', stream, '>',
        );
        if (!result) return null;

        const entries: StreamEntry[] = [];
        for (const [, streamEntries] of result) {
            for (const [id, fields] of streamEntries) {
                entries.push({ id, fields });
            }
        }
        return entries;
    }

    /** Подтвердить обработку записи. */
    async ack(stream: string, group: string, id: string): Promise<void> {
        await this.client.xack(stream, group, id);
    }
}

export interface StreamEntry {
    id: string;
    fields: string[]; // flat: ['key','value','key2','value2',...]
}
```

### `redis-stream.interface.ts` — опции формата событий (опционально)

Если хотим абстрагироваться от плоского формата полей — добавить нормализацию:

```typescript
export interface StreamEntryNormalized {
    id: string;
    data: Record<string, string>;
}
```

```typescript
// метод в RedisStreamService
async readGroupData(...): Promise<StreamEntryNormalized[] | null> {
    const entries = await this.readGroup(...);
    if (!entries) return null;
    return entries.map(({ id, fields }) => {
        const data: Record<string, string> = {};
        for (let i = 0; i < fields.length; i += 2) data[fields[i]] = fields[i + 1] ?? '';
        return { id, data };
    });
}
```

## Что добавить в модуль

`redis.module.ts` — зарегистрировать `RedisStreamService` как провайдер:

```typescript
const streamProvider: Provider = {
    provide: RedisStreamService,
    useFactory: (client: Redis) => new RedisStreamService(client),
    inject: [REDIS_CLIENT],
};
// добавить в providers + exports обоих forRoot/forRootAsync
```

## Сравнение подходов

### Raw ioredis (в media-service напрямую)

```typescript
const result = await this.redis.xreadgroup(
    'GROUP', 'media-verify', 'consumer-1',
    'COUNT', 10, 'BLOCK', 5000,
    'STREAMS', 'minio-events', '>'
);
for (const [, entries] of result ?? []) {
    await this.handleStream(entries);  // парсинг flat полей руками
}
```

Плюсы: без wrapper. Минусы: магические строки, хрупкий парсинг.

### RedisStreamService (в common)

```typescript
const entries = await this.streamService.readGroupData(
    'minio-events', 'media-verify', 'consumer-1',
);
for (const entry of entries ?? []) {
    await this.handleEvent(entry.data, entry.id);
    await this.streamService.ack('minio-events', 'media-verify', entry.id);
}
```

Плюсы: чистая семантика, типизированные entries, тестируемость (mock сервиса). Минус: wrapper.

**Выбор: `RedisStreamService`** — методы тривиальны, но убирают хрупкость raw X-команд и повторение. Тестируется mock'ом.

## Файлы

| Файл | Действие |
|------|----------|
| `common/modules/redis/redis-stream.service.ts` | Создать |
| `common/modules/redis/redis-stream.interface.ts` | Создать (типы StreamEntry) |
| `common/modules/redis/redis.module.ts` | Добавить provider + export |
| `common/modules/redis/index.ts` | Добавить экспорт |

## Итог

```
common/modules/redis/
├── redis.module.ts           # + RedisStreamService provider
├── redis.lifecycle.ts
├── redis.tokens.ts
├── redis.interface.ts
├── redis-stream.service.ts   # ← НОВЫЙ
├── redis-stream.interface.ts # ← НОВЫЙ
└── index.ts                  # + экспорт
```

Потребитель (`RedisConnectorService` из external/03) инжектит `RedisStreamService` вместо raw `REDIS_CLIENT` для Stream-операций.
