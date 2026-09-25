# Media-service 02 — Провайдеры (direct + presigned)

> **Что это:** две реализации `IFileHandler` — стратегии загрузки.
> **Подчиняется:** `03-media-service-structure.md` → providers/

## Интерфейс IFileHandler

### `interfaces/file-handler.interface.ts`

```typescript
export interface IFileHandler {
    /** Прямая загрузка файла (bytes). Возвращает key. */
    handleUpload(params: FileHandlerUploadParams): Promise<FileHandlerResult>;

    /** Presigned: генерация URL. Опционально, только для presigned. */
    createUpload(params: FileHandlerUploadParams): Promise<FileHandlerResult>;
}

export interface FileHandlerUploadParams {
    purpose: string;
    ownerAccountId: string;
    file?: { buffer: Buffer; mimetype?: string };  // direct mode
    filename?: string;
    metadata?: Record<string, string>;
}

export interface FileHandlerResult {
    key: string;
    url?: string;                    // presigned POST URL (presigned mode)
    fields?: Record<string, string>; // form fields (presigned mode)
    expiresAt?: string;              // ISO string
}
```

**Замечание:** прямой (direct) и presigned — оба возвращают `FileHandlerResult`. Direct возвращает только `{ key }`, presigned — `{ key, url, fields, expiresAt }`. MediaService по RPC (UploadFile / RequestUploadUrl) решает какие поля отдать потребителю.

---

## Direct провайдер

### `providers/direct-file-handler/direct.service.ts`

```typescript
@Injectable()
export class DirectFileHandler implements IFileHandler {
    constructor(
        @Inject(I_S3_CLIENT) private readonly s3: IS3ClientService,
        @Inject(S3_BUCKET) private readonly bucket: string,
        @Inject(I_MEDIA_EVENTS_PRODUCER) private readonly producer: IMediaEventsProducer,
    ) {}

    async handleUpload(params: FileHandlerUploadParams): Promise<FileHandlerResult> {
        const buffer = params.file!.buffer;
        const mime = sniffMime(buffer.subarray(0, 512));
        const profile = getPurposeProfile(params.purpose);

        const confirmed = mime !== null && profile.allowedMimes.includes(mime);
        const key = buildObjectKey(profile, params.ownerAccountId, randomUUID());

        await this.s3.uploadFile(this.bucket, {
            file: { buffer, mimetype: mime ?? undefined },
            folder: profile.prefix,
            name: `${params.ownerAccountId}/${fileIdFromKey(key)}`,
        });

        await this.s3.putObjectTagging(this.bucket, key, {
            [STATUS_TAG]: confirmed ? STATUS_TAG_VALUES.CONFIRMED : STATUS_TAG_VALUES.REJECTED,
        });

        if (confirmed) {
            await this.producer.emitUploadCompleted(key);
        } else {
            await this.producer.emitUploadRejected(key, mime ?? 'unrecognized');
        }

        return { key };
    }
}
```

### Flow

```
Client → UploadFile (bytes) → MediaService → DirectFileHandler.handleUpload()
    → sniffMime → validate → uploadFile → putObjectTagging → BullMQ
    → return { key }
```

**Нет PENDING.** Файл сразу confirmed/rejected.

---

## Presigned провайдер

### `providers/presigned-file-handler/presigned.service.ts`

```typescript
@Injectable()
export class PresignedFileHandler implements IFileHandler {
    constructor(
        @Inject(I_S3_CLIENT) private readonly s3: IS3ClientService,
        @Inject(S3_BUCKET) private readonly bucket: string,
    ) {}

    async createUpload(params: FileHandlerUploadParams): Promise<FileHandlerResult> {
        const profile = getPurposeProfile(params.purpose);
        const key = buildObjectKey(profile, params.ownerAccountId, randomUUID());

        const presigned = await createPresignedPost(this.s3, {
            Bucket: this.bucket,
            Key: key,
            Expires: 900,
            Conditions: [
                ['content-length-range', 1, profile.maxSizeBytes],
                ['starts-with', '$Content-Type', getAllowedMimesNamespace(profile.allowedMimes)],
            ],
        });

        return {
            key,
            url: presigned.url,
            fields: presigned.fields,
            expiresAt: new Date(Date.now() + 900 * 1000).toISOString(),
        };
    }
}
```

### Flow

```
Client → RequestUploadUrl → MediaService → PresignedFileHandler.createUpload()
    → createPresignedPost → return { key, url, fields, expiresAt }
```

Верификация — отдельно, через connectors (external/03 + media-service/03).

---

## Регистрация в media.module — НЕ factory

Провайдер выбирается **явно**, подставляется при сборке. Без `uploadMode` конфига и без Factory.

```typescript
// media.module.ts — выбор DIRECT
@Module({
    imports: [DirectFileHandlerModule],
    providers: [MediaService],
})
```

```typescript
// media.module.ts — выбор PRESIGNED
@Module({
    imports: [PresignedFileHandlerModule],  // внутри — ConnectorsModule + BullMQ queue
    providers: [MediaService],
})
```

**Смена режима = правка media.module.ts** (одна строка импорта).

---

## Итог

| | Direct | Presigned |
|--|--------|-----------|
| Метод IFileHandler | `handleUpload()` | `createUpload()` |
| Возвращает | `{ key }` | `{ key, url, fields, expiresAt }` |
| Верификация | On the spot | Async via connectors |
| Модуль | `DirectFileHandlerModule` | `PresignedFileHandlerModule` |
| Токен | `I_MEDIA_EVENTS_PRODUCER` + others | без producer (верификация не тут) |
