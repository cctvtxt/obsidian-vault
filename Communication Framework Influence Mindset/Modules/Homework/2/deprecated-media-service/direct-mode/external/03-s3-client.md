# Direct Mode — External 03: Требования к common/modules/s3-client

> **Область:** `common/modules/s3-client`.
> **Подчиняется:** `direct-mode/index.md` (шаг 1), `external/02-s3-client-extension.md` (базовое расширение).

## Что нужно для client streaming (multipart upload)

Direct-режим принимает файл чанками через gRPC stream. Чтобы не держать весь файл в памяти перед загрузкой в S3, нужен **multipart upload**: чанки пишутся напрямую в S3 по мере прихода.

### Новый метод: `uploadMultipart` (или `createMultipartUpload` / `uploadPart` / `complete` / `abort`)

Поток чанков требует управление multipart-сессией. Варианты API:

**Вариант A (инкапсулированный, рекомендуется):**
```typescript
interface IMultipartUploadSession {
    uploadPart(data: Buffer): Promise<void>;
    complete(): Promise<void>;
    abort(): Promise<void>;
}

// в S3ClientService
createMultipartUpload(bucket: string, key: string, opts: {
    contentType?: string;
}): Promise<IMultipartUploadSession>;
```

`S3ClientService` инкапсулирует `CreateMultipartUploadCommand` / `UploadPartCommand` / `CompleteMultipartUploadCommand` / `AbortMultipartUploadCommand`. Media-service просто вызывает `session.uploadPart(chunk)` по мере приёма чанков из gRPC.

**Вариант B (низкоуровневые методы):**
```typescript
createMultipartUpload(bucket, key): Promise<string>;      // → uploadId
uploadPart(bucket, key, uploadId, partNumber, data): Promise<string>; // → ETag
completeMultipartUpload(bucket, key, uploadId, parts): Promise<void>;
abortMultipartUpload(bucket, key, uploadId): Promise<void>;
```

Вариант B — больше контроля, но media-service должен хранить части/ETag'и и собирать complete. Для шага 1 достаточно **Варианта A** (инкапсулированный session), т.к. backpressure и буфер — на стороне media-service, а low-level S3 multipart — внутри s3-client.

### Требование: не буферизовать весь файл

`S3ClientService` должен писать каждый чанк в S3 **сразу по приходу** (multipart part), не накапливая все части в памяти. Т.е. `uploadPart` должен синхронно (или с малым in-flight) отправлять чанк в S3.

---

## Концепция backpressure на уровне s3-client

Чтобы не накапливать чанки в памяти между gRPC-stream и S3:

- `uploadPart` возвращает Promise, который резолвится после отправки части в S3
- Media-service `await`'ит каждый `uploadPart` → клиент gRPC не шлёт следующий чанк, пока S3 не принял текущий
- Пул in-flight частей ограничен (`STREAM_MAX_PENDING_CHUNKS`) — часть отправляется в S3, остальные буферизуются в лимите

### Параметры (в s3-client)

```typescript
interface MultipartOptions {
    minPartSizeBytes?: number;   // например 5MB (min S3 part)
    maxConcurrentParts?: number; // максимальное число in-flight частей
}
```

**Замечание:** минимальный размер части S3 = 5MB (кроме последней). Для маленьких файлов single-upload (не multipart). s3-client решает: если файл ≤ лимиту → один `PutObjectCommand`; если больше → multipart. Media-service не должен выбирать.

---

## Итоговый набор методов s3-client для direct-mode

| Метод | Цель |
|-------|------|
| `createMultipartUpload(bucket, key, opts)` → session | прямой чанкованный upload (client streaming) |
| (внутри session) `uploadPart` / `complete` / `abort` | multipart |
| `uploadFile(bucket, payload)` → key | **уже есть** — для малых файлов (single Put) |
| `removeFile(bucket, payload)` | **уже есть** |
| `headObject`, `listObjects`, `putObjectTagging`, `getObjectTags`, `getObject`, `getSignedDownloadUrl` | **уже в external/02** (для metadata/download/list) |

## Примечание

`createMultipartUpload` возвращает **инкапсулированную сессию** (Вариант A) — по умолчанию. Низкоуровневые части не экспортируются за пределы s3-client. Это сохраняет принцип «media-service не знает про AWS SDK-типы».
