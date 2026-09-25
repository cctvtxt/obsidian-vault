# Presigned Mode — External 01: Proto добавляет RequestUploadUrl

> **Область:** `common/contracts/proto/media.proto`.
> **Подчиняется:** `presigned-mode/index.md`, поверх `direct-mode/external/01-media-proto.md`.

## Дополнение к proto (шаг 3)

К direct client-streaming RPC `UploadFile` добавляется presigned RPC:

```protobuf
service MediaService {
    // DIRECT (остаётся)
    rpc UploadFile (stream UploadChunk) returns (UploadFileResponse);

    // PRESIGNED (добавляется)
    rpc RequestUploadUrl (RequestUploadUrlRequest) returns (RequestUploadUrlResponse);

    // unary (остаются)
    rpc GetFileMetadata ...
    rpc GetDownloadUrl ...
    rpc DeleteFile ...
    rpc ListFiles ...
}

message RequestUploadUrlRequest {
    string purpose = 1;
    string ownerAccountId = 2;
    optional string filename = 3;
    map<string,string> metadata = 4;
    optional uint64 sizeBytes = 5;   // для presigned POST content-length-range
}

message RequestUploadUrlResponse {
    string key = 1;
    string url = 2;                    // presigned POST URL
    map<string,string> fields = 3;     // form fields
    string expiresAt = 4;
}
```

## Единый интерфейс vs два RPC

Вопрос был: «можно ли один unary, клиент шлёт одинаковые данные, режим решает конфиг».

**Вывод:** для direct используется **client streaming** `UploadFile` (не unary bytes). поэтому объединить нельзя — transport разный:
- direct = `stream<UploadChunk>`
- presigned = `unary{RequestUploadUrl}`

Оба RPC принимают **один и тот же набор метаданных** (`purpose/ownerAccountId/filename/metadata`) — клиентская семантика единая. Разница только в том, как доставляются байты (stream vs минуя сервис). Режим выбирается **провайдером в media module**, не протоколом — клиент не знает какой бэкенд подключён.

## Прохождение нагрузки (presigned)

```
Client → RequestUploadUrl (unary, метаданные)
Client → POST url (MinIO) — байты напрямую, НЕ через gateway/media
```

**Gateway НЕ блокируется байтами:** он проксирует только unary-запрос метаданных (малый). Байты идут клиент → MinIO напрямую. Presigned — единственный способ полностью разгрузить gateway/media от файлов.

## Backpressure (presigned)

- Нет буфера файла → нет проблемы backpressure по данным
- Ограничение: лимит на число выдаваемых URL (расход presigned), TTL URL, лимит размера через `sizeBytes`
- Верификация — асинхронная через connectors (Redis Stream)
