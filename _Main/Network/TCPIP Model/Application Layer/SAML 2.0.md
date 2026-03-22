[[SAML-Based Auth]]


SAML 2.0 описан в OASIS-стандарте "Assertions and Protocols for the OASIS Security Assertion Markup Language (SAML) V2.0" (апрель 2005 г., с обновлениями). Это набор спецификаций: SAML Core (ядро с XML-ассерциями, протоколами AuthnRequest/Response), SAML Bindings (HTTP-Redirect, HTTP-POST, SOAP), Profiles (Web Browser SSO, Single Logout) и Metadata.

## Архитектура без OAuth

SAML фокусируется на федеративной аутентификации/авторизации между Identity Provider (IdP) и Service Provider (SP) через браузер: обмен SAML Request/Response в XML, подпись/шифрование, SSO без прямого API-вызова. В отличие от OAuth (grant types, token endpoint), здесь нет access/refresh-токенов — только ассерции для SSO.