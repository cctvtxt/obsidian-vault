# Media-service 05 — Миграция (configs → common, удаление legacy)

> **Что это:** механика переноса и удаления. Выполняется ПОСЛЕ external модулей и новой сборки.
> **Подчиняется:** `03-media-service-structure.md`

## Перенос configs в common

### Было: `media-service/src/configs/`

```
media-service/src/configs/
├── media.config.ts
└── media.scheme.ts
```

### Стало: `common/configs/`

```
common/configs/
├── media.config.ts          ← перенесено
└── media.scheme.ts          ← перенесено
```

**Почему:** конфиги — общие, переиспользуются gateway и другими сервисами (как redis.config, grpc.config).

### Изменения в `common/configs/media.config.ts`

Добавить getters для Redis Stream (из media.constants):

```typescript
get minioStream()      { return process.env.MINIO_STREAM ?? 'minio-events'; }
get consumerGroup()    { return process.env.MEDIA_CONSUMER_GROUP ?? 'media-verify'; }
```

### Изменения в `common/configs/media.scheme.ts`

```typescript
MINIO_STREAM: Joi.string().default('minio-events'),
MEDIA_CONSUMER_GROUP: Joi.string().default('media-verify'),
```

### Обновить `common/configs/index.ts`

Добавить экспорт `media.config` + `media.scheme`.

---

## Удаление legacy файлов

После успешной новой сборки удалить:

```
media-service/src/configs/        # → common/configs
media-service/src/webhook/        # → modules/connectors (business logic)
media-service/src/queue/          # -> перенос MediaEventsProducer в providers/shared
media-service/src/sniffing/       # → utils/mime-sniffer.ts
media-service/src/purpose/        # → utils/object-key.ts + media.types.ts
media-service/src/media-service.service.ts       # → media.service.ts
media-service/src/media-service.controller.ts    # → media.controller.ts
media-service/src/media-service.module.ts        # → media.module.ts
media-service/src/media-service.constants.ts     # → media.constants.ts
media-service/src/media-service.service.spec.ts  # → media.service.spec.ts (обновить)
```

## Новая структура после миграции

```
media-service/src/
├── interfaces/
│   ├── s3-client.interface.ts
│   ├── file-handler.interface.ts
│   ├── media-service.interface.ts
│   └── index.ts
├── utils/
│   ├── mime-sniffer.ts
│   ├── object-key.ts
│   └── index.ts
├── providers/
│   ├── direct-file-handler/
│   │   ├── direct.module.ts
│   │   ├── direct.service.ts
│   │   └── index.ts
│   └── presigned-file-handler/
│       ├── presigned.module.ts
│       ├── presigned.service.ts
│       ├── interfaces/
│       │   └── index.ts
│       └── index.ts
├── modules/
│   └── connectors/
│       ├── connectors.module.ts
│       ├── connectors.service.ts
│       └── index.ts
├── media.module.ts
├── media.controller.ts
├── media.service.ts
├── media.constants.ts
└── main.ts
```

## Разнесение MediaEventsProducer

Было: `queue/media-events.producer.ts` в media-service, используется и direct- и presigned-провайдером.

**Как разместить:** буos независим — оба провайдера эмитят BullMQ события. Варианты:
1. Общий провайдер в `providers/` (если оба режима)
2. В `interfaces/media-events.interface.ts` — интерфейс, реализация `MediaEventsProducer` регистрируется в общем месте

**Рекомендация:** интерфейс `IMediaEventsProducer` в `interfaces/`, реализация регистрируется в `media.module.ts` (общая для обоих провайдеров). Провайдеры инжектят интерфейс.

```typescript
// media.module.ts
providers: [
    MediaService,
    MediaEventsProducer,   // implementation
    { provide: I_MEDIA_EVENTS_PRODUCER, useExisting: MediaEventsProducer },
    ...
]
```

---

## Порядок миграции (checklist)

- [ ] 1. Перенести `configs` → `common`, добавить stream-конфиги, обновить index.ts
- [ ] 2. Создать `interfaces/` (IS3ClientService, IFileHandler, IMediaServiceController, IMediaEventsProducer, index)
- [ ] 3. Создать `utils/` (mime-sniffer, object-key, index) — из sniffing/purpose
- [ ] 4. Создать `providers/direct-file-handler/`
- [ ] 5. Создать `providers/presigned-file-handler/`
- [ ] 6. Создать `modules/connectors/` (из webhook)
- [ ] 7. Создать `media.service.ts`, `media.controller.ts`, `media.constants.ts`, `media.module.ts`
- [ ] 8. Обновить `main.ts` (импорты MediaModule)
- [ ] 9. Удалить legacy (configs, webhook, queue, sniffing, purpose, media-service.*)
- [ ] 10. Запустить сборку + обновить/добавить spec
