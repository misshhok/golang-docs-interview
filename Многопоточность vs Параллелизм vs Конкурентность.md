# Многопоточность vs Параллелизм vs Конкурентность

Это три разных, но связанных концепции, которые часто путают. Понимание различий критически важно для работы с [[Go - Горутины (goroutines)|горутинами]] и [[Go - Каналы (channels)|каналами]] в Go.

## Определения

### Конкурентность (Concurrency)

**Конкурентность** - это когда несколько задач **могут** выполняться в перекрывающиеся периоды времени, но необязательно одновременно.

> "Concurrency is about **dealing with** lots of things at once."
> — Rob Pike

**Пример**: Один повар готовит три блюда:
- Ставит картошку вариться
- Пока варится, режет овощи
- Пока овощи жарятся, месит тесто

Повар один, но **управляет** множеством задач, переключаясь между ними.

### Параллелизм (Parallelism)

**Параллелизм** - это когда несколько задач выполняются **одновременно** в один момент времени.

> "Parallelism is about **doing** lots of things at once."
> — Rob Pike

**Пример**: Три повара готовят три блюда:
- Первый повар варит картошку
- Второй повар жарит овощи
- Третий повар месит тесто

Все происходит **одновременно**.

### Многопоточность (Multithreading)

**Многопоточность** - это техническая реализация конкурентности/параллелизма через **потоки операционной системы**.

**Пример**: Программа создает 3 OS thread'а для выполнения задач.

## Визуальное сравнение

### Последовательное выполнение (Sequential)

```
CPU: [====A====][====B====][====C====]
     │          │          │
     0         10         20         30 (время)

Одна задача за раз, одна за другой
```

### Конкурентное выполнение (Concurrent)

```
CPU: [==A==][B][=A=][==C==][A][B][==C==]
     │     │ │ │   │     │ │ │ │     │
     0    2 3 5   8    12 14 15 17   20 (время)

Одно ядро, но задачи чередуются (context switching)
```

### Параллельное выполнение (Parallel)

```
CPU1: [====A====][====A====]
CPU2: [====B====][====B====]
CPU3: [====C====][====C====]
      │          │          │
      0         10         20 (время)

Три ядра, задачи выполняются одновременно
```

### Конкурентное И параллельное

```
CPU1: [==A==][=A=][==C==][A]
CPU2: [==B==][C][====B====][B]
      │      │   │         │
      0      5  10        15 (время)

Два ядра, задачи и чередуются, и выполняются параллельно
```

## Ключевые различия

| Аспект | Конкурентность | Параллелизм |
|--------|----------------|-------------|
| **Определение** | Структура программы | Выполнение программы |
| **Про что** | Composition of tasks | Simultaneous execution |
| **Может быть на** | 1 ядре | Требует 2+ ядер |
| **Цель** | Структурирование кода | Ускорение вычислений |
| **Когда полезно** | I/O-bound задачи | CPU-bound задачи |

## В контексте Go

### Горутины - Конкурентность

[[Go - Горутины (goroutines)|Горутины]] обеспечивают **конкурентность**:

```go
func main() {
    go task1() // Запускаем конкурентно
    go task2()
    go task3()

    time.Sleep(time.Second)
}
```

Эти горутины могут:
- Выполняться на одном ядре (чередуясь)
- Выполняться параллельно на разных ядрах
- Комбинация обоих

### GOMAXPROCS - Параллелизм

`GOMAXPROCS` определяет сколько OS threads используется = сколько горутин может выполняться **параллельно**:

```go
import "runtime"

func main() {
    // По умолчанию = количество CPU cores
    fmt.Println(runtime.GOMAXPROCS(0))

    // Ограничить до 1 потока = нет параллелизма (только конкурентность)
    runtime.GOMAXPROCS(1)

    // Максимальный параллелизм
    runtime.GOMAXPROCS(runtime.NumCPU())
}
```

**Пример:**
```
GOMAXPROCS = 1:  Конкурентность БЕЗ параллелизма
                 [G1][G2][G1][G3][G2]... на одном ядре

GOMAXPROCS = 4:  Конкурентность С параллелизмом
                 CPU1: [G1][G1]...
                 CPU2: [G2][G2]...
                 CPU3: [G3][G3]...
                 CPU4: [G4][G4]...
```

## I/O-bound vs CPU-bound

### I/O-bound задачи

Задачи, ограниченные вводом-выводом (network, disk, database):

```go
// I/O-bound: большую часть времени ждем I/O
func fetchURL(url string) {
    resp, _ := http.Get(url) // Ждем сеть
    defer resp.Body.Close()
    io.ReadAll(resp.Body)
}
```

**Для I/O-bound:**
- ✅ Конкурентность помогает (пока одна горутина ждет, другие работают)
- ⚠️ Параллелизм помогает меньше (т.к. узкое место - I/O)

```go
// 1000 конкурентных HTTP запросов - быстро!
for i := 0; i < 1000; i++ {
    go fetchURL(urls[i])
}
```

### CPU-bound задачи

Задачи, ограниченные вычислениями:

```go
// CPU-bound: вычисления, нет I/O
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}
```

**Для CPU-bound:**
- ⚠️ Конкурентность без параллелизма НЕ помогает (CPU занят постоянно)
- ✅ Параллелизм помогает (распределяем нагрузку на ядра)

```go
// Распараллеливание CPU-bound задачи
numCPU := runtime.NumCPU()
chunkSize := len(data) / numCPU

var wg sync.WaitGroup
for i := 0; i < numCPU; i++ {
    wg.Add(1)
    go func(chunk []int) {
        defer wg.Done()
        process(chunk) // Каждая горутина на своем ядре
    }(data[i*chunkSize:(i+1)*chunkSize])
}
wg.Wait()
```

## Примеры паттернов

### 1. Конкурентность для I/O

```go
// Конкурентная загрузка URL'ов
func fetchAll(urls []string) []Result {
    results := make(chan Result, len(urls))
    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            results <- fetch(u) // Пока один ждет I/O, другие работают
        }(url)
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    var out []Result
    for r := range results {
        out = append(out, r)
    }
    return out
}
```

### 2. Параллелизм для CPU-bound

```go
// Параллельная обработка данных
func processParallel(data []int) []int {
    numWorkers := runtime.NumCPU()
    chunkSize := len(data) / numWorkers
    results := make([][]int, numWorkers)

    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            start := workerID * chunkSize
            end := start + chunkSize
            if workerID == numWorkers-1 {
                end = len(data)
            }
            results[workerID] = processChunk(data[start:end])
        }(i)
    }

    wg.Wait()

    // Объединяем результаты
    var final []int
    for _, r := range results {
        final = append(final, r...)
    }
    return final
}
```

### 3. Pipeline - конкурентная композиция

```go
// Конкурентный pipeline
func pipeline() {
    // Stage 1: Генерация
    generate := func() <-chan int {
        out := make(chan int)
        go func() {
            defer close(out)
            for i := 0; i < 100; i++ {
                out <- i
            }
        }()
        return out
    }

    // Stage 2: Обработка
    square := func(in <-chan int) <-chan int {
        out := make(chan int)
        go func() {
            defer close(out)
            for n := range in {
                out <- n * n
            }
        }()
        return out
    }

    // Stage 3: Фильтрация
    filter := func(in <-chan int) <-chan int {
        out := make(chan int)
        go func() {
            defer close(out)
            for n := range in {
                if n%2 == 0 {
                    out <- n
                }
            }
        }()
        return out
    }

    // Соединяем stages
    nums := generate()
    squared := square(nums)
    filtered := filter(squared)

    // Потребляем результат
    for n := range filtered {
        fmt.Println(n)
    }
}
```

## Распространенные заблуждения

### ❌ "Больше горутин = быстрее"

```go
// ❌ Слишком много горутин
for i := 0; i < 1000000; i++ {
    go process(i) // Миллион горутин - overhead!
}

// ✅ Worker pool с разумным количеством
numWorkers := runtime.NumCPU()
jobs := make(chan int, 100)

for i := 0; i < numWorkers; i++ {
    go worker(jobs)
}

for i := 0; i < 1000000; i++ {
    jobs <- i
}
```

### ❌ "Конкурентность = параллелизм"

Нет! Конкурентность - это **структура**, параллелизм - это **выполнение**.

```go
// Конкурентная программа может работать на 1 ядре
runtime.GOMAXPROCS(1)
go task1()
go task2() // Конкурентность есть, параллелизма нет
```

### ❌ "Параллелизм всегда быстрее"

Не всегда! Overhead на синхронизацию может быть больше выигрыша.

```go
// Для маленьких задач параллелизм может быть медленнее!
// Sequential: 100 ns
// Parallel (2 cores): 50 ns + 100 ns overhead = 150 ns!
```

## Когда что использовать

### Используйте конкурентность когда:

- ✅ I/O-bound задачи (network, file, database)
- ✅ Нужна отзывчивость (UI, servers)
- ✅ Логически независимые задачи
- ✅ Ожидание событий

**Примеры:**
- Web servers (обработка множества requests)
- Crawlers (загрузка множества страниц)
- Chat servers (множество клиентов)

### Используйте параллелизм когда:

- ✅ CPU-bound задачи
- ✅ Большие объемы данных для обработки
- ✅ Независимые вычисления
- ✅ Есть несколько CPU cores

**Примеры:**
- Image processing (обработка пикселей)
- Data analysis (анализ больших датасетов)
- Machine learning (обучение моделей)
- Видео кодирование

### Комбинируйте оба когда:

```go
// Конкурентность для I/O + Параллелизм для обработки
func processURLs(urls []string) {
    // Конкурентно загружаем (I/O-bound)
    dataChan := make(chan Data, 100)
    for _, url := range urls {
        go func(u string) {
            data := fetch(u) // I/O
            dataChan <- data
        }(url)
    }

    // Параллельно обрабатываем (CPU-bound)
    numWorkers := runtime.NumCPU()
    for i := 0; i < numWorkers; i++ {
        go func() {
            for data := range dataChan {
                process(data) // CPU
            }
        }()
    }
}
```

## Benchmark примеры

### Sequential vs Concurrent vs Parallel

```go
// Sequential
func sequential(data []int) {
    for _, item := range data {
        process(item)
    }
}
// 1000 items: ~1000ms

// Concurrent (GOMAXPROCS=1)
func concurrent(data []int) {
    var wg sync.WaitGroup
    for _, item := range data {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            process(i)
        }(item)
    }
    wg.Wait()
}
// 1000 items: ~1000ms (нет параллелизма, только overhead!)

// Parallel (GOMAXPROCS=4)
func parallel(data []int) {
    numWorkers := 4
    chunkSize := len(data) / numWorkers
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(start int) {
            defer wg.Done()
            for j := start; j < start+chunkSize; j++ {
                process(data[j])
            }
        }(i * chunkSize)
    }
    wg.Wait()
}
// 1000 items: ~250ms (4x ускорение на 4 ядрах!)
```

## Best Practices

1. **Для I/O** - используйте много горутин (конкурентность)
2. **Для CPU** - количество горутин = количеству cores (параллелизм)
3. **Worker pools** - ограничивайте количество горутин
4. **Профилирование** - измеряйте реальную производительность
5. **Context** - всегда управляйте жизненным циклом горутин

```go
// ✅ Хороший паттерн
func process(ctx context.Context, data []Data) {
    numWorkers := runtime.NumCPU()
    jobs := make(chan Data, len(data))
    results := make(chan Result, len(data))

    // Workers
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    results <- processOne(job)
                }
            }
        }()
    }

    // Producer
    go func() {
        for _, d := range data {
            jobs <- d
        }
        close(jobs)
    }()

    // Cleanup
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect
    for r := range results {
        handle(r)
    }
}
```

## Связанные темы

- [[Go - Горутины (goroutines)]] - реализация конкурентности в Go
- [[Go - Каналы (channels)]] - коммуникация между горутинами
- [[Go - Context]] - управление жизненным циклом
- [[Go - Пакет sync]] - примитивы синхронизации
- [[Go - Mutex и RWMutex]] - защита shared state
