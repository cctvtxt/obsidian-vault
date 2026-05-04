**Отказоустойчивость** — это способность системы продолжать выполнять свои функции после возникновения сбоев, отказов отдельных компонентов или неисправностей. При этом система должна скрывать от пользователя отказ отдельных элементов и минимизировать влияние сбоев на общую работоспособность. Отказоустойчивость достигается за счёт избыточности (резервирования, дублирования компонентов), автоматического обнаружения ошибок и переключения на резервные ресурсы.

Способность API адаптироваться, восстанавливаться и продолжать работу при частичных отказах.

| Характеристика           | Описание                                                | Паттерны                                         | Инструменты                                          |
| ------------------------ | ------------------------------------------------------- | ------------------------------------------------ | ---------------------------------------------------- |
| **Fault Isolation**      | Локализация сбоя в одном компоненте                     | Bulkhead, circuit breaker, timeout               | Resilience4j, Hystrix, Envoy retry policies          |
| **Graceful Degradation** | Снижение функциональности вместо полного отказа         | Feature flags, fallback responses, cached data   | LaunchDarkly, config maps, stale-while-revalidate    |
| **Retry & Backoff**      | Повтор запросов при временных сбоях                     | Exponential backoff, jitter, retry limits        | Client retry logic, service mesh retry policies      |
| **Circuit Breaker**      | Временное отключение неработающего бэкенда              | Closed → Open → Half-Open states                 | Istio circuit breaking, Polly, go-hystrix            |
| **Load Shedding**        | Отказ в обслуживании при перегрузке для сохранения ядра | Priority queues, rate limiting, request dropping | Token bucket, adaptive concurrency, L7 load shedding |
| **Chaos Readiness**      | Способность выдерживать контролируемые сбои             | Chaos experiments, failure injection             | Chaos Mesh, Gremlin, AWS Fault Injection Simulator   |