WS-SecurityPolicy — это стандарт WS-\* для SOAP, который определяет политику безопасности в виде XML-ассерций, прикрепляемых к WSDL или endpoint'ам, чтобы клиенты и серверы согласовывали требования к защите сообщений.

## Модель политик

Политики выражаются через WS-Policy фреймворк: набор утверждений (assertions) описывает обязательные механизмы, такие как тип токенов (X.509, SAML), алгоритмы подписи/шифрования (AES-256, SHA-512), использование WS-Trust или WS-SecureConversation. Например, [sp:AsymmetricBinding](https://www.perplexity.ai/search/perechisli-mekhanizkhmmy-obesp-LmBGdQq6RVGAntXrUbGNtw) требует асимметричного шифрования с подписью. Клиент запрашивает WSDL, парсит политику и подбирает подходящий binding, обеспечивая интероперабельность.

## Применение в SOAP

Ассерции встраиваются в WSDL ([wsdl:binding](https://www.perplexity.ai/search/perechisli-mekhanizkhmmy-obesp-LmBGdQq6RVGAntXrUbGNtw) или [sp:Policy](https://www.perplexity.ai/search/perechisli-mekhanizkhmmy-obesp-LmBGdQq6RVGAntXrUbGNtw)), указывая, какие части SOAP-сообщения (Body, Header) защищать, режим (transport, message) и дополнительные опции вроде Timestamp или SignedInfo. Сервер отвергает несоответствующие запросы на этапе negotiation. Это автоматизирует конфигурацию в enterprise-системах (WCF, Axis2, Spring WS).

## Преимущества

Упрощает декларативную настройку безопасности без кода, поддерживает сложные сценарии (federated identity, multi-token). Обеспечивает строгую проверку compliance перед обменом, минимизируя ошибки в B2B-интеграциях.
