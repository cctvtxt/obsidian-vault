# Media-service 01 — Структура модуля

> **Что это:** физическое расположение файлов media-service после рефакторинга.
> **Подчиняется:** `03-media-service-structure.md`

## Структура файлов

```
common/configs/
├── media.config.ts          ← перенос из media-service/src/configs
└── media.scheme.ts          ← перенос из media-service/src/configs

media-service/src/
├── interfaces/
│   ├── s3-client.interface.ts        # IS3ClientService (контракт к s3 модулю)
│   ├── file-handler.interface.ts     # IFileHandler
│   ├── media-service.interface.ts    # IMediaServiceController
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
│       ├── connectors.types.ts
│       └── index.ts
├── media.module.ts
├── media.controller.ts
├── media.service.ts
├── media.constants.ts
└── main.ts
```

## Удаляем из текущей структуры

```
media-service/src/configs/      → перенос в common/configs (конфиги общие)
media-service/src/webhook/      → удалить (заменён connectors через Redis Stream)
media-service/src/queue/        → перенос логики в providers (producer)
media-service/src/sniffing/     → перенос в utils/mime-sniffer.ts
media-service/src/purpose/      → перенос в utils/object-key.ts + media.types.ts
media-service/src/media-service.*  → переименовать в media.* (service/controller/module/constants)
```

## Логика размещения

| Папка | Содержит | Почему |
|-------|----------|--------|
| `interfaces/` | Контракты к внешним модулям (S3), провайдерам (IFileHandler), транспорт (IMediaServiceController) | Все "какие структуры нам нужны" в одном месте |
| `utils/` | Чистые функции без DI (sniff-mime, object-key) | Переиспользуются обоими провайдерами и connectors |
| `providers/` | Реализации стратегий загрузки (direct / presigned) | media.module выбирает одну из двух |
| `modules/` | Подключаемые модули (connectors) | Изолированные функциональные блоки |

## media.module.ts — точка сборки

```typescript
@Module({
    imports: [
        ConfigModule.forRoot(grpcSchema, redisSchema, mediaSchema),
        S3ClientModule.forRootAsync({ ... }),
        RedisModule.forRoot(redisConfig),
        // выбираем ОДИН провайдер:
        DirectFileHandlerModule,
        // или PresignedFileHandlerModule (+ ConnectorsModule внутри)
    ],
    controllers: [MediaController],
    providers: [MediaService, ...],
})
export class MediaModule {}
```

## Переименование файлов

| Было | Стало |
|------|-------|
| `media-service.service.ts` | `media.service.ts` |
| `media-service.controller.ts` | `media.controller.ts` |
| `media-service.module.ts` | `media.module.ts` |
| `media-service.constants.ts` | `media.constants.ts` |

**Имя файла = имя класса** (`MediaService`, `MediaController`, `MediaModule`).
