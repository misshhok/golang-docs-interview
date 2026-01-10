# System Design - Основы

System Design — это проектирование архитектуры масштабируемых распределённых систем. На собеседованиях Senior уровня эта тема обязательна и проверяет способность принимать архитектурные решения.

## Подход к System Design интервью

### Структура ответа (45-60 минут)

```
1. Уточнение требований (5-10 мин)
   ├── Функциональные требования (что делает система)
   ├── Нефункциональные требования (масштаб, latency, availability)
   └── Ограничения (бюджет, технологии)

2. Оценка нагрузки (5 мин)
   ├── Количество пользователей
   ├── QPS (queries per second)
   └── Объём данных

3. High-Level Design (10-15 мин)
   ├── Основные компоненты
   ├── Потоки данных
   └── API дизайн

4. Deep Dive (15-20 мин)
   ├── Детали критических компонентов
   ├── Trade-offs
   └── Оптимизации

5. Масштабирование (5-10 мин)
   ├── Bottlenecks
   ├── Scaling strategies
   └── Fault tolerance
```

## Ключевые концепции

### Масштабирование (Scaling)

```
┌─────────────────────────────────────────────────────────────────┐
│                      ВЕРТИКАЛЬНОЕ                                │
│                 (Scale Up / Scale Down)                          │
│  ┌──────┐         ┌──────────────┐                              │
│  │Server│   →→→   │Bigger Server │                              │
│  │ 4GB  │         │    32GB      │                              │
│  └──────┘         └──────────────┘                              │
│  + Простота                                                      │
│  - Ограничения железа                                           │
│  - Single point of failure                                       │
├─────────────────────────────────────────────────────────────────┤
│                      ГОРИЗОНТАЛЬНОЕ                              │
│                 (Scale Out / Scale In)                           │
│  ┌──────┐         ┌──────┐ ┌──────┐ ┌──────┐                    │
│  │Server│   →→→   │Server│ │Server│ │Server│                    │
│  └──────┘         └──────┘ └──────┘ └──────┘                    │
│  + Неограниченный рост                                          │
│  + Отказоустойчивость                                           │
│  - Сложность                                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Load Balancing

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
     ┌──────────┐      ┌──────────┐      ┌──────────┐
     │ Server 1 │      │ Server 2 │      │ Server 3 │
     └──────────┘      └──────────┘      └──────────┘
```

**Алгоритмы балансировки:**

| Алгоритм | Описание | Когда использовать |
|----------|----------|-------------------|
| Round Robin | По очереди | Серверы одинаковой мощности |
| Weighted Round Robin | С весами | Разная мощность серверов |
| Least Connections | Меньше всего соединений | Разная длительность запросов |
| IP Hash | По IP клиента | Sticky sessions |
| Consistent Hashing | По хешу ключа | Распределённые кеши |

### Кэширование

```
┌────────────────────────────────────────────────────────────────┐
│                     УРОВНИ КЭШИРОВАНИЯ                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Client  ───▶  CDN  ───▶  Load Balancer  ───▶  App Cache      │
│                                                  (Redis)        │
│                                                     │           │
│                                                     ▼           │
│                                                  Database       │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Чем ближе к клиенту — тем быстрее, но сложнее инвалидировать
```

**Стратегии кэширования:**

```go
// 1. Cache-Aside (Lazy Loading)
func GetUser(id string) (*User, error) {
    // Сначала проверяем кэш
    if user := cache.Get(id); user != nil {
        return user, nil
    }

    // Если нет — читаем из БД
    user, err := db.GetUser(id)
    if err != nil {
        return nil, err
    }

    // Сохраняем в кэш
    cache.Set(id, user, TTL)
    return user, nil
}

// 2. Write-Through
func UpdateUser(user *User) error {
    // Сначала обновляем кэш
    cache.Set(user.ID, user, TTL)

    // Потом БД
    return db.UpdateUser(user)
}

// 3. Write-Behind (Write-Back)
func UpdateUser(user *User) error {
    // Только кэш, БД обновится асинхронно
    cache.Set(user.ID, user, TTL)
    queue.Push(user)  // Асинхронная запись в БД
    return nil
}
```

### Database Sharding

```
┌─────────────────────────────────────────────────────────────────┐
│                        SHARDING                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  User ID: 1-1000000                                             │
│                                                                  │
│  Shard Key: user_id % 3                                         │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Shard 0    │  │   Shard 1    │  │   Shard 2    │          │
│  │ user_id % 3  │  │ user_id % 3  │  │ user_id % 3  │          │
│  │     = 0      │  │     = 1      │  │     = 2      │          │
│  │              │  │              │  │              │          │
│  │ IDs: 3,6,9.. │  │ IDs: 1,4,7.. │  │ IDs: 2,5,8.. │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Типы шардинга:**

| Тип | Плюсы | Минусы |
|-----|-------|--------|
| Range-based | Простота, range queries | Hotspots |
| Hash-based | Равномерное распределение | Нет range queries |
| Directory-based | Гибкость | Lookup overhead |
| Geo-based | Локальность | Сложность миграции |

### Репликация

```
┌─────────────────────────────────────────────────────────────────┐
│                    MASTER-SLAVE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│         ┌──────────┐                                            │
│         │  Master  │ ◄─── Writes                                │
│         └────┬─────┘                                            │
│              │ Replication                                       │
│      ┌───────┼───────┐                                          │
│      ▼       ▼       ▼                                          │
│  ┌───────┐┌───────┐┌───────┐                                    │
│  │ Slave ││ Slave ││ Slave │ ◄─── Reads                         │
│  └───────┘└───────┘└───────┘                                    │
│                                                                  │
│  + Read scalability                                              │
│  + Failover (promote slave to master)                           │
│  - Write bottleneck                                              │
│  - Replication lag                                               │
└─────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   MASTER-MASTER                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    ┌──────────┐         ┌──────────┐                            │
│    │ Master 1 │ ◄─────▶ │ Master 2 │                            │
│    └──────────┘         └──────────┘                            │
│         ▲                    ▲                                   │
│         │                    │                                   │
│     Writes/Reads         Writes/Reads                           │
│                                                                  │
│  + Write scalability                                             │
│  + No single point of failure                                    │
│  - Conflict resolution                                           │
│  - Complexity                                                    │
└─────────────────────────────────────────────────────────────────┘
```

## Паттерны распределённых систем

### Circuit Breaker

```go
type CircuitBreaker struct {
    failures    int
    threshold   int
    state       State  // Closed, Open, HalfOpen
    lastFailure time.Time
    timeout     time.Duration
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    if cb.state == Open {
        if time.Since(cb.lastFailure) > cb.timeout {
            cb.state = HalfOpen
        } else {
            return ErrCircuitOpen
        }
    }

    err := fn()
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.threshold {
            cb.state = Open
        }
        return err
    }

    cb.failures = 0
    cb.state = Closed
    return nil
}
```

```
     ┌──────────────────────────────────────────────────────┐
     │                  CIRCUIT BREAKER                      │
     └──────────────────────────────────────────────────────┘

         success              failure threshold
     ┌─────────────┐       ┌─────────────────┐
     │             │       │                 │
     │   CLOSED    │──────▶│      OPEN       │
     │             │       │                 │
     └─────────────┘       └────────┬────────┘
           ▲                        │
           │                        │ timeout
           │ success                ▼
           │               ┌─────────────────┐
           │               │                 │
           └───────────────│   HALF-OPEN     │
                           │                 │
                           └─────────────────┘
                                   │
                                   │ failure
                                   ▼
                              back to OPEN
```

### Retry с Exponential Backoff

```go
func RetryWithBackoff(fn func() error, maxRetries int) error {
    var err error
    for i := 0; i < maxRetries; i++ {
        err = fn()
        if err == nil {
            return nil
        }

        // Exponential backoff: 1s, 2s, 4s, 8s...
        delay := time.Duration(1<<i) * time.Second
        // Добавляем jitter чтобы избежать thundering herd
        jitter := time.Duration(rand.Intn(1000)) * time.Millisecond
        time.Sleep(delay + jitter)
    }
    return err
}
```

### Rate Limiting

```go
// Token Bucket
type TokenBucket struct {
    tokens     float64
    capacity   float64
    rate       float64  // tokens per second
    lastUpdate time.Time
    mu         sync.Mutex
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(tb.lastUpdate).Seconds()
    tb.tokens = min(tb.capacity, tb.tokens + elapsed*tb.rate)
    tb.lastUpdate = now

    if tb.tokens >= 1 {
        tb.tokens--
        return true
    }
    return false
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                      RATE LIMITING                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Token Bucket:                                                   │
│  ┌─────────────────┐                                            │
│  │ ● ● ● ● ○ ○ ○ ○ │  capacity=8, tokens=4                      │
│  └─────────────────┘                                            │
│     ↑                   ↓                                        │
│  +rate/sec         -1 per request                                │
│                                                                  │
│  Leaky Bucket:                                                   │
│  ┌─────────────────┐                                            │
│  │ ● ● ● ● ● ● ● ● │  requests queue                            │
│  └────────┬────────┘                                            │
│           │ fixed rate out                                       │
│           ▼                                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Идемпотентность

```go
// Idempotency Key для предотвращения дублирования
type PaymentService struct {
    processed map[string]bool  // idempotency_key -> bool
    mu        sync.Mutex
}

func (s *PaymentService) ProcessPayment(idempotencyKey string, amount int) error {
    s.mu.Lock()
    defer s.mu.Unlock()

    // Проверяем, обрабатывался ли уже этот запрос
    if s.processed[idempotencyKey] {
        return nil  // Уже обработано — возвращаем успех
    }

    // Обрабатываем платёж
    if err := s.doPayment(amount); err != nil {
        return err
    }

    // Помечаем как обработанный
    s.processed[idempotencyKey] = true
    return nil
}
```

## Оценка нагрузки (Back-of-Envelope)

### Полезные числа

```
Latency:
- L1 cache:           ~1 ns
- L2 cache:           ~4 ns
- RAM:                ~100 ns
- SSD random read:    ~100 μs
- HDD random read:    ~10 ms
- Network (same DC):  ~0.5 ms
- Network (cross DC): ~100 ms

Throughput:
- SSD sequential:     ~500 MB/s
- HDD sequential:     ~100 MB/s
- 1 Gbps network:     ~100 MB/s
- 10 Gbps network:    ~1 GB/s

Storage:
- 1 char = 1 byte (ASCII)
- 1 char = 2-4 bytes (UTF-8)
- UUID = 36 chars = 36 bytes
- Timestamp = 8 bytes
- Average tweet = ~300 bytes
- Average photo = ~500 KB
- Average video = ~50 MB/min
```

### Пример расчёта

```
Задача: Twitter-like система

Требования:
- 100M DAU (daily active users)
- Каждый пользователь читает 100 tweets/день
- 10% пользователей пишут 2 tweets/день

Расчёт QPS:
- Read QPS = 100M * 100 / 86400 ≈ 115,000 QPS
- Write QPS = 100M * 0.1 * 2 / 86400 ≈ 230 QPS
- Peak QPS = 3x average ≈ 350,000 read QPS

Расчёт хранилища (в год):
- Tweets/год = 100M * 0.1 * 2 * 365 = 7.3B tweets
- Размер = 7.3B * 300 bytes = 2.2 TB/год
- С репликацией (3x) = 6.6 TB/год
```

## Типовые архитектуры

### URL Shortener

```
┌─────────────────────────────────────────────────────────────────┐
│                      URL SHORTENER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Client ──▶ LB ──▶ API Servers ──▶ Cache (Redis)               │
│                          │              │                        │
│                          ▼              ▼                        │
│                    ID Generator    Database                      │
│                    (Counter/UUID)  (short_url → long_url)       │
│                                                                  │
│   Алгоритм:                                                      │
│   1. Генерируем unique ID                                        │
│   2. Конвертируем в base62 (a-zA-Z0-9)                          │
│   3. Сохраняем mapping                                           │
│                                                                  │
│   ID: 12345 → Base62: "dnh"                                     │
│   short.url/dnh → example.com/very-long-url                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Rate Limiter

```
┌─────────────────────────────────────────────────────────────────┐
│                      RATE LIMITER                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Client ──▶ Rate Limiter ──▶ API Servers                       │
│                   │                                              │
│                   ▼                                              │
│              Redis                                               │
│         (user_id → tokens)                                       │
│                                                                  │
│   Алгоритмы:                                                     │
│   - Token Bucket (burst friendly)                                │
│   - Leaky Bucket (smooth output)                                 │
│   - Sliding Window (accurate)                                    │
│   - Fixed Window (simple)                                        │
│                                                                  │
│   Redis команды:                                                 │
│   INCR user:123:requests                                         │
│   EXPIRE user:123:requests 60                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Distributed Cache

```
┌─────────────────────────────────────────────────────────────────┐
│                   DISTRIBUTED CACHE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Consistent Hashing:                                            │
│                                                                  │
│              Node A                                              │
│                ●                                                 │
│           /        \                                             │
│          /          \                                            │
│      ●───────────────────●                                       │
│   Node D                 Node B                                  │
│          \          /                                            │
│           \        /                                             │
│                ●                                                 │
│              Node C                                              │
│                                                                  │
│   key.hash() → позиция на кольце → ближайший node по часовой    │
│                                                                  │
│   Virtual Nodes: каждый физический node = 100-200 virtual       │
│   → равномерное распределение                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Вопросы с собеседований

### Вопрос 1: Как масштабировать базу данных?

<details>
<summary>Ответ</summary>

**Вертикально:**
- Больше RAM, CPU, SSD
- Ограничено железом

**Горизонтально:**
1. **Read replicas** — для read-heavy нагрузки
2. **Sharding** — разбиение по ключу
3. **Partitioning** — разбиение по range/hash

**Дополнительно:**
- Кэширование (Redis)
- Денормализация
- Архивирование старых данных

</details>

### Вопрос 2: CAP теорема — что выбрать?

<details>
<summary>Ответ</summary>

**CAP:** Consistency, Availability, Partition tolerance — можно выбрать только 2 из 3.

**В реальности:** Partition tolerance обязателен (сеть ненадёжна), выбор между C и A.

- **CP (Consistency):** Банковские системы, inventory
- **AP (Availability):** Социальные сети, кэши

Многие системы позволяют настраивать per-operation (eventual vs strong consistency).

</details>

### Вопрос 3: Как спроектировать Rate Limiter?

<details>
<summary>Ответ</summary>

**Алгоритмы:**
1. Token Bucket — хорош для burst
2. Leaky Bucket — smooth output
3. Sliding Window — точный подсчёт
4. Fixed Window — простой, но имеет edge cases

**Распределённый:**
- Redis для хранения счётчиков
- Lua scripts для атомарности
- Consistent hashing для шардинга

**HTTP заголовки:**
- X-RateLimit-Limit
- X-RateLimit-Remaining
- Retry-After

</details>

### Вопрос 4: Как обеспечить отказоустойчивость?

<details>
<summary>Ответ</summary>

1. **Репликация** — данные на нескольких серверах
2. **Load Balancer** — распределение и failover
3. **Circuit Breaker** — изоляция сбоев
4. **Retry с backoff** — повторные попытки
5. **Graceful degradation** — частичная работа при сбоях
6. **Health checks** — мониторинг состояния
7. **Multi-region** — географическое распределение

</details>

### Вопрос 5: Когда использовать синхронную vs асинхронную коммуникацию?

<details>
<summary>Ответ</summary>

**Синхронная (REST, gRPC):**
- Нужен немедленный ответ
- Простые CRUD операции
- Транзакции

**Асинхронная (Kafka, RabbitMQ):**
- Длительные операции
- Decoupling сервисов
- Пиковые нагрузки (буферизация)
- Event-driven архитектура
- Гарантия доставки

</details>

## Связанные темы

- [[Микросервисная архитектура]]
- [[CAP теорема]]
- [[Redis - Кэширование стратегии]]
- [[Apache Kafka]]
- [[PostgreSQL - Оптимизация запросов]]
- [[Задача - Rate Limiter]]
- [[Clean Architecture]]
