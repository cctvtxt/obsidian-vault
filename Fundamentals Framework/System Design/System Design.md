cap teorem 


Анализ задачи
- функциональные требования
- нефункциональные 
Оптимизация производительности
- latency
- throughput
Обеспечение безопасности 
Отказоустойчивость 
Maintenance






- **Обеспечение безопасности**: Защита данных и доступа с помощью аутентификации (OAuth/JWT), шифрования (TLS/mTLS), авторизации и защиты от атак (CSRF, DDoS).[](https://agaltsovav.ru/docs/architecture/system-design/)[](https://www.youtube.com/watch?v=-udz9s-Z7xs)
    
- **Отказоустойчивость**: Восстановление после сбоев via репликация, резервирование, circuit breakers и graceful degradation.
    
- **Масштабируемость**: Горизонтальное/вертикальное расширение, шардинг, load balancing для роста нагрузки без потери качества.
    
- **Доступность**: Высокий uptime (99.9%+), обработка сетевых разделений по CAP-теореме, failover-механизмы.
    
- **Консистентность данных**: Выбор модели (strong/eventual consistency), ACID/CAPELC для синхронизации в распределённых БД.[](https://www.youtube.com/watch?v=jAi0hZTmOag)[](https://interview-ai.dev/resources/system-design)
    
- **Управление данными**: Выбор БД (SQL/NoSQL), схемы хранения, шардинг/репликация для эффективности и объёма.
    
- **Интеграция компонентов**: API-дизайн (REST/GraphQL/gRPC), очереди сообщений, микросервисы для взаимодействия.[](https://www.youtube.com/watch?v=-udz9s-Z7xs)[](https://otus.ru/lessons/system-design/)
    
- **Мониторинг и наблюдаемость**: Логи, метрики, tracing (Prometheus/ELK) для диагностики и proactive maintenance.[](https://agaltsovav.ru/docs/architecture/system-design/)
    
- **Простота и поддерживаемость**: Минимизация сложности, модульность, автоматизация деплоя (CI/CD, IaC) для быстрой разработки.
    
- **Эффективность ресурсов**: Оптимизация CPU/памяти/сети, cost-efficiency в облаке (auto-scaling).[](https://skyeng.ru/it-industry/design/osnovy-proyektirovaniya-slozhnykh-it-arkhitektur-znakomstvo/)
    
- **Соответствие требованиям**: Сбор func/non-func требований, моделирование нагрузки (QPS, пики) на старте.