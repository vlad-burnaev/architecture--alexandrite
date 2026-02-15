# Архитектурное решение по логированию

## Анализ системы в контексте логирования

### Системы требующие логирования

1. Internet Shop (Shop API)
   - Пользовательские действия
   - Создание заказов
   - Загрузка 3D файлов
   - Взаимодействие с MES

2. CRM (CRM API)
   - Работа продавцов с заказами
   - Подтверждение/отклонение заказов
   - Изменение статусов
   - Коммуникация с клиентами

3. MES (MES API)
   - Расчет стоимости
   - Производственные операции
   - Работа операторов
   - Изменение статусов заказов

4. RabbitMQ
   - Публикация сообщений
   - Доставка сообщений
   - Ошибки обработки

5. Инфраструктура
   - Load Balancers
   - Nginx/API Gateway
   - Базы данных (PostgreSQL logs)

---

## Список логов с уровнем INFO

### Internet Shop

1. Создание заказа
   - Время: timestamp
   - Order ID: идентификатор заказа
   - Customer ID: идентификатор клиента
   - Order Type: custom_3d, constructor, standard
   - Source: web, mobile, api

2. Загрузка 3D файла
   - Время: timestamp
   - Order ID
   - File ID
   - File Name
   - File Size
   - File Format: stl, obj, fbx
   - Upload Status: started, completed, failed

3. Отправка заказа на расчет
   - Время: timestamp
   - Order ID
   - Customer ID
   - Message ID (RabbitMQ)
   - Target Queue: orders_to_calculate

4. Получение результата расчета
   - Время: timestamp
   - Order ID
   - Calculated Price
   - Calculation Duration
   - Status: success, timeout, error

### CRM

1. Получение заказа для подтверждения
   - Время: timestamp
   - Order ID
   - Customer ID
   - Assigned To: seller_id
   - Source: queue, manual

2. Изменение статуса заказа
   - Время: timestamp
   - Order ID
   - Previous Status
   - New Status
   - Changed By: user_id
   - Reason (если отклонен)

3. Подтверждение заказа
   - Время: timestamp
   - Order ID
   - Approved By: user_id
   - Expected Completion Date
   - Priority: normal, high, urgent

4. Закрытие заказа
   - Время: timestamp
   - Order ID
   - Closure Reason: delivered, cancelled, returned
   - Closed By: user_id

### MES

1. Начало расчета стоимости
   - Время: timestamp
   - Order ID
   - Model Complexity: simple, medium, complex
   - Polygon Count
   - Expected Duration

2. Завершение расчета стоимости
   - Время: timestamp
   - Order ID
   - Calculated Price
   - Actual Duration
   - Status: success, timeout, error

3. Взятие заказа в работу оператором
   - Время: timestamp
   - Order ID
   - Operator ID
   - Previous Status
   - New Status: MANUFACTURING_STARTED

4. Изменение статуса производства
   - Время: timestamp
   - Order ID
   - Operator ID
   - Previous Status
   - New Status
   - Operation: manufacturing, packaging, shipping

5. Завершение производства
   - Время: timestamp
   - Order ID
   - Operator ID
   - Actual Production Time
   - Quality Check: passed, failed

### RabbitMQ операции

1. Публикация сообщения
   - Время: timestamp
   - Message ID
   - Exchange
   - Routing Key
   - Queue Name
   - Payload Size
   - Correlation ID (для трейсинга)

2. Доставка сообщения
   - Время: timestamp
   - Message ID
   - Queue Name
   - Consumer
   - Delivery Tag
   - Redelivery Count

3. Подтверждение обработки (ACK)
   - Время: timestamp
   - Message ID
   - Consumer
   - Processing Duration
   - Status: ack, nack, reject

### Общие технические логи

1. HTTP запросы
   - Время: timestamp
   - Request ID
   - Method: GET, POST, PUT, DELETE
   - Endpoint
   - Status Code
   - Duration
   - User Agent
   - Client IP

2. База данных операции
   - Время: timestamp
   - Query Type: select, insert, update, delete
   - Table
   - Duration
   - Affected Rows
   - Transaction ID (если есть)

---

## Уровни логирования

### DEBUG
Использование: только в dev окружении
- Детальная информация для отладки
- Значения переменных
- Промежуточные состояния
- Детали алгоритмов

Примеры:
- Детали расчета стоимости (каждый шаг)
- Парсинг 3D модели (детали полигонов)
- Промежуточные результаты валидации

### INFO (основной уровень)
Использование: production
- Нормальный flow операций
- Бизнес-события
- Изменения состояния

Примеры (описаны выше):
- Создание заказа
- Изменение статуса
- Публикация в RabbitMQ

### WARN
Использование: потенциальные проблемы
- Ситуации требующие внимания
- Retry операций
- Приближение к лимитам
- Deprecated API usage

Примеры:
- Расчет стоимости > 15 минут (близко к таймауту)
- RabbitMQ queue size > 100 (растет)
- Database connection pool > 80% используется
- Файл 3D модели > 100MB (очень большой)
- Retry операции после временной ошибки

### ERROR
Использование: ошибки требующие реакции
- Невозможность выполнить операцию
- Исключения
- Бизнес-логика нарушена

Примеры:
- Не удалось создать заказ (БД недоступна)
- Расчет стоимости failed
- RabbitMQ сообщение не доставлено
- Файл 3D модели не загружен (timeout S3)
- Database query timeout

### CRITICAL/FATAL
Использование: критические сбои системы
- Сервис не может функционировать
- Требуется немедленное вмешательство

Примеры:
- База данных недоступна
- RabbitMQ connection потерян
- Out of Memory
- Disk full
- Критическая ошибка в core логике

---

## Мотивация

### Почему нужно централизованное логирование

Текущая ситуация:
- Логи разбросаны по трем системам (Shop, CRM, MES)
- Каждая система пишет в свои файлы на диске
- Невозможно найти что происходило с заказом
- Разбор инцидента занимает часы - нужно заходить на каждый сервер
- Логи теряются при рестарте или падении сервиса

Логирование решает:
- Все логи в одном месте (централизованно)
- Быстрый поиск по order_id, customer_id, error
- Correlation между системами (тот же заказ в Shop, MES, CRM)
- Сохранность логов (не теряются при рестарте)
- Аналитика и паттерны

### Влияние на технические метрики

1. Mean Time To Resolve (MTTR)
   - Сейчас: часы на разбор проблемы (поиск логов на каждом сервере)
   - С централизованным логированием: 10-15 минут (быстрый поиск)
   - Улучшение: 4-6x ускорение

2. Debug Efficiency
   - Сейчас: разработчик запрашивает логи у DevOps, ждет, анализирует
   - С логированием: разработчик сам ищет в Kibana/UI
   - Результат: автономность команды

3. Root Cause Analysis Speed
   - Сейчас: непонятно где началась проблема (Shop? MES? RabbitMQ?)
   - С логированием: видим последовательность событий
   - Результат: быстрое выявление корневой причины

4. Proactive Issue Detection
   - Сейчас: узнаем о проблемах от клиентов
   - С логированием + алертинг: видим аномалии до жалоб
   - Результат: предотвращение инцидентов

5. Incident Postmortem Quality
   - Сейчас: сложно восстановить что произошло (логи потеряны/неполные)
   - С логированием: полная картина инцидента
   - Результат: качественный анализ и предотвращение повторов

### Влияние на бизнес-метрики

1. Support Efficiency
   - Проблема: support не может помочь клиенту ("где мой заказ?")
   - С логированием: support видит все действия с заказом
   - Результат: -40-50% времени на тикет, больше довольных клиентов

2. Customer Satisfaction
   - Проблема: клиенты ждут ответа часами
   - С логированием: быстрые ответы, проактивные уведомления
   - Результат: рост CSAT на 15-20%

3. Developer Productivity
   - Проблема: разработчики тратят время на поиск логов
   - С логированием: фокус на решении, не на сборе данных
   - Результат: +20-30% производительности

4. Cost of Downtime
   - Проблема: долгое восстановление после инцидентов
   - С логированием: быстрая диагностика и fix
   - Результат: меньше потерь от простоя

---

## Приоритизация: логирование vs трейсинг

Команда не может внедрить все сразу. Рекомендуемая последовательность:

### Фаза 1: Критичные системы с логированием и трейсингом (параллельно)

MES (Manufacturing Execution System)
- Почему первым: самая проблемная система (тормозит, теряются заказы)
- Логирование: расчет стоимости, изменения статусов, операции операторов
- Трейсинг: полный путь заказа через MES
- Срок: 1-2 недели

RabbitMQ
- Почему: центральная коммуникация, потеря сообщений = потеря заказов
- Логирование: публикация, доставка, ACK/NACK, errors
- Трейсинг: передача trace context
- Срок: 1 неделя (параллельно с MES)

### Фаза 2: Входные системы (2-3 недели после Фазы 1)

Internet Shop
- Почему: точка входа B2C и B2B клиентов
- Логирование: создание заказов, загрузки файлов, API запросы
- Трейсинг: начало trace для каждого заказа

CRM
- Почему: управление заказами, критично для бизнеса
- Логирование: подтверждения, изменения статусов, коммуникация
- Трейсинг: продолжение trace из Shop

### Обоснование приоритетов:

1. MES первым:
   - Самая большая боль (производительность, операторы)
   - Долгие операции (расчет стоимости) требуют visibility
   - Если MES работает хорошо → довольные операторы

2. RabbitMQ параллельно:
   - Потеря сообщений = потеря заказов
   - Критично для всей системы
   - Относительно просто внедрить

3. Shop и CRM вторыми:
   - Менее проблемные чем MES
   - Но необходимы для полной картины
   - Можно делать параллельно

---

## Предлагаемое решение

### Выбор технологии: ELK Stack

Elastic Stack (ELK):
- Elasticsearch: хранение и индексация логов
- Logstash: сбор и обработка логов
- Kibana: визуализация и поиск

Почему ELK:
- Open source (бесплатно для базового использования)
- Мощный поиск и фильтрация
- Масштабируемость
- Богатая экосистема
- Опыт использования во многих компаниях
- Уже выбран для трейсинга (Jaeger использует Elasticsearch)

Альтернативы:
- OpenSearch (форк Elasticsearch, более открытая лицензия)
- Loki (от Grafana, проще но менее функционален)
- CloudWatch Logs (AWS, vendor lock-in)

### Архитектура решения

Компоненты:

1. Filebeat / Fluent Bit (на каждом инстансе приложения)
   - Легковесный агент для сбора логов
   - Читает файлы логов или STDOUT
   - Отправляет в Logstash

2. Logstash (центральный)
   - Получает логи от всех Filebeat агентов
   - Парсинг (JSON, multiline, grok patterns)
   - Обогащение (добавление environment, service tags)
   - Фильтрация
   - Отправка в Elasticsearch

3. Elasticsearch Cluster
   - Хранение логов
   - Индексация для быстрого поиска
   - Минимум 3 ноды для отказоустойчивости

4. Kibana
   - UI для поиска логов
   - Дашборды
   - Алертинг (Elastic Alerting)
   - Доступ для: DevOps, backend devs, QA, support

### Формат логов

Структурированные JSON логи (обязательно):

Стандартные поля в каждом логе:
- timestamp (ISO 8601)
- level (DEBUG, INFO, WARN, ERROR, CRITICAL)
- service (shop, crm, mes, rabbitmq)
- environment (dev, release, prod)
- instance_id (какой инстанс)
- message (человекочитаемое описание)

Дополнительные поля (context):
- order_id (если логирование связано с заказом)
- customer_id
- user_id / operator_id
- request_id (для HTTP запросов)
- trace_id (интеграция с трейсингом!)
- span_id

Пример:
```
{
  "timestamp": "2026-02-15T10:30:45.123Z",
  "level": "INFO",
  "service": "shop",
  "environment": "prod",
  "instance_id": "shop-prod-01",
  "message": "Order created",
  "order_id": "ORD-12345",
  "customer_id": "CUST-67890",
  "order_type": "custom_3d",
  "trace_id": "abc123def456",
  "request_id": "req-xyz789"
}
```

### Корреляция с трейсингом

Критично: trace_id из трейсинга должен быть в логах!

Преимущества:
- Найти trace в Jaeger → скопировать trace_id → найти все логи этого заказа в Kibana
- Или наоборот: найти ERROR лог → trace_id → посмотреть trace в Jaeger
- Полная картина: метрики + traces + logs

---

## Политика безопасности

### Работа с чувствительными данными

Не логировать:
- Пароли
- Номера кредитных карт
- CVV коды
- API ключи / tokens
- Полные адреса клиентов
- Email адреса (можно только домен)
- Телефоны

Логировать только идентификаторы:
- customer_id вместо имени
- order_id вместо деталей заказа
- Хэши вместо чувствительных данных

Маскирование:
- Email: ivan@example.com → i***@example.com
- Телефон: +7 999 123 45 67 → +7 999 *** ** 67

Sanitization на уровне приложения:
- Автоматические фильтры для known sensitive fields
- Code review для предотвращения случайного логирования

### Доступ к логам

Role-Based Access Control (RBAC):

1. DevOps / SRE: full access
   - Все логи всех систем
   - Все окружения (dev, release, prod)
   - Настройка алертов

2. Backend Developers: read-only к своим сервисам
   - Shop devs → shop логи
   - MES devs → mes логи
   - Только dev и release окружения (prod только при инциденте)

3. Support: read-only для помощи клиентам
   - Поиск по order_id, customer_id
   - Только последние 7 дней
   - Только INFO, WARN, ERROR уровни (без DEBUG)

4. QA: read-only для тестирования
   - Логи тестовых окружений
   - Все уровни логирования

5. Тимлид, продакт: read-only для мониторинга
   - Дашборды с агрегированной информацией
   - Без доступа к детальным логам с customer_id

Реализация:
- Kibana Spaces для разделения доступа
- Или OAuth2 Proxy + LDAP/AD интеграция
- Audit log кто и что искал

### Compliance

GDPR:
- При удалении данных клиента → удалить логи с его customer_id
- Скрипт очистки логов: delete logs where customer_id = X
- Retention policy помогает (автоудаление старых логов)

---

## Политика хранения

### Индексы

Отдельный индекс на систему и окружение:
- logs-shop-prod-2026.02.15
- logs-crm-prod-2026.02.15
- logs-mes-prod-2026.02.15
- logs-shop-dev-2026.02.15
- ...

Rotation: ежедневно (новый индекс каждый день)

Преимущества:
- Легко удалять старые логи (удалить индекс целиком)
- Изоляция между системами
- Оптимизация поиска (ищем только в нужных индексах)

### Retention (срок хранения)

Production:
- ERROR, CRITICAL: 90 дней (долго, для анализа паттернов)
- WARN: 30 дней
- INFO: 14 дней
- DEBUG: не логируем в prod

Dev/Release:
- Все уровни: 7 дней
- DEBUG: 3 дня (много данных)

Автоматическое удаление:
- Elasticsearch Index Lifecycle Management (ILM)
- Или cron job для удаления старых индексов

### Размер логов (оценка)

Текущая нагрузка Александрита: ~500-1000 RPS

Оценка логов:
- 1 HTTP запрос = ~5-10 логов (request, processing, response, DB queries, RabbitMQ)
- 1 лог = ~1KB (JSON)
- 1000 RPS × 7 логов × 1KB = 7MB/sec = 600GB/день

Хранение:
- Prod (14 дней INFO): 600GB × 14 = 8.4TB
- Prod (30 дней WARN): 60GB × 30 = 1.8TB
- Prod (90 дней ERROR): 10GB × 90 = 0.9TB
- Итого prod: ~11TB

С компрессией (Elasticsearch сжимает ~3-5x): ~3TB

Storage:
- Elasticsearch: 3TB × 3 replicas = 9TB
- Стоимость: Yandex Cloud Disk ~8-10₽/GB/месяц = ~90-100k₽/месяц

Optimization:
- Sampling для DEBUG логов (если включаем)
- Агрегация INFO логов после 7 дней (только статистика)
- Hot-Warm-Cold архитектура Elasticsearch

---

## Система анализа логов

### Алертинг

Настроить алерты на аномальные паттерны:

1. Рост ERROR логов
   - Условие: ERROR count > 100 в 5 минут
   - Действие: алерт DevOps (Telegram/Slack)
   - Пример: всплеск ошибок после деплоя

2. Критические ошибки
   - Условие: CRITICAL level
   - Действие: немедленный алерт + PagerDuty
   - Пример: database unavailable

3. Замедление операций
   - Условие: "calculate_price" duration > 20 минут
   - Действие: алерт DevOps
   - Пример: MES перегружен

4. RabbitMQ проблемы
   - Условие: "message delivery failed" > 10 в минуту
   - Действие: алерт backend devs
   - Пример: consumer не работает

5. Аномалии создания заказов
   - Условие: "Order created" count > 1000 в минуту (обычно ~10-20)
   - Действие: алерт DevOps + security
   - Пример: DDoS атака или bot

### Поиск аномалий

Elastic Machine Learning для автоматического выявления:

1. Аномалии в объеме логов
   - Обычно: 1000 INFO логов/минуту
   - Аномалия: 10,000 логов/минуту
   - Возможная причина: атака, bug с бесконечными логами

2. Аномалии во времени операций
   - Обычно: расчет стоимости 2-5 минут
   - Аномалия: расчет стоимости 25 минут
   - Причина: проблемы с MES или БД

3. Аномалии в распределении ошибок
   - Обычно: 1-2% ERROR
   - Аномалия: 20% ERROR
   - Причина: проблема после деплоя или инфраструктурный сбой

4. Необычные паттерны
   - Внезапное появление нового типа ошибки
   - Изменение в распределении логов по сервисам
   - Пример: все ошибки только от одного инстанса (проблема с ним)

### Дашборды для разных ролей

DevOps Dashboard:
- Количество логов по уровням (DEBUG, INFO, WARN, ERROR)
- Top 10 ошибок
- Логи по сервисам и инстансам
- Response times по эндпоинтам
- RabbitMQ статистика

Support Dashboard:
- Поиск по order_id
- Timeline заказа (все действия)
- Статус заказа
- Ошибки связанные с заказом

Business Dashboard:
- Количество созданных заказов (по часам)
- Количество завершенных заказов
- Среднее время обработки
- Top ошибки влияющие на пользователей

Developer Dashboard:
- Ошибки в своем сервисе
- Performance метрики (database query times)
- Последние деплои и correlation с ошибками

---

## План внедрения

### Фаза 1: Инфраструктура (неделя 1)

1. Развернуть Elasticsearch cluster
   - 3 ноды для отказоустойчивости
   - Настроить index lifecycle management

2. Развернуть Logstash
3. Развернуть Kibana
4. Настроить OAuth2/LDAP authentication

### Фаза 2: MES и RabbitMQ (неделя 2-3)

5. MES логирование
   - Настроить Serilog с структурированными логами
   - Интегрировать trace_id из OpenTelemetry
   - Развернуть Filebeat на MES инстансах

6. RabbitMQ логирование
   - Настроить RabbitMQ audit logs
   - Filebeat для сбора

### Фаза 3: Shop и CRM (неделя 3-4)

7. Internet Shop
   - Logback с JSON encoder
   - Filebeat

8. CRM
   - Аналогично Shop

### Фаза 4: Дашборды и алертинг (неделя 4-5)

9. Создать дашборды в Kibana
10. Настроить алерты
11. Интеграция с PagerDuty/Jira

### Фаза 5: Обучение (неделя 5-6)

12. Обучить команду работе с Kibana
13. Создать runbooks для типичных сценариев

Общий срок: 5-6 недель

Ресурсы: DevOps + 2-3 backend dev

---

## Метрики успеха

После внедрения логирования:

1. Время поиска логов: < 1 минута (сейчас 10-30 минут)
2. MTTR: -50% (быстрее находим проблемы)
3. Support ticket resolution time: -40%
4. Developer productivity: +20-30%
5. Proactive issue detection: выявление 80% проблем до жалоб клиентов
6. Log coverage: 100% критичных операций
