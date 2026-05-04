[[HTTP Headers]]

Заголовки, связанные с **источником запроса (Origin)**, являются ключевой частью механизма **Cross-Origin Resource Sharing (CORS)**. Они позволяют серверу определять, каким веб-страницам разрешено получать доступ к его ресурсам, обеспечивая безопасность в браузере.

1. `Origin` — источник запроса для CORS и политики происхождения.
2. `Access-Control-Allow-Origin` — какие origin разрешены.
3. `Access-Control-Allow-Credentials` — можно ли отправлять cookies и креды.
4. `Access-Control-Allow-Headers` — какие заголовки разрешены в запросе.
5. `Access-Control-Allow-Methods` — какие методы разрешены.
6. `Access-Control-Expose-Headers` — какие заголовки доступны JS-коду.
7. `Access-Control-Max-Age` — срок кэширования preflight-ответа.
8. `Access-Control-Request-Method` — запрашиваемый метод в preflight.
9. `Access-Control-Request-Headers` — запрашиваемые заголовки в preflight.
10. `Timing-Allow-Origin` — разрешает доступ к timing-данным.
