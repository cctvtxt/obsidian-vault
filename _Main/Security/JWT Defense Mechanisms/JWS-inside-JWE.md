[[JSON Web Token]]

JWS-inside-JWE — это вложенная структура: сначала создаётся JWS-токен (с подписью), а затем его полная строка шифруется как payload внутри JWE

## Структура

1. **Внутренний JWS**: `header.payload.signature` (стандартный подписанный JWT, например, `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`).
    
2. **Внешний JWE**: `protected.encrypted_key.iv.ciphertext.tag` — где `ciphertext` содержит **закодированный JWS целиком** как зашифрованный payload.
    

## Зачем нужно

Обеспечивает **двойную защиту**: подпись (JWS) гарантирует целостность и аутентичность, шифрование (JWE) — конфиденциальность (payload скрыт даже от прокси/логов). Получатель сначала расшифровывает JWE, затем проверяет подпись JWS.

На диаграмме видно, как JWT реализуется через JWS/JWE, а nesting (JWS-inside-JWE) — частый паттерн для sensitive claims.