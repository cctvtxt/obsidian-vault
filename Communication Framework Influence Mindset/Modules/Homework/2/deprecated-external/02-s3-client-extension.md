# Расширение s3-client модуля

## Текущее состояние

`s3-client.service.ts` имеет 2 метода:

```typescript
async uploadFile(bucket: string, payload: UploadFilePayload): Promise<string> { ... }
async removeFile(bucket: string, payload: RemoveFilePayload): Promise<void> { ... }
```

## Проблема

`media-service` работает с raw `S3_CLIENT` для:
- `HeadObjectCommand` — получение метаданных файла
- `ListObjectsV2Command` — список файлов с пагинацией
- `PutObjectTaggingCommand` — установка тегов (статус верификации)
- `GetObjectCommand` — чтение первых байт для определения MIME
- `getSignedUrl` (presigner) — генерация presigned URL для скачивания

**MediaService инжектит 3 разных S3-объекта:**
- `S3ClientService` (для upload/remove)
- `S3_CLIENT` (raw SDK для head/list/tag/get)
- `S3_CLIENT_OPTIONS` (для bucket)

Это смешение абстракций. Весь S3-доступ должен быть через один сервис.

## Решение

Расширить `S3ClientService` методами, которые нужны media-service (и потенциально другим потребителям).

## Новые методы

### headObject

```typescript
async headObject(bucket: string, key: string): Promise<HeadObjectOutput> {
    this.logger.log(`HeadObject: "${bucket}/${key}"`);

    try {
        return await this.client.send(
            new HeadObjectCommand({ Bucket: bucket, Key: key }),
        );
    } catch (error) {
        this.logger.error(`HeadObject failed for "${bucket}/${key}"`, error);
        throw error;
    }
}
```

**Зачем:** Получение метаданных файла (ContentType, ContentLength, LastModified, Metadata) без скачивания всего файла. Используется в:
- `getFileMetadata` — возвращает метаданные consumer
- `webhook.service.ts` — проверка что файл существует после загрузки

### listObjects

```typescript
async listObjects(bucket: string, prefix: string): Promise<ListedObject[]> {
    this.logger.log(`ListObjects: "${bucket}/${prefix}"`);

    const collected: ListedObject[] = [];
    let continuationToken: string | undefined;

    try {
        do {
            const response = await this.client.send(
                new ListObjectsV2Command({
                    Bucket: bucket,
                    Prefix: prefix,
                    MaxKeys: 1000,
                    ContinuationToken: continuationToken,
                }),
            );

            for (const entry of response.Contents ?? []) {
                if (!entry.Key) continue;
                collected.push({
                    key: entry.Key,
                    sizeBytes: entry.Size ?? 0,
                    lastModified: entry.LastModified,
                });
            }

            continuationToken = response.IsTruncated
                ? response.NextContinuationToken
                : undefined;
        } while (continuationToken);
    } catch (error) {
        this.logger.error(`ListObjects failed for "${bucket}/${prefix}"`, error);
        throw error;
    }

    return collected;
}
```

**Зачем:** Получение списка файлов с автоматической пагинацией. S3 возвращает максимум 1000 объектов за раз, метод обрабатывает `ContinuationToken` автоматически. Используется в:
- `listFiles` — возвращает paginated list consumer

### putObjectTagging

```typescript
async putObjectTagging(bucket: string, key: string, tags: Record<string, string>): Promise<void> {
    this.logger.log(`PutObjectTagging: "${bucket}/${key}"`);

    try {
        await this.client.send(
            new PutObjectTaggingCommand({
                Bucket: bucket,
                Key: key,
                Tagging: {
                    TagSet: Object.entries(tags).map(([Key, Value]) => ({ Key, Value })),
                },
            }),
        );
    } catch (error) {
        this.logger.error(`PutObjectTagging failed for "${bucket}/${key}"`, error);
        throw error;
    }
}
```

**Зачем:** Установка/обновление тегов на объекте. Теги — единственный способ хранить статус верификации в S3 (PENDING/CONFIRMED/REJECTED). Используется в:
- `media.service.ts` — помечает файл как confirmed/rejected после верификации
- `webhook.service.ts` — аналогично при webhook verification

### getObjectTags

```typescript
async getObjectTags(bucket: string, key: string): Promise<Record<string, string>> {
    this.logger.log(`GetObjectTags: "${bucket}/${key}"`);

    try {
        const response = await this.client.send(
            new GetObjectTaggingCommand({ Bucket: bucket, Key: key }),
        );

        const tags: Record<string, string> = {};
        for (const tag of response.TagSet ?? []) {
            if (tag.Key && tag.Value) tags[tag.Key] = tag.Value;
        }
        return tags;
    } catch (error) {
        this.logger.error(`GetObjectTags failed for "${bucket}/${key}"`, error);
        throw error;
    }
}
```

**Зачем:** Чтение тегов объекта. Используется в:
- `getFileMetadata` — определение статуса файла из тега `status`

### getObject

```typescript
async getObject(bucket: string, key: string, range?: string): Promise<GetObjectOutput> {
    this.logger.log(`GetObject: "${bucket}/${key}"${range ? ` range=${range}` : ''}`);

    try {
        return await this.client.send(
            new GetObjectCommand({
                Bucket: bucket,
                Key: key,
                ...(range ? { Range: range } : {}),
            }),
        );
    } catch (error) {
        this.logger.error(`GetObject failed for "${bucket}/${key}"`, error);
        throw error;
    }
}
```

**Зачем:** Чтение содержимого объекта (или его части через Range). Используется в:
- `webhook.service.ts` — чтение первых 512 байт для определения MIME (sniff)

### getSignedDownloadUrl

```typescript
async getSignedDownloadUrl(bucket: string, key: string, expiresIn: number): Promise<string> {
    this.logger.log(`GetSignedUrl: "${bucket}/${key}" expiresIn=${String(expiresIn)}`);

    try {
        return await getSignedUrl(
            this.client,
            new GetObjectCommand({ Bucket: bucket, Key: key }),
            { expiresIn },
        );
    } catch (error) {
        this.logger.error(`GetSignedUrl failed for "${bucket}/${key}"`, error);
        throw error;
    }
}
```

**Зачем:** Генерация presigned URL для скачивания файла. URL действителен ограниченное время. Используется в:
- `getDownloadUrl` — возвращает URL consumer для скачивания файла

## Новые интерфейсы

### В `interfaces/`

```typescript
// interfaces/listed-object.interface.ts
export interface ListedObject {
    key: string;
    sizeBytes: number;
    lastModified?: Date;
}
```

```typescript
// interfaces/get-object-options.interface.ts
export interface GetObjectOptions {
    range?: string;  // e.g. "bytes=0-511"
}
```

## Итог

### Было

```
MediaService
├── S3ClientService          → uploadFile(), removeFile()
├── S3_CLIENT (raw SDK)      → HeadObjectCommand, ListObjectsV2Command,
│                              PutObjectTaggingCommand, GetObjectCommand,
│                              GetObjectTaggingCommand
├── S3_CLIENT_OPTIONS        → options.bucket
└── getSignedUrl (presigner) → GetDownloadUrl
```

**3 источника S3 + 1 прямой import AWS SDK.**

### Стало

```
MediaService
└── S3ClientService
    ├── uploadFile()
    ├── removeFile()
    ├── headObject()
    ├── listObjects()
    ├── putObjectTagging()
    ├── getObjectTags()
    ├── getObject()
    └── getSignedDownloadUrl()
```

**1 источник S3.** MediaService не знает про `S3Client`, `PutObjectCommand` и прочие AWS SDK типы.

## Файлы для изменения

| Файл | Действие |
|------|----------|
| `common/modules/s3-client/s3-client.service.ts` | Добавить 6 методов |
| `common/modules/s3-client/interfaces/listed-object.interface.ts` | Создать |
| `common/modules/s3-client/interfaces/get-object-options.interface.ts` | Создать |
| `common/modules/s3-client/index.ts` | Добавить экспорт новых типов |

## Импорты

В `s3-client.service.ts` добавить:

```typescript
import {
    S3Client,
    PutObjectCommand,
    DeleteObjectCommand,
    HeadObjectCommand,          // ← новый
    ListObjectsV2Command,       // ← новый
    PutObjectTaggingCommand,    // ← новый
    GetObjectTaggingCommand,    // ← новый
    GetObjectCommand,           // ← новый
} from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';  // ← новый
```

## Важно

- Все новые методы работают **только с raw S3_CLIENT** (как и uploadFile/removeFile)
- `bucket` — required параметр (как в текущих методах)
- Ошибки оборачиваются в BadRequestException (единообразно с текущими методами)
- Logger используется для всех операций (trace/debug level)
