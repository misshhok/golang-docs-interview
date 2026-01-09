# Задача - Rate Limiter

Реализовать ограничитель частоты запросов (Rate Limiter).

## Условие

Разработайте Rate Limiter, который ограничивает количество запросов в единицу времени.

**Требования:**
- Ограничить N запросов в единицу времени (например, 10 запросов в секунду)
- Потокобезопасность
- Эффективность

## Алгоритмы

### 1. Token Bucket (Корзина токенов)

**Принцип:**
- Корзина вмещает N токенов
- Токены добавляются с постоянной скоростью (rate)
- Запрос забирает 1 токен
- Если токенов нет → отклонить запрос

```
Capacity = 10
Rate = 10 tokens/sec

[●●●●●●●●●●] → Request → [●●●●●●●●●_]
   10 tokens              9 tokens

После 0.1 sec: [●●●●●●●●●●] (пополнили до capacity)
```

#### Реализация

```go
type TokenBucket struct {
    capacity  int64         // Максимум токенов
    tokens    int64         // Текущие токены
    rate      int64         // Токенов в секунду
    lastTime  time.Time     // Последнее обновление
    mu        sync.Mutex
}

func NewTokenBucket(capacity, rate int64) *TokenBucket {
    return &TokenBucket{
        capacity: capacity,
        tokens:   capacity,
        rate:     rate,
        lastTime: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(tb.lastTime).Seconds()

    // Добавить токены за прошедшее время
    tb.tokens += int64(elapsed * float64(tb.rate))
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }

    tb.lastTime = now

    // Проверить наличие токена
    if tb.tokens >= 1 {
        tb.tokens--
        return true
    }

    return false
}

// Использование
func main() {
    limiter := NewTokenBucket(10, 10)  // 10 req/sec

    for i := 0; i < 15; i++ {
        if limiter.Allow() {
            fmt.Println("Request", i, "allowed")
        } else {
            fmt.Println("Request", i, "rate limited")
        }
        time.Sleep(50 * time.Millisecond)
    }
}
```

### 2. Leaky Bucket (Дырявая корзина)

**Принцип:**
- Запросы попадают в очередь (корзину)
- Обрабатываются с постоянной скоростью
- Если очередь полна → отклонить

```
Input Rate: Variable
Output Rate: Constant (rate)

Requests → [Queue] → Process at fixed rate
```

#### Реализация

```go
type LeakyBucket struct {
    capacity int
    rate     time.Duration
    queue    chan struct{}
}

func NewLeakyBucket(capacity int, rate time.Duration) *LeakyBucket {
    lb := &LeakyBucket{
        capacity: capacity,
        rate:     rate,
        queue:    make(chan struct{}, capacity),
    }

    go lb.leak()

    return lb
}

func (lb *LeakyBucket) leak() {
    ticker := time.NewTicker(lb.rate)
    defer ticker.Stop()

    for range ticker.C {
        select {
        case <-lb.queue:
            // Обработан один запрос
        default:
            // Очередь пуста
        }
    }
}

func (lb *LeakyBucket) Allow() bool {
    select {
    case lb.queue <- struct{}{}:
        return true
    default:
        return false  // Очередь полна
    }
}
```

### 3. Fixed Window Counter

**Принцип:**
- Разбиваем время на фиксированные окна (например, минуты)
- Считаем запросы в текущем окне
- Если превышен лимит → отклонить

```
Window 1 (00:00-00:59): 10 requests ✅
Window 2 (01:00-01:59): 12 requests ❌ (limit exceeded)
```

#### Реализация

```go
type FixedWindowCounter struct {
    limit       int
    windowSize  time.Duration
    currentTime time.Time
    count       int
    mu          sync.Mutex
}

func NewFixedWindowCounter(limit int, windowSize time.Duration) *FixedWindowCounter {
    return &FixedWindowCounter{
        limit:       limit,
        windowSize:  windowSize,
        currentTime: time.Now(),
        count:       0,
    }
}

func (fwc *FixedWindowCounter) Allow() bool {
    fwc.mu.Lock()
    defer fwc.mu.Unlock()

    now := time.Now()

    // Новое окно?
    if now.Sub(fwc.currentTime) >= fwc.windowSize {
        fwc.currentTime = now
        fwc.count = 0
    }

    if fwc.count < fwc.limit {
        fwc.count++
        return true
    }

    return false
}
```

**Проблема:** Burst в границе окна

```
00:50 - 10 requests ✅
01:10 - 10 requests ✅
Итого 20 requests за 20 секунд! (должно быть 10/min)
```

### 4. Sliding Window Log

**Принцип:**
- Храним timestamp каждого запроса
- Удаляем старые (вне окна)
- Считаем количество в окне

```
Window = 1 min
Requests: [10:00:10, 10:00:30, 10:01:00, 10:01:20]

At 10:01:30:
Remove 10:00:10 (> 1 min ago)
Count: 3 requests in last minute
```

#### Реализация

```go
type SlidingWindowLog struct {
    limit      int
    windowSize time.Duration
    requests   []time.Time
    mu         sync.Mutex
}

func NewSlidingWindowLog(limit int, windowSize time.Duration) *SlidingWindowLog {
    return &SlidingWindowLog{
        limit:      limit,
        windowSize: windowSize,
        requests:   []time.Time{},
    }
}

func (swl *SlidingWindowLog) Allow() bool {
    swl.mu.Lock()
    defer swl.mu.Unlock()

    now := time.Now()
    cutoff := now.Add(-swl.windowSize)

    // Удалить старые запросы
    validRequests := []time.Time{}
    for _, t := range swl.requests {
        if t.After(cutoff) {
            validRequests = append(validRequests, t)
        }
    }
    swl.requests = validRequests

    // Проверить лимит
    if len(swl.requests) < swl.limit {
        swl.requests = append(swl.requests, now)
        return true
    }

    return false
}
```

**Проблема:** O(n) память (для каждого запроса)

### 5. Sliding Window Counter (Гибрид)

**Принцип:**
- Комбинация Fixed Window + Sliding
- Взвешенная сумма текущего и предыдущего окна

```
Previous window: 80 requests
Current window: 20 requests
Position in current: 70%

Estimate = 80 × (1 - 0.7) + 20 = 24 + 20 = 44
```

#### Реализация

```go
type SlidingWindowCounter struct {
    limit      int
    windowSize time.Duration
    prevCount  int
    currCount  int
    prevTime   time.Time
    currTime   time.Time
    mu         sync.Mutex
}

func NewSlidingWindowCounter(limit int, windowSize time.Duration) *SlidingWindowCounter {
    now := time.Now()
    return &SlidingWindowCounter{
        limit:      limit,
        windowSize: windowSize,
        prevTime:   now.Add(-windowSize),
        currTime:   now,
    }
}

func (swc *SlidingWindowCounter) Allow() bool {
    swc.mu.Lock()
    defer swc.mu.Unlock()

    now := time.Now()

    // Новое окно?
    if now.Sub(swc.currTime) >= swc.windowSize {
        swc.prevCount = swc.currCount
        swc.prevTime = swc.currTime
        swc.currCount = 0
        swc.currTime = now
    }

    // Вычислить взвешенное количество
    elapsed := now.Sub(swc.currTime)
    progress := float64(elapsed) / float64(swc.windowSize)

    estimate := int(float64(swc.prevCount)*(1-progress)) + swc.currCount

    if estimate < swc.limit {
        swc.currCount++
        return true
    }

    return false
}
```

## Сравнение алгоритмов

| Алгоритм | Точность | Память | Burst | Сложность |
|----------|----------|--------|-------|-----------|
| Token Bucket | ✅ | O(1) | ✅ Разрешает | Простая |
| Leaky Bucket | ✅ | O(n) | ❌ Сглаживает | Средняя |
| Fixed Window | ❌ | O(1) | ❌ Проблема границ | Простая |
| Sliding Log | ✅✅ | O(n) | ✅ | Средняя |
| Sliding Counter | ✅ | O(1) | ✅ | Средняя |

## Использование в Go

### golang.org/x/time/rate

Стандартная библиотека предоставляет Token Bucket:

```go
import "golang.org/x/time/rate"

func main() {
    // 10 запросов в секунду, burst = 5
    limiter := rate.NewLimiter(10, 5)

    // Блокирующий
    err := limiter.Wait(context.Background())
    if err != nil {
        log.Fatal(err)
    }

    // Неблокирующий
    if limiter.Allow() {
        fmt.Println("Request allowed")
    } else {
        fmt.Println("Rate limited")
    }

    // Резервировать N токенов
    reservation := limiter.Reserve()
    if !reservation.OK() {
        fmt.Println("Rate limited")
    } else {
        time.Sleep(reservation.Delay())
        // Обработать запрос
    }
}
```

### HTTP Middleware

```go
func RateLimitMiddleware(limiter *rate.Limiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

// Использование
limiter := rate.NewLimiter(10, 5)
http.Handle("/api", RateLimitMiddleware(limiter)(apiHandler))
```

### Per-User Rate Limiting

```go
type UserRateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewUserRateLimiter(r rate.Limit, b int) *UserRateLimiter {
    return &UserRateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    b,
    }
}

func (url *UserRateLimiter) GetLimiter(userID string) *rate.Limiter {
    url.mu.RLock()
    limiter, exists := url.limiters[userID]
    url.mu.RUnlock()

    if exists {
        return limiter
    }

    url.mu.Lock()
    defer url.mu.Unlock()

    // Double-check
    limiter, exists = url.limiters[userID]
    if exists {
        return limiter
    }

    limiter = rate.NewLimiter(url.rate, url.burst)
    url.limiters[userID] = limiter

    return limiter
}

func (url *UserRateLimiter) Allow(userID string) bool {
    limiter := url.GetLimiter(userID)
    return limiter.Allow()
}
```

### Distributed Rate Limiting (Redis)

```go
import "github.com/go-redis/redis/v8"

type RedisRateLimiter struct {
    client *redis.Client
    limit  int
    window time.Duration
}

func (rrl *RedisRateLimiter) Allow(ctx context.Context, key string) (bool, error) {
    now := time.Now().UnixNano()
    windowStart := now - int64(rrl.window)

    pipe := rrl.client.Pipeline()

    // Удалить старые
    pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprint(windowStart))

    // Подсчитать
    pipe.ZCard(ctx, key)

    // Добавить текущий
    pipe.ZAdd(ctx, key, &redis.Z{
        Score:  float64(now),
        Member: now,
    })

    // Установить TTL
    pipe.Expire(ctx, key, rrl.window)

    results, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }

    count := results[1].(*redis.IntCmd).Val()

    return count < int64(rrl.limit), nil
}
```

## Применение

### API Rate Limiting

```go
type APIRateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
}

func (arl *APIRateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        apiKey := r.Header.Get("X-API-Key")

        if apiKey == "" {
            http.Error(w, "Missing API Key", http.StatusUnauthorized)
            return
        }

        limiter := arl.GetLimiter(apiKey)

        if !limiter.Allow() {
            w.Header().Set("X-RateLimit-Limit", "100")
            w.Header().Set("X-RateLimit-Remaining", "0")
            w.Header().Set("Retry-After", "60")
            http.Error(w, "Rate Limit Exceeded", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

### DDoS Protection

```go
type DDoSProtection struct {
    ipLimiters *UserRateLimiter
}

func (ddos *DDoSProtection) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := getClientIP(r)

        if !ddos.ipLimiters.Allow(ip) {
            http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

## Best Practices

1. ✅ Token Bucket для большинства случаев
2. ✅ Используйте `golang.org/x/time/rate`
3. ✅ Per-user limiting для API
4. ✅ Redis для distributed системы
5. ✅ Graceful degradation (503 вместо drop)
6. ✅ Возвращайте Retry-After header
7. ❌ Не забывайте про cleanup старых limiters
8. ❌ Тестируйте с realistic load

## Связанные темы

- [[Go - Горутины (goroutines)]]
- [[Go - Каналы (channels)]]
- [[Go - Пакет sync]]
- [[Go - Context]]
- [[Алгоритмическая сложность (Big O)]]
