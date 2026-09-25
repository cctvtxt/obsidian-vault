# Direct Mode — External 02: Генерация типов из proto

> **Область:** `common/contracts/generated/media.ts`.
> **Подчиняется:** `direct-mode/index.md` (шаг 1), `direct-mode/external/01-media-proto.md`.

## Что

Сгенерировать TS-типы из нового proto (client streaming `UploadFile`) в `common/contracts/generated/media.ts`.

## Типы (сгенерированные)

- `MediaServiceController` interface (RPC-методы: `uploadFile`, `getFileMetadata`, `getDownloadUrl`, `deleteFile`, `listFiles`)
- `UploadChunk`, `UploadMeta`, `UploadFileResponse`
- `FileMetadata`, `FileStatus`
- `GetFileMetadataRequest`/`Response`, `GetDownloadUrlRequest`/`Response`, `DeleteFileRequest`, `ListFilesRequest`/`Response`

## Примечание для client streaming

Контракт `MediaServiceController.uploadFile` при client streaming не совпадает с unary. В NestJS-gRPC client streaming обрабатывается через `@GrpcStreamMethod()` и `Observable<UploadChunk>` / async iterator:

```typescript
// генерируется контракт RPC (unary-стиль для controller interface)
interface MediaServiceController {
    uploadFile(request: Observable<UploadChunk> | UploadChunk): void;
    ...
}
```

**На шаге 1** media-service реализует `uploadFile` как streaming-обработчик (не через типичный unary), поэтому сгенерированный интерфейс — ориентир, фактический streaming-handler пишется вручную в `upload-stream.controller.ts`. Прото-типы (`UploadChunk`, `UploadMeta`) — обязательны для валидной генерации.

## Зависимость

Это самый узкий блокер: без `generated/media.ts` media-service не компилируется. Выполняется сразу после обновления proto.
