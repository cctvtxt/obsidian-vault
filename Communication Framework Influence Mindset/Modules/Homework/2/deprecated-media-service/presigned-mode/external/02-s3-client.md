# Presigned Mode — External 02: Требования к common/modules/s3-client (presigned)

> **Область:** `common/modules/s3-client`.
> **Подчиняется:** `presigned-mode/index.md`, поверх `direct-mode/external/03-s3-client.md`.

## Требование: presigned POST генерация

Presigned-режим выдает клиенту URL для загрузки файла напрямую в MinIO. нужен метод в `S3ClientService`:

### `createPreSignedPostUpload` (или `createPresignedUpload`)

```typescript
interface PresignedUploadResult {
    key: string;
    url: string;
    fields: Record<string,string>;
    expiresAt: string;
}

interface PresignedUploadOptions {
    key: string;
    expiresInSeconds: number;
    maxSizeBytes?: number;        // → content-length-range
    allowedMimeTypes?: readonly string[];  // → starts-with $Content-Type
    metadata?: Record<string,string>;      // → x-amz-meta-*
}

// в S3ClientService
async createPreSignedPostUpload(
    bucket: string,
    opts: PresignedUploadOptions,
): Promise<PresignedUploadResult>;
```

Инкапсулирует `@aws-sdk/s3-presigned-post` → `createPresignedPost`. Media-service не знает про presigner/поля формы.

## Что уже есть (сумма требований к s3-client)

| Метод | direct (шаг1/2) | presigned (шаг3) |
|-------|-----------------|------------------|
| `createMultipartUpload(bucket, key, opts)` → session | ✅ | — |
| `uploadPart`/`complete`/`abort` (в session) | ✅ | — |
| `uploadFile(bucket, payload)` (single) | ✅ малые | — |
| `removeFile` | ✅ | ✅ |
| `createPreSignedPostUpload(bucket, opts)` → {url, fields} | — | ✅ |
| `getSignedDownloadUrl` | ✅ (download) | ✅ |
| `headObject` / `getObject` (range) | ✅ (metadata/verify) | ✅ (verify) |
| `listObjects` | ✅ | ✅ |
| `putObjectTagging` / `getObjectTags` | ✅ | ✅ (verify status) |

## Принцип

media-service работает только через `IS3ClientService`. Никаких raw `S3Client`, `createPresignedPost` прямых импортов. Весь AWS SDK спрятан в s3-client.

## Зависимость connectors

Верификация после presigned загрузки использует `headObject`, `getObject` (range), `putObjectTagging` — всё уже в s3-client. Connector-модуль (см. top-level `external/03-connector-interface.md`) использует `IS3ClientService` для этих операций.

## Redis Stream (для connectors)

Поток MinIO → Redis. Требования к `common/modules/redis` (RedisStreamService: ensureGroup/readGroup/ack) — см. top-level `external/04-redis-stream-module.md`.
