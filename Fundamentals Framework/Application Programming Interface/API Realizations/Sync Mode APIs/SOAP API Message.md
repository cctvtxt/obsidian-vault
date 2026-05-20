Способ отправки запросов зависит от SOAP-библиотеки. Если библиотека поддерживает WSDL, достаточно указать адрес WSDL-описания. Из него библиотека получает адрес сервиса и выполняет необходимые действия для отправки запроса. Если библиотека не поддерживает WSDL, необходимо явно указывать адрес сервиса. Адрес WSDL и адрес для запросов указаны в документации каждого сервиса.

#### Структура SOAP сообщения:

- **Конверт (Envelope)** — корневой и обязательный элемент, инкапсулирует все остальные части сообщения и определяет XML-пространство имён для его интерпретации.
- **Заголовок (Header)** — необязательный элемент, используется для передачи дополнительной информации, не связанной напрямую с основным содержимым сообщения. В заголовке могут содержаться данные для аутентификации, управления транзакциями или маршрутизации.
- **Тело (Body)** — обязательный элемент, содержит основную информацию, предназначенную для конечного получателя. В теле сообщения находятся данные для вызова удалённой процедуры и её параметры или информация об ответе.
- **Ошибка (Fault)** — необязательный элемент, появляется в теле сообщения только в случае возникновения ошибки. Предоставляет стандартизированную информацию о том, что пошло не так, включая код ошибки, её описание и источник.

# Пример запроса

Вызов метода `Ads.add` для добавления объявлений рекламодателю `agrom`, выполняемый от имени рекламного агентства.

```
POST /v5/ads/ HTTP/1.1
Host: api.direct.yandex.com
Authorization: Bearer 0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f
Accept-Language: en
Client-Login: agrom
Content-Type: text/xml; charset=utf-8
SOAPAction: "https://api.direct.yandex.com/v5/ads/add"

`<?xml version="1.0" encoding="UTF-8"?`
<SOAP-ENV:Envelope
    xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:ns1="http://api.direct.yandex.com/v5/ads"
  <SOAP-ENV:Body
    <ns1:AddRequest
      <Ads
        <AdGroupId1234567</AdGroupId
        <TextAd
          <TextСлоны всех пород. Сертифицированный питомник</Text
          <TitleКупи слона!</Title
          <Hrefhttp://exotic-farm.com/elefants</Href
          <MobileNO</Mobile
        </TextAd
      </Ads
      <Ads
        <AdGroupId1234567</AdGroupId
        <TextAd
          <TextНосороги с доставкой. Весенняя распродажа</Text
          <TitleКупи носорога!</Title
          <MobileNO</Mobile
        </TextAd
      </Ads
    </ns1:AddRequest
  </SOAP-ENV:Body
</SOAP-ENV:Envelope
```

# Пример ответа

```
HTTP/1.1 200 OK
Connection:close
Content-Type:text/xml; charset=utf-8
Date:Fri, 28 Nov 2014 17:07:02 GMT
RequestId:1010101010101010101
Units:10/20828/64000
Units-Used-Login:agrom
Server:nginx
Transfer-Encoding:chunked

`<?xml version="1.0" encoding="UTF-8"?`<SOAP-ENV:Envelope
    xmlns:SOAP-ENV="http://schemas.xmlsoap.org/soap/envelope/"
  <SOAP-ENV:Body
    <ns3:AddResponse xmlns:ns3="http://api.direct.yandex.com/v5/ads"
      <AddResults
        <Id7654321</Id
      </AddResults<!-- Объявление успешно создано, возвращен его идентификатор --
      <AddResults
        <Errors
          <Code6000</Code
          <MessageНеконсистентное состояние объекта</Message
          <DetailsВ объявлении должна быть указана или визитка или основная ссылка</Details
        </Errors
      </AddResults<!-- Ошибка создания объявления --
    </ns3:AddResponse
  </SOAP-ENV:Body
</SOAP-ENV:Envelope
```
