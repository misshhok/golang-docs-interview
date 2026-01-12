# Задача - First Response

Популярный паттерн на собеседованиях. Проверяет понимание select, context и гонки горутин для получения первого успешного результата.

## Условие задачи

```
Реализовать функцию, которая отправляет запрос к нескольким серверам
и возвращает первый успешный ответ.
Требования:
1. Запросы выполняются параллельно
2. Возвращается первый успешный результат
3. Остальные запросы отменяются
4. Обработка ошибок (что если все запросы упали?)
```

```go
func FirstResponse(ctx context.Context, urls []string) (string, error)
```

## Решение 1: Базовое с select

```go
func FirstResponse(ctx context.Context, urls []string) (string, error) {
    if len(urls) == 0 {
        return "", errors.New("no urls provided")
    }

    ctx, cancel := context.WithCancel(ctx)
    defer cancel() // Отменяем остальные запросы

    type result struct {
        body string
        err  error
    }

    results := make(chan result, len(urls))

    for _, url := range urls {
        go func(u string) {
            body, err := fetch(ctx, u)
            results <- result{body, err}
        }(url)
    }

    // Ждём первый успешный
    var lastErr error
    for i := 0; i < len(urls); i++ {
        select {
        case r := <-results:
            if r.err == nil {
                return r.body, nil // Первый успешный!
            }
            lastErr = r.err
        case <-ctx.Done():
            return "", ctx.Err()
        }
    }

    return "", fmt.Errorf("all requests failed: %w", lastErr)
}

func fetch(ctx context.Context, url string) (string, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return "", err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    return string(body), err
}
```

## Решение 2: С errgroup и отменой

```go
import "golang.org/x/sync/errgroup"

func FirstResponseErrgroup(ctx context.Context, urls []string) (string, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    var result atomic.Value
    g, ctx := errgroup.WithContext(ctx)

    for _, url := range urls {
        url := url
        g.Go(func() error {
            body, err := fetch(ctx, url)
            if err != nil {
                return nil // Не прерываем, пробуем другие
            }

            // Первый успешный сохраняем и отменяем остальных
            if result.CompareAndSwap(nil, body) {
                cancel()
            }
            return nil
        })
    }

    g.Wait()

    if val := result.Load(); val != nil {
        return val.(string), nil
    }
    return "", errors.New("all requests failed")
}
```

## Решение 3: Race pattern (чистый)

```go
func Race(ctx context.Context, urls []string) (string, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    winner := make(chan string, 1) // Буфер 1 для неблокирующей записи
    errs := make(chan error, len(urls))

    for _, url := range urls {
        go func(u string) {
            body, err := fetch(ctx, u)
            if err != nil {
                errs <- err
                return
            }

            // Пытаемся записать — только первый успеет
            select {
            case winner <- body:
                cancel() // Отменяем остальных
            default:
                // Кто-то уже победил
            }
        }(url)
    }

    // Ждём победителя или все ошибки
    errCount := 0
    for {
        select {
        case body := <-winner:
            return body, nil
        case <-errs:
            errCount++
            if errCount == len(urls) {
                return "", errors.New("all requests failed")
            }
        case <-ctx.Done():
            return "", ctx.Err()
        }
    }
}
```

## Решение 4: С reflect.Select для динамического количества каналов

```go
func FirstResponseReflect(ctx context.Context, urls []string) (string, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    // Создаём канал для каждого URL
    channels := make([]chan string, len(urls))
    for i := range channels {
        channels[i] = make(chan string, 1)
    }

    // Запускаем горутины
    for i, url := range urls {
        go func(idx int, u string) {
            body, err := fetch(ctx, u)
            if err == nil {
                channels[idx] <- body
            }
            close(channels[idx])
        }(i, url)
    }

    // Создаём cases для reflect.Select
    cases := make([]reflect.SelectCase, len(channels)+1)
    for i, ch := range channels {
        cases[i] = reflect.SelectCase{
            Dir:  reflect.SelectRecv,
            Chan: reflect.ValueOf(ch),
        }
    }
    // Добавляем ctx.Done()
    cases[len(channels)] = reflect.SelectCase{
        Dir:  reflect.SelectRecv,
        Chan: reflect.ValueOf(ctx.Done()),
    }

    // Ждём первый результат
    for remaining := len(channels); remaining > 0; {
        chosen, value, ok := reflect.Select(cases)

        if chosen == len(channels) { // ctx.Done()
            return "", ctx.Err()
        }

        if ok && value.String() != "" {
            return value.String(), nil
        }

        // Канал закрыт без результата — убираем из выборки
        cases[chosen].Chan = reflect.ValueOf(nil)
        remaining--
    }

    return "", errors.New("all requests failed")
}
```

## Решение 5: С таймаутом для каждого запроса

```go
func FirstResponseWithTimeout(urls []string, timeout time.Duration) (string, error) {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    type result struct {
        url  string
        body string
        err  error
        took time.Duration
    }

    results := make(chan result, len(urls))

    for _, url := range urls {
        go func(u string) {
            start := time.Now()
            body, err := fetch(ctx, u)
            results <- result{
                url:  u,
                body: body,
                err:  err,
                took: time.Since(start),
            }
        }(url)
    }

    var lastErr error
    for i := 0; i < len(urls); i++ {
        select {
        case r := <-results:
            if r.err == nil {
                log.Printf("Winner: %s (took %v)", r.url, r.took)
                return r.body, nil
            }
            lastErr = r.err
        case <-ctx.Done():
            return "", fmt.Errorf("timeout: %w", lastErr)
        }
    }

    return "", lastErr
}
```

## Пример использования

```go
func main() {
    urls := []string{
        "https://api1.example.com/data",
        "https://api2.example.com/data",
        "https://api3.example.com/data",
    }

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    body, err := FirstResponse(ctx, urls)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("Got response: %s\n", body[:100])
}
```

## Визуализация

```
URLs: [url1, url2, url3]

Время ──────────────────────────────────────▶

url1: │████████████████│ (600ms) - слишком долго
url2: │████│✓           - ПОБЕДИТЕЛЬ (200ms)
url3: │████████│✗       - ошибка (400ms)

              │
              ▼
         cancel() — отменяем url1 и url3

Результат: ответ от url2
```

## Типичные ошибки

### Ошибка 1: Не отменять остальные запросы

```go
// ❌ НЕПРАВИЛЬНО: остальные горутины продолжают работать
func FirstResponseBad(urls []string) (string, error) {
    results := make(chan string, len(urls))

    for _, url := range urls {
        go func(u string) {
            body, _ := fetch(context.Background(), u) // Без отмены!
            results <- body
        }(url)
    }

    return <-results, nil
}

// ✅ ПРАВИЛЬНО: используем context с cancel
ctx, cancel := context.WithCancel(ctx)
defer cancel() // Отменяем при выходе
```

### Ошибка 2: Утечка горутин при небуферизированном канале

```go
// ❌ НЕПРАВИЛЬНО: горутины заблокируются навсегда
results := make(chan string) // Небуферизированный!

for _, url := range urls {
    go func(u string) {
        body, _ := fetch(ctx, u)
        results <- body // Заблокируется если никто не читает!
    }(url)
}

return <-results, nil // Читаем только один раз

// ✅ ПРАВИЛЬНО: буферизированный канал
results := make(chan string, len(urls))
```

### Ошибка 3: Гонка при записи результата

```go
// ❌ НЕПРАВИЛЬНО: data race
var result string
for _, url := range urls {
    go func(u string) {
        body, _ := fetch(ctx, u)
        result = body // DATA RACE!
    }(url)
}

// ✅ ПРАВИЛЬНО: используем канал или atomic
winner := make(chan string, 1)
select {
case winner <- body:
default:
}
```

### Ошибка 4: Не обрабатывать случай когда все упали

```go
// ❌ НЕПРАВИЛЬНО: вечное ожидание
body := <-results // Что если все вернули ошибки?

// ✅ ПРАВИЛЬНО: считаем ошибки
errCount := 0
for {
    select {
    case r := <-results:
        if r.err == nil {
            return r.body, nil
        }
        errCount++
        if errCount == len(urls) {
            return "", errors.New("all failed")
        }
    }
}
```

## Применение в реальных проектах

### 1. Геораспределённые сервисы

```go
// Запрос к ближайшему дата-центру
endpoints := []string{
    "https://eu.api.example.com",
    "https://us.api.example.com",
    "https://asia.api.example.com",
}
body, _ := FirstResponse(ctx, endpoints)
```

### 2. DNS резолвинг

```go
// Запрос к нескольким DNS серверам
dnsServers := []string{"8.8.8.8", "1.1.1.1", "9.9.9.9"}
```

### 3. Хеджированные запросы (Hedged Requests)

```go
// Отправить второй запрос если первый слишком долгий
func HedgedRequest(ctx context.Context, url string, hedgeDelay time.Duration) (string, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    results := make(chan string, 2)

    // Первый запрос сразу
    go func() {
        body, err := fetch(ctx, url)
        if err == nil {
            results <- body
        }
    }()

    // Второй запрос через hedgeDelay
    go func() {
        select {
        case <-time.After(hedgeDelay):
            body, err := fetch(ctx, url)
            if err == nil {
                results <- body
            }
        case <-ctx.Done():
        }
    }()

    select {
    case body := <-results:
        return body, nil
    case <-ctx.Done():
        return "", ctx.Err()
    }
}
```

## Сложность

| Аспект | Значение |
|--------|----------|
| Время | O(min(t1, t2, ..., tn)) — время самого быстрого |
| Память | O(n) для каналов и горутин |
| Горутины | n горутин (но большинство отменяются) |

## Вопросы с собеседований

### Вопрос 1: Почему важно использовать буферизированный канал?

**Ответ:**

Без буфера горутины, которые не успели первыми, заблокируются навсегда на отправке в канал. Это утечка горутин.

```go
// Буфер = количество горутин
results := make(chan result, len(urls))
```

Или использовать select с default:
```go
select {
case results <- body:
default: // Не блокируемся
}
```

---

### Вопрос 2: Как убедиться что все горутины завершились?

**Ответ:**

1. **Context с cancel** — при вызове cancel() все запросы с этим контекстом получат ошибку.

2. **WaitGroup** (если нужно дождаться всех):
```go
var wg sync.WaitGroup
for _, url := range urls {
    wg.Add(1)
    go func(u string) {
        defer wg.Done()
        // ...
    }(url)
}
wg.Wait()
```

---

### Вопрос 3: Как реализовать "первые N успешных"?

**Ответ:**

```go
func FirstN(ctx context.Context, urls []string, n int) ([]string, error) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    results := make(chan string, len(urls))
    var collected []string

    for _, url := range urls {
        go func(u string) {
            body, err := fetch(ctx, u)
            if err == nil {
                results <- body
            }
        }(url)
    }

    for i := 0; i < len(urls) && len(collected) < n; i++ {
        select {
        case body := <-results:
            collected = append(collected, body)
        case <-ctx.Done():
            return collected, ctx.Err()
        }
    }

    return collected, nil
}
```

---

## Связанные темы

- [[Go - Select statement]]
- [[Go - Context]]
- [[Go - Каналы (channels)]]
- [[Задача - Parallel URL Fetcher]]
- [[Задача - Timeout pattern]]
- [[Go - Пакет net-http]]
