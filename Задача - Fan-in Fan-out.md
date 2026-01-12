# Задача - Fan-in Fan-out

Fan-out и Fan-in - паттерны для параллельной обработки данных и объединения результатов. Часто спрашивают вместе с Worker Pool на собеседованиях!

## Концепция

### Fan-out (распределение)

**Fan-out** - распределение работы от одного источника на несколько обработчиков для параллельной обработки.

```
          ┌─→ Worker 1
Input ────┼─→ Worker 2
          └─→ Worker 3
```

**Применение:** Когда задачу можно разбить на независимые подзадачи.

### Fan-in (объединение)

**Fan-in** - объединение результатов из нескольких источников в один канал.

```
Worker 1 ─┐
Worker 2 ─┼─→ Output
Worker 3 ─┘
```

**Применение:** Сбор результатов от нескольких параллельных процессов.

## Fan-out: Базовый пример

```go
package main

import (
    "fmt"
    "time"
)

// Генератор задач
func generator() <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for i := 1; i <= 10; i++ {
            out <- i
        }
    }()

    return out
}

// Worker - обработчик
func worker(id int, jobs <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for job := range jobs {
            fmt.Printf("Worker %d processing job %d\n", id, job)
            time.Sleep(100 * time.Millisecond)  // Симуляция работы
            out <- job * job
        }
    }()

    return out
}

func main() {
    // Генерируем задачи
    jobs := generator()

    // Fan-out: распределяем на 3 workers
    // Все workers читают из ОДНОГО канала jobs
    worker1 := worker(1, jobs)
    worker2 := worker(2, jobs)
    worker3 := worker(3, jobs)

    // Читаем из всех workers
    for i := 0; i < 10; i++ {
        select {
        case result := <-worker1:
            fmt.Println("From worker 1:", result)
        case result := <-worker2:
            fmt.Println("From worker 2:", result)
        case result := <-worker3:
            fmt.Println("From worker 3:", result)
        }
    }
}
```

**Проблема:** Нужно вручную читать из каждого канала → неудобно при большом количестве workers.

## Fan-in: Объединение результатов

```go
package main

import (
    "fmt"
    "sync"
)

// Fan-in: объединяет несколько каналов в один
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup
    wg.Add(len(channels))

    // Для каждого канала запускаем горутину
    for _, ch := range channels {
        go func(c <-chan int) {
            defer wg.Done()

            // Читаем всё из канала
            for n := range c {
                out <- n
            }
        }(ch)
    }

    // Закрываем out когда все каналы прочитаны
    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func main() {
    jobs := generator()

    // Fan-out
    worker1 := worker(1, jobs)
    worker2 := worker(2, jobs)
    worker3 := worker(3, jobs)

    // Fan-in: объединяем результаты
    results := fanIn(worker1, worker2, worker3)

    // Читаем из одного канала
    for result := range results {
        fmt.Println("Result:", result)
    }
}
```

**Преимущества:**
- Читаем из одного канала вместо трёх
- Порядок результатов не гарантируется (зависит от скорости workers)
- Автоматическое закрытие выходного канала

## Полный пример: Fan-out + Fan-in

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

// Генератор чисел
func generateNumbers(max int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for i := 1; i <= max; i++ {
            out <- i
        }
    }()

    return out
}

// Тяжёлая обработка (долгая операция)
func process(id int, in <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for num := range in {
            fmt.Printf("Worker %d processing %d\n", id, num)

            // Симуляция тяжёлой работы
            time.Sleep(200 * time.Millisecond)
            result := num * num

            out <- result
            fmt.Printf("Worker %d finished %d = %d\n", id, num, result)
        }

        fmt.Printf("Worker %d exiting\n", id)
    }()

    return out
}

// Fan-out: создаём N workers читающих из одного канала
func fanOut(in <-chan int, numWorkers int) []<-chan int {
    channels := make([]<-chan int, numWorkers)

    for i := 0; i < numWorkers; i++ {
        channels[i] = process(i+1, in)
    }

    return channels
}

// Fan-in: объединяем результаты
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup
    wg.Add(len(channels))

    for _, ch := range channels {
        go func(c <-chan int) {
            defer wg.Done()
            for n := range c {
                out <- n
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func main() {
    start := time.Now()

    // Pipeline с fan-out/fan-in
    numbers := generateNumbers(10)

    // Fan-out: 5 параллельных workers
    workers := fanOut(numbers, 5)

    // Fan-in: объединяем результаты
    results := fanIn(workers...)

    // Собираем результаты
    total := 0
    for result := range results {
        total += result
        fmt.Printf("Got result: %d (total: %d)\n", result, total)
    }

    fmt.Printf("\nTotal: %d\n", total)
    fmt.Printf("Time: %v\n", time.Since(start))
}
```

**Результат:**
```
Worker 1 processing 1
Worker 2 processing 2
Worker 3 processing 3
Worker 4 processing 4
Worker 5 processing 5
...
Time: ~400ms  (вместо 2 секунд без параллелизма)
```

## Fan-in с select (альтернативный подход)

```go
func fanInSelect(ch1, ch2 <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for {
            select {
            case val, ok := <-ch1:
                if !ok {
                    ch1 = nil  // Отключаем закрытый канал
                } else {
                    out <- val
                }

            case val, ok := <-ch2:
                if !ok {
                    ch2 = nil  // Отключаем закрытый канал
                } else {
                    out <- val
                }
            }

            // Выходим когда оба канала закрыты
            if ch1 == nil && ch2 == nil {
                return
            }
        }
    }()

    return out
}

// Обобщённая версия
func fanInSelectMulti(channels ...<-chan int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup
    wg.Add(len(channels))

    // Для каждого канала - своя горутина
    for _, ch := range channels {
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                out <- val
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

## Реальный пример: Параллельная обработка URL

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
    "time"
)

type Result struct {
    URL    string
    Status int
    Size   int
    Err    error
}

// Генератор URL
func generateURLs(urls []string) <-chan string {
    out := make(chan string)

    go func() {
        defer close(out)
        for _, url := range urls {
            out <- url
        }
    }()

    return out
}

// Fetcher - скачивает URL
func fetcher(id int, urls <-chan string) <-chan Result {
    out := make(chan Result)

    go func() {
        defer close(out)

        client := &http.Client{Timeout: 10 * time.Second}

        for url := range urls {
            fmt.Printf("Fetcher %d: fetching %s\n", id, url)

            result := Result{URL: url}

            resp, err := client.Get(url)
            if err != nil {
                result.Err = err
                out <- result
                continue
            }

            defer resp.Body.Close()
            body, err := io.ReadAll(resp.Body)

            result.Status = resp.StatusCode
            result.Size = len(body)
            result.Err = err

            out <- result
        }

        fmt.Printf("Fetcher %d: done\n", id)
    }()

    return out
}

// Fan-out
func fanOut(in <-chan string, workers int) []<-chan Result {
    channels := make([]<-chan Result, workers)

    for i := 0; i < workers; i++ {
        channels[i] = fetcher(i+1, in)
    }

    return channels
}

// Fan-in
func fanIn(channels ...<-chan Result) <-chan Result {
    out := make(chan Result)

    var wg sync.WaitGroup
    wg.Add(len(channels))

    for _, ch := range channels {
        go func(c <-chan Result) {
            defer wg.Done()
            for result := range c {
                out <- result
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}

func main() {
    urls := []string{
        "https://golang.org",
        "https://github.com",
        "https://stackoverflow.com",
        "https://reddit.com",
        "https://news.ycombinator.com",
    }

    start := time.Now()

    // Pipeline
    urlChan := generateURLs(urls)

    // Fan-out: 3 параллельных fetchers
    fetchers := fanOut(urlChan, 3)

    // Fan-in: объединяем результаты
    results := fanIn(fetchers...)

    // Обрабатываем результаты
    successCount := 0
    totalSize := 0

    for result := range results {
        if result.Err != nil {
            fmt.Printf("❌ %s: %v\n", result.URL, result.Err)
        } else {
            fmt.Printf("✅ %s: %d (%d bytes)\n", result.URL, result.Status, result.Size)
            successCount++
            totalSize += result.Size
        }
    }

    fmt.Printf("\nSuccess: %d/%d\n", successCount, len(urls))
    fmt.Printf("Total size: %d bytes\n", totalSize)
    fmt.Printf("Time: %v\n", time.Since(start))
}
```

## Вопрос 1: В чём разница между Fan-out и Worker Pool?

**Ответ:**

| Fan-out | Worker Pool |
|---------|-------------|
| Несколько workers читают из **одного** канала | Несколько workers читают из **одного** канала |
| Каждый worker возвращает **свой** канал результатов | Все workers пишут в **один** канал результатов |
| Нужен Fan-in для объединения | Результаты уже в одном канале |
| Сложнее (2 операции: fan-out + fan-in) | Проще (готовый паттерн) |

**Fan-out + Fan-in:**
```go
jobs := generator()
worker1 := worker(jobs)  // Свой канал
worker2 := worker(jobs)  // Свой канал
worker3 := worker(jobs)  // Свой канал
results := fanIn(worker1, worker2, worker3)
```

**Worker Pool:**
```go
jobs := generator()
results := make(chan Result)

for i := 0; i < 3; i++ {
    go worker(jobs, results)  // Все пишут в results
}
```

**Когда использовать:**
- Fan-out/Fan-in → когда нужна промежуточная обработка результатов каждого worker'а
- Worker Pool → когда результаты можно сразу объединить

---

## Вопрос 2: Реализуйте Fan-in с приоритетом

Нужно читать из канала с высоким приоритетом чаще чем из обычных.

**Решение:**

```go
func fanInWithPriority(high <-chan int, normal ...<-chan int) <-chan int {
    out := make(chan int)

    var wg sync.WaitGroup
    wg.Add(len(normal) + 1)

    // High priority: читаем в отдельной горутине
    go func() {
        defer wg.Done()
        for val := range high {
            out <- val
        }
    }()

    // Normal priority: по горутине на канал
    for _, ch := range normal {
        go func(c <-chan int) {
            defer wg.Done()
            for val := range c {
                // Сначала проверяем high priority
                select {
                case highVal := <-high:
                    out <- highVal
                    out <- val
                default:
                    out <- val
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

Или с использованием select с приоритетом:
```go
func fanInPriority(high, low <-chan int) <-chan int {
    out := make(chan int)

    go func() {
        defer close(out)

        for {
            select {
            case val, ok := <-high:
                if !ok {
                    high = nil
                } else {
                    out <- val
                }

            default:
                select {
                case val, ok := <-high:
                    if !ok {
                        high = nil
                    } else {
                        out <- val
                    }

                case val, ok := <-low:
                    if !ok {
                        low = nil
                    } else {
                        out <- val
                    }
                }
            }

            if high == nil && low == nil {
                return
            }
        }
    }()

    return out
}
```

---

## Вопрос 3: Как ограничить количество одновременных workers в Fan-out?

**Решение:**

**Проблема:** Fan-out может создать слишком много горутин.

**Решение: Используем semaphore**
```go
func fanOutLimited(in <-chan int, maxWorkers int) []<-chan int {
    sem := make(chan struct{}, maxWorkers)  // Semaphore
    channels := make([]<-chan int, maxWorkers)

    for i := 0; i < maxWorkers; i++ {
        out := make(chan int)
        channels[i] = out

        go func(id int, out chan<- int) {
            defer close(out)

            for job := range in {
                sem <- struct{}{}  // Acquire semaphore

                fmt.Printf("Worker %d processing %d\n", id, job)
                result := job * job
                out <- result

                <-sem  // Release semaphore
            }
        }(i, out)
    }

    return channels
}
```

**Или используйте Worker Pool вместо Fan-out** - он уже имеет ограничение.

---

## Best Practices

1. ✅ **Используйте WaitGroup для синхронизации fan-in**
   ```go
   var wg sync.WaitGroup
   wg.Add(len(channels))

   // ...

   go func() {
       wg.Wait()
       close(out)
   }()
   ```

2. ✅ **Всегда закрывайте выходной канал**
   ```go
   defer close(out)
   ```

3. ✅ **Fan-in должен дождаться всех входных каналов**
   ```go
   for _, ch := range channels {
       go func(c <-chan int) {
           defer wg.Done()
           for val := range c {
               out <- val
           }
       }(ch)
   }
   ```

4. ✅ **Используйте буферизированные каналы для снижения блокировок**
   ```go
   out := make(chan int, 100)
   ```

5. ✅ **Для простых случаев используйте Worker Pool**
   - Fan-out + Fan-in сложнее
   - Worker Pool проще и часто достаточен

6. ✅ **Контролируйте количество горутин**
   ```go
   maxWorkers := runtime.NumCPU() * 2
   workers := fanOut(jobs, maxWorkers)
   ```

7. ❌ **Не забывайте закрывать каналы**
   - Незакрытый канал → goroutine leak
   - Range не завершится → получатель зависнет

## Где спрашивают

- Яндекс
- Авито
- VK
- Т-Банк
- Google (Go позиции)

## Связанные темы

- [[Задача - Worker Pool]]
- [[Задача - Pipeline pattern]]
- [[Go - Каналы (channels)]]
- [[Go - WaitGroup и Once]]
- [[Go - Select statement]]
