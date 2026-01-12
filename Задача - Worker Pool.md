# Задача - Worker Pool

Worker Pool - один из самых популярных паттернов конкурентности в Go. Постоянно спрашивают на собеседованиях!

## Зачем нужен Worker Pool

### Проблема: слишком много горутин

```go
// ❌ Плохо: создаём миллион горутин
func processURLs(urls []string) {
    for _, url := range urls {
        go fetchURL(url)  // 1 000 000 горутин!
    }
}
```

**Проблемы:**
- Каждая горутина потребляет память (~2KB стек)
- Миллион горутин = ~2GB памяти минимум
- Overhead на создание/удаление горутин
- Перегрузка CPU (context switching)
- Можем исчерпать системные ресурсы (файловые дескрипторы, сокеты)

### Решение: Worker Pool

**Worker Pool** - фиксированное количество горутин (workers), которые обрабатывают задачи из очереди.

**Преимущества:**
- **Ограничение параллелизма** - контролируем нагрузку
- **Переиспользование горутин** - не создаём/удаляем постоянно
- **Предсказуемое использование ресурсов**
- **Back pressure** - если задач больше чем workers могут обработать, они ждут в очереди

## Базовая реализация

```go
package main

import (
    "fmt"
    "time"
)

// Job - задача для выполнения
type Job struct {
    ID   int
    Data string
}

// Result - результат выполнения
type Result struct {
    Job    Job
    Output string
}

// worker - функция воркера
func worker(id int, jobs <-chan Job, results chan<- Result) {
    for job := range jobs {  // Читаем пока канал не закроется
        fmt.Printf("Worker %d started job %d\n", id, job.ID)

        // Симуляция работы
        time.Sleep(time.Second)
        output := fmt.Sprintf("Processed: %s", job.Data)

        fmt.Printf("Worker %d finished job %d\n", id, job.ID)

        results <- Result{Job: job, Output: output}
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 9

    // Каналы для задач и результатов
    jobs := make(chan Job, numJobs)
    results := make(chan Result, numJobs)

    // Запускаем workers
    for w := 1; w <= numWorkers; w++ {
        go worker(w, jobs, results)
    }

    // Отправляем задачи
    for j := 1; j <= numJobs; j++ {
        jobs <- Job{ID: j, Data: fmt.Sprintf("task-%d", j)}
    }
    close(jobs)  // Закрываем канал задач

    // Собираем результаты
    for a := 1; a <= numJobs; a++ {
        result := <-results
        fmt.Printf("Result: %s\n", result.Output)
    }
}
```

**Вывод:**
```
Worker 1 started job 1
Worker 2 started job 2
Worker 3 started job 3
Worker 1 finished job 1
Worker 1 started job 4
Worker 2 finished job 2
Worker 2 started job 5
...
```

### Ключевые моменты

1. **Буферизированные каналы** - `make(chan Job, numJobs)`
   - Позволяют отправить все задачи сразу без блокировки
   - Размер буфера = количество задач

2. **Range по каналу** - `for job := range jobs`
   - Автоматически выходит когда канал закрыт и пуст
   - Не нужно проверять `ok`

3. **Закрытие канала** - `close(jobs)`
   - Сигнал workers'ам что задач больше не будет
   - Важно: закрывает отправитель, не получатель

4. **Однонаправленные каналы**
   - `jobs <-chan Job` - только чтение
   - `results chan<- Result` - только запись
   - Компилятор проверяет корректность использования

## Расширенная реализация с WaitGroup

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID int
}

func worker(id int, jobs <-chan Job, wg *sync.WaitGroup) {
    defer wg.Done()  // Уменьшаем счётчик при выходе

    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job.ID)
        time.Sleep(500 * time.Millisecond)
        fmt.Printf("Worker %d completed job %d\n", id, job.ID)
    }

    fmt.Printf("Worker %d exiting\n", id)
}

func main() {
    const numWorkers = 3
    jobs := make(chan Job, 100)

    var wg sync.WaitGroup

    // Запускаем workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, &wg)
    }

    // Отправляем задачи
    for j := 1; j <= 10; j++ {
        jobs <- Job{ID: j}
    }
    close(jobs)

    // Ждём завершения всех workers
    wg.Wait()
    fmt.Println("All workers finished")
}
```

**Преимущества WaitGroup:**
- Не нужен отдельный канал результатов
- Гарантия что все workers завершились
- Проще для случаев когда результат не нужен

## Worker Pool с Graceful Shutdown

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID int
}

// WorkerPool - структура пула
type WorkerPool struct {
    workers int
    jobs    chan Job
    wg      sync.WaitGroup
    ctx     context.Context
    cancel  context.CancelFunc
}

// NewWorkerPool - создание пула
func NewWorkerPool(workers int, bufferSize int) *WorkerPool {
    ctx, cancel := context.WithCancel(context.Background())

    return &WorkerPool{
        workers: workers,
        jobs:    make(chan Job, bufferSize),
        ctx:     ctx,
        cancel:  cancel,
    }
}

// Start - запуск workers
func (wp *WorkerPool) Start() {
    for i := 1; i <= wp.workers; i++ {
        wp.wg.Add(1)
        go wp.worker(i)
    }
}

// worker - логика воркера
func (wp *WorkerPool) worker(id int) {
    defer wp.wg.Done()

    fmt.Printf("Worker %d started\n", id)

    for {
        select {
        case job, ok := <-wp.jobs:
            if !ok {
                // Канал закрыт
                fmt.Printf("Worker %d: jobs channel closed\n", id)
                return
            }

            // Обработка задачи
            fmt.Printf("Worker %d processing job %d\n", id, job.ID)
            time.Sleep(500 * time.Millisecond)
            fmt.Printf("Worker %d completed job %d\n", id, job.ID)

        case <-wp.ctx.Done():
            // Получен сигнал отмены
            fmt.Printf("Worker %d received shutdown signal\n", id)
            return
        }
    }
}

// Submit - добавление задачи
func (wp *WorkerPool) Submit(job Job) error {
    select {
    case wp.jobs <- job:
        return nil
    case <-wp.ctx.Done():
        return fmt.Errorf("worker pool is shutting down")
    }
}

// Shutdown - graceful shutdown
func (wp *WorkerPool) Shutdown() {
    fmt.Println("Initiating shutdown...")

    close(wp.jobs)  // Закрываем канал задач
    wp.wg.Wait()    // Ждём завершения текущих задач

    fmt.Println("All workers finished")
}

// ShutdownNow - немедленная остановка
func (wp *WorkerPool) ShutdownNow() {
    fmt.Println("Initiating immediate shutdown...")

    wp.cancel()  // Отменяем context
    wp.wg.Wait() // Ждём выхода workers

    fmt.Println("All workers stopped")
}

func main() {
    pool := NewWorkerPool(3, 10)
    pool.Start()

    // Отправляем задачи
    for i := 1; i <= 10; i++ {
        pool.Submit(Job{ID: i})
    }

    // Graceful shutdown через 3 секунды
    time.Sleep(3 * time.Second)
    pool.Shutdown()

    // Для немедленной остановки:
    // pool.ShutdownNow()
}
```

**Два типа shutdown:**

1. **Graceful Shutdown** (`Shutdown`)
   - Закрываем канал задач
   - Ждём пока workers обработают текущие задачи
   - Используется когда важно завершить работу

2. **Immediate Shutdown** (`ShutdownNow`)
   - Отменяем context
   - Workers прекращают работу немедленно
   - Используется при критических ошибках

## Worker Pool с Rate Limiting

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID int
}

// RateLimitedWorkerPool - пул с ограничением скорости
type RateLimitedWorkerPool struct {
    workers   int
    jobs      chan Job
    wg        sync.WaitGroup
    rateLimit time.Duration  // Минимальный интервал между задачами
}

func NewRateLimitedPool(workers int, bufferSize int, rateLimit time.Duration) *RateLimitedWorkerPool {
    return &RateLimitedWorkerPool{
        workers:   workers,
        jobs:      make(chan Job, bufferSize),
        rateLimit: rateLimit,
    }
}

func (p *RateLimitedWorkerPool) Start() {
    for i := 1; i <= p.workers; i++ {
        p.wg.Add(1)
        go p.worker(i)
    }
}

func (p *RateLimitedWorkerPool) worker(id int) {
    defer p.wg.Done()

    ticker := time.NewTicker(p.rateLimit)
    defer ticker.Stop()

    for job := range p.jobs {
        <-ticker.C  // Ждём разрешения по rate limit

        fmt.Printf("Worker %d processing job %d\n", id, job.ID)
        time.Sleep(100 * time.Millisecond)  // Симуляция работы
        fmt.Printf("Worker %d completed job %d\n", id, job.ID)
    }
}

func (p *RateLimitedWorkerPool) Submit(job Job) {
    p.jobs <- job
}

func (p *RateLimitedWorkerPool) Shutdown() {
    close(p.jobs)
    p.wg.Wait()
}

func main() {
    // Максимум 2 запроса в секунду на воркер
    pool := NewRateLimitedPool(2, 10, 500*time.Millisecond)
    pool.Start()

    for i := 1; i <= 10; i++ {
        pool.Submit(Job{ID: i})
    }

    pool.Shutdown()
}
```

**Применение:**
- API с rate limits (например, 100 запросов/минуту)
- Предотвращение DDoS на собственный сервис
- Контроль нагрузки на внешние системы

## Worker Pool с обработкой ошибок

```go
package main

import (
    "fmt"
    "sync"
)

type Job struct {
    ID   int
    Data string
}

type Result struct {
    Job   Job
    Value string
    Error error
}

func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job.ID)

        // Симуляция работы с возможной ошибкой
        var result Result
        result.Job = job

        if job.ID%3 == 0 {
            result.Error = fmt.Errorf("job %d failed", job.ID)
        } else {
            result.Value = fmt.Sprintf("Success: %s", job.Data)
        }

        results <- result
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10

    jobs := make(chan Job, numJobs)
    results := make(chan Result, numJobs)

    var wg sync.WaitGroup

    // Запускаем workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // Отправляем задачи
    for j := 1; j <= numJobs; j++ {
        jobs <- Job{ID: j, Data: fmt.Sprintf("task-%d", j)}
    }
    close(jobs)

    // Закрываем results когда все workers завершатся
    go func() {
        wg.Wait()
        close(results)
    }()

    // Собираем результаты
    successCount := 0
    errorCount := 0

    for result := range results {
        if result.Error != nil {
            fmt.Printf("❌ Job %d failed: %v\n", result.Job.ID, result.Error)
            errorCount++
        } else {
            fmt.Printf("✅ Job %d: %s\n", result.Job.ID, result.Value)
            successCount++
        }
    }

    fmt.Printf("\nTotal: %d success, %d errors\n", successCount, errorCount)
}
```

## Вопрос 1: Реализуйте динамический Worker Pool

Задача: пул должен уметь увеличивать/уменьшать количество workers в runtime.

**Решение:**

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

type DynamicPool struct {
    jobs          chan Job
    activeWorkers int32
    mu            sync.Mutex
    wg            sync.WaitGroup
    shutdown      chan struct{}
}

func NewDynamicPool(bufferSize int) *DynamicPool {
    return &DynamicPool{
        jobs:     make(chan Job, bufferSize),
        shutdown: make(chan struct{}),
    }
}

func (p *DynamicPool) AddWorker() {
    p.mu.Lock()
    defer p.mu.Unlock()

    atomic.AddInt32(&p.activeWorkers, 1)
    p.wg.Add(1)

    workerID := atomic.LoadInt32(&p.activeWorkers)

    go func(id int32) {
        defer p.wg.Done()
        defer atomic.AddInt32(&p.activeWorkers, -1)

        fmt.Printf("Worker %d started\n", id)

        for {
            select {
            case job, ok := <-p.jobs:
                if !ok {
                    fmt.Printf("Worker %d exiting (channel closed)\n", id)
                    return
                }
                fmt.Printf("Worker %d processing job %d\n", id, job.ID)

            case <-p.shutdown:
                fmt.Printf("Worker %d exiting (shutdown signal)\n", id)
                return
            }
        }
    }(workerID)
}

func (p *DynamicPool) GetWorkerCount() int32 {
    return atomic.LoadInt32(&p.activeWorkers)
}

func (p *DynamicPool) Submit(job Job) {
    p.jobs <- job
}

func (p *DynamicPool) Shutdown() {
    close(p.shutdown)
    close(p.jobs)
    p.wg.Wait()
}

func main() {
    pool := NewDynamicPool(10)

    // Начинаем с 2 workers
    pool.AddWorker()
    pool.AddWorker()

    // Отправляем задачи
    for i := 1; i <= 5; i++ {
        pool.Submit(Job{ID: i})
    }

    fmt.Printf("Active workers: %d\n", pool.GetWorkerCount())

    // Добавляем ещё worker'а
    pool.AddWorker()
    fmt.Printf("Active workers: %d\n", pool.GetWorkerCount())

    for i := 6; i <= 10; i++ {
        pool.Submit(Job{ID: i})
    }

    pool.Shutdown()
    fmt.Printf("Final workers: %d\n", pool.GetWorkerCount())
}
```

---

## Вопрос 2: Чем отличается Worker Pool от Semaphore?

**Ответ:**

| Worker Pool | Semaphore |
|-------------|-----------|
| Фиксированное количество горутин-workers | Ограничение одновременных операций |
| Workers переиспользуются | Горутины создаются/удаляются |
| Задачи в очереди (канал) | Нет очереди задач |
| Явное управление lifecycle | Просто acquire/release |

**Worker Pool:**
```go
// 3 воркера обрабатывают 100 задач
workers := 3
jobs := make(chan Job, 100)

for w := 1; w <= workers; w++ {
    go worker(jobs)
}
```

**Semaphore:**
```go
// Максимум 3 одновременные горутины
sem := make(chan struct{}, 3)

for i := 0; i < 100; i++ {
    sem <- struct{}{}  // Acquire
    go func() {
        defer func() { <-sem }()  // Release
        doWork()
    }()
}
```

**Когда использовать:**
- Worker Pool → долгоживущие воркеры, управление очередью
- Semaphore → простое ограничение параллелизма

---

## Вопрос 3: Проблема с этой реализацией?

```go
func main() {
    jobs := make(chan Job)
    results := make(chan Result)

    // Запускаем workers
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    // Отправляем задачи
    for j := 1; j <= 9; j++ {
        jobs <- Job{ID: j}  // Блокируется!
    }
    close(jobs)

    // Читаем результаты
    for a := 1; a <= 9; a++ {
        <-results
    }
}
```

**Ответ:**

**Deadlock!**

**Почему:**
- Канал `jobs` небуферизированный
- `jobs <- Job{ID: j}` блокируется пока кто-то не прочитает
- Workers пытаются отправить в `results`
- Но main блокирован на отправке в `jobs` и не читает `results`
- Circular dependency → deadlock

**Исправление 1: Буферизированный канал**
```go
jobs := make(chan Job, 9)
```

**Исправление 2: Отправка в горутине**
```go
go func() {
    for j := 1; j <= 9; j++ {
        jobs <- Job{ID: j}
    }
    close(jobs)
}()

for a := 1; a <= 9; a++ {
    <-results
}
```

---

## Best Practices

1. ✅ **Буферизируйте каналы задач**
   ```go
   jobs := make(chan Job, expectedJobs)
   ```

2. ✅ **Используйте WaitGroup для синхронизации**
   ```go
   var wg sync.WaitGroup
   wg.Add(numWorkers)
   ```

3. ✅ **Закрывайте каналы правильно**
   ```go
   close(jobs)      // Отправитель закрывает jobs
   wg.Wait()        // Ждём workers
   close(results)   // Закрываем results после workers
   ```

4. ✅ **Используйте однонаправленные каналы**
   ```go
   func worker(jobs <-chan Job, results chan<- Result)
   ```

5. ✅ **Реализуйте graceful shutdown**
   ```go
   select {
   case job := <-jobs:
       process(job)
   case <-ctx.Done():
       return
   }
   ```

6. ✅ **Обрабатывайте панику в workers**
   ```go
   defer func() {
       if r := recover(); r != nil {
           log.Printf("Worker panic: %v", r)
       }
   }()
   ```

7. ✅ **Выбирайте правильное количество workers**
   ```go
   // CPU-bound задачи
   numWorkers := runtime.NumCPU()

   // I/O-bound задачи
   numWorkers := 100  // Может быть больше CPU
   ```

## Где спрашивают

- Яндекс
- Авито
- Т-Банк
- Озон
- VK
- Практически везде на Go позициях!

## Связанные темы

- [[Go - Горутины (goroutines)]]
- [[Go - Каналы (channels)]]
- [[Go - WaitGroup и Once]]
- [[Go - Context]]
- [[Задача - Semaphore pattern]]
- [[Задача - Graceful Shutdown]]
- [[Задача - Pipeline pattern]]
