# Задача - Retry with Backoff

Важный паттерн для работы с внешними сервисами. Спрашивают в Авито, Т-Банк, Ozon. Проверяет понимание обработки ошибок, таймаутов и защиты от перегрузки.

## Условие задачи

```
Реализовать функцию retry с экспоненциальным backoff.
Требования:
1. Повторять операцию при ошибке до maxRetries раз
2. Увеличивать задержку между попытками экспоненциально
3. Добавить jitter (случайный разброс) для предотвращения thundering herd
4. Поддержать отмену через context
5. Различать retriable и non-retriable ошибки
```

## Решение 1: Базовый Exponential Backoff

```go
func Retry(ctx context.Context, maxRetries int, fn func() error) error {
    var err error
    baseDelay := 100 * time.Millisecond
    maxDelay := 10 * time.Second

    for attempt := 0; attempt <= maxRetries; attempt++ {
        err = fn()
        if err == nil {
            return nil
        }

        if attempt == maxRetries {
            break
        }

        // Экспоненциальная задержка: 100ms, 200ms, 400ms, 800ms...
        delay := baseDelay * time.Duration(1<<attempt)
        if delay > maxDelay {
            delay = maxDelay
        }

        select {
        case <-time.After(delay):
            // Продолжаем
        case <-ctx.Done():
            return fmt.Errorf("retry cancelled: %w", ctx.Err())
        }
    }

    return fmt.Errorf("max retries exceeded: %w", err)
}
```

## Решение 2: С Jitter (рандомизация)

```go
func RetryWithJitter(ctx context.Context, maxRetries int, fn func() error) error {
    var err error
    baseDelay := 100 * time.Millisecond
    maxDelay := 10 * time.Second

    for attempt := 0; attempt <= maxRetries; attempt++ {
        err = fn()
        if err == nil {
            return nil
        }

        if attempt == maxRetries {
            break
        }

        // Экспоненциальная задержка
        delay := baseDelay * time.Duration(1<<attempt)
        if delay > maxDelay {
            delay = maxDelay
        }

        // Добавляем jitter: ±25%
        jitter := time.Duration(rand.Int63n(int64(delay / 2)))
        delay = delay - delay/4 + jitter

        select {
        case <-time.After(delay):
        case <-ctx.Done():
            return fmt.Errorf("retry cancelled: %w", ctx.Err())
        }
    }

    return fmt.Errorf("max retries exceeded: %w", err)
}
```

## Решение 3: С конфигурацией и retriable ошибками

```go
type RetryConfig struct {
    MaxRetries  int
    BaseDelay   time.Duration
    MaxDelay    time.Duration
    Multiplier  float64
    JitterRatio float64 // 0.0 - 1.0
}

func DefaultConfig() RetryConfig {
    return RetryConfig{
        MaxRetries:  3,
        BaseDelay:   100 * time.Millisecond,
        MaxDelay:    10 * time.Second,
        Multiplier:  2.0,
        JitterRatio: 0.25,
    }
}

// RetriableError интерфейс для определения можно ли повторять
type RetriableError interface {
    error
    IsRetriable() bool
}

type retriableError struct {
    err       error
    retriable bool
}

func (e *retriableError) Error() string    { return e.err.Error() }
func (e *retriableError) IsRetriable() bool { return e.retriable }
func (e *retriableError) Unwrap() error    { return e.err }

func NewRetriableError(err error) error {
    return &retriableError{err: err, retriable: true}
}

func NewNonRetriableError(err error) error {
    return &retriableError{err: err, retriable: false}
}

func RetryWithConfig(ctx context.Context, cfg RetryConfig, fn func() error) error {
    var err error
    delay := cfg.BaseDelay

    for attempt := 0; attempt <= cfg.MaxRetries; attempt++ {
        err = fn()
        if err == nil {
            return nil
        }

        // Проверяем, можно ли повторять
        if re, ok := err.(RetriableError); ok && !re.IsRetriable() {
            return err // Не повторяем
        }

        if attempt == cfg.MaxRetries {
            break
        }

        // Вычисляем задержку с jitter
        actualDelay := calculateDelay(delay, cfg.MaxDelay, cfg.JitterRatio)

        select {
        case <-time.After(actualDelay):
        case <-ctx.Done():
            return fmt.Errorf("retry cancelled: %w", ctx.Err())
        }

        // Увеличиваем базовую задержку
        delay = time.Duration(float64(delay) * cfg.Multiplier)
    }

    return fmt.Errorf("max retries (%d) exceeded: %w", cfg.MaxRetries, err)
}

func calculateDelay(base, max time.Duration, jitterRatio float64) time.Duration {
    delay := base
    if delay > max {
        delay = max
    }

    if jitterRatio > 0 {
        jitter := time.Duration(rand.Float64() * jitterRatio * float64(delay))
        delay = delay + jitter
    }

    return delay
}
```

## Решение 4: Функциональный стиль с опциями

```go
type RetryOption func(*retryOptions)

type retryOptions struct {
    maxRetries    int
    baseDelay     time.Duration
    maxDelay      time.Duration
    isRetriable   func(error) bool
    onRetry       func(attempt int, err error, delay time.Duration)
}

func WithMaxRetries(n int) RetryOption {
    return func(o *retryOptions) { o.maxRetries = n }
}

func WithBaseDelay(d time.Duration) RetryOption {
    return func(o *retryOptions) { o.baseDelay = d }
}

func WithMaxDelay(d time.Duration) RetryOption {
    return func(o *retryOptions) { o.maxDelay = d }
}

func WithRetriableCheck(fn func(error) bool) RetryOption {
    return func(o *retryOptions) { o.isRetriable = fn }
}

func WithOnRetry(fn func(int, error, time.Duration)) RetryOption {
    return func(o *retryOptions) { o.onRetry = fn }
}

func Do(ctx context.Context, fn func() error, opts ...RetryOption) error {
    // Значения по умолчанию
    o := &retryOptions{
        maxRetries:  3,
        baseDelay:   100 * time.Millisecond,
        maxDelay:    30 * time.Second,
        isRetriable: func(error) bool { return true },
    }

    for _, opt := range opts {
        opt(o)
    }

    var err error
    delay := o.baseDelay

    for attempt := 0; attempt <= o.maxRetries; attempt++ {
        err = fn()
        if err == nil {
            return nil
        }

        if !o.isRetriable(err) {
            return err
        }

        if attempt == o.maxRetries {
            break
        }

        // Jitter
        actualDelay := delay + time.Duration(rand.Int63n(int64(delay/4)))
        if actualDelay > o.maxDelay {
            actualDelay = o.maxDelay
        }

        if o.onRetry != nil {
            o.onRetry(attempt+1, err, actualDelay)
        }

        select {
        case <-time.After(actualDelay):
        case <-ctx.Done():
            return ctx.Err()
        }

        delay *= 2
    }

    return fmt.Errorf("max retries exceeded: %w", err)
}
```

## Решение 5: Generic версия

```go
func RetryWithResult[T any](ctx context.Context, maxRetries int, fn func() (T, error)) (T, error) {
    var zero T
    var result T
    var err error

    baseDelay := 100 * time.Millisecond

    for attempt := 0; attempt <= maxRetries; attempt++ {
        result, err = fn()
        if err == nil {
            return result, nil
        }

        if attempt == maxRetries {
            break
        }

        delay := baseDelay * time.Duration(1<<attempt)

        select {
        case <-time.After(delay):
        case <-ctx.Done():
            return zero, ctx.Err()
        }
    }

    return zero, fmt.Errorf("max retries exceeded: %w", err)
}
```

## Пример использования

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // Базовое использование
    err := Retry(ctx, 3, func() error {
        return fetchData()
    })

    // С конфигурацией
    cfg := RetryConfig{
        MaxRetries:  5,
        BaseDelay:   200 * time.Millisecond,
        MaxDelay:    5 * time.Second,
        Multiplier:  2.0,
        JitterRatio: 0.25,
    }

    err = RetryWithConfig(ctx, cfg, func() error {
        return callExternalAPI()
    })

    // Функциональный стиль
    err = Do(ctx, func() error {
        return sendRequest()
    },
        WithMaxRetries(5),
        WithBaseDelay(500*time.Millisecond),
        WithRetriableCheck(isRetriableHTTPError),
        WithOnRetry(func(attempt int, err error, delay time.Duration) {
            log.Printf("Retry %d: %v (waiting %v)", attempt, err, delay)
        }),
    )

    // Generic версия
    result, err := RetryWithResult(ctx, 3, func() (string, error) {
        return httpGet("https://api.example.com/data")
    })
}

func isRetriableHTTPError(err error) bool {
    // 5xx — можно повторять
    // 4xx — нельзя (кроме 429 Too Many Requests)
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        return httpErr.StatusCode >= 500 || httpErr.StatusCode == 429
    }
    // Сетевые ошибки — можно повторять
    return true
}
```

## Визуализация задержек

```
Exponential Backoff с jitter:

Attempt  Base Delay   With Jitter (±25%)
   1       100ms        75ms - 125ms
   2       200ms       150ms - 250ms
   3       400ms       300ms - 500ms
   4       800ms       600ms - 1000ms
   5      1600ms      1200ms - 2000ms
   ...

Без jitter (thundering herd):          С jitter (распределённые запросы):
   │                                      │
   ▼▼▼▼▼▼ (все клиенты разом)            ▼ ▼  ▼ ▼ ▼  ▼ (распределены)
   │                                      │
```

## Типичные ошибки

### Ошибка 1: Retry без backoff

```go
// ❌ НЕПРАВИЛЬНО: DDos своего сервера
for i := 0; i < maxRetries; i++ {
    err := fn()
    if err == nil {
        return nil
    }
    // Нет задержки!
}

// ✅ ПРАВИЛЬНО: экспоненциальная задержка
for i := 0; i < maxRetries; i++ {
    err := fn()
    if err == nil {
        return nil
    }
    time.Sleep(baseDelay * time.Duration(1<<i))
}
```

### Ошибка 2: Retry non-retriable ошибок

```go
// ❌ НЕПРАВИЛЬНО: 404 не исправится повторным запросом
err := Retry(ctx, 5, func() error {
    resp, err := http.Get(url)
    if resp.StatusCode == 404 {
        return errors.New("not found") // Будет ретраиться!
    }
    return nil
})

// ✅ ПРАВИЛЬНО: проверяем тип ошибки
err := Do(ctx, fn, WithRetriableCheck(func(err error) bool {
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        return httpErr.StatusCode >= 500 // Только 5xx
    }
    return true
}))
```

### Ошибка 3: Забыть про context

```go
// ❌ НЕПРАВИЛЬНО: игнорируем отмену
for i := 0; i < maxRetries; i++ {
    err := fn()
    if err == nil {
        return nil
    }
    time.Sleep(delay) // Не проверяем ctx!
}

// ✅ ПРАВИЛЬНО: проверяем context
select {
case <-time.After(delay):
case <-ctx.Done():
    return ctx.Err()
}
```

### Ошибка 4: Одинаковый jitter для всех клиентов

```go
// ❌ НЕПРАВИЛЬНО: seed не инициализирован — все клиенты получат одинаковый jitter
jitter := rand.Int63n(100)

// ✅ ПРАВИЛЬНО: инициализировать генератор или использовать crypto/rand
func init() {
    rand.Seed(time.Now().UnixNano())
}

// Или лучше — использовать rand.New с отдельным источником
rng := rand.New(rand.NewSource(time.Now().UnixNano()))
```

## Стратегии Backoff

### 1. Constant Backoff

```go
delay := 1 * time.Second // Всегда одинаковая задержка
```

### 2. Linear Backoff

```go
delay := baseDelay * time.Duration(attempt) // 1s, 2s, 3s, 4s...
```

### 3. Exponential Backoff

```go
delay := baseDelay * time.Duration(1<<attempt) // 1s, 2s, 4s, 8s...
```

### 4. Decorrelated Jitter (AWS рекомендация)

```go
// Лучше распределяет нагрузку чем обычный jitter
sleep := min(maxDelay, rand.Int63n(int64(prevSleep*3)))
```

## Вопросы с собеседований

### Вопрос 1: Зачем нужен jitter?

**Ответ:**

**Thundering Herd Problem**: Если много клиентов упали одновременно и используют одинаковый backoff без jitter, они все будут ретраиться в одно время, создавая пики нагрузки.

Jitter рандомизирует время retry, распределяя нагрузку равномернее.

```
Без jitter:     ▼▼▼▼▼▼▼▼▼▼ (spike)
С jitter:       ▼  ▼ ▼  ▼ ▼  ▼ (distributed)
```

---

### Вопрос 2: Какие ошибки стоит ретраить?

**Ответ:**

**Retriable (временные):**
- 5xx ошибки сервера
- 429 Too Many Requests
- Таймауты
- Сетевые ошибки (connection refused, DNS failure)
- Deadlock в БД

**Non-retriable (постоянные):**
- 4xx ошибки клиента (400, 401, 403, 404)
- Ошибки валидации
- Бизнес-логические ошибки

---

### Вопрос 3: Как ограничить общее время retry?

**Ответ:**

Использовать context с timeout:

```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

err := Retry(ctx, 10, fn)
// Даже если 10 попыток не завершились, через 30 секунд прервётся
```

---

## Связанные темы

- [[Go - Context]]
- [[Go - Пакет time]]
- [[Задача - Circuit Breaker]]
- [[Задача - Timeout pattern]]
- [[Go - Обработка ошибок]]
- [[System Design - Основы]]
