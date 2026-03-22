[[TLS-Based Auth]]

Сертификат X.509 — это структурированный бинарный объект в формате ASN.1 (DER), часто представленный в PEM как Base64-текст между `-----BEGIN CERTIFICATE-----` и `-----END CERTIFICATE-----`. Он содержит поля вроде Version (v3), Serial Number, Signature Algorithm (например, sha256WithRSAEncryption), Issuer DN (имя CA), Validity (Not Before/Not After), Subject DN (имя владельца, типа `CN=username,O=company`), Public Key (открытый ключ с алгоритмом, например RSA 2048), Extensions (Key Usage, SubjectAltName) и подпись CA.

Расшифровка: парсится библиотеками вроде OpenSSL (`openssl x509 -in cert.pem -text -noout`), показывая человекочитаемый вывод с DN как `/C=RU/ST=Moscow/L=Moscow/O=Org/CN=username`, публичным ключом в hex и цепочкой доверия.[](https://habr.com/ru/articles/194664/)​

Сертификат X.509 имеет три основные версии: v1 (1988 г., базовая структура без расширений), v2 (1993 г., добавлены поля issuerUniqueID и subjectUniqueID для идентификации) и v3 (наиболее распространённая, с расширениями вроде Key Usage, Subject Alternative Name).

**509** — это номер рекомендации (recommendation) Международного союза электросвязи (ITU-T) в серии X.200–X.509, описывающей архитектуру открытых систем (OSI) и PKI; стандарт ITU-T X.509 определяет формат сертификатов, CRL и атрибутов.