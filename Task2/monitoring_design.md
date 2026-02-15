# Выбор и настройка мониторинга в системе

## Мотивация

### Текущая ситуация

Компания "Александрит" сталкивается с критическими проблемами в production:
- Потеря заказов - клиенты жалуются, что заказы не поступают в работу
- Медленная работа MES - операторы ждут загрузки страницы
- Жалобы на API - B2B клиенты не получают заказы
- Невозможность диагностировать проблемы - разбор инцидентов занимает часы/дни

Яндекс Метрика дает информацию только о поведении пользователей на сайте (B2C), но не показывает:
- Что происходит с API запросами (B2B)
- Как работают внутренние сервисы (CRM, MES)
- Где возникают ошибки и задержки
- Состояние инфраструктуры (БД, RabbitMQ)

### Почему нужен технический мониторинг

1. Быстрая диагностика проблем
   - Сейчас: часы/дни на разбор инцидента
   - С мониторингом: минуты на выявление корневой причины
   - Пример: "MES тормозит" → дашборд показывает, что БД под нагрузкой

2. Проактивное выявление проблем
   - Обнаружение проблем до жалоб клиентов
   - Алерты при превышении порогов
   - Предотвращение каскадных сбоев

3. Понимание реального состояния системы
   - Какая нагрузка на каждый сервис
   - Где узкие места
   - Как растет потребление ресурсов

4. Обоснованные решения по масштабированию
   - Данные вместо догадок
   - Когда добавлять ресурсы
   - Оценка эффективности оптимизаций

5. SLA и качество сервиса
   - Объективные метрики доступности
   - Время ответа для B2B клиентов
   - Отслеживание деградации сервиса

### Бизнес-ценность

Прямая выгода:
- Сокращение времени простоя → меньше потерянных заказов → сохранение выручки
- Быстрое решение проблем API → удержание B2B клиентов → миллионы рублей контрактов
- Предотвращение инцидентов → меньше репутационного ущерба

Косвенная выгода:
- Команда тратит меньше времени на разбор инцидентов → больше времени на развитие продукта
- Уверенность в стабильности при масштабировании
- Прозрачность для бизнеса - понятно, где инвестировать в улучшения

Оценка ROI:
- Затраты: 2-3 недели работы DevOps + 1 backend dev + инфраструктура (~50-100k₽/месяц)
- Выгода: сохранение одного крупного B2B контракта (5-10 млн₽) окупает год работы мониторинга

---

## Выбор подхода к мониторингу

Для разных частей системы Александрита будем использовать разные подходы:

### 1. RED метрики - для всех приложений (Internet Shop, CRM, MES)

RED - Rate, Errors, Duration. Фокус на запросах к сервису.

Применение:
- Мониторинг API эндпоинтов
- Отслеживание пользовательских запросов
- Производительность веб-приложений

Почему RED для приложений:
- Простота - три ключевые метрики дают полную картину здоровья сервиса
- Фокус на user experience - измеряем то, что чувствует пользователь
- Быстрая диагностика - рост ошибок или latency сразу виден

Метрики:
- Rate (запросы/сек) - понимаем нагрузку
- Errors (% ошибок) - видим проблемы
- Duration (время ответа) - измеряем производительность

### 2. USE метрики - для инфраструктуры (БД, RabbitMQ, серверы)

USE - Utilization, Saturation, Errors. Фокус на ресурсах.

Применение:
- Базы данных (Shop DB, CRM DB, MES DB)
- RabbitMQ
- EC2 инстансы
- S3 storage

Почему USE для инфраструктуры:
- Показывает насыщенность ресурсов
- Предсказывает проблемы до их возникновения
- Помогает планировать масштабирование

Метрики:
- Utilization (% использования) - CPU, Memory, Disk, Network
- Saturation (очередь ожидания) - сколько запросов ждут
- Errors (количество ошибок) - сбои ресурса

### 3. Четыре золотых сигнала - для критичных бизнес-процессов

Применение:
- Lifecycle заказа (от создания до производства)
- API для B2B клиентов
- Расчет стоимости через MES

Четыре сигнала:
1. Latency - сколько времени занимает операция
2. Traffic - сколько операций в секунду
3. Errors - сколько операций завершились ошибкой
4. Saturation - насколько система загружена

Почему для бизнес-процессов:
- Комплексный взгляд на критичные операции
- Связь технических метрик с бизнес-результатами
- Измеряем end-to-end процессы (не только отдельные сервисы)

---

## Метрики для отслеживания

### Группа 1: Приложения (RED метрики)

#### Internet Shop

1. http_requests_total (counter)
   - Зачем: общее количество запросов, понимаем нагрузку на онлайн-магазин
   - Labels: method (GET/POST), endpoint (/api/orders, /api/products), status_code (200, 404, 500)

2. http_request_duration_seconds (histogram)
   - Зачем: время обработки запросов, выявляем медленные эндпоинты
   - Labels: method, endpoint
   - Бакеты: 0.1s, 0.5s, 1s, 2s, 5s, 10s

3. http_requests_errors_total (counter)
   - Зачем: количество ошибок, быстро видим проблемы
   - Labels: method, endpoint, status_code, error_type

4. orders_created_total (counter)
   - Зачем: бизнес-метрика - сколько заказов создано
   - Labels: order_type (custom_3d, constructor, standard)

5. file_uploads_total (counter)
   - Зачем: количество загрузок 3D моделей
   - Labels: file_format (stl, obj, fbx), status (success, failed)

6. file_upload_size_bytes (histogram)
   - Зачем: размер загружаемых файлов, планирование storage
   - Labels: file_format
   - Бакеты: 1MB, 10MB, 50MB, 100MB, 500MB

#### CRM

1. http_requests_total (counter)
   - Зачем: общая нагрузка на CRM
   - Labels: method, endpoint, status_code, user_role (seller, admin)

2. http_request_duration_seconds (histogram)
   - Зачем: производительность CRM для продавцов
   - Labels: method, endpoint, user_role

3. orders_approved_total (counter)
   - Зачем: сколько заказов подтверждено для производства
   - Labels: approval_type (auto, manual)

4. orders_closed_total (counter)
   - Зачем: сколько заказов завершено
   - Labels: closure_reason (delivered, cancelled, returned)

5. crm_active_users (gauge)
   - Зачем: сколько продавцов работают одновременно
   - Labels: user_role

#### MES

1. http_requests_total (counter)
   - Зачем: нагрузка на MES
   - Labels: method, endpoint, status_code

2. http_request_duration_seconds (histogram)
   - Зачем: производительность MES - критично для операторов
   - Labels: method, endpoint
   - Важно: алерт если p95 > 2s для /orders endpoint

3. price_calculation_duration_seconds (histogram)
   - Зачем: время расчета стоимости (основная проблема - 2-30 минут)
   - Labels: complexity (simple, medium, complex)
   - Бакеты: 30s, 1m, 2m, 5m, 10m, 20m, 30m

4. price_calculation_total (counter)
   - Зачем: количество расчетов стоимости
   - Labels: status (success, timeout, error), source (shop, api), complexity

5. orders_in_production_by_status (gauge)
   - Зачем: количество заказов в каждом статусе
   - Labels: status (manufacturing_started, manufacturing_completed, packaging, shipped)

6. operator_active_sessions (gauge)
   - Зачем: сколько операторов работают
   - Labels: none

7. orders_taken_by_operator_total (counter)
   - Зачем: производительность операторов
   - Labels: operator_id

### Группа 2: API для B2B (RED + дополнительные)

1. api_requests_total (counter)
   - Зачем: нагрузка от B2B клиентов
   - Labels: api_key, endpoint, status_code, client_tier (tier1, tier2, tier3)

2. api_request_duration_seconds (histogram)
   - Зачем: производительность API для клиентов
   - Labels: api_key, endpoint, client_tier

3. api_rate_limit_exceeded_total (counter)
   - Зачем: кто превышает лимиты
   - Labels: api_key, endpoint, client_tier

4. api_quota_usage (gauge)
   - Зачем: сколько квоты использовано
   - Labels: api_key, quota_type (requests_per_minute, requests_per_day), client_tier

5. api_errors_total (counter)
   - Зачем: ошибки для B2B клиентов
   - Labels: api_key, endpoint, error_code, client_tier

### Группа 3: RabbitMQ (USE метрики)

1. rabbitmq_queue_messages (gauge)
   - Зачем: размер очередей - критично для выявления проблем с обработкой
   - Labels: queue_name (orders_created, orders_approved, price_calculated)

2. rabbitmq_queue_messages_ready (gauge)
   - Зачем: сколько сообщений ждут обработки
   - Labels: queue_name

3. rabbitmq_queue_messages_unacked (gauge)
   - Зачем: сколько сообщений в обработке (не подтверждены)
   - Labels: queue_name

4. rabbitmq_queue_consumers (gauge)
   - Зачем: сколько консьюмеров обрабатывают очередь
   - Labels: queue_name

5. rabbitmq_messages_published_total (counter)
   - Зачем: сколько сообщений опубликовано
   - Labels: queue_name, source_service

6. rabbitmq_messages_delivered_total (counter)
   - Зачем: сколько сообщений доставлено
   - Labels: queue_name, consumer_service

7. rabbitmq_messages_acked_total (counter)
   - Зачем: сколько сообщений подтверждено (успешно обработано)
   - Labels: queue_name, consumer_service

8. rabbitmq_messages_nacked_total (counter)
   - Зачем: сколько сообщений отклонено (ошибки обработки)
   - Labels: queue_name, consumer_service, reason

9. rabbitmq_messages_redelivered_total (counter)
   - Зачем: сколько сообщений переотправлено (retry)
   - Labels: queue_name

10. rabbitmq_message_processing_duration_seconds (histogram)
    - Зачем: сколько времени занимает обработка сообщения
    - Labels: queue_name, consumer_service

### Группа 4: Базы данных (USE метрики)

#### Общие метрики для всех БД (Shop DB, CRM DB, MES DB)

1. postgres_connections_active (gauge)
   - Зачем: количество активных подключений, мониторинг connection pool
   - Labels: database (shop, crm, mes), state (active, idle, waiting)

2. postgres_connections_max (gauge)
   - Зачем: максимум подключений, алерт при приближении к лимиту
   - Labels: database

3. postgres_query_duration_seconds (histogram)
   - Зачем: время выполнения запросов, выявление slow queries
   - Labels: database, query_type (select, insert, update, delete)

4. postgres_slow_queries_total (counter)
   - Зачем: количество медленных запросов (> 1s)
   - Labels: database, table

5. postgres_transactions_total (counter)
   - Зачем: количество транзакций
   - Labels: database, status (commit, rollback)

6. postgres_deadlocks_total (counter)
   - Зачем: deadlocks в БД
   - Labels: database

7. postgres_table_size_bytes (gauge)
   - Зачем: размер таблиц, планирование storage
   - Labels: database, table

8. postgres_cache_hit_ratio (gauge)
   - Зачем: эффективность кеша БД
   - Labels: database

9. postgres_replication_lag_seconds (gauge)
   - Зачем: отставание read replica от master (после внедрения replicas)
   - Labels: database, replica_name

### Группа 5: Инфраструктура (USE метрики)

#### EC2 инстансы (для всех приложений)

1. node_cpu_usage_percent (gauge)
   - Зачем: загрузка CPU
   - Labels: instance_id, service (shop, crm, mes), environment (dev, release, prod)

2. node_memory_usage_percent (gauge)
   - Зачем: использование памяти
   - Labels: instance_id, service, environment

3. node_disk_usage_percent (gauge)
   - Зачем: заполненность диска
   - Labels: instance_id, service, environment, mount_point

4. node_disk_io_read_bytes (counter)
   - Зачем: чтение с диска
   - Labels: instance_id, service, environment

5. node_disk_io_write_bytes (counter)
   - Зачем: запись на диск
   - Labels: instance_id, service, environment

6. node_network_receive_bytes (counter)
   - Зачем: входящий трафик
   - Labels: instance_id, service, environment

7. node_network_transmit_bytes (counter)
   - Зачем: исходящий трафик
   - Labels: instance_id, service, environment

#### S3 Storage (3D files)

1. s3_objects_total (gauge)
   - Зачем: количество файлов
   - Labels: bucket_name

2. s3_bucket_size_bytes (gauge)
   - Зачем: размер хранилища
   - Labels: bucket_name

3. s3_requests_total (counter)
   - Зачем: количество запросов к S3
   - Labels: operation (put, get, delete), status_code

4. s3_request_duration_seconds (histogram)
   - Зачем: производительность S3
   - Labels: operation

### Группа 6: Бизнес-метрики (для дашбордов руководства)

1. orders_by_status (gauge)
   - Зачем: распределение заказов по статусам
   - Labels: status

2. orders_processing_time_seconds (histogram)
   - Зачем: время обработки заказа от создания до производства
   - Labels: order_type

3. revenue_total (counter)
   - Зачем: выручка (опционально, если доступна в системе)
   - Labels: order_type, source (b2c, b2b)

4. sla_violations_total (counter)
   - Зачем: количество нарушений SLA (заказ дольше обещанного срока)
   - Labels: sla_type (calculation, production, delivery)

---

## План действий

### Фаза 1: Инфраструктура мониторинга (неделя 1-2)

1. Развернуть Prometheus
   - Создать инстанс Prometheus в Yandex Cloud (2 CPU, 4GB RAM, 100GB SSD)
   - Настроить retention policy: 30 дней для детальных метрик, 1 год для агрегатов
   - Настроить backup в Yandex Object Storage

2. Развернуть Grafana
   - Создать инстанс Grafana в Yandex Cloud (2 CPU, 4GB RAM)
   - Настроить authentication через корпоративный LDAP/SSO
   - Подключить Prometheus как data source

3. Развернуть Alertmanager
   - Настроить для отправки алертов
   - Интегрировать с Telegram/Slack для команды
   - Настроить эскалации для критичных алертов

### Фаза 2: Инструментация приложений (неделя 2-3)

4. Добавить библиотеки для метрик
   - Internet Shop, CRM (Java): Micrometer + Prometheus registry
   - MES (C#): prometheus-net.AspNetCore
   - Экспорт метрик на /actuator/prometheus (Java) или /metrics (C#)

5. Добавить кастомные бизнес-метрики в код
   - orders_created, file_uploads, orders_approved и т.д.

6. Настроить метрики для RabbitMQ и PostgreSQL
   - RabbitMQ: установить Prometheus plugin
   - PostgreSQL: развернуть postgres_exporter для каждой БД

### Фаза 3: Настройка сбора метрик (неделя 3)

7. Настроить Prometheus scraping для всех сервисов
8. Установить Node Exporter на EC2 инстансы
9. Настроить мониторинг S3 через Yandex Cloud Monitoring API

### Фаза 4: Дашборды (неделя 4)

10. Создать дашборды в Grafana
    - "Обзор системы": статус сервисов, RPS, error rate, latency
    - "Internet Shop": RED метрики, созданные заказы, загрузки файлов
    - "CRM": RED метрики, подтвержденные заказы
    - "MES": RED метрики, время расчета стоимости, заказы по статусам
    - "API": запросы по клиентам, rate limiting, quota usage
    - "RabbitMQ": размер очередей, скорость обработки
    - "Базы данных": подключения, slow queries, cache hit ratio
    - "Инфраструктура": CPU/Memory/Disk по инстансам
    - "Бизнес-метрики": заказы по статусам, время обработки

### Фаза 5: Алертинг (неделя 4-5)

11. Настроить алерты в Prometheus

    Критичные (PagerDuty):
    - Сервис недоступен > 1 минуты
    - Error rate > 5% в течение 5 минут
    - RabbitMQ очередь > 1000 сообщений
    - БД недоступна

    Важные (Telegram/Slack):
    - Latency p95 > 2s для MES /orders
    - CPU > 80% в течение 10 минут
    - Memory > 85%
    - Disk > 80%

    Предупреждения (Telegram):
    - Latency p95 > 1s
    - Error rate > 1%
    - CPU > 70% в течение 15 минут

12. Настроить escalation policy
    - Критичные: DevOps on-call → тимлид (если не отвечает 5 минут)
    - Важные: DevOps → backend devs
    - Предупреждения: общий канал команды

### Фаза 6: Документация и обучение (неделя 5-6)

14. Создать документацию
    - Runbook для каждого типа алерта
    - Инструкции по добавлению новых метрик
    - Описание дашбордов
    - Troubleshooting guide

15. Обучить команду
    - Обзор системы мониторинга
    - Как читать дашборды
    - Как реагировать на алерты
    - Как добавлять метрики в новый код

---

## Показатели насыщенности (Saturation)

### CPU

Порог: CPU > 80% в течение 10+ минут

Почему этот порог:
- 80% - еще есть запас для обработки пиковой нагрузки
- 10 минут - фильтруем краткосрочные всплески

Действия при превышении:
1. Автоматический алерт → DevOps (Telegram)
2. Если CPU > 90% в течение 5 минут → критичный алерт (PagerDuty)
3. Manual action:
   - Проверить что потребляет CPU (top, профайлинг)
   - Если нагрузка легитимная → добавить инстанс (horizontal scaling)
   - Если проблема в коде → завести тикет high priority
4. Автоматическое действие (после внедрения auto-scaling):
   - Запустить дополнительный инстанс
   - Перенаправить часть трафика через Load Balancer

### Memory

Порог: Memory > 85%

Почему этот порог:
- 85% - критичный уровень, близко к OOM
- Память обычно не освобождается быстро (в отличие от CPU)

Действия:
1. Алерт → DevOps (Telegram)
2. Если Memory > 95% → критичный алерт (PagerDuty)
3. Manual action:
   - Проверить memory leak
   - Рестарт сервиса если критично
   - Завести тикет для исследования
4. Долгосрочно:
   - Увеличить память инстанса
   - Или добавить инстанс и распределить нагрузку

### Disk

Порог: Disk > 80%

Почему этот порог:
- 80% - еще есть время на реакцию
- Полный диск критичен (логи, БД не смогут писать)

Действия:
1. Алерт → DevOps (Telegram)
2. Если Disk > 90% → критичный алерт
3. Manual action:
   - Очистить старые логи (если не ротируются)
   - Проверить что занимает место
   - Увеличить размер диска
4. Автоматическое действие:
   - Ротация логов с удалением старых (> 30 дней)
   - Завести тикет для увеличения диска

### RabbitMQ

Порог: Размер очереди > 500 сообщений в течение 5 минут

Почему этот порог:
- 500 сообщений - индикатор что консьюмеры не успевают
- 5 минут - фильтруем временные всплески

Действия:
1. Алерт → DevOps + backend devs (Telegram)
2. Если очередь > 1000 → критичный алерт (PagerDuty)
3. Manual action:
   - Проверить что консьюмеры живы
   - Проверить нет ли ошибок в обработке (логи)
   - Добавить инстанс консьюмера
   - Если проблема в MES расчетах (долгие) - временно увеличить workers
4. Автоматическое действие (после внедрения):
   - Auto-scale консьюмеров при росте очереди

### RabbitMQ unacked messages

Порог: Unacked > 100 в течение 10 минут
Действия: алерт → backend devs, проверить логи, рестарт при необходимости

### Database connections

Порог: Active connections > 80% от max_connections
Действия: алерт → backend devs, проверить connection leak, настроить connection pool

### Database replication lag

Порог: Replication lag > 30 секунд
Действия: алерт → DevOps, проверить нагрузку на master и сеть

### API rate limit violations

Порог: Rate limit exceeded > 100 раз/минуту для API ключа
Действия: алерт → продакт + DevOps, связаться с клиентом, временная блокировка при > 1000/минуту
