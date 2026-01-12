# Задача - Bounded Parallelism

Фундаментальный паттерн конкурентности в Go. Спрашивают на каждом втором собеседовании в Яндекс, Авито, Т-Банк. Проверяет умение ограничивать параллелизм.

## Условие задачи

```
Реализовать обработку списка задач с ограничением количества
одновременно выполняемых операций.
Требования:
1. Обработать N задач
2. Не более M задач выполняются одновременно
3. Собрать все результаты
4. Поддержать отмену через context
```

## Решение 1: Семафор через буферизированный канал

```go
func ProcessWithLimit(ctx context.Context, items []int, maxConcurrency int) []int {
    results := make([]int, len(items))
    var wg sync.WaitGroup

    // Семафор — буферизированный канал
    sem := make(chan struct{}, maxConcurrency)

    for i, item := range items {
        wg.Add(1)

        go func(idx int, val int) {
            defer wg.Done()

            // Захватываем слот
            select {
            case sem <- struct{}{}:
                defer func() { <-sem }()
            case <-ctx.Done():
                return
            }

            // Обрабатываем
            results[idx] = process(val)
        }(i, item)
    }

    wg.Wait()
    return results
}

func process(val int) int {
    time.Sleep(100 * time.Millisecond) // Имитация работы
    return val * 2
}
```

## Решение 2: Worker Pool

```go
func ProcessWorkerPool(ctx context.Context, items []int, numWorkers int) []int {
    results := make([]int, len(items))
    jobs := make(chan int, len(items))

    var wg sync.WaitGroup

    // Запускаем воркеров
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case idx, ok := <-jobs:
                    if !ok {
                        return
                    }
                    results[idx] = process(items[idx])
                case <-ctx.Done():
                    return
                }
            }
        }()
    }

    // Отправляем задачи
    for i := range items {
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

## Решение 3: С errgroup.SetLimit

```go
import "golang.org/x/sync/errgroup"

func ProcessErrgroup(ctx context.Context, items []int, maxConcurrency int) ([]int, error) {
    results := make([]int, len(items))

    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(maxConcurrency) // Ограничиваем параллелизм

    for i, item := range items {
        i, item := i, item // Захват переменных
        g.Go(func() error {
            select {
            case <-ctx.Done():
                return ctx.Err()
            default:
            }

            results[i] = process(item)
            return nil
        })
    }

    err := g.Wait()
    return results, err
}
```

## Решение 4: С semaphore.Weighted

```go
import "golang.org/x/sync/semaphore"

func ProcessSemaphore(ctx context.Context, items []int, maxConcurrency int64) []int {
    results := make([]int, len(items))
    sem := semaphore.NewWeighted(maxConcurrency)

    var wg sync.WaitGroup

    for i, item := range items {
        // Захватываем слот (блокирующий)
        if err := sem.Acquire(ctx, 1); err != nil {
            break // Context отменён
        }

        wg.Add(1)
        go func(idx int, val int) {
            defer wg.Done()
            defer sem.Release(1)

            results[idx] = process(val)
        }(i, item)
    }

    wg.Wait()
    return results
}
```

## Решение 5: Generator + Fan-Out

```go
func ProcessFanOut(ctx context.Context, items []int, numWorkers int) <-chan int {
    // Generator
    input := make(chan int)
    go func() {
        defer close(input)
        for _, item := range items {
            select {
            case input <- item:
            case <-ctx.Done():
                return
            }
        }
    }()

    // Fan-Out: несколько воркеров читают из одного канала
    workers := make([]<-chan int, numWorkers)
    for i := 0; i < numWorkers; i++ {
        workers[i] = worker(ctx, input)
    }

    // Fan-In: объединяем результаты
    return merge(ctx, workers...)
}

func worker(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for val := range input {
            select {
            case out <- process(val):
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

func merge(ctx context.Context, channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                select {
                case out <- val:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

## Решение 6: С rate limiting (запросы в секунду)

```go
import "golang.org/x/time/rate"

func ProcessRateLimited(ctx context.Context, items []int, rps int) []int {
    results := make([]int, len(items))
    limiter := rate.NewLimiter(rate.Limit(rps), 1)

    var wg sync.WaitGroup

    for i, item := range items {
        // Ждём разрешения
        if err := limiter.Wait(ctx); err != nil {
            break
        }

        wg.Add(1)
        go func(idx int, val int) {
            defer wg.Done()
            results[idx] = process(val)
        }(i, item)
    }

    wg.Wait()
    return results
}
```

## Пример использования

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    items := make([]int, 100)
    for i := range items {
        items[i] = i
    }

    start := time.Now()

    // Обрабатываем 100 элементов, максимум 10 параллельно
    results := ProcessWithLimit(ctx, items, 10)

    fmt.Printf("Processed %d items in %v\n", len(results), time.Since(start))
    // Output: Processed 100 items in ~1s (100 items / 10 workers * 100ms)
}
```

## Визуализация

```
Items: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
maxConcurrency: 3

Время ──────────────────────────────────────▶

Slot 1: │ 1 ██│ 4 ██│ 7 ██│ 10 ██│
Slot 2: │ 2 ████│ 5 ██│ 8 ████│
Slot 3: │ 3 ███│ 6 ███│ 9 ███│

Семафор: [■ ■ ■]  (3 слота)

Когда 1 завершается → слот освобождается → 4 начинает
```

## Сравнение подходов

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| Семафор (канал) | Просто, нет зависимостей | N горутин создаются сразу |
| Worker Pool | Фиксированное число горутин | Сложнее реализовать |
| errgroup | Чистый API, обработка ошибок | Внешняя зависимость |
| semaphore.Weighted | Гибкие веса | Внешняя зависимость |
| Rate limiter | Контроль RPS | Не ограничивает concurrency |

## Типичные ошибки

### Ошибка 1: Создание горутины до захвата семафора

```go
// ❌ НЕПРАВИЛЬНО: все горутины создаются сразу
for i, item := range items {
    go func(idx int, val int) {
        sem <- struct{}{} // Все горутины уже созданы!
        defer func() { <-sem }()
        process(val)
    }(i, item)
}

// ✅ ПРАВИЛЬНО: захватываем ДО создания горутины
for i, item := range items {
    sem <- struct{}{} // Блокируемся здесь
    go func(idx int, val int) {
        defer func() { <-sem }()
        process(val)
    }(i, item)
}
```

### Ошибка 2: Не проверять context при захвате

```go
// ❌ НЕПРАВИЛЬНО: можем застрять навсегда
sem <- struct{}{}

// ✅ ПРАВИЛЬНО: с проверкой context
select {
case sem <- struct{}{}:
case <-ctx.Done():
    return ctx.Err()
}
```

### Ошибка 3: Неправильный размер буфера

```go
// ❌ НЕПРАВИЛЬНО: слишком большой буфер не ограничивает
sem := make(chan struct{}, 1000000)

// ❌ НЕПРАВИЛЬНО: небуферизированный блокирует всё
sem := make(chan struct{})

// ✅ ПРАВИЛЬНО: размер = maxConcurrency
sem := make(chan struct{}, maxConcurrency)
```

### Ошибка 4: Утечка при панике

```go
// ❌ НЕПРАВИЛЬНО: при панике слот не освободится
sem <- struct{}{}
process(val) // Может паниковать!
<-sem

// ✅ ПРАВИЛЬНО: defer гарантирует освобождение
sem <- struct{}{}
defer func() { <-sem }()
process(val)
```

## Выбор числа воркеров

```go
// CPU-bound задачи
numWorkers := runtime.NumCPU()

// I/O-bound задачи (сетевые запросы)
numWorkers := runtime.NumCPU() * 10 // или больше

// Ограничение внешнего API
numWorkers := apiRateLimit // Согласно документации API

// Ограничение базы данных
numWorkers := maxDBConnections
```

## Практический пример: Загрузка файлов

```go
type DownloadResult struct {
    URL   string
    Size  int64
    Error error
}

func DownloadFiles(ctx context.Context, urls []string, maxConcurrency int) []DownloadResult {
    results := make([]DownloadResult, len(urls))
    sem := make(chan struct{}, maxConcurrency)
    var wg sync.WaitGroup

    for i, url := range urls {
        wg.Add(1)

        go func(idx int, u string) {
            defer wg.Done()

            select {
            case sem <- struct{}{}:
                defer func() { <-sem }()
            case <-ctx.Done():
                results[idx] = DownloadResult{URL: u, Error: ctx.Err()}
                return
            }

            size, err := downloadFile(ctx, u)
            results[idx] = DownloadResult{URL: u, Size: size, Error: err}
        }(i, url)
    }

    wg.Wait()
    return results
}

func downloadFile(ctx context.Context, url string) (int64, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return 0, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return 0, err
    }
    defer resp.Body.Close()

    return io.Copy(io.Discard, resp.Body)
}
```

## Вопросы с собеседований

### Вопрос 1: В чём разница между семафором и Worker Pool?

**Ответ:**

**Семафор (канал):**
- Создаёт N горутин (по количеству задач)
- Ограничивает сколько из них работают одновременно
- Проще в реализации
- Больше памяти при большом N

**Worker Pool:**
- Создаёт фиксированное число воркеров
- Задачи отправляются через канал
- Эффективнее по памяти
- Сложнее в реализации

---

### Вопрос 2: Когда использовать errgroup.SetLimit vs семафор?

**Ответ:**

**errgroup.SetLimit:**
- Нужна обработка ошибок (первая ошибка отменяет всё)
- Хотите автоматическую отмену context
- Не критична внешняя зависимость

**Семафор (канал):**
- Нужен полный контроль
- Не хотите внешних зависимостей
- Нужно продолжать при ошибках отдельных задач

---

### Вопрос 3: Как выбрать оптимальное число воркеров?

**Ответ:**

Зависит от типа задачи:

1. **CPU-bound**: `runtime.NumCPU()` — больше не даст выигрыша
2. **I/O-bound**: `NumCPU * 10-100` — горутины ждут I/O
3. **Внешние ограничения**: rate limit API, пул соединений БД

Лучший способ — бенчмарки на реальной нагрузке.

---

## Связанные темы

- [[Задача - Semaphore pattern]]
- [[Задача - Worker Pool]]
- [[Go - Context]]
- [[Go - WaitGroup и Once]]
- [[Задача - Rate Limiter]]
- [[Задача - Parallel URL Fetcher]]
