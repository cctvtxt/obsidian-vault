Пример Создан W3C в 1998 году (XML 1.0) как универсальный формат разметки и данных. **Характеристика: Теговый (tag-based), иерархический, строго структурированный с валидацией — "eXtensible Markup Language" (XML).** Цель — переносимость данных, расширяемость, замена SGML для веба. Версии эволюционировали: 1.0 (1998), 1.1 (2004, редко). XML использует теги <tag>value</tag>, атрибуты, пространства имён, поддерживает любые типы через схемы (XSD). Комментарии `<!-- -->`, CDATA для сырого текста.

```xml
<dependencies>
  <dependency>
    <name>serde</name>
    <version>1.0</version>
    <features>["derive"]</features>
  </dependency>
  <dependency name="toml" version="0.8" />
</dependencies>

<build>
  <version>0.1.0</version>
  <date>2026-03-04T20:39:00Z</date>
  <authors>
    <author>dev@example.com</author>
  </authors>
</build>

```

Формат позиционируется как стандартизированный (W3C), альтернатива JSON/YAML (строже, с схемами), CSV (иерархия). Часто сравнивают с HTML для документов, но с данными.

## Преимущества

Строгая валидация (XSD, DTD, Schematron) и типизация.  
Поддержка namespaces, атрибутов, вложенности любой глубины.  
Универсальность: документы (SVG, RSS), SOAP API, конфиги.  
Широкая поддержка (Java JAXB, Rust quick-xml, Python lxml).  
Расширяемость без ломки обратной совместимости.

## Недостатки

Высокая вербозность — много boilerplate тегов.  
Медленный парсинг и большой размер файлов.  
Сложный синтаксис для простых данных.  
Устаревание в вебе (JSON/YAML проще).  
Кривая обучения для схем и namespaces.

## Область применения

XML доминирует в enterprise: SOAP webservices, Android manifests, Maven pom.xml, Office OpenXML (.docx), RSS/Atom фиды. Идеален для legacy-систем, где нужна валидация и стандарты.
