[[API]]

Способность API обрабатывать запросы с требуемой скоростью и эффективностью.


|Характеристика|Описание|Метрики|Оптимизация|
|---|---|---|---|
|**Latency (Задержка)**|Время от запроса до полного ответа|p50, p95, p99 (мс)|Кэширование, async processing, CDN, DB indexing|
|**Throughput (Пропускная способность)**|Макс. число запросов в единицу времени|RPS/QPS, Mbps|Горизонтальное масштабирование, connection pooling|
|**Time to First Byte (TTFB)**|Время до начала получения ответа|Мс до первого байта|TLS session resumption, HTTP/2, early hints|
|**Эффективность кэширования**|Снижение нагрузки за счёт повторного использования|Cache hit ratio %, avg TTL|`Cache-Control`, ETag, Redis/Memcached, CDN|
|**Payload Efficiency**|Оптимальный размер передаваемых данных|Avg response size (KB), compression ratio|GZIP/Brotli, protobuf, field selection, pagination|
|**Concurrency Handling**|Обработка параллельных запросов без деградации|Max concurrent connections, queue depth|Async I/O, worker pools, backpressure mechanisms|