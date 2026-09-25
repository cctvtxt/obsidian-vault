# Оптимизация media.proto

## Текущий proto (6 RPC)

```protobuf
service MediaService {
    rpc CreateUploadIntent (CreateUploadIntentRequest) returns (CreateUploadIntentResponse);
    rpc ConfirmUpload (ConfirmUploadRequest) returns (ConfirmUploadResponse);
    rpc GetFileMetadata (GetFileMetadataRequest) returns (GetFileMetadataResponse);
    rpc GetDownloadUrl (GetDownloadUrlRequest) returns (GetDownloadUrlResponse);
    rpc DeleteFile (DeleteFileRequest) returns (google.protobuf.Empty);
    rpc ListFiles (ListFilesRequest) returns (ListFilesResponse);
}
```

## Проблемы текущего proto

### 1. Proto диктует архитектуру загрузки

`CreateUploadIntent` жёстко привязан к presigned URL:
- Request содержит `purpose`, `ownerAccountId` — это нормально
- Response содержит `postUrl`, `fields` — это **реализация** (presigned POST URL + form fields)

Если захотеть перейти на direct upload (клиент отправляет файл через gRPC) — прото не поддерживает. Придётся переписывать.

### 2. ConfirmUpload — workflow-specific

`ConfirmUpload` существует только для presigned mode:
- Клиент загрузил файл в MinIO
- MinIO отправил webhook → верификация
- Клиент вызывает `ConfirmUpload` чтобы узнать статус

В direct upload `ConfirmUpload` не нужен — файл уже загружен и верифицирован.

### 3. FileStatus избыточен

`FileStatus` содержит 5 состояний: UNSPECIFIED, PENDING, CONFIRMED, REJECTED, DELETED.

- `DELETED` — не нужен: `DeleteFile` уже удаляет файл
- `UNSPECIFIED` — дефолт protobuf, не нужен в enum
- `PENDING` / `CONFIRMED` / `REJECTED` — нужны, но только для presigned mode

### 4. GetDownloadUrlResponse.expiresAt

`expiresAt` — implementation detail. Потребителю нужен URL, а не время истечения. Время истечения他知道 из TTL.

## Предлагаемый proto

```protobuf
syntax = "proto3";
package media.v1;

import "google/protobuf/empty.proto";

service MediaService {
    // Direct upload: клиент отправляет файл бинарником через gRPC
    rpc UploadFile (UploadFileRequest) returns (UploadFileResponse);

    // Presigned mode: генерация presigned URL для загрузки напрямую в S3
    rpc RequestUploadUrl (RequestUploadUrlRequest) returns (RequestUploadUrlResponse);

    // Метаданные файла (works for both modes)
    rpc GetFileMetadata (GetFileMetadataRequest) returns (GetFileMetadataResponse);

    // Presigned URL для скачивания
    rpc GetDownloadUrl (GetDownloadUrlRequest) returns (GetDownloadUrlResponse);

    // Удаление файла
    rpc DeleteFile (DeleteFileRequest) returns (google.protobuf.Empty);

    // Список файлов с пагинацией
    rpc ListFiles (ListFilesRequest) returns (ListFilesResponse);
}

enum FileStatus {
    FILE_STATUS_UNSPECIFIED = 0;
    FILE_STATUS_PENDING = 1;      // presigned: файл загружен, ожидает верификации
    FILE_STATUS_CONFIRMED = 2;    // файл верифицирован
    FILE_STATUS_REJECTED = 3;     // файл не прошёл верификацию
}

message FileMetadata {
    string key = 1;
    string ownerAccountId = 2;
    string mimeType = 3;
    uint64 sizeBytes = 4;
    optional string originalFilename = 5;
    FileStatus status = 6;
    string createdAt = 7;
}

// === Direct upload ===

message UploadFileRequest {
    string purpose = 1;
    string ownerAccountId = 2;
    bytes fileData = 3;
    optional string filename = 4;
    map<string, string> metadata = 5;
}

message UploadFileResponse {
    string key = 1;
}

// === Presigned mode ===

message RequestUploadUrlRequest {
    string purpose = 1;
    string ownerAccountId = 2;
    optional string filename = 3;
    map<string, string> metadata = 4;
}

message RequestUploadUrlResponse {
    string key = 1;
    string url = 2;                    // presigned POST URL
    map<string, string> fields = 3;    // form fields для presigned POST
    string expiresAt = 4;
}

// === Метаданные ===

message GetFileMetadataRequest {
    string key = 1;
    string ownerAccountId = 2;
}

message GetFileMetadataResponse {
    FileMetadata fileMetadata = 1;
}

// === Скачивание ===

message GetDownloadUrlRequest {
    string key = 1;
    string ownerAccountId = 2;
    optional uint32 expiresInSeconds = 3;
}

message GetDownloadUrlResponse {
    string url = 1;
}

// === Удаление ===

message DeleteFileRequest {
    string key = 1;
    string ownerAccountId = 2;
}

// === Список ===

message ListFilesRequest {
    optional string prefix = 1;
    uint32 page = 2;
    uint32 limit = 3;
}

message ListFilesResponse {
    repeated FileMetadata files = 1;
    uint32 total = 2;
    uint32 page = 3;
    uint32 limit = 4;
}
```

## Что изменилось

| Было | Стало | Зачем |
|------|-------|-------|
| `CreateUploadIntent` | `UploadFile` + `RequestUploadUrl` | Поддержка обоих режимов загрузки |
| `ConfirmUpload` | Удалён | Статус определяется через `GetFileMetadata` |
| `FileStatus.DELETED` | Удалён | `DeleteFile` уже удаляет, дублирование не нужно |
| `CreateUploadIntentResponse.postUrl + fields` | `RequestUploadUrlResponse.url + fields` | Переименовано для ясности |
| `GetDownloadUrlResponse.expiresAt` | Удалён | Implementation detail,consumer знает TTL |

## Преимущества нового proto

1. **Архитектурная гибкость**: `UploadFile` (direct) и `RequestUploadUrl` (presigned) — два разных RPC, конфиг выбирает какой использовать
2. **Нет привязки к implementation**: Proto описывает бизнес-операции, не как они реализованы
3. **Проще**: 6 RPC вместо 6 (то же количество), но без workflow-specific операций
4. **GetFileMetadata универсален**: Работает для обоих режимов, возвращает статус
5. **Убраны implementation details**: `expiresAt` в `GetDownloadUrlResponse` — consumer не должен знать об этом
