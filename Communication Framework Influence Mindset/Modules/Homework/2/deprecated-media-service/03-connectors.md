# Media-service 03 — Модуль connectors (Redis Stream)

> **Что это:** внутренняя обвязка media-service вокруг общего `IConnector` из common (external/03).
> **Подчиняется:** `03-media-service-structure.md` → modules/connectors/

## Отличие от common-connector

У нас **два слоя**:

| Слой | Где | Что делает |
|------|-----|-----------|
| `common/modules/connectors` | common | `IConnector` + `RedisConnectorService`. Транспорт-agnet. Конвертирует MinIO → StorageEvent. **Не знает** про purpose/mime/BullMQ. |
| `media-service/modules/connectors` | media-service | Реализует потребитель событий: берёт StorageEvent и выполняет бизнес-логику (sniff, validate, tag, emit). |

То есть: **common** даёт "как получить событие", **media-service** даёт "что делать с событием".

## Интерфейс для бизнес-логики

### `modules/connectors/connectors.service.ts`

```typescript
import type { IConnector, StorageEvent } from '@kirillasyamov/common/connectors';

@Injectable()
export class ConnectorsService implements IConnector {
    constructor(
        @Inject(I_S3_CLIENT) private readonly s3: IS3ClientService,
        @Inject(I_MEDIA_EVENTS_PRODUCER) private readonly producer: IMediaEventsProducer,
    ) {}

    async start(): Promise<void> {
        // нет собственного цикла — стартует RedisConnectorService из common
        // этот сервис только обрабатывает события
    }

    async stop(): Promise<void> {}

    async handleStorageEvent(event: StorageEvent): Promise<void> {
        if (event.eventType !== 's3:ObjectCreated:Put') return;

        const parsed = parseObjectKey(event.key);
        if (!parsed) return;

        const purpose = findPurposeByPrefix(parsed.prefix);
        if (!purpose) {
            this.logger.warn(`Skipping event for unknown purpose: key=${event.key}`);
            return;
        }
        const profile = getPurposeProfile(purpose);

        // 1. файл существует в S3?
        try {
            await this.s3.headObject(event.bucket, event.key);
        } catch {
            this.logger.warn(`Object not reachable, skipping: key=${event.key}`);
            return;
        }

        // 2. sniff MIME по первым 512 байтам
        const bytes = await this.sniffFirstBytes(event.bucket, event.key);
        const mime = bytes ? sniffMime(bytes) : null;

        // 3. валидация against purpose profile
        const confirmed = mime !== null && profile.allowedMimes.includes(mime);

        // 4. тег статуса
        await this.s3.putObjectTagging(event.bucket, event.key, {
            [STATUS_TAG]: confirmed ? STATUS_TAG_VALUES.CONFIRMED : STATUS_TAG_VALUES.REJECTED,
        });

        // 5. emit
        if (confirmed) {
            await this.producer.emitUploadCompleted(event.key);
        } else {
            const reason = mime === null ? 'unrecognized file signature' : `mime "${mime}" is not allowed`;
            await this.producer.emitUploadRejected(event.key, reason);
        }
    }

    private async sniffFirstBytes(bucket: string, key: string): Promise<Uint8Array | null> {
        try {
            const obj = await this.s3.getObject(bucket, key, 'bytes=0-511');
            return await obj.Body.transformToByteArray().then((arr) => arr.slice(0, 512));
        } catch {
            return null;
        }
    }
}
```

## Модуль

### `modules/connectors/connectors.module.ts`

```typescript
@Module({
    imports: [
        // берёт I_CONNECTOR (Redis слой) из common
        RedisConnectorModule.forRoot({ stream, consumerGroup, consumerName }),
    ],
    providers: [ConnectorsService],   // бизнес-слой
})
export class ConnectorsModule {}
```

**Вопрос DI:** `ConnectorsService` и `RedisConnectorService` оба имплементят `IConnector`.
- `I_CONNECTOR` (транспортный) регистрирует common → `RedisConnectorService`
- `ConnectorsService` — свой класс-провайдер, инжектится напрямую в media.service (не через I_CONNECTOR)
- Либо: `ConnectorsService` оборачивает `RedisConnectorService` (decorator pattern)

Уточнить при реализации: два токена (`I_RAW_CONNECTOR` + `I_CONNECTOR`) либо один. Оптимально — один токен, `ConnectorsService` оборачивает Redis-слой.

## Полный flow (presigned mode)

```
1. Client → RequestUploadUrl → PresignedFileHandler.createUpload()
   → { key, url, fields }

2. Client → POST url (MinIO) — прямая загрузка

3. MinIO → XADD minio-events → Redis Stream

4. RedisConnectorService (common) — XREADGROUP → конвертация fields → StorageEvent
   → вызывает IConnector.handleStorageEvent(StorageEvent)

5. ConnectorsService (media-service) — business logic:
   headObject → sniffMime → validate → putObjectTagging → emit BullMQ

6. Client → GetFileMetadata → MediaService → headObject + getObjectTags
   → { status: CONFIRMED | PENDING }
```

## Файлы

| Файл | Действие |
|------|----------|
| `modules/connectors/connectors.module.ts` | Создать |
| `modules/connectors/connectors.service.ts` | Создать (перенос логики из `webhook/minio-webhook.service.ts`) |
| `modules/connectors/connectors.types.ts` | Удалить/не нужен (типы формата теперь в common) |
| `modules/connectors/index.ts` | Создать |

## Перенос из webhook

Логика `minio-webhook.service.ts` (headObject → sniff → tag → emit) переезжает в `ConnectorsService.handleStorageEvent()`. Разница:

| Было (webhook) | Стало (connectors) |
|----------------|--------------------|
| HTTP POST payload `MinioWebhookPayload` | `StorageEvent` из common |
| `extractCreatedEvents()` | Транспорт уже даёт готовые StorageEvent |
| S3 через raw `S3_CLIENT` + `S3_CLIENT_OPTIONS` | S3 через `IS3ClientService` (единый интерфейс) |
| Controller + секрет | Нет HTTP вообще |
