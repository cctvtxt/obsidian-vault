Ниже — карта интеграции **Matt Pocock skills** с OpenSpec. Я предполагаю, что под `matprococ` имеется в виду репозиторий `mattpocock/skills`; его состав постоянно меняется: README описывает основной набор, а публичный индекс содержит также дополнительные, устаревшие и экспериментальные skills. Поэтому некоторые названия ниже могут быть legacy-алиасами в зависимости от установленной версии.[[github](https://github.com/mattpocock/skills)][[claudemarketplaces](https://claudemarketplaces.com/skills/mattpocock/skills)]

Главный принцип:

```
PRD
  ↓
grill / domain modeling / research
  ↓
OpenSpec proposal + specs + design + tasks
  ↓
tickets
  ↓
TDD + implementation
  ↓
code review + verification
  ↓
archive
```

Skills не становятся автоматически частью OpenSpec lifecycle. Они соединяются через общий контекст, файлы и явно заданный workflow. В самом Matt-репозитории user-invoked skill может вызывать model-invoked skills, но другой user-invoked skill обычно нужно запускать явно.[[github](https://github.com/mattpocock/skills)]

## Главные engineering skills

`/grill-me` — проводит подробное интервью по плану или дизайну и раскрывает неразрешённые ветки решения. Для OpenSpec он полезен до `/opsx:propose`: найденные решения превращаются в requirements, scenarios, ограничения и out-of-scope. Сам по себе новых файлов не создаёт, но повышает качество `proposal.md` и `spec.md`.

`/grill-with-docs` — более ценный вариант для OpenSpec-проекта: кроме интервью, он проверяет термины относительно существующей модели и сохраняет решения в `CONTEXT.md`, glossary и ADR. Это лучший preflight перед OpenSpec для сложной доменной задачи, потому что часть контекста остаётся в репозитории, а не только в истории чата.[[raw.githubusercontent](https://raw.githubusercontent.com/mattpocock/skills/main/skills/engineering/grill-with-docs/SKILL.md)][[github](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md)]

`/improve-codebase-architecture` — анализирует кодовую базу, находит места архитектурного «углубления» и предлагает кандидатов для улучшения. Для OpenSpec каждый выбранный кандидат можно оформить отдельным change с `proposal`, `design`, specs и tasks; дополнительный артефакт — HTML-отчёт анализа или список архитектурных инициатив.

`/tdd` — заставляет идти через red-green-refactor и тестировать поведение вертикальными срезами. Он усиливает OpenSpec тем, что превращает сценарии из `spec.md` в реальные тестовые проверки; добавляемые артефакты — unit/integration/e2e tests и иногда `testing-strategy.md`.[[github](https://github.com/mattpocock/skills/blob/main/skills/engineering/tdd/SKILL.md)]

`/setup-matt-pocock-skills` — настраивает `AGENTS.md` или `CLAUDE.md`, `docs/agents/` и сведения о tracker, labels и расположении доменных документов. Для OpenSpec это полезно как слой project context, но он не заменяет `openspec/config.yaml`; его артефакты — инструкции для агентов и конфигурационная документация.[[github](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/SKILL.md)]

`/handoff` — сжимает текущую сессию в документ передачи работы другому агенту или будущей сессии. В связке с OpenSpec он полезен при длинной реализации, когда один change не помещается в контекст; добавляет `HANDOFF.md` или аналогичный session handoff с текущим состоянием, решениями, блокерами и следующими действиями.

`/triage` — переводит issue через состояния процесса: новая задача, исследование, готово к работе, блокирована, выполнена и так далее. Он не расширяет `spec.md` напрямую, но делает внешний tracker согласованным с OpenSpec change; основным артефактом становятся labels, статус и комментарии issue.

`/prototype` — создаёт временный прототип, чтобы проверить техническую или UX-гипотезу до фиксации окончательного дизайна. Для OpenSpec это полезно перед `proposal` или `design`, если неизвестно, возможна ли идея; артефакт — throwaway HTML, экспериментальный код или prototype branch, который после решения нужно удалить либо явно пометить как disposable.

`/teach` — организует многошаговое обучение в текущем workspace и сохраняет учебные материалы. Для OpenSpec-проекта он полезен только косвенно: можно создать `docs/learning/`, explanations и упражнения для команды, но эти файлы не являются system specification и не должны смешиваться с production requirements.

`/grilling` — model-invoked примитив для последовательного интервью, который используется другими skills. Самостоятельного пользовательского артефакта не добавляет; его ценность для OpenSpec — быть внутренним механизмом `grill-with-docs`, `wayfinder`, `triage` или собственного orchestration-skill.

`/domain-modeling` — формирует и уточняет ubiquitous language, проверяет термины на edge cases и сверяет их с кодом. Он добавляет или обновляет `CONTEXT.md`, glossary и ADR; для OpenSpec это особенно важно, потому что одинаковые термины в requirements и коде уменьшают риск неверной реализации.[[raw.githubusercontent](https://raw.githubusercontent.com/mattpocock/skills/main/skills/engineering/domain-modeling/SKILL.md)]

`/codebase-design` — помогает проектировать deep modules: сложное поведение прячется за небольшим стабильным интерфейсом. В OpenSpec он усиливает `design.md` и помогает сформулировать границы модулей, но обычно не создаёт отдельный файл; решения следует сохранять в `design.md`, ADR или architecture documentation.

`/diagnosing-bugs` — использует цикл воспроизведение → минимизация → гипотезы → instrumentation → fix → regression test. Для OpenSpec bug fix можно оформить как change с изменённым scenario и regression test; дополнительные артефакты — минимальный repro, captured trace, postmortem или regression test.[[raw.githubusercontent](https://raw.githubusercontent.com/mattpocock/skills/main/skills/engineering/diagnosing-bugs/SKILL.md)]

`/ask-matt` — маршрутизатор, который рекомендует подходящий skill или workflow. Сам ничего не специфицирует и файлов не создаёт, но снижает вероятность выбрать `/to-spec` для архитектурной проблемы или `/implement` для ещё не проработанной задачи.

`/implement` — реализует спецификацию или набор tickets, обычно подключая `/tdd` и завершая работу `/code-review`. В OpenSpec он соответствует этапу после согласования change; добавляет production code, tests, commits и, возможно, обновление статусов tickets, но не должен молча менять requirements без `/opsx:update`.

`/to-prd` — превращает текущий контекст в Product Requirements Document и публикует его в issue tracker. Для OpenSpec это хороший верхний слой перед `proposal.md`: PRD фиксирует проблему, пользователей, цели и scope, а OpenSpec затем переводит это в проверяемое поведение. Артефакт — PRD в tracker или `docs/prd/*.md`.[[github](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-prd/SKILL.md)]

`/code-review` — проверяет diff по двум независимым осям: соответствие стандартам репозитория и соответствие исходной spec/PRD. Для OpenSpec это полезнее обычного review checklist, потому что напрямую выявляет missing requirements, scope creep и implementation, не соответствующую scenario; результатом является review report или комментарии в PR.[[raw.githubusercontent](https://raw.githubusercontent.com/mattpocock/skills/main/skills/engineering/code-review/SKILL.md)]

`/to-issues` — разбивает план, PRD или spec на independently-grabbable tickets, обычно вертикальными срезами. Для OpenSpec он полезен при командной разработке, но может дублировать `tasks.md`; лучше выбрать source of truth: либо локальные tasks, либо внешний tracker с ссылкой из change.

`/wayfinder` — предназначен для большой работы, которая превышает одну agent-сессию: строит карту решений, decision tickets и блокирующие связи. Он хорошо дополняет OpenSpec для миграций и крупных архитектурных изменений; артефакты — map issue, decision tickets, resolution comments и context pointers.[[github](https://github.com/mattpocock/skills/blob/main/skills/engineering/wayfinder/SKILL.md)]

`/research` — исследует вопрос по первичным источникам и сохраняет результаты с цитатами в Markdown. Для OpenSpec он закрывает внешние неизвестные: протокольные ограничения, поведение библиотек, security requirements и cloud provider semantics; артефакт — `docs/research/*.md`, который можно указать в `design.md` или ADR.

`/to-spec` — синтезирует текущую беседу в большую spec и публикует её в issue tracker. Для OpenSpec он может быть источником PRD или discovery-документа, но его результат не следует считать валидным `openspec/spec.md` без преобразования в `Requirement`/`Scenario` формат.

## Планирование и качество

`/writing-great-skills` — методика создания предсказуемых skills с хорошими descriptions, triggers и progressive disclosure. Для OpenSpec он полезен, если ты хочешь написать собственный `/openspec-secure-change` или wrapper `explore → grill → propose → review`; артефакт — `SKILL.md`, references и при необходимости evals.

`/resolving-merge-conflicts` — разрешает конфликты по intent каждой стороны, а не механическим выбором одной версии. OpenSpec он усиливает косвенно: помогает не потерять изменения в `specs`, ADR и `design.md`; отдельный документ обычно не создаёт, но сложные решения стоит записывать в ADR или merge notes.

`/to-tickets` — разбивает spec на tracer-bullet tickets и явно указывает blocking edges. В отличие от простого `to-issues`, он лучше подходит для dependency-aware плана; добавляет один Markdown-файл на ticket или issues в tracker с зависимостями.[[github](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-tickets/SKILL.md)]

`/diagnose` — в некоторых версиях является коротким или legacy-вариантом bug-diagnosis workflow. Если он доступен одновременно с `/diagnosing-bugs`, лучше выбрать последний; ценность для OpenSpec та же: reproducible bug artifact, regression test и уточнение поведения.

`/git-guardrails-claude-code` — добавляет защитные правила вокруг опасных Git-операций. Для OpenSpec он защищает незакоммиченные `proposal`, `specs`, ADR и tasks от потери, но не добавляет продуктовую документацию; артефакты — инструкции, hooks или agent guardrails.

`/setup-pre-commit` — настраивает pre-commit проверки. В SDD-процессе его можно использовать для автоматической проверки Markdown, OpenSpec formatting, links, secrets, generated contracts и tests; добавляет `.pre-commit-config.yaml`, scripts и CI configuration.

`/write-a-skill` — создаёт новый skill по формату Agent Skills. Для OpenSpec это способ автоматизировать повторяющиеся правила: например, проверять, что у каждого requirement есть scenarios, а у каждого task есть ссылка на requirement; артефакты — `SKILL.md`, references, scripts и поведенческие тесты skill.

`/zoom-out` — переводит работу с локального файла на уровень всей системы: зависимости, границы сервисов, соседние модули и последствия. Он полезен перед OpenSpec change, чтобы не специфицировать изолированный endpoint, не заметив event consumers и migration impact; результатом может быть context map, architecture notes или список затронутых компонентов.

`/request-refactor-plan` — готовит безопасный инкрементальный план рефакторинга. В OpenSpec его план можно преобразовать в change с MODIFIED requirements, `design.md` и задачами по этапам; ценность — уменьшение риска большого refactor-bang. В публичном репозитории он отмечен как планировщик безопасного incremental refactor.[[github](https://github.com/mattpocock/skills/blob/main/skills/deprecated/request-refactor-plan/agents/openai.yaml)]

`/qa` — проводит QA-проверку результата относительно требований и ожидаемых сценариев. Он может дополнить OpenSpec evidence report, checklist, test matrix и список пропущенных cases; это полезно после `/opsx:apply`, но не заменяет `/opsx:verify`.

`/ubiquitous-language` — помогает сформировать единый словарь терминов. Для OpenSpec это слой между доменом и requirements: он добавляет glossary или записи в `CONTEXT.md`, благодаря чему `User`, `Account`, `Identity`, `Session` и `Tenant` не смешиваются в одной спецификации.

`/wizard` — создаёт интерактивный shell wizard для действий, которые должен выполнить человек: credentials, provisioning, cutover, one-off migration. Для OpenSpec он особенно полезен для operational tasks; артефакт — проверяемый script, а в `tasks.md` должна быть ссылка на него и человеческие шаги.

`/setup-ts-deep-modules` — настраивает TypeScript-проект и проверки структуры зависимостей для deep modules. В OpenSpec это помогает сделать архитектурные ограничения машино-проверяемыми; артефакты — config dependency graph tool, scripts и CI check. Если проверка находит плохую границу, результат можно оформить отдельным architecture change.

`/review` — вероятно, общий или legacy-вариант code review. Для OpenSpec его следует использовать только если `/code-review` недоступен; желательно явно передавать путь к originating `spec.md`, иначе review может проверить стиль, но не соответствие behavior contract.

`/decision-mapping` — раскладывает большую инициативу в дерево или карту решений. Это близко к `wayfinder`, но полезно именно на discovery-этапе; артефакты — decision map, unresolved decisions и links на будущие ADR/OpenSpec changes.

`/prd-to-issues` — переводит PRD в tracker issues. Для OpenSpec это верхнеуровневое разбиение продуктовой работы, но tickets ещё не обязательно являются implementation-ready; после него полезно создать отдельные OpenSpec changes для конкретных capabilities.

`/prd-to-plan` — переводит PRD в инженерный план. Он может предварить `/opsx:propose`, но нужно затем отделить behavior requirements от implementation details; артефакт — plan document или issue checklist.

`/write-a-prd` — старый или более узкий вариант генерации PRD. По смыслу пересекается с `/to-prd`; использовать оба не нужно. Для OpenSpec ценность та же: зафиксировать problem, users, goals, scope, non-goals и success criteria до инженерной спецификации.

## Остальные и вспомогательные skills

`/scaffold-exercises` — создаёт структуру учебных exercise с `problem`, `solution` и `explainer`. Для OpenSpec-production почти не нужен, но полезен для внутреннего обучения команды OpenSpec, security или distributed systems; артефакты — каталоги упражнений и Markdown README.[[github](https://github.com/mattpocock/skills/blob/main/skills/misc/scaffold-exercises/SKILL.md)]

`/writing-shape` — помогает сначала определить форму и структуру документа, а затем писать содержание. Для OpenSpec его можно применить к PRD, ADR, RFC и runbook, чтобы не смешивать sections; отдельный workflow-артефакт обычно не добавляет.

`/writing-fragments` — создаёт и редактирует переиспользуемые текстовые фрагменты. Для OpenSpec полезен для стандартных формулировок security constraints, compatibility policy, error semantics и operational requirements; артефакт — fragments library, но её лучше не превращать в скрытый второй source of truth.

`/writing-beats` — раскладывает повествовательный или обучающий материал на последовательность смысловых шагов. Для инженерного OpenSpec имеет низкую ценность, но может помочь писать tutorial, migration guide или incident postmortem.

`/migrate-to-shoehorn` — заменяет небезопасные TypeScript `as` assertions в тестах на `shoehorn`, сохраняя типобезопасность частичных test fixtures. Для OpenSpec это не planning skill, но улучшает качество тестов, которыми проверяются scenarios; артефакт — изменённые tests и dependency/config changes.[[github](https://github.com/mattpocock/skills/blob/main/skills/misc/migrate-to-shoehorn//SKILL.md?plain=1)]

`/caveman` — режим ультракомпактной коммуникации с уменьшением лишнего текста. Для OpenSpec он полезен только в больших сессиях, где нужно экономить context window; никаких артефактов не создаёт и не должен применяться к самим requirements, где краткость не должна ухудшать точность.[[github](https://github.com/mattpocock/skills/blob/main/skills/productivity/caveman/SKILL.md)]

`/design-an-interface` — ориентирован на проектирование пользовательских интерфейсов. Для backend OpenSpec почти не нужен; для frontend change может добавлять design proposal, UI states, accessibility checklist и prototype, но эти материалы следует связывать с behavior specs.

`/obsidian-vault` — ищет, создаёт и связывает заметки в Obsidian. Для OpenSpec можно отправлять туда долгоживущие research notes, learning notes и индекс ADR, но repository `openspec/` должен оставаться source of truth для реализуемого поведения; артефакты — Obsidian notes и wikilinks.[[github](https://github.com/mattpocock/skills/blob/main/skills/personal/obsidian-vault/SKILL.md)]

`/edit-article` — редактирует статью или технический текст. Для OpenSpec применим после завершения implementation: из specs и ADR можно сделать release note, migration guide или публичную документацию, но это производный, а не нормативный документ.

`/loop-me` — экспериментальный skill для формализации повторяемого рабочего цикла через trigger, checkpoint, push и brief. Для OpenSpec его можно использовать как operating loop для длинной реализации, но он, судя по текущему состоянию, ещё не является стабильным core skill; возможный артефакт — loop protocol или checklist.[[github](https://github.com/mattpocock/skills/issues/657)]

`/claude-handoff` — старый или специализированный вариант `/handoff` для передачи контекста Claude-сессии. Если оба доступны, достаточно `/handoff`; результатом является handoff document с текущим change, решениями, изменёнными файлами и следующими действиями.

`/triage-issue` — обработка одной issue вместо полного triage workflow. Для OpenSpec может добавить classification, missing information, priority, owner и label; полезен до создания change, но не заменяет requirements analysis.

`/github-triage` — GitHub-ориентированный вариант triage. Его ценность — синхронизация issue state, labels и assignee с OpenSpec workflow; постоянных файлов обычно не создаёт, зато добавляет metadata и комментарии в GitHub.

`/domain-model` — legacy-вариант или короткое имя для domain-modeling. Для новых проектов лучше использовать `/domain-modeling`, потому что он явно описывает работу с `CONTEXT.md`, glossary и ADR.

## Другие популярные skills

### obra/superpowers

`obra/superpowers` — наиболее близкий внешний набор к OpenSpec по идее «сначала согласовать, потом планировать, потом реализовывать». Его базовый workflow включает `brainstorming`, `writing-plans`, TDD, subagent-driven development, code review и verification. Он сохраняет design document, implementation plan и тесты, поэтому может дополнить OpenSpec, но не стоит одновременно использовать два независимых источника плана.[[github](https://github.com/obra/superpowers)]

`brainstorming` — аналог preflight-дискуссии перед кодом. Его можно использовать вместо `/grill-me` перед `/opsx:propose`; результат лучше сохранять как исходный design или переносить в `proposal.md`, а не оставлять параллельным планом.

`writing-plans` — создаёт детальный implementation plan с маленькими шагами, путями файлов и verification commands. Он может дополнять `tasks.md`, но чаще всего дублирует OpenSpec tasks; выбирай его, если execution-план Superpowers удобнее native OpenSpec.

`test-driven-development` — строгий red-green-refactor workflow. Он функционально пересекается с Matt `/tdd`; одновременная установка двух TDD skills может привести к дублирующим инструкциям, поэтому лучше выбрать один.

`systematic-debugging` — дисциплинирует debugging через reproduction, hypothesis и verification. Он хорошо сочетается с OpenSpec bug changes и regression scenarios, но пересекается с `/diagnosing-bugs`.

`subagent-driven-development` — распределяет отдельные implementation tasks между свежими subagents и запускает review после каждого шага. Для OpenSpec полезен после утверждения `tasks.md`, но не должен позволять subagent изменять spec без явного обновления change.

`requesting-code-review` — создаёт checkpoint перед завершением работы и направляет код на review. Это полезно поверх `/opsx:verify`, потому что OpenSpec проверяет спецификацию, а review может дополнительно проверять качество модулей и standards.

`verification-before-completion` — требует доказательств перед заявлением «готово»: команды, тесты, diff и фактический результат. Для OpenSpec это идеальный финальный guardrail перед archive.

### Anthropic и Vercel

Anthropic публикует официальный набор Agent Skills, включая `webapp-testing`, `mcp-builder` и document skills. Для OpenSpec-проекта наиболее полезен `webapp-testing`: он может превращать UI scenarios в browser-level verification, но для чистого backend обычно не нужен.[[github](https://github.com/anthropics/skills)]

`mcp-builder` полезен, если OpenSpec workflow нужно расширить собственным MCP-сервером: например, читать issue tracker, запускать validation или собирать сведения о runtime. Это уже инфраструктурный слой, а не дополнительная product spec.

`vercel-labs/agent-skills` содержит `react-best-practices`, `web-design-guidelines`, `writing-guidelines` и другие frontend-oriented skills. Они полезны для repository с React/Next.js: могут добавлять UI review reports, accessibility checklists и performance findings, но для NestJS/gRPC/Kafka backend ценность ограничена.[[github](https://github.com/vercel-labs/agent-skills)]

## Рекомендуемый набор для OpenSpec

Для твоего backend/microservices workflow я бы не устанавливал все 53 skills. Практичный набор:

```
/setup-matt-pocock-skills
/opsx:explore
/grill-with-docs
/opsx:propose
/research
/to-tickets
/tdd
/opsx:apply
/diagnosing-bugs
/code-review
/handoff
/opsx:archive
```

Распределение артефактов:

```
docs/
├── prd/
├── research/
├── runbooks/
└── testing/

CONTEXT.md
docs/adr/
openspec/
├── specs/
└── changes/

AGENTS.md
HANDOFF.md
```

Рекомендуемый flow для сложного изменения:

```
/opsx:explore
→ /grill-with-docs
→ /research при внешних неизвестных
→ /opsx:propose
→ /to-tickets если работа большая
→ /tdd
→ /opsx:apply
→ /code-review
→ /opsx:verify
→ /opsx:archive
```

Не стоит одновременно складывать один и тот же план в `to-prd`, `to-spec`, `proposal.md`, `writing-plans` и `tasks.md` без разграничения ролей. Иначе получится несколько конкурирующих source of truth. Для проекта лучше заранее зафиксировать:

```
PRD             → docs/prd/
Domain context  → CONTEXT.md
Architecture    → docs/adr/
Behavior        → openspec/specs/
Current change  → openspec/changes/
Execution       → tasks.md или tracker
Evidence        → tests, review, verification report
```

Для максимального усиления OpenSpec я бы начал именно с комбинации:

```
grill-with-docs
+ domain-modeling
+ research
+ OpenSpec
+ tdd
+ code-review
+ verification-before-completion
```

Она закрывает почти все важные пробелы: неясные требования, терминологию, внешние факты, поведенческий контракт, тесты и проверку результата.