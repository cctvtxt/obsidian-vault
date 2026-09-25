# Media-service 04 — Core service (media.service, controller, constants, utils)

> **Что это:** центральная логика и вспомогательные файлы.
> **Подчиняется:** `03-media-service-structure.md` → root + utils + interfaces

## media.service.ts

Бизнес-логика без зависимости от S3 raw SDK и gRPC-транспорта.

```typescript
export interface IMediaServiceController { ... } // контракт из proto

@Injectable()
export class MediaService implements IMediaServiceController {
    constructor(
        @Inject(I_S3_CLIENT) private readonly s3: IS3ClientService,
        @Inject(I_S3_BUCKET) private readonly bucket: string,
        @Inject(I_FILE_HANDLER) private readonly fileHandler: IFileHandler,
    ) {}

    // ===== UploadFile (direct) =====
    async uploadFile(req: UploadFileRequest): Promise<UploadFileResponse> {
        this.assertPurpose(req.purpose);
        const { key } = await this.fileHandler.handleUpload({
            purpose: req.purpose,
            ownerAccountId: req.ownerAccountId,
            file: { buffer: Buffer.from(req.fileData), mimetype: req.filename ? undefined : undefined },
            filename: req.filename,
            metadata: req.metadata,
        });
        return { key };
    }

    // ===== RequestUploadUrl (presigned) =====
    async requestUploadUrl(req: RequestUploadUrlRequest): Promise<RequestUploadUrlResponse> {
        this.assertPurpose(req.purpose);
        const result = await this.fileHandler.createUpload({
            purpose: req.purpose,
            ownerAccountId: req.ownerAccountId,
            filename: req.filename,
            metadata: req.metadata,
        });
        return { key: result.key, url: result.url!, fields: result.fields!, expiresAt: result.expiresAt! };
    }

    // ===== GetFileMetadata =====
    async getFileMetadata(req: GetFileMetadataRequest): Promise<GetFileMetadataResponse> {
        this.assertOwnership(req.key, req.ownerAccountId);
        const head = await this.s3.headObject(this.bucket, req.key);
        const tags = await this.s3.getObjectTags(this.bucket, req.key);
        const parsed = parseObjectKey(req.key);
        return {
            fileMetadata: {
                key: req.key,
                ownerAccountId: parsed!.ownerAccountId,
                mimeType: head.ContentType ?? '',
                sizeBytes: head.ContentLength ?? 0,
                originalFilename: head.Metadata?.['original-filename'],
                status: this.mapStatusTag(tags[STATUS_TAG]),
                createdAt: head.LastModified?.toISOString() ?? '',
            },
        };
    }

    // ===== GetDownloadUrl =====
    async getDownloadUrl(req: GetDownloadUrlRequest): Promise<GetDownloadUrlResponse> {
        this.assertOwnership(req.key, req.ownerAccountId);
        const ttl = this.clampTtl(req.expiresInSeconds ?? mediaConfig.downloadUrlTtl);
        const url = await this.s3.getSignedDownloadUrl(this.bucket, req.key, ttl);
        return { url };   // expiresAt убран из proto
    }

    // ===== DeleteFile =====
    async deleteFile(req: DeleteFileRequest): Promise<void> {
        this.assertOwnership(req.key, req.ownerAccountId);
        await this.s3.removeFile(this.bucket, { path: req.key });
    }

    // ===== ListFiles =====
    async listFiles(req: ListFilesRequest): Promise<ListFilesResponse> {
        const page = req.page > 0 ? req.page : DEFAULT_PAGE;
        const limit = req.limit > 0 ? Math.min(req.limit, MAX_LIMIT) : DEFAULT_LIMIT;
        const collected: ListedObject[] = [];
        for (const name of getPurposeNames()) {
            const prefix = `${getPurposeProfile(name).prefix}/${req.ownerAccountId}/`;
            collected.push(...(await this.s3.listObjects(this.bucket, prefix)));
        }
        collected.sort((a, b) => (b.lastModified?.getTime() ?? 0) - (a.lastModified?.getTime() ?? 0));
        const start = (page - 1) * limit;
        return {
            total: collected.length,
            page,
            limit,
            files: collected.slice(start, start + limit).map((o) => this.toMetadata(o)),
        };
    }
}
```

## media.controller.ts

gRPC handler, thin — только делегирует.

```typescript
@Controller()
export class MediaController implements IMediaServiceController {
    constructor(private readonly mediaService: MediaService) {}

    @GrpcMethod('MediaService', 'UploadFile')
    uploadFile(req: UploadFileRequest) { return this.mediaService.uploadFile(req); }

    @GrpcMethod('MediaService', 'RequestUploadUrl')
    requestUploadUrl(req: RequestUploadUrlRequest) { return this.mediaService.requestUploadUrl(req); }

    // ... остальные thin делегирования
}
```

## media.constants.ts

```typescript
export const MEDIA_EVENTS_QUEUE = 'media-events';
export const UPLOAD_COMPLETED_JOB = 'upload.completed';
export const UPLOAD_REJECTED_JOB = 'upload.rejected';
export const STATUS_TAG = 'status';
export const STATUS_TAG_VALUES = { CONFIRMED: 'confirmed', REJECTED: 'rejected' } as const;
export const DEFAULT_PAGE = 1;
export const DEFAULT_LIMIT = 20;
export const MAX_LIMIT = 100;
export const HEALTH_STATUS_SERVING = 1;

// Redis Stream (для connectors)
export const MINIO_STREAM = 'minio-events';
export const CONSUMER_GROUP = 'media-verify';
export const CONSUMER_NAME = 'media-consumer-1';

// S3 метаданные
export const SIGNATURE_SNIFF_BYTES = 512;
export const MIN_URL_TTL_SECONDS = 60;
export const LIST_MAX_KEYS = 1000;
```

## utils/

### `utils/mime-sniffer.ts`

Перенос из `sniffing/sniff-mime.ts` (без изменений, те же magic bytes).

### `utils/object-key.ts`

Перенос из части `purpose/purpose-profiles.ts`:

```typescript
export interface ParsedObjectKey { prefix: string; ownerAccountId: string; fileId: string; }
export const parseObjectKey = (key: string): ParsedObjectKey | null => ...;
export const buildObjectKey = (profile, owner, fileId): string => ...;
export const ownsObjectKey = (key, owner): boolean => ...;
export const findPurposeByPrefix = (prefix): PurposeName | undefined => ...;
```

### `utils/index.ts`

```typescript
export { sniffMime } from './mime-sniffer';
export { parseObjectKey, buildObjectKey, ownsObjectKey, findPurposeByPrefix } from './object-key';
```

## media.types.ts (опционально, если нужны PurposeProfile)

`PurposeProfile` + `PURPOSE_PROFILES` + `isPurpose` + `getPurposeProfile` — остаются в `media.types.ts` (не в utils, т.к. это бизнес-данные, не чистая утилита).

---

## Что связывает всё

```
media.controller.ts → media.service.ts → IFileHandler (provider)
                                         → IS3ClientService (common)
                                         → IMediaEventsProducer (BullMQ → gateway)
media.service.ts   → utils (sniff, object-key)
media.module.ts    → собирает: ConfigModule, S3ClientModule, RedisModule,
                     один provider module, ConnectorsModule
```
