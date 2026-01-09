# Service Mesh

Service Mesh — это инфраструктурный слой для управления коммуникацией между микросервисами. Он обеспечивает функциональность вроде service discovery, load balancing, encryption, observability, без изменения кода приложений.

## Проблема, которую решает Service Mesh

### Без Service Mesh

Каждый микросервис должен реализовывать:

```go
// Order Service - дублирование логики в каждом сервисе
type OrderService struct {
    userClient    *UserClient
    circuitBreaker *CircuitBreaker    // Защита от сбоев
    retryPolicy    *RetryPolicy       // Retry logic
    timeout        time.Duration       // Таймауты
    metrics        *Prometheus         // Метрики
    tracer         *Jaeger            // Трейсинг
}

func (s *OrderService) GetUser(ctx context.Context, userID int) (*User, error) {
    // 1. Timeout
    ctx, cancel := context.WithTimeout(ctx, s.timeout)
    defer cancel()

    // 2. Circuit Breaker
    result, err := s.circuitBreaker.Execute(func() (interface{}, error) {
        // 3. Retry
        var user *User
        err := s.retryPolicy.Do(func() error {
            // 4. Трейсинг
            span, ctx := opentracing.StartSpanFromContext(ctx, "get_user")
            defer span.Finish()

            // 5. Метрики
            start := time.Now()
            defer func() {
                s.metrics.ObserveLatency("get_user", time.Since(start))
            }()

            // 6. Сам запрос
            user, err = s.userClient.Get(ctx, userID)
            return err
        })
        return user, err
    })

    if err != nil {
        return nil, err
    }
    return result.(*User), nil
}
```

**Проблемы:**
- ❌ Дублирование логики в каждом сервисе
- ❌ Сложность кода
- ❌ Разные языки → разные реализации
- ❌ Сложно обновлять (нужно патчить все сервисы)

### С Service Mesh

```go
// Order Service - чистый бизнес-код
type OrderService struct {
    userClient *UserClient
}

func (s *OrderService) GetUser(ctx context.Context, userID int) (*User, error) {
    // Просто делаем запрос
    // Service Mesh автоматически добавляет:
    // - Circuit breaker
    // - Retry
    // - Timeout
    // - Трейсинг
    // - Метрики
    // - mTLS encryption
    return s.userClient.Get(ctx, userID)
}
```

Service Mesh берет на себя всю инфраструктурную логику!

## Архитектура Service Mesh

### Data Plane (Sidecar Proxy)

Каждый под (pod) в Kubernetes получает sidecar proxy (обычно Envoy).

```
┌──────────────────────────────────┐
│          Pod                     │
│  ┌───────────┐   ┌───────────┐  │
│  │  Order    │──▶│  Envoy    │──┼──▶ Network
│  │  Service  │   │  Proxy    │  │
│  └───────────┘   └───────────┘  │
└──────────────────────────────────┘
```

**Весь сетевой трафик проходит через Envoy:**

```
Order Service → User Service

Реально:
Order Service → Envoy (Order) → Envoy (User) → User Service
                  ↑                    ↑
              Data Plane          Data Plane
```

### Control Plane

Управляет всеми Envoy proxy:

```
┌─────────────────────────────────┐
│     Control Plane (Istiod)      │
│   - Configuration               │
│   - Service Discovery           │
│   - Certificate Management      │
└────────┬────────────────────────┘
         │
    Конфигурирует
         │
    ┌────┴──────┬──────┬──────┐
    ▼           ▼      ▼      ▼
  Envoy      Envoy  Envoy  Envoy
  (Pod 1)    (Pod 2) ...
```

## Istio (самый популярный Service Mesh)

### Установка Istio в Kubernetes

```bash
# 1. Установка istioctl
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.x.x
export PATH=$PWD/bin:$PATH

# 2. Установка Istio в кластер
istioctl install --set profile=demo -y

# 3. Включение автоматического injection sidecar
kubectl label namespace default istio-injection=enabled
```

### Пример: Traffic Management

**Canary Deployment (постепенный роллаут):**

```yaml
# virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  hosts:
  - order-service
  http:
  - match:
    - headers:
        user-type:
          exact: "beta"
    route:
    - destination:
        host: order-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 90   # 90% трафика на старую версию
    - destination:
        host: order-service
        subset: v2
      weight: 10   # 10% трафика на новую версию
---
# destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: order-service
spec:
  host: order-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Применение:**

```bash
kubectl apply -f virtual-service.yaml
kubectl apply -f destination-rule.yaml

# Теперь:
# - Beta пользователи → v2 (100%)
# - Обычные пользователи → v1 (90%), v2 (10%)
```

### Retry и Timeout

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
  - user-service
  http:
  - route:
    - destination:
        host: user-service
    timeout: 3s              # Таймаут 3 секунды
    retries:
      attempts: 3            # Retry до 3 раз
      perTryTimeout: 1s      # Таймаут на каждую попытку
      retryOn: 5xx,reset,connect-failure,refused-stream
```

### Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: user-service
spec:
  host: user-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100        # Максимум 100 одновременных соединений
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 2
    outlierDetection:
      consecutiveErrors: 5         # После 5 ошибок подряд
      interval: 30s
      baseEjectionTime: 30s        # Исключить на 30 секунд
      maxEjectionPercent: 50       # Максимум 50% подов можно исключить
```

**Как это работает:**

```
Order Service → User Service

1. User Service pod-1 вернул 5 ошибок подряд
2. Istio автоматически исключает pod-1 на 30 секунд
3. Весь трафик идет на здоровые поды (pod-2, pod-3)
4. Через 30 секунд Istio проверяет pod-1 снова
```

### mTLS (Mutual TLS) - автоматическое шифрование

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT  # Все коммуникации только через mTLS
```

**Что происходит:**

```
Order Service → User Service

Без Istio:
Order Service → (HTTP) → User Service
                 ↑
            Незашифровано!

С Istio mTLS:
Order Service → Envoy → (mTLS) → Envoy → User Service
                         ↑
                  Автоматически зашифровано!
```

**В коде Go ничего менять не нужно:**

```go
// Код остается простым HTTP
user, err := http.Get("http://user-service:8080/users/123")

// Istio автоматически:
// 1. Перехватывает запрос
// 2. Шифрует через mTLS
// 3. Отправляет на user-service
// 4. Расшифровывает на другой стороне
```

## Observability (наблюдаемость)

### 1. Distributed Tracing (Jaeger)

Istio автоматически создает traces:

```yaml
# Включение Jaeger
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100.0  # 100% запросов
        zipkin:
          address: jaeger-collector.istio-system:9411
```

**Что видно в Jaeger:**

```
HTTP GET /orders/123
├─ Order Service (10ms)
│  ├─ GET User Service /users/456 (30ms)
│  │  └─ PostgreSQL Query (20ms)
│  ├─ GET Payment Service /payments/789 (50ms)
│  │  └─ Redis GET (5ms)
│  └─ PostgreSQL Query (15ms)
└─ Total: 105ms
```

**Код в Go остается простым:**

```go
// Istio автоматически добавляет trace headers
func (s *OrderService) GetOrder(w http.ResponseWriter, r *http.Request) {
    // Просто делаем запросы
    user := s.userClient.Get(r.Context(), userID)
    payment := s.paymentClient.Get(r.Context(), paymentID)

    // Istio автоматически:
    // - Создает spans
    // - Передает trace context
    // - Отправляет в Jaeger
}
```

### 2. Метрики (Prometheus + Grafana)

Istio автоматически экспортирует метрики:

```bash
# Istio собирает метрики автоматически
# Открываем Grafana
istioctl dashboard grafana
```

**Доступные метрики:**

```
istio_requests_total
istio_request_duration_milliseconds
istio_request_bytes
istio_response_bytes
istio_tcp_connections_opened_total
```

**Примеры запросов в Prometheus:**

```promql
# Request rate (RPS)
rate(istio_requests_total{destination_service="order-service"}[5m])

# P95 latency
histogram_quantile(0.95,
  rate(istio_request_duration_milliseconds_bucket[5m])
)

# Error rate
sum(rate(istio_requests_total{response_code=~"5.*"}[5m]))
/
sum(rate(istio_requests_total[5m]))
```

### 3. Service Graph

Istio автоматически строит граф зависимостей:

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ▼
┌────────────┐
│ API Gateway│
└─┬────┬────┘
  │    │
  │    └─────────────┐
  │                  │
  ▼                  ▼
┌────────┐      ┌────────┐
│ Order  │─────▶│  User  │
│Service │      │Service │
└───┬────┘      └────────┘
    │
    ▼
┌────────┐
│Payment │
│Service │
└────────┘
```

## Authorization (авторизация)

```yaml
# Разрешить только Order Service вызывать Payment Service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-policy
  namespace: default
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/order-service"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["/payments/*"]
```

**Что происходит:**

```
Order Service → Payment Service ✅ (разрешено)
User Service  → Payment Service ❌ (запрещено, 403 Forbidden)
```

## Rate Limiting

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: rate-limit
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_INBOUND
      listener:
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
    patch:
      operation: INSERT_BEFORE
      value:
        name: envoy.filters.http.local_ratelimit
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
          stat_prefix: http_local_rate_limiter
          token_bucket:
            max_tokens: 100          # Максимум 100 запросов
            tokens_per_fill: 100
            fill_interval: 60s       # В минуту
```

## Fault Injection (тестирование отказоустойчивости)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service-fault
spec:
  hosts:
  - user-service
  http:
  - fault:
      delay:
        percentage:
          value: 10.0        # 10% запросов
        fixedDelay: 5s       # Задержка 5 секунд
      abort:
        percentage:
          value: 5.0         # 5% запросов
        httpStatus: 503      # Возвращаем 503 Service Unavailable
    route:
    - destination:
        host: user-service
```

**Используется для:**
- Тестирование Circuit Breaker
- Тестирование Retry logic
- Chaos Engineering

## Istio в Go приложении

**До Istio:**

```go
// main.go - сложный код с инфраструктурой
func main() {
    // 1. Distributed tracing
    tracer := initJaeger()
    defer tracer.Close()
    opentracing.SetGlobalTracer(tracer)

    // 2. Метрики
    metrics := initPrometheus()

    // 3. Circuit breaker
    breaker := gobreaker.NewCircuitBreaker(settings)

    // 4. Retry
    retryPolicy := retry.NewPolicy(3, 1*time.Second)

    // 5. Timeout
    timeout := 5 * time.Second

    // Куча boilerplate кода...

    orderService := NewOrderService(db, tracer, metrics, breaker, retryPolicy, timeout)

    http.ListenAndServe(":8080", nil)
}
```

**После Istio:**

```go
// main.go - чистый код
func main() {
    db := connectDB()

    // Просто бизнес-логика
    orderService := NewOrderService(db)

    http.HandleFunc("/orders", orderService.Handler)

    // Istio автоматически добавляет:
    // - Tracing
    // - Metrics
    // - Circuit breaker
    // - Retry
    // - Timeout
    // - mTLS
    http.ListenAndServe(":8080", nil)
}
```

## Альтернативы Istio

### 1. Linkerd

- Легче чем Istio
- Написан на Rust (производительнее)
- Проще в настройке

```bash
# Установка Linkerd
curl -sL https://run.linkerd.io/install | sh
linkerd install | kubectl apply -f -

# Добавление приложения в mesh
kubectl get deploy -o yaml | linkerd inject - | kubectl apply -f -
```

### 2. Consul Connect

- Service Mesh от HashiCorp
- Интегрируется с Consul service discovery

### 3. AWS App Mesh

- Managed service от AWS
- Работает с EKS, ECS, EC2

## Service Mesh vs API Gateway

| Функция | API Gateway | Service Mesh |
|---------|-------------|--------------|
| **Назначение** | External → Internal | Internal ↔ Internal |
| **Местоположение** | Граница сети | Внутри кластера |
| **Аутентификация** | Пользователей | Сервисов (mTLS) |
| **Rate Limiting** | Per user | Per service |
| **Routing** | Внешний трафик | Внутренний трафик |

**Обычно используют вместе:**

```
Client → API Gateway → Service Mesh (Order → User → Payment)
           ↑                ↑
      External       Internal mesh
```

## Когда использовать Service Mesh

**✅ Используйте когда:**

1. **Много микросервисов (> 10-20)**
   - Сложная коммуникация между сервисами

2. **Разные языки программирования**
   - Go, Python, Java, Node.js
   - Service Mesh работает на сетевом уровне (не зависит от языка)

3. **Нужна наблюдаемость**
   - Трейсинг, метрики, service graph

4. **Нужна безопасность**
   - mTLS между всеми сервисами
   - Zero-trust networking

5. **Сложные deployment patterns**
   - Canary releases
   - A/B testing
   - Traffic mirroring

**❌ Не используйте когда:**

1. **Мало сервисов (< 5)**
   - Overhead не оправдан

2. **Монолит**
   - Service Mesh для микросервисов

3. **Простая инфраструктура**
   - Если нет Kubernetes

4. **Малая команда без опыта**
   - Service Mesh сложен в поддержке

## Недостатки Service Mesh

1. **Сложность**
   - Еще один слой абстракции
   - Нужна экспертиза для настройки

2. **Performance overhead**
   - Каждый запрос проходит через 2 Envoy proxy
   - Добавляет 1-5ms латентности

3. **Ресурсы**
   - Sidecar в каждом поде → больше памяти/CPU
   - 50 подов → 50 Envoy proxy

4. **Debugging сложнее**
   - Больше компонентов, где может быть проблема

## Best Practices

1. ✅ **Начните с малого** - включите mesh для 2-3 сервисов сначала
2. ✅ **Постепенный rollout** - не включайте mTLS STRICT сразу для всех
3. ✅ **Мониторинг обязателен** - следите за latency и resource usage
4. ✅ **Тестируйте в staging** - Service Mesh может сломать существующую логику
5. ✅ **Документируйте политики** - Authorization policies должны быть понятны команде
6. ❌ **Не используйте как замену кода** - бизнес-логика остается в приложении

## Связанные темы

- [[Микросервисная архитектура]]
- [[Монолит vs Микросервисы]]
- [[Kubernetes - Основы]]
- [[gRPC]]
- [[REST API - Основы]]
- [[Docker - Основы]]
- [[Метрики и наблюдаемость]]
- [[Prometheus]]
