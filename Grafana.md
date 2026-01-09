# Grafana

Grafana - это open-source платформа для визуализации и анализа метрик. Используется для создания дашбордов, которые отображают данные из различных источников (Prometheus, InfluxDB, Elasticsearch и др.).

## Основные концепции

### Что такое Grafana

Grafana - это веб-приложение для:
- Визуализации метрик (графики, таблицы, heatmaps)
- Создания дашбордов
- Настройки алертов
- Исследования данных (explore mode)

**Типичный стек:**
```
Prometheus (хранит метрики)
    ↓
Grafana (визуализирует метрики)
    ↓
Пользователь (смотрит дашборды)
```

### Data Sources

Data Source - это источник данных для Grafana.

**Популярные data sources:**
- Prometheus (самый популярный для метрик)
- InfluxDB
- Elasticsearch
- MySQL/PostgreSQL
- Graphite
- Loki (для логов)
- Jaeger (для трейсов)

**Подключение Prometheus:**
```
Configuration → Data Sources → Add data source → Prometheus

URL: http://localhost:9090
Access: Server (default)
```

## Dashboards

Dashboard - это набор panels, организованных в строки.

### Структура Dashboard

```
┌─────────────────────────────────────────────┐
│ Dashboard Title                              │
├─────────────────────────────────────────────┤
│ Row 1: API Metrics                          │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│ │ Panel 1 │ │ Panel 2 │ │ Panel 3 │       │
│ │ Requests│ │ Latency │ │ Errors  │       │
│ └─────────┘ └─────────┘ └─────────┘       │
├─────────────────────────────────────────────┤
│ Row 2: System Metrics                       │
│ ┌─────────┐ ┌─────────┐                    │
│ │ Panel 4 │ │ Panel 5 │                    │
│ │ CPU     │ │ Memory  │                    │
│ └─────────┘ └─────────┘                    │
└─────────────────────────────────────────────┘
```

### Создание Dashboard

**1. Через UI:**
```
Dashboards → New → New Dashboard → Add visualization
```

**2. Через JSON:**
```json
{
  "dashboard": {
    "title": "API Metrics",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ]
      }
    ]
  }
}
```

## Panels и визуализации

### Типы Panels

#### 1. Time Series (Graph)

Для отображения метрик во времени.

**Пример: Request Rate**
```promql
rate(http_requests_total[5m])
```

**Настройки:**
- Legend: `{{method}} {{endpoint}}`
- Y-axis unit: requests/sec
- Calculation: Mean, Max, Last

#### 2. Stat

Одно большое число (текущее значение).

**Пример: Active Users**
```promql
active_users
```

**Настройки:**
- Show: Value + Trend
- Color: Thresholds (green < 100, yellow < 500, red >= 500)

#### 3. Gauge

Шкала с диапазоном.

**Пример: CPU Usage**
```promql
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Настройки:**
- Min: 0, Max: 100
- Thresholds: 60 (yellow), 80 (red)

#### 4. Bar Gauge

Горизонтальные/вертикальные полосы.

**Пример: Queue Sizes**
```promql
worker_queue_size
```

#### 5. Table

Таблица с данными.

**Пример: Top Endpoints by Error Rate**
```promql
topk(5,
  sum by (endpoint) (rate(http_requests_total{status=~"5.."}[5m]))
)
```

#### 6. Heatmap

Распределение значений.

**Пример: Latency Distribution**
```promql
rate(http_request_duration_seconds_bucket[5m])
```

#### 7. Logs

Отображение логов из Loki.

```logql
{job="my-app"} |= "error"
```

## Практические примеры дашбордов

### Dashboard 1: API Monitoring

```json
{
  "title": "API Monitoring",
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (endpoint)",
          "legendFormat": "{{endpoint}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps"
        }
      }
    },
    {
      "title": "Error Rate",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))",
          "legendFormat": "Error Rate"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "thresholds": {
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 0.01, "color": "yellow"},
              {"value": 0.05, "color": "red"}
            ]
          }
        }
      }
    },
    {
      "title": "P95 Latency",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint))",
          "legendFormat": "{{endpoint}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "s"
        }
      }
    },
    {
      "title": "Active Connections",
      "type": "stat",
      "targets": [
        {
          "expr": "active_connections"
        }
      ]
    }
  ]
}
```

### Dashboard 2: System Resources

```json
{
  "title": "System Resources",
  "panels": [
    {
      "title": "CPU Usage",
      "type": "gauge",
      "targets": [
        {
          "expr": "100 - (avg(irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "min": 0,
          "max": 100,
          "thresholds": {
            "steps": [
              {"value": 0, "color": "green"},
              {"value": 60, "color": "yellow"},
              {"value": 80, "color": "red"}
            ]
          }
        }
      }
    },
    {
      "title": "Memory Usage",
      "type": "timeseries",
      "targets": [
        {
          "expr": "(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent"
        }
      }
    },
    {
      "title": "Disk I/O",
      "type": "timeseries",
      "targets": [
        {
          "expr": "rate(node_disk_read_bytes_total[5m])",
          "legendFormat": "Read"
        },
        {
          "expr": "rate(node_disk_written_bytes_total[5m])",
          "legendFormat": "Write"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "Bps"
        }
      }
    }
  ]
}
```

## Variables (Переменные)

Variables позволяют создавать динамические дашборды.

### Типы переменных

#### Query Variable

Значения из запроса к data source.

**Пример: Выбор instance**
```
Name: instance
Type: Query
Data source: Prometheus
Query: label_values(up, instance)
```

Использование в запросе:
```promql
rate(http_requests_total{instance="$instance"}[5m])
```

#### Custom Variable

Список предопределенных значений.

```
Name: environment
Values: production,staging,development
```

#### Interval Variable

Временной интервал.

```
Name: interval
Values: 1m,5m,10m,30m,1h
Auto: true
```

Использование:
```promql
rate(http_requests_total[$interval])
```

### Пример с переменными

```json
{
  "templating": {
    "list": [
      {
        "name": "datasource",
        "type": "datasource",
        "query": "prometheus"
      },
      {
        "name": "instance",
        "type": "query",
        "datasource": "$datasource",
        "query": "label_values(up, instance)",
        "multi": true
      },
      {
        "name": "interval",
        "type": "interval",
        "auto": true,
        "auto_count": 30,
        "auto_min": "10s"
      }
    ]
  },
  "panels": [
    {
      "targets": [
        {
          "expr": "rate(http_requests_total{instance=~\"$instance\"}[$interval])"
        }
      ]
    }
  ]
}
```

## Alerting

Grafana может отправлять алерты при достижении пороговых значений.

### Создание алерта

**1. В panel настройках:**
```
Alert → Create alert from this panel
```

**2. Конфигурация условия:**
```
WHEN: avg() OF query(A, 5m, now)
IS ABOVE: 100

Expression: avg(http_request_duration_seconds) > 0.5
For: 5m
```

### Contact Points

**Настройка уведомлений:**
```
Alerting → Contact points → New contact point

Name: Team Slack
Type: Slack
Webhook URL: https://hooks.slack.com/services/XXX
```

**Типы contact points:**
- Email
- Slack
- PagerDuty
- Telegram
- Webhook
- OpsGenie

### Notification Policies

```
Alerting → Notification policies

Label matchers: severity = critical
Contact point: Team PagerDuty
Group by: alertname
Group wait: 30s
Group interval: 5m
Repeat interval: 4h
```

## Query Editor

### Prometheus Query Editor

**Builder mode:**
```
Metric: http_requests_total
Label filters: status = 200, method = GET
Operations: rate, 5m
```

**Code mode (PromQL):**
```promql
rate(http_requests_total{status="200", method="GET"}[5m])
```

### Функции и трансформации

**Transformations:**
- Rename fields
- Filter by value
- Group by
- Join by field
- Calculate field

**Пример: Calculate Error Rate**
```
Query A: sum(rate(http_requests_total{status=~"5.."}[5m]))
Query B: sum(rate(http_requests_total[5m]))

Transform: Add field from calculation
Field: Error Rate
Mode: Binary operation
Operation: A / B
```

## Annotations

Annotations - это маркеры событий на графике.

**Пример: Deployment annotations**
```
Query: deployments{job="kubernetes"}
Title: Deployment
Text: Version {{version}}
Tags: deployment
```

Показывает на графике вертикальные линии в моменты деплоя.

## Best Practices

### Dashboard Design

1. ✅ **Используйте Rows** для группировки связанных panels
2. ✅ **Сверху ставьте ключевые метрики** (RED metrics)
3. ✅ **Используйте единообразные цвета** (красный = плохо, зеленый = хорошо)
4. ✅ **Добавляйте описания** к panels (Description field)
5. ✅ **Используйте переменные** для гибкости
6. ✅ **Группируйте по сервисам**, а не по типам метрик

### Query Optimization

1. ✅ **Используйте recording rules** в Prometheus для тяжелых запросов
2. ✅ **Ограничивайте временной диапазон** (не запрашивайте год данных)
3. ✅ **Избегайте `count()`** без labels - дорого
4. ❌ **Не делайте слишком много sub-queries**

### Alerting

1. ✅ **Алерты на симптомы**, а не на причины
2. ✅ **Используйте разные severity** (critical, warning, info)
3. ✅ **Добавляйте контекст** в annotations
4. ✅ **Группируйте алерты** по service/team
5. ❌ **Не создавайте алерты на каждую метрику**

## Типичные ошибки

### Ошибка 1: Слишком много panels

```
❌ Плохо: 50 panels на одном dashboard
✅ Хорошо: 10-15 panels, разделенные на несколько dashboards
```

### Ошибка 2: Неправильный тип визуализации

```
❌ Плохо: Table для отображения time series
✅ Хорошо: Time Series (Graph) для метрик во времени
```

### Ошибка 3: Игнорирование null values

```promql
# ❌ Плохо: метрика исчезает при отсутствии данных
rate(http_requests_total[5m])

# ✅ Хорошо: заполняем 0 при отсутствии данных
rate(http_requests_total[5m]) or vector(0)
```

## Примеры полезных запросов

### Top N Endpoints by Traffic

```promql
topk(10, sum by (endpoint) (rate(http_requests_total[5m])))
```

### Error Rate by Endpoint

```promql
sum by (endpoint) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (endpoint) (rate(http_requests_total[5m]))
```

### P50, P90, P99 Latency

```promql
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.90, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

### Request Rate per Minute

```promql
sum(rate(http_requests_total[1m])) * 60
```

### Apdex Score

```promql
(
  sum(rate(http_request_duration_seconds_bucket{le="0.1"}[5m]))
  +
  sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m])) / 2
)
/
sum(rate(http_request_duration_seconds_count[5m]))
```

## Provisioning

Автоматическое создание dashboards через файлы.

### Dashboard Provisioning

```yaml
# /etc/grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

```json
# /var/lib/grafana/dashboards/api-dashboard.json
{
  "dashboard": {
    "title": "API Metrics",
    "panels": [...]
  }
}
```

### Data Source Provisioning

```yaml
# /etc/grafana/provisioning/datasources/datasources.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

## Вопросы с собеседований

**Вопрос:** В чем разница между Grafana и Prometheus?

**Ответ:**
- **Prometheus** - система сбора и хранения метрик (TSDB)
- **Grafana** - визуализация метрик
- Prometheus может жить без Grafana (свой UI)
- Grafana не может без data source (Prometheus, InfluxDB, etc.)

**Вопрос:** Как экспортировать dashboard?

**Ответ:**
- Dashboard settings → JSON Model → Copy to clipboard
- Или через API: `GET /api/dashboards/uid/:uid`
- Можно использовать Grafana provisioning для version control

**Вопрос:** Что такое recording rules и зачем они нужны?

**Ответ:**
- Recording rules - это предвычисленные запросы в Prometheus
- Grafana использует их для ускорения тяжелых запросов
- Вместо выполнения сложного PromQL каждый раз, запрашивается готовая метрика
- Настраиваются в Prometheus, а не в Grafana

## Связанные темы

- [[Prometheus]]
- [[Метрики и наблюдаемость]]
- [[Kubernetes - Основы]]
- [[Микросервисная архитектура]]
