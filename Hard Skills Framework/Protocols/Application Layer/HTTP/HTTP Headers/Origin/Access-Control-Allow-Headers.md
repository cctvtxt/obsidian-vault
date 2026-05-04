[[Hard Skills Framework/Protocols/Application Layer/HTTP/HTTP Headers/Origin]]

`Access-Control-Allow-Headers` — это заголовок ответа, который используется в ответе на preflight-запрос. Он сообщает браузеру, какие HTTP-заголовки могут быть использованы в фактическом запросе.

# Описание

Когда клиентский код пытается сделать кросс-доменный запрос с "непростыми" заголовками (например, кастомными, как `X-Custom-Header`, или стандартными, но требующими preflight, как `Authorization`), браузер сначала отправляет preflight-запрос методом `OPTIONS`.

В этом `OPTIONS`-запросе браузер использует заголовок `Access-Control-Request-Headers`, перечисляя заголовки, которые он собирается отправить.

Сервер в ответ должен вернуть `Access-Control-Allow-Headers`, перечислив те из запрошенных заголовков, которые он разрешает.

`Access-Control-Allow-Headers: Content-Type, Authorization, X-Requested-With`

# Юзкейсы

*   **Разрешение кастомных заголовков**: Позволяет API принимать кастомные заголовки от клиентских приложений (например, `X-CSRF-Token`, `X-API-Key`).
*   **Разрешение `Authorization`**: Заголовок `Authorization` не является "простым", поэтому для его использования в кросс-доменных запросах сервер должен явно его разрешить через `Access-Control-Allow-Headers`.
*   **Разрешение `Content-Type`**: Если `Content-Type` запроса отличается от `application/x-www-form-urlencoded`, `multipart/form-data` или `text/plain` (например, `application/json`), он также требует разрешения.

# Проблемы и особенности

*   **Только для preflight**: Этот заголовок имеет смысл только в ответе на `OPTIONS` (preflight) запрос. В ответе на основной запрос (`GET`, `POST` и т.д.) он игнорируется.
*   **Список заголовков**: Значение — это список имен заголовков, разделенных запятыми.
*   **`*` (wildcard)**: В старых браузерах `*` не поддерживался. В современных браузерах `Access-Control-Allow-Headers: *` означает, что разрешены любые заголовки, которые клиент запрашивает в `Access-Control-Request-Headers`. Однако это не разрешает `Authorization` при запросах с `credentials`.
*   **Динамический ответ**: Часто серверная логика просто "отражает" значение из `Access-Control-Request-Headers` в `Access-Control-Allow-Headers`, разрешая все, что запросил клиент.
