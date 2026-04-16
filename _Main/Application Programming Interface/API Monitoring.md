[[API]]


Способность предоставлять данные для понимания внутреннего состояния и поведения API.


| Характеристика          | Описание                               | Метрики / Сигналы                                                   | Инструменты                                  |
| ----------------------- | -------------------------------------- | ------------------------------------------------------------------- | -------------------------------------------- |
| **Logging**             | Детальная фиксация событий             | Structured logs, correlation IDs, log levels                        | ELK/EFK, Loki, CloudWatch Logs, Datadog Logs |
| **Metrics**             | Количественные показатели работы       | RED (Rate, Errors, Duration), USE (Utilization, Saturation, Errors) | Prometheus, StatsD, OpenTelemetry Metrics    |
| **Distributed Tracing** | Отслеживание запроса через сервисы     | Trace ID, span duration, service map                                | Jaeger, Zipkin, X-Ray, OpenTelemetry Tracing |
| **Health & Readiness**  | Статус работоспособности компонентов   | `/health`, `/ready`, `/live` с детализацией зависимостей            | Kubernetes probes, custom health endpoints   |
| **Alerting**            | Автоматическое уведомление о проблемах | SLO burn rate, anomaly detection, multi-signal alerts               | Prometheus Alertmanager, PagerDuty, Opsgenie |
| **Dashboarding**        | Визуализация состояния для команд      | Real-time SLO dashboards, business KPIs                             | Grafana, Kibana, custom internal portals     |