# Direct Mode — План реализации (ШАГ 1)

> **Что это:** первая итерация media-service — **только direct download через gRPC client streaming**.
> **Статус:** БЕЗ провайдеров. Прямая загрузка байтов через stream.
> **Подчиняется:** `03-media-service-structure.md`

## Итерации (phase)

- [x] **Шаг 1 (этот файл):** direct через client streaming. Новый proto, backpressure, артефакты. Без папки providers.
- [ ] **Шаг 2:** структура + interfaces + папка `providers/direct-mode`, подключение провайдера.
- [ ] **Шаг 3 (presigned-mode папка):** новый proto (presigned RPC), presigned провайдер, подстановка реализации через provider.

---

## Целевая структура (шаг 1 — direct, без providers)

```
media-service/src/
├── interfaces/
│   ├── s3-client.interface.ts
│   ├── upload-stream.interface.ts     # IUploadStreamService (обработка чанков)
│   └── index.ts
├── utils/
│   ├── mime-sniffer.ts
│   ├── object-key.ts
│   └── index.ts
├── modules/
│   └── uploader/
│       ├── uploader.module.ts         # UploaderModule (регистрирует контроллер+сервис)
│       ├── uploader.service.ts        # IUploadStreamService impl (multipart orchestration)
│       └── upload-stream.controller.ts# gRPC client-streaming handler
├── media.module.ts
├── media.controller.ts                # unary методы (metadata, download, list, delete)
├── media.service.ts
├── media.constants.ts
└── main.ts
```

**Ключевое: на шаге 1 НЕТ `providers/`.** Загрузка байтов напрямую в `modules/uploader/upload-stream.controller` → `uploader.service` → S3 client streaming/multipart. Uploader — автономный module (`UploaderModule`), импортируется в media.module.

---

## Client streaming gRPC

### proto (изменён — см. external/01 в этой папке)

```protobuf
rpc UploadFile (stream UploadChunk) returns (UploadFileResponse);

message UploadChunk {
    string uploadId = 1;   // id сессии (продолжение)
    bytes data = 2;        // чанк, не весь файл
}

message UploadFileRequestMeta {  // первый чанк несёт метаданные
    string purpose = 1;
    string ownerAccountId = 2;
    string filename = 3;
    map<string,string> metadata = 4;
}

message UploadFileResponse {
    string key = 1;
}
```

Первый чанк потока несёт метаданные, последующие — байты `data`.

---

## Backpressure (настройка)

### Как работает

```
Client ──stream──► UploadStreamController ──► UploaderService ──► S3 (multipart upload)
        чанк за чанком              читает по мере готовности         пишет по мере приёма
```

Client streaming даёт **настоящий backpressure**: сервер читает следующий чанк только когда обработал текущий. Клиент не может переполнить буфер сервера быстрее, чем тот успевает.

`UploadStreamController` (gRPC handler) только принимает чанки и делегирует в `UploaderService` (business: сессии, лимиты, multipart).

### Настройки (media.constants.ts / media.config.ts + proto)

```typescript
export const STREAM_MAX_CHUNK_BYTES = 1024 * 1024;     // 1MB на чанк
export const STREAM_MAX_TOTAL_BYTES = ...;             // лимит всего файла (зависит от purpose)
export const STREAM_MAX_CONCURRENCY = 32;              // макс параллельных upload-сессий
export const STREAM_IDLE_TIMEOUT_MS = 30_000;          // таймаут без данных → abort
export const STREAM_MAX_PENDING_CHUNKS = 4;            // ограничение in-flight чанков
```

### Двойной механизм backpressure

1. **На протоколе:** клиент не шлёт быстрее, чем сервер ack'ает (client streaming).
2. **На уровне сервиса:** лимит параллельных сессий → при превышении `RESOURCE_EXHAUSTED` (не принимать новые uploadId).

### Реализация лимитов

- `uploader.service` ведёт счётчик активных сессий
- При максимальной конкуренции — gRPC handler возвращает `RESOURCE_EXHAUSTED` (backpressure на входе)
- Каждый чанк проверяет `uploadId` против активной сессии (текущая сессия таймаутом завершается по `STREAM_IDLE_TIMEOUT_MS`)

---

## Прохождение нагрузки: будет ли блокироваться API Gateway?

**Комментарий к архитектуре.**

### Поток данных

```
Client ──► API Gateway (gRPC proxy) ──► MediaService (gRPC client streaming)
            ↑                                 ↑
      проксирует stream                 принимает stream
```

### Блокировка Gateway: ДА, частично

Gateway — **прозрачный gRPC прокси**. При client streaming:

1. **Gateway баферизует входящие чанки** — он тоже должен буферизовать поток и переслать его дальше.
2. **Память Gateway** — растёт на объём in-flight чанков (не весь файл, если прокси поддерживает streaming, а только «скользящее окно»).
3. **Время жизни соединения** — Gateway держит gRPC-соединение открытым до конца upload (долго для больших файлов).
4. **Worker'ы Gateway** — одно соединение = один занятый worker на всё время транзакции.

### Сравнение unary vs client-streaming (для Gateway)

| | unary `bytes fileData` | client streaming |
|--|------------------------|------------------|
| Gateway память | Весь файл в буфере (двойной) | Скользящее окно чанков |
| Gateway worker занят | Только на время приёма | Всю транзакцию (дольше) |
| Backpressure | Нет | Да |
| Масштаб файлов | Малые (≤ пары MB) | Средние |

**Вывод:** client streaming **убирает главную проблему unary** (весь файл в RAM на gateway/media). Но gateway **всё ещё держит соединение и worker** на время upload, и буферизует окно чанков. Если главное — **не грузить gateway байтами вовсе**, это **presigned** (байты идут клиент → MinIO напрямую, шаг 3).

**Для шага 1 (direct, client streaming):** gateway проксирует stream, buffering окна, но не весь файл. Мемльная нагрузка gateway ограничена `STREAM_MAX_PENDING_CHUNKS × chunkSize`.

---

## Внешние изменения на шаг 1

См. `external/` в этой папке:
- `external/01-media.proto.md` — новый proto (client streaming)
- `external/02-generated-types.md` — генерация `common/contracts/generated/media.ts`
- `external/03-s3-client.md` — требования к common/modules/s3-client (multipart upload)

## Порядок шага 1

1. proto → client streaming (external/01)
2. генерация типов (external/02)
3. common/modules/s3-client: multipart (external/03)
4. media-service: interfaces, utils, modules/uploader, module, controller, service
