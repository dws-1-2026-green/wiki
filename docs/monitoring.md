# Подсистема мониторинга

Мониторинг построен на связке **Prometheus + Grafana** и является единым для всех трёх сервисов системы доставки вебхуков.

## Архитектура

```
event-receiver   :8080/metrics  ──┐
subscriptions-api:8082/metrics  ──┤
subscriptions-worker:9091/metrics─┤──► Prometheus :9090 ──► Grafana :3000
delivery-service :9095/metrics  ──┘
```

Prometheus и Grafana подключены к отдельной Docker-сети `monitoring-net`. Все три сервиса также включены в эту сеть, что позволяет Prometheus обращаться к ним по имени контейнера.

Grafana подключена к Prometheus как datasource и загружает дашборды автоматически через provisioning при старте контейнера.

## Prometheus

**Адрес:** http://localhost:9090

Scrape-интервал: **15 секунд**.

### Targets

| Job | Адрес | Описание |
|-----|-------|----------|
| `event-receiver` | `event-receiver:8080/metrics` | Сервис приёма событий |
| `subscriptions-api` | `subscriptions-api:8082/metrics` | HTTP API подписок |
| `subscriptions-worker` | `subscriptions-worker:9091/metrics` | Воркер маршрутизации |
| `delivery-service` | `delivery-service:9095/metrics` | Сервис доставки |

Состояние всех targets доступно на странице **Status → Targets** в интерфейсе Prometheus.

### Ключевые метрики

**event-receiver**

| Метрика | Тип | Описание |
|---------|-----|----------|
| `event_receiver_events_received_total` | counter | Входящие события (лейблы: `source`, `event_type`) |
| `event_receiver_events_published_total` | counter | Публикации в Kafka (лейбл: `status`: success/error) |
| `event_receiver_http_requests_total` | counter | HTTP запросы (лейблы: `method`, `path`, `status_code`) |
| `event_receiver_http_request_duration_seconds` | histogram | Длительность HTTP запросов |

**subscriptions-worker**

| Метрика | Тип | Описание |
|---------|-----|----------|
| `subscriptions_events_processed_total` | counter | Обработанные события из Kafka |
| `subscriptions_deliveries_dispatched_total` | counter | Задачи доставки, отправленные в Kafka |
| `subscriptions_no_matches_total` | counter | События без совпадений в подписках |
| `subscriptions_routing_errors_total` | counter | Ошибки маршрутизации |
| `subscriptions_db_query_duration_seconds` | histogram | Длительность запросов к БД (лейбл: `backend`) |

**delivery-service**

| Метрика | Тип | Описание |
|---------|-----|----------|
| `delivery_messages_received_total` | counter | Задачи доставки, полученные из Kafka |
| `delivery_attempts_total` | counter | Попытки доставки (лейбл: `status`: success/failure) |
| `delivery_final_status_total` | counter | Итоговый статус после всех попыток (лейбл: `status`: success/exhausted) |
| `delivery_attempt_duration_seconds` | histogram | Длительность одной попытки HTTP-доставки |
| `delivery_retries_total` | counter | Повторные попытки доставки (планировщик) |
| `delivery_pending_total` | gauge | Доставки в статусе pending в БД |

## Grafana

**Адрес:** http://localhost:3000  
**Логин / пароль:** `admin` / `admin`

Дашборды подгружаются автоматически при старте контейнера из директории `Environment/grafana/provisioning/dashboards/`.

### Структура дашбордов

| Папка | Дашборд | Содержимое |
|-------|---------|------------|
| `event-service` | Event Receiver | Входящие события, публикации в Kafka, HTTP-метрики |
| `subscription-service` | Subscriptions | Маршрутизация событий, dispatched deliveries, ошибки, задержки БД |
| `delivery-service` | Delivery Service | Попытки и итоги доставки, ретраи, длительность |
| `overview` | Overview | Все сервисы на одном экране, разбитые по секциям |

Дашборд **Overview** содержит секции для каждого сервиса, которые можно сворачивать. В нём также есть панель **Pipeline: End-to-End Event Flow** — сравнение трёх ключевых метрик сквозного потока: `received → routed → delivered`.

### Datasource

Grafana автоматически подключается к Prometheus через provisioning-файл `grafana/provisioning/datasources/prometheus.yml`. Ручная настройка datasource не требуется.

## Деплой

### Локально (Docker Compose)

Prometheus и Grafana поднимаются вместе со всем стеком:

```bash
cd Environment
docker compose up -d
```

Конфигурация Prometheus: `Environment/prometheus/prometheus.yml`  
Дашборды Grafana: `Environment/grafana/provisioning/dashboards/`

### Kubernetes (будущее)

При деплое в Kubernetes статические targets в `prometheus.yml` заменяются на **ServiceMonitor**-манифесты (Prometheus Operator). Prometheus автоматически обнаруживает поды через Kubernetes Service Discovery, что поддерживает горизонтальное масштабирование сервисов без изменения конфигурации.

PromQL-запросы в дашбордах при этом потребуют добавления `sum()` для агрегации метрик по всем инстансам, и опциональной группировки `by (pod)` для drill-down на уровень отдельного пода.
