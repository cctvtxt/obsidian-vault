Способность API справляться с ростом нагрузки путём добавления ресурсов.


| Характеристика           | Описание                                    | Тип масштабирования               | Ключевые требования                                    |
| ------------------------ | ------------------------------------------- | --------------------------------- | ------------------------------------------------------ |
| **Horizontal Scaling**   | Добавление инстансов под нагрузку           | Scale-out / scale-in              | Stateless дизайн, shared-nothing, externalized session |
| **Vertical Scaling**     | Увеличение ресурсов одного инстанса         | Scale-up / scale-down             | Поддержка более мощных нод, лимиты облака              |
| **Elasticity**           | Автоматическое масштабирование под паттерны | Auto-scaling по метрикам          | HPA/VPA, serverless, event-driven triggers             |
| **Geo-Scalability**      | Обслуживание глобальной аудитории           | Multi-region, edge deployment     | CDN, latency-based routing, data replication           |
| **Data Scalability**     | Рост объёмов данных без деградации          | Sharding, partitioning, archiving | DB sharding, time-series partitioning, cold storage    |
| **Protocol Scalability** | Эффективность при разных типах трафика      | HTTP/2, gRPC streaming, WebSocket | Multiplexing, binary protocols, push notifications     |