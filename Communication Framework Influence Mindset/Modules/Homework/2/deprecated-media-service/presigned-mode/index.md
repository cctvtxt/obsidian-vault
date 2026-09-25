# Presigned Mode — Шаг 3: обновление proto + presigned провайдер

> **Статус:** после direct-mode шага 2. Добавляет presigned-режим.
> **Цель:** расширить media-service на presigned — новый RPC, presigned провайдер, подстановка реализации вместо direct.

## Что добавляется

1. Новый RPC в proto: `RequestUploadUrl` (presigned URL для загрузки напрямую в MinIO)
2. Новый провайдер: `providers/presigned-mode/presigned.provider.ts`
3. Подстановка `PresignedProviderModule` вместо `DirectProviderModule` в media.module
4. Connectors (Redis Stream) для верификации после загрузки

## Целевая структура

```
media-service/src/
├── interfaces/
│   ├── s3-client.interface.ts
│   ├── file-handler.interface.ts
│   ├── media-events.interface.ts
│   └── index.ts
├── utils/
│   ├── mime-sniffer.ts
│   ├── object-key.ts
│   └── index.ts
├── providers/
│   ├── direct-mode/
│   │   ├── direct.module.ts
│   │   ├── direct.provider.ts
│   │   ├── upload-stream.controller.ts
│   │   ├── uploader.service.ts
│   │   └── index.ts
│   └── presigned-mode/
│       ├── presigned.module.ts
│       ├── presigned.provider.ts
│       ├── interfaces/
│       │   └── index.ts
│       └── index.ts
├── modules/                    # (NEW)
│   └── connectors/
│       ├── connectors.module.ts
│       ├── connectors.service.ts
│       └── index.ts
├── media.module.ts             # подключает PresignedProviderModule
├── media.controller.ts
├── media.service.ts
├── media.constants.ts
└── main.ts
```

## IFileHandler — расширенный контракт (одинаковые данные 100%)

Клиент всегда передаёт одни и те же данные. Режим решает провайдер (не протокол).

```typescript
export interface IFileHandler {
    // direct: стрим байтов
    uploadStream(): IUploadStreamService;
    // presigned: выдать URL. Клиент шлёт те же purpose/owner, байты сам в MinIO.
    requestUploadUrl(params: UploadUrlParams): Promise<UploadUrlResult>;
}
```

```typescript
export interface UploadUrlParams {
    purpose: string;
    ownerAccountId: string;
    filename?: string;
    metadata?: Record<string,string>;
    sizeBytes?: number;   // чтобы presigned POST ограничил размер
}
export interface UploadUrlResult {
    key: string;
    url: string;
    fields: Record<string,string>;
    expiresAt: string;
}
```

**Единый интерфейс:** клиент всегда передаёт `purpose/ownerAccountId/filename/metadata`. В direct тот же набор + байты. В presigned — без байт, получает URL и сам грузит. media-service по конфигу (не proto) выбирает провайдера.

## Подстановка реализации (media.module)

```typescript
@Module({
    imports: [PresignedProviderModule],   // меняем direct → presigned
    ...
})
export class MediaModule {}
```

`PresignedProviderModule` регистрирует `PresignedFileHandler` как `I_FILE_HANDLER`, импортирует `ConnectorsModule` + регистрирует BullMQ-очередь.

---

## External на эту итерацию

См. `external/` в этой папке:
- `external/01-media-proto.md` — добавить RPC `RequestUploadUrl`
- `external/02-s3-client.md` — requirements (presigned POST / createPresignedPost)
- `external/03-connectors.md` — Redis Stream connector (верификация)
- `external/04-redis-stream.md` — RedisStreamService в common

## Full presigned flow

```
1. Client → RequestUploadUrl → PresignedProvider.requestUploadUrl()
   → { key, url, fields }
2. Client → POST url (MinIO) — прямая загрузка (байты не через наши сервисы)
3. MinIO → Redis Stream → ConnectorsService.handleStorageEvent()
   → headObject → sniff → tag → BullMQ
4. Client → GetFileMetadata → { status: CONFIRMED | PENDING }
```
