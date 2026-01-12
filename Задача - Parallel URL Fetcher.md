# Задача - Parallel URL Fetcher

Популярная задача на собеседованиях в Авито, Т-Банк, Ozon. Проверяет понимание context, семафоров, ограничения параллелизма и обработки ошибок.

## Условие задачи

```
Написать функцию, которая параллельно запрашивает список URL.
Требования:
1. Ограничить количество одновременных запросов (maxWorkers)
2. Поддержать отмену через context
3. Вернуть результаты всех запросов (успешных и неуспешных)
4. Корректно обрабатывать ошибки
```

```go
type Result struct {
    URL   string
    Body  string
    Error error
}

func FetchURLs(ctx context.Context, urls []string, maxWorkers int) []Result
```

## Решение 1: Семафор через буферизированный канал

```go
type Result struct {
    URL   string
    Body  string
    Error error
}

func FetchURLs(ctx context.Context, urls []string, maxWorkers int) []Result {
    results := make([]Result, len(urls))
    var wg sync.WaitGroup

    // Семафор — буферизированный канал
    semaphore := make(chan struct{}, maxWorkers)

    for i, url := range urls {
        wg.Add(1)

        go func(idx int, u string) {
            defer wg.Done()

            // Захватываем слот семафора
            select {
            case semaphore <- struct{}{}:
                defer func() { <-semaphore }() // Освобождаем слот
            case <-ctx.Done():
                results[idx] = Result{URL: u, Error: ctx.Err()}
                return
            }

            // Делаем запрос
            results[idx] = fetchURL(ctx, u)
        }(i, url)
    }

    wg.Wait()
    return results
}

func fetchURL(ctx context.Context, url string) Result {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return Result{URL: url, Error: err}
    }

    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return Result{URL: url, Error: err}
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return Result{URL: url, Error: err}
    }

    return Result{URL: url, Body: string(body)}
}
```

## Решение 2: Worker Pool

```go
func FetchURLsWorkerPool(ctx context.Context, urls []string, maxWorkers int) []Result {
    jobs := make(chan int, len(urls))
    results := make([]Result, len(urls))

    var wg sync.WaitGroup

    // Запускаем воркеров
    for w := 0; w < maxWorkers; w++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case idx, ok := <-jobs:
                    if !ok {
                        return
                    }
                    results[idx] = fetchURL(ctx, urls[idx])
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    // Отправляем задачи
    for i := range urls {
        select {
        case jobs <- i:
        case <-ctx.Done():
            break
        }
    }
    close(jobs)

    wg.Wait()
    return results
}
```

## Решение 3: С использованием errgroup

```go
import "golang.org/x/sync/errgroup"

func FetchURLsErrgroup(ctx context.Context, urls []string, maxWorkers int) []Result {
    results := make([]Result, len(urls))

    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(maxWorkers) // Ограничиваем параллелизм

    for i, url := range urls {
        i, url := i, url // Захватываем переменные
        g.Go(func() error {
            results[i] = fetchURL(ctx, url)
            return nil // Не прерываем при ошибках отдельных URL
        })
    }

    g.Wait()
    return results
}
```

## Решение 4: С отменой при первой ошибке

```go
func FetchURLsFailFast(ctx context.Context, urls []string, maxWorkers int) ([]Result, error) {
    results := make([]Result, len(urls))
    var mu sync.Mutex

    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(maxWorkers)

    for i, url := range urls {
        i, url := i, url
        g.Go(func() error {
            result := fetchURL(ctx, url)

            mu.Lock()
            results[i] = result
            mu.Unlock()

            if result.Error != nil {
                return result.Error // Отменяет остальные запросы
            }
            return nil
        })
    }

    err := g.Wait()
    return results, err
}
```

## Пример использования

```go
func main() {
    urls := []string{
        "https://httpbin.org/delay/1",
        "https://httpbin.org/delay/2",
        "https://httpbin.org/get",
        "https://httpbin.org/status/404",
        "https://invalid-url",
    }

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    results := FetchURLs(ctx, urls, 2) // Максимум 2 параллельных запроса

    for _, r := range results {
        if r.Error != nil {
            fmt.Printf("ERROR %s: %v\n", r.URL, r.Error)
        } else {
            fmt.Printf("OK %s: %d bytes\n", r.URL, len(r.Body))
        }
    }
}
```

## Типичные ошибки

### Ошибка 1: Не передавать context в HTTP запрос

```go
// ❌ НЕПРАВИЛЬНО: запрос не отменится при ctx.Done()
resp, err := http.Get(url)

// ✅ ПРАВИЛЬНО: используем NewRequestWithContext
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := client.Do(req)
```

### Ошибка 2: Утечка горутин при отмене

```go
// ❌ НЕПРАВИЛЬНО: горутины продолжат работу после отмены
for _, url := range urls {
    go func(u string) {
        resp, _ := http.Get(u) // Не проверяем ctx
        // ...
    }(url)
}

// ✅ ПРАВИЛЬНО: проверяем ctx везде
select {
case semaphore <- struct{}{}:
case <-ctx.Done():
    return // Выходим сразу
}
```

### Ошибка 3: Data race при записи результатов

```go
// ❌ НЕПРАВИЛЬНО: гонка при записи в один слайс
results := []Result{}
for _, url := range urls {
    go func(u string) {
        results = append(results, fetchURL(ctx, u)) // DATA RACE!
    }(url)
}

// ✅ ПРАВИЛЬНО: каждая горутина пишет в свой индекс
results := make([]Result, len(urls))
for i, url := range urls {
    go func(idx int, u string) {
        results[idx] = fetchURL(ctx, u) // Разные индексы — безопасно
    }(i, url)
}
```

### Ошибка 4: Не закрыть body ответа

```go
// ❌ НЕПРАВИЛЬНО: утечка ресурсов
resp, err := client.Do(req)
body, _ := io.ReadAll(resp.Body)
// resp.Body не закрыт!

// ✅ ПРАВИЛЬНО
resp, err := client.Do(req)
if err != nil {
    return Result{Error: err}
}
defer resp.Body.Close()
```

## Визуализация

```
URLs: [url1, url2, url3, url4, url5]
maxWorkers: 2

Время ──────────────────────────────────────▶

Slot 1: │ url1 ████│ url3 ██│ url5 ███│
Slot 2: │ url2 █████████│ url4 ████│

Семафор: [■ ■]  (2 слота)

           url1 занял    url1 освободил
                │              │
                ▼              ▼
Slots:    [■ □] ──▶ [■ ■] ──▶ [□ ■] ──▶ [■ ■]
                     ▲
                     │
               url2 занял
```

## Вопросы с собеседований

### Вопрос 1: Зачем ограничивать количество горутин?

**Ответ:**

1. **Ресурсы**: Каждая HTTP-соединение использует файловый дескриптор. Их количество ограничено (ulimit).

2. **Сервер**: Слишком много параллельных запросов могут положить целевой сервер (DDoS себя).

3. **Память**: Много горутин = много памяти на стеки.

4. **Rate limiting**: Внешние API часто имеют лимиты запросов.

---

### Вопрос 2: Чем errgroup лучше WaitGroup + семафор?

**Ответ:**

**errgroup преимущества:**
- `SetLimit(n)` — встроенное ограничение параллелизма
- Автоматическая отмена context при ошибке
- Возвращает первую ошибку
- Меньше boilerplate кода

**WaitGroup + семафор:**
- Не нужна внешняя зависимость
- Больше контроля над поведением
- Проще для понимания

---

### Вопрос 3: Как обработать timeout отдельного запроса?

**Ответ:**

```go
// Способ 1: Timeout в HTTP Client
client := &http.Client{Timeout: 5 * time.Second}

// Способ 2: Context с timeout для каждого запроса
reqCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
req, _ := http.NewRequestWithContext(reqCtx, "GET", url, nil)
```

---

## Связанные темы

- [[Go - Context]]
- [[Задача - Worker Pool]]
- [[Задача - Semaphore pattern]]
- [[Задача - Timeout pattern]]
- [[Go - Пакет net-http]]
- [[Задача - Rate Limiter]]
