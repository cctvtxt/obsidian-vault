# Direct Mode — External 01: Новый media.proto (client streaming)

> **Область:** `common/contracts/proto/media.proto` + генерация.
> **Подчиняется:** `direct-mode/index.md` (шаг 1)

## Смена транспорта: unary `bytes` → client streaming

Было (в 03 главном плане) — unary `UploadFileRequest { bytes fileData }`.

**Проблема unary:** весь файл в память (RAM gateway + media), нет backpressure. Для больших файлов — антипаттерн.

**Решение (шаг 1):** client streaming `stream<UploadChunk>` — чанки, реальный backpressure, окно буфера вместо всего файла.

## Новый proto

```protobuf
syntax = "proto3";
package media.v1;

import "google/protobuf/empty.proto";

service MediaService {
    // DIRECT: клиент стримит файл чанками. Первый чанк — метаданные.
    rpc UploadFile (stream UploadChunk) returns (UploadFileResponse);

    // Унарные — metadata / download / list / delete (остаются unary)
    rpc GetFileMetadata (GetFileMetadataRequest) returns (GetFileMetadataResponse);
    rpc GetDownloadUrl (GetDownloadUrlRequest) returns (GetDownloadUrlResponse);
    rpc DeleteFile (DeleteFileRequest) returns (google.protobuf.Empty);
    rpc ListFiles (ListFilesRequest) returns (ListFilesResponse);
}

// === Direct upload (client streaming) ===

message UploadChunk {
    // Первый чанк: uploadId = "" (или генерируется сервером), несёт meta.
    // Последующие: uploadId = id сессии, несёт data.
    string uploadId = 1;
    bytes data = 2;
    optional UploadMeta meta = 3;    // только в первом чанке
}

message UploadMeta {
    string purpose = 1;
    string ownerAccountId = 2;
    string filename = 3;
    map<string,string> metadata = 4;
}

message UploadFileResponse {
    string key = 1;
}

// === Метаданные ===
message FileMetadata {
    string key = 1;
    string ownerAccountId = 2;
    string mimeType = 3;
    uint64 sizeBytes = 4;
    optional string originalFilename = 5;
    FileStatus status = 6;
    string createdAt = 7;
}

enum FileStatus {
    FILE_STATUS_UNSPECIFIED = 0;
    FILE_STATUS_PENDING = 1;
    FILE_STATUS_CONFIRMED = 2;
    FILE_STATUS_REJECTED = 3;
}

message GetFileMetadataRequest { string key = 1; string ownerAccountId = 2; }
message GetFileMetadataResponse { FileMetadata fileMetadata = 1; }

// === Скачивание / удаление / список ===
message GetDownloadUrlRequest { string key = 1; string ownerAccountId = 2; optional uint32 expiresInSeconds = 3; }
message GetDownloadUrlResponse { string url = 1; }

message DeleteFileRequest { string key = 1; string ownerAccountId = 2; }

message ListFilesRequest { optional string prefix = 1; uint32 page = 2; uint32 limit = 3; }
message ListFilesResponse { repeated FileMetadata files = 1; uint32 total = 2; uint32 page = 3; uint32 limit = 4; }
```

## Передача нагрузки и backpressure (комментарий в прото)

См. подробный разбор в `direct-mode/index.md` → «Прохождение нагрузки».

Кратко:
- Весь файл **не** в памяти. Сервер читает чанки по мере обработки → backpressure.
- Gateway **проксирует stream**: буфер = окно чанков (не файл), но worker и соединение заняты на всё время upload.
- Unary-методы (metadata/download/list/delete) остаются unary.
- Для полного освобождения gateway/медиа от байт — presigned (шаг 3).

## Что изменилось относительно прошлого proto (03)

| Было (03) | Стало (шаг 1) |
|-----------|---------------|
| `UploadFile` unary `bytes fileData` | `UploadFile` client streaming `stream<UploadChunk>` |
| `RequestUploadUrl` RPC | **нет на шаге 1** (добавится в presigned, шаг 3) |
| `FileStatus` | Без изменений (нужен PENDING для presigned позже) |
| `GetDownloadUrlResponse.expiresAt` | Убрано (implementation detail) |

## Backpressure — параметры (в proto комментариях + конфиг)

- `STREAM_MAX_PENDING_CHUNKS` — ограничение in-flight чанков
- `STREAM_MAX_CONCURRENCY` — макс параллельных сессий (→ `RESOURCE_EXHAUSTED`)
- `STREAM_IDLE_TIMEOUT_MS` — abort сессии без данных
- Лимит размера всего файла пер-провоider (purpose profile)

Эти params фиксируются в `media.constants.ts` / конфиге, не в proto (proto не должен нести реализацию).
