# Direct Mode — Шаг 2: interfaces + providers/direct-mode

> **Статус:** после успешного шага 1 (client streaming работает напрямую).
> **Цель:** ввести провайдеры — сначала `providers/direct-mode`, подключить через `IPROVIDER`-механизм.

## Что меняется

На шаге 1 загрузка была напрямую: `upload-stream.controller → uploader.service → S3 multipart`.

Шаг 2 — вводим **provider abstraction**: media-service больше не знает деталей загрузки, работает через интерфейс `IUploadProvider` (файл-хендлер). Провайдер — это стратегия (direct это или presigned).

## Новая структура

```
media-service/src/
├── interfaces/
│   ├── s3-client.interface.ts          # IS3ClientService
│   ├── file-handler.interface.ts       # IFileHandler (upload/verify)
│   ├── upload-stream.interface.ts      # IUploadStreamService (оркестратор чанков)
│   ├── media-events.interface.ts       # IMediaEventsProducer
│   └── index.ts                        # токены IS3_CLIENT, I_FILE_HANDLER, ...
├── utils/
│   ├── mime-sniffer.ts
│   ├── object-key.ts
│   └── index.ts
├── modules/
│   └── uploader/                      # из шага 1 (без изменений)
│       ├── uploader.module.ts
│       ├── uploader.service.ts        # multipart orchestration (IUploadStreamService)
│       └── upload-stream.controller.ts# gRPC client-streaming
├── providers/
│   └── direct-mode/
│       ├── direct.module.ts           # DirectProviderModule
│       ├── direct.provider.ts         # DirectFileHandler implements IFileHandler
│       └── index.ts
├── media.module.ts
├── media.controller.ts
├── media.service.ts
├── media.constants.ts
└── main.ts
```

**Ключевое:** uploader остаётся в `modules/uploader` (механика стрима S3-multipart). `providers/direct-mode` — только стратегия: выбирает/компонует uploader, реализует IFileHandler.

## IFileHandler — интерфейс загрузки

```typescript
export interface IFileHandler {
    uploadStream(): IUploadStreamService;   // для client streaming
}
```

**На шаге 2 настраиваем** провайдер direct: `DirectFileHandler` реализует потоковую загрузку (через `uploadStream` + s3 multipart).

## Подключение в media.module (НЕ factory)

```typescript
@Module({
    imports: [DirectProviderModule],   // подставляем провider
    controllers: [MediaController],
    providers: [MediaService, ...],
})
export class MediaModule {}
```

`DirectProviderModule` регистрирует `DirectFileHandler` как `I_FILE_HANDLER`. Смена на presigned (шаг 3) = поменять `DirectProviderModule` на `PresignedProviderModule` — одна строка.

## Модуль direct

`direct.module.ts` регистрирует `DirectFileHandler` как `I_FILE_HANDLER`, импортирует `UploaderModule` (для доступа к `IUploadStreamService`).

## Инкремент relative шага 1

- Добавлена папка `providers/direct-mode/`
- Добавлены interfaces: `file-handler.interface.ts`, `media-events.interface.ts`
- `modules/uploader/` остаётся на месте (механика стрима не меняется)
- `upload-stream.controller` переходит на `IFileHandler` (через DirectFileHandler)
- media.module подключает `DirectProviderModule` (импортирует `UploaderModule` внутри)

## Примечание

Ключ: **провайдер = стратегия загрузки**. Direct — стримит байты в S3 multipart (использует `modules/uploader`). Presigned — выдаёт URL (шаг 3). Интерфейс `IFileHandler` общий, а механика (uploader) переиспользуется.
