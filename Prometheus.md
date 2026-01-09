# Prometheus

Prometheus - это open-source система мониторинга и алертинга, разработанная в SoundCloud. Стала стандартом де-факто для мониторинга в cloud-native экосистеме, особенно в связке с Kubernetes.

## Основные концепции

### Pull Model

Prometheus использует **pull model** - сам опрашивает (scrape) targets для сбора метрик.

```
┌─────────────┐      HTTP GET /metrics      ┌─────────────┐
│ Prometheus  │ ───────────────────────────> │ Application │
│   Server    │ <─────────────────────────── │ (exporter)  │
└─────────────┘      Metrics in text format  └─────────────┘
```

**Преимущества pull model:**
- Централизованное управление (Prometheus знает все targets)
- Легко обнаружить "мертвые" сервисы (если scrape fail)
- Приложение не знает о Prometheus (меньше coupling)
- Service discovery из коробки (Kubernetes, Consul, etc.)

**Альтернатива:** Push model (как в Graphite) - приложение само отправляет метрики.

### Time Series Database

Prometheus хранит данные как временные ряды (time series):

```
metric_name{label1="value1", label2="value2"} value timestamp
```

**Пример:**
```
http_requests_total{method="GET", endpoint="/api/users", status="200"} 1234 1642424242
```

### Targets и Service Discovery

**Статическая конфигурация:**
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'my-app'
    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
```

**Service Discovery (Kubernetes):**
```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

## Типы метрик

### 1. Counter

Счетчик, который только увеличивается (или сбрасывается при рестарте).

**Когда использовать:** Количество событий (запросы, ошибки, выполненные задачи).

```go
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
)

func handleRequest(method, endpoint, status string) {
    // Инкремент счетчика
    requestsTotal.WithLabelValues(method, endpoint, status).Inc()

    // Инкремент на произвольное значение
    // requestsTotal.WithLabelValues(method, endpoint, status).Add(5)
}
```

**PromQL запросы:**
```promql
# Текущее значение counter
http_requests_total{endpoint="/api/users"}

# Rate (запросов в секунду за последние 5 минут)
rate(http_requests_total[5m])

# Increase (общий прирост за 5 минут)
increase(http_requests_total[5m])
```

### 2. Gauge

Метрика, которая может увеличиваться и уменьшаться.

**Когда использовать:** Текущие значения (температура, memory usage, размер очереди).

```go
var (
    activeConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )

    queueSize = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "worker_queue_size",
            Help: "Current queue size",
        },
        []string{"queue_name"},
    )

    temperature = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "cpu_temperature_celsius",
            Help: "Current CPU temperature",
        },
    )
)

func main() {
    // Установить значение
    activeConnections.Set(42)

    // Увеличить/уменьшить
    activeConnections.Inc()
    activeConnections.Dec()
    activeConnections.Add(10)
    activeConnections.Sub(5)

    // Установить в текущее время Unix
    temperature.SetToCurrentTime()

    // Gauge с labels
    queueSize.WithLabelValues("orders").Set(100)
}
```

**PromQL запросы:**
```promql
# Текущее значение
active_connections

# Средний размер очереди за 5 минут
avg_over_time(worker_queue_size[5m])

# Максимальный за час
max_over_time(worker_queue_size[1h])
```

### 3. Histogram

Выборка наблюдений (обычно длительность или размеры) и их распределение в buckets.

**Когда использовать:** Latency, размеры запросов/ответов.

```go
var (
    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "HTTP request duration in seconds",
            Buckets: []float64{0.001, 0.01, 0.05, 0.1, 0.5, 1.0, 2.5, 5.0, 10.0},
            // Buckets: prometheus.DefBuckets, // [0.005, 0.01, 0.025, 0.05, ...]
            // Buckets: prometheus.LinearBuckets(0, 0.1, 10), // [0, 0.1, 0.2, ..., 0.9]
            // Buckets: prometheus.ExponentialBuckets(0.001, 2, 10), // [0.001, 0.002, 0.004, ...]
        },
        []string{"method", "endpoint"},
    )
)

func handleRequest(method, endpoint string, duration float64) {
    // Записать наблюдение
    requestDuration.WithLabelValues(method, endpoint).Observe(duration)
}
```

**Что создается:**
```
# Buckets
http_request_duration_seconds_bucket{method="GET",endpoint="/api/users",le="0.001"} 10
http_request_duration_seconds_bucket{method="GET",endpoint="/api/users",le="0.01"} 50
http_request_duration_seconds_bucket{method="GET",endpoint="/api/users",le="0.05"} 95
http_request_duration_seconds_bucket{method="GET",endpoint="/api/users",le="0.1"} 100
http_request_duration_seconds_bucket{method="GET",endpoint="/api/users",le="+Inf"} 100

# Sum всех наблюдений
http_request_duration_seconds_sum{method="GET",endpoint="/api/users"} 4.5

# Count наблюдений
http_request_duration_seconds_count{method="GET",endpoint="/api/users"} 100
```

**PromQL запросы:**
```promql
# Средняя latency
rate(http_request_duration_seconds_sum[5m])
/
rate(http_request_duration_seconds_count[5m])

# 95th percentile (p95)
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# 99th percentile (p99)
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### 4. Summary

Похож на Histogram, но вычисляет квантили на стороне клиента.

**Когда использовать:** Когда нужны точные квантили, но их нельзя агрегировать.

```go
var (
    requestDuration = promauto.NewSummaryVec(
        prometheus.SummaryOpts{
            Name: "http_request_duration_seconds",
            Help: "HTTP request duration in seconds",
            Objectives: map[float64]float64{
                0.5:  0.05,  // 50th percentile с погрешностью 5%
                0.9:  0.01,  // 90th percentile с погрешностью 1%
                0.99: 0.001, // 99th percentile с погрешностью 0.1%
            },
        },
        []string{"method", "endpoint"},
    )
)

func handleRequest(method, endpoint string, duration float64) {
    requestDuration.WithLabelValues(method, endpoint).Observe(duration)
}
```

**Что создается:**
```
http_request_duration_seconds{method="GET",endpoint="/api/users",quantile="0.5"} 0.05
http_request_duration_seconds{method="GET",endpoint="/api/users",quantile="0.9"} 0.15
http_request_duration_seconds{method="GET",endpoint="/api/users",quantile="0.99"} 0.45
http_request_duration_seconds_sum{method="GET",endpoint="/api/users"} 4.5
http_request_duration_seconds_count{method="GET",endpoint="/api/users"} 100
```

### Histogram vs Summary

| Характеристика | Histogram | Summary |
|----------------|-----------|---------|
| Квантили на стороне | Сервера (Prometheus) | Клиента (приложение) |
| Можно агрегировать | ✅ Да | ❌ Нет |
| Точность | Зависит от buckets | Точная (с заданной погрешностью) |
| Производительность | Лучше | Хуже (хранит sliding window) |
| Рекомендация | **Используй Histogram** | Только если нужна абсолютная точность |

## PromQL Basics

### Селекторы

```promql
# Все метрики с именем
http_requests_total

# С фильтром по labels
http_requests_total{method="GET"}
http_requests_total{method="GET", status="200"}

# Regex
http_requests_total{endpoint=~"/api/.*"}
http_requests_total{status!~"5.."}  # исключить 5xx
```

### Range Vectors

```promql
# Instant vector (одно значение на метрику)
http_requests_total{method="GET"}

# Range vector (все значения за период)
http_requests_total{method="GET"}[5m]
```

### Функции

```promql
# Rate - скорость изменения counter в секунду
rate(http_requests_total[5m])

# Irate - мгновенная скорость (последние 2 точки)
irate(http_requests_total[5m])

# Increase - прирост за период
increase(http_requests_total[1h])

# Sum - сумма по labels
sum(rate(http_requests_total[5m])) by (endpoint)

# Avg - среднее
avg(http_request_duration_seconds) by (endpoint)

# Max/Min
max(http_request_duration_seconds)
min(http_request_duration_seconds)

# Histogram quantile
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### Операторы

```promql
# Арифметические
rate(http_requests_total[5m]) * 60  # запросов в минуту

# Сравнение
http_request_duration_seconds > 1.0

# Агрегация
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

## Практический пример

### Полный Go-сервис с метриками

```go
package main

import (
    "log"
    "math/rand"
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // Counter: общее количество запросов
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "api_requests_total",
            Help: "Total API requests",
        },
        []string{"method", "endpoint", "status"},
    )

    // Histogram: длительность запросов
    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "api_request_duration_seconds",
            Help:    "API request duration",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )

    // Gauge: активные соединения
    activeConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )

    // Gauge: размер очереди
    queueSize = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "worker_queue_size",
            Help: "Current queue size",
        },
    )
)

func metricsMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // Увеличиваем счетчик активных соединений
        activeConnections.Inc()
        defer activeConnections.Dec()

        wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}
        next(wrapped, r)

        duration := time.Since(start).Seconds()

        status := "2xx"
        if wrapped.statusCode >= 500 {
            status = "5xx"
        } else if wrapped.statusCode >= 400 {
            status = "4xx"
        }

        requestsTotal.WithLabelValues(r.Method, r.URL.Path, status).Inc()
        requestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    }
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func main() {
    // Симуляция работы с очередью
    go func() {
        for {
            size := rand.Intn(100)
            queueSize.Set(float64(size))
            time.Sleep(5 * time.Second)
        }
    }()

    // Метрики для Prometheus
    http.Handle("/metrics", promhttp.Handler())

    // API endpoints
    http.HandleFunc("/api/users", metricsMiddleware(handleUsers))
    http.HandleFunc("/api/slow", metricsMiddleware(handleSlow))
    http.HandleFunc("/api/error", metricsMiddleware(handleError))

    log.Println("Server starting on :8080")
    log.Println("Metrics available at http://localhost:8080/metrics")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

func handleUsers(w http.ResponseWriter, r *http.Request) {
    time.Sleep(time.Duration(rand.Intn(50)) * time.Millisecond)
    w.Write([]byte(`{"users": []}`))
}

func handleSlow(w http.ResponseWriter, r *http.Request) {
    time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
    w.Write([]byte("OK"))
}

func handleError(w http.ResponseWriter, r *http.Request) {
    if rand.Float32() < 0.3 {
        w.WriteHeader(http.StatusInternalServerError)
        w.Write([]byte("Error"))
        return
    }
    w.Write([]byte("OK"))
}
```

### Prometheus конфигурация

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'my-go-app'
    static_configs:
      - targets: ['localhost:8080']
```

## Alerting

### Правила алертов

```yaml
# alert_rules.yml
groups:
  - name: api_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(api_requests_total{status="5xx"}[5m]))
            /
            sum(rate(api_requests_total[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # Slow requests
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(api_request_duration_seconds_bucket[5m])
          ) > 1.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"
          description: "95th percentile is {{ $value }}s"

      # Service down
      - alert: ServiceDown
        expr: up{job="my-go-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"

      # High queue size
      - alert: HighQueueSize
        expr: worker_queue_size > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Queue size is {{ $value }}"
```

### Alertmanager конфигурация

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'team-slack'
  routes:
    - match:
        severity: critical
      receiver: 'team-pagerduty'

receivers:
  - name: 'team-slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX'
        channel: '#alerts'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'team-pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR-KEY'
```

## Best Practices

1. ✅ **Используйте Histogram для latency**, а не Summary
2. ✅ **Не создавайте метрики с высокой cardinality** (user_id, request_id в labels)
3. ✅ **Называйте метрики правильно**: `namespace_subsystem_metric_unit`
4. ✅ **Counter должны заканчиваться на `_total`**
5. ✅ **Используйте `rate()` для counter**, а не голые значения
6. ✅ **Алерты на симптомы**, а не на метрики (alert на slow requests, а не на CPU)
7. ❌ **Не используйте timestamps в метриках** - Prometheus добавит их сам
8. ❌ **Не экспортируйте метрики слишком часто** (15-60 секунд достаточно)

## Вопросы с собеседований

**Вопрос:** В чем разница между rate() и irate()?

**Ответ:**
- `rate()` - средняя скорость за весь временной диапазон (smoothed)
- `irate()` - мгновенная скорость между последними двумя точками (sensitive to spikes)
- Для алертов лучше `rate()`, для графиков - по ситуации

**Вопрос:** Почему Prometheus использует pull model?

**Ответ:**
- Централизованный контроль (Prometheus знает все targets)
- Легко определить, что сервис умер (scrape fail)
- Service discovery из коробки
- Приложение не зависит от доступности Prometheus

**Вопрос:** Когда использовать Histogram, а когда Summary?

**Ответ:**
- **Histogram** (почти всегда): можно агрегировать, вычислять квантили на сервере
- **Summary**: только если нужна абсолютная точность квантилей и не нужна агрегация

## Связанные темы

- [[Метрики и наблюдаемость]]
- [[Grafana]]
- [[Kubernetes - Основы]]
- [[Микросервисная архитектура]]
