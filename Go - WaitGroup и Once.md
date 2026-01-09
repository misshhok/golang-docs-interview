# Go - WaitGroup и Once

`sync.WaitGroup` и `sync.Once` - два важных примитива из пакета [[Go - Пакет sync]] для координации [[Go - Горутины (goroutines)|горутин]]: ожидание завершения группы горутин и гарантия одноразового выполнения.

## sync.WaitGroup

WaitGroup используется для ожидания завершения группы горутин. Это аналог `Promise.all()` в JavaScript или `threading.join()` в Python.

### Базовое использование

```go
import "sync"

func main() {
    var wg sync.WaitGroup

    // Запускаем 5 горутин
    for i := 0; i < 5; i++ {
        wg.Add(1) // Увеличиваем счетчик

        go func(id int) {
            defer wg.Done() // Уменьшаем счетчик при завершении
            fmt.Printf("Worker %d starting\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Worker %d done\n", id)
        }(i)
    }

    wg.Wait() // Блокируется пока счетчик не станет 0
    fmt.Println("All workers completed")
}
```

### API методы

```go
type WaitGroup struct {
    // ...
}

// Увеличить счетчик на delta (обычно 1)
func (wg *WaitGroup) Add(delta int)

// Уменьшить счетчик на 1 (эквивалентно Add(-1))
func (wg *WaitGroup) Done()

// Блокироваться пока счетчик не станет 0
func (wg *WaitGroup) Wait()
```

### Правильный порядок Add/Done/Wait

```go
// ✅ ПРАВИЛЬНО
for i := 0; i < 10; i++ {
    wg.Add(1) // Add ДО запуска горутины
    go func() {
        defer wg.Done() // Done внутри горутины
        // Работа
    }()
}
wg.Wait() // Wait после всех Add и go

// ❌ НЕПРАВИЛЬНО - race condition
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1) // Add внутри горутины - race!
        defer wg.Done()
    }()
}
wg.Wait() // Может вызваться до Add!
```

**Правило**: `Add()` всегда вызывается **до** `go func()`.

### Передача по указателю

WaitGroup **ВСЕГДА** передается по указателю:

```go
// ✅ Правильно - указатель
func worker(wg *sync.WaitGroup) {
    defer wg.Done()
    // Работа
}

func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go worker(&wg)
    wg.Wait()
}

// ❌ Неправильно - копия
func worker(wg sync.WaitGroup) {
    defer wg.Done() // Работает с копией!
    // Работа
}
```

## Паттерны использования WaitGroup

### 1. Worker Pool

```go
func workerPool(jobs []Job) {
    var wg sync.WaitGroup
    numWorkers := 10

    jobsChan := make(chan Job, len(jobs))
    for _, job := range jobs {
        jobsChan <- job
    }
    close(jobsChan)

    // Запускаем workers
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobsChan {
                process(job)
            }
        }()
    }

    wg.Wait() // Ждем завершения всех workers
    fmt.Println("All jobs processed")
}
```

### 2. Fan-Out Pattern

```go
func fanOut(data []Data) []Result {
    var wg sync.WaitGroup
    results := make([]Result, len(data))

    for i, d := range data {
        wg.Add(1)
        go func(index int, item Data) {
            defer wg.Done()
            results[index] = process(item)
        }(i, d)
    }

    wg.Wait()
    return results
}
```

### 3. Ограничение параллелизма с WaitGroup

```go
func processWithLimit(items []Item, limit int) {
    var wg sync.WaitGroup
    semaphore := make(chan struct{}, limit)

    for _, item := range items {
        wg.Add(1)
        semaphore <- struct{}{} // Захватываем слот

        go func(i Item) {
            defer wg.Done()
            defer func() { <-semaphore }() // Освобождаем слот

            process(i)
        }(item)
    }

    wg.Wait()
}
```

### 4. Error группа (errgroup)

```go
// Используя golang.org/x/sync/errgroup
import "golang.org/x/sync/errgroup"

func fetchAll(urls []string) error {
    g := new(errgroup.Group)

    for _, url := range urls {
        url := url // Захватываем переменную
        g.Go(func() error {
            return fetch(url)
        })
    }

    // Ждем всех и возвращаем первую ошибку
    if err := g.Wait(); err != nil {
        return err
    }

    return nil
}
```

## sync.Once

`sync.Once` гарантирует что функция выполнится **ровно один раз**, даже при конкурентных вызовах.

### Базовое использование

```go
import "sync"

var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        fmt.Println("Creating singleton")
        instance = &Singleton{}
    })
    return instance
}

func main() {
    var wg sync.WaitGroup

    // 100 горутин пытаются получить instance
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            inst := GetInstance()
            fmt.Println(inst)
        }()
    }

    wg.Wait()
    // "Creating singleton" напечатается только ОДИН раз
}
```

### API

```go
type Once struct {
    // ...
}

// Выполнить f ровно один раз
func (o *Once) Do(f func())
```

**Важно**: Все конкурентные вызовы `Do()` блокируются пока первый вызов не завершится!

### Как работает Once

```go
// Упрощенная реализация
type Once struct {
    done uint32
    m    Mutex
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

## Паттерны использования Once

### 1. Lazy Initialization (Singleton)

```go
type Config struct {
    // ...
}

var (
    config *Config
    once   sync.Once
)

func GetConfig() *Config {
    once.Do(func() {
        fmt.Println("Loading config...")
        config = loadConfig() // Дорогая операция
    })
    return config
}
```

### 2. One-Time Setup

```go
var (
    db   *Database
    once sync.Once
)

func initDB() {
    once.Do(func() {
        fmt.Println("Connecting to database...")
        db = connectDB()
        // Эта функция выполнится только один раз
    })
}

func GetDB() *Database {
    initDB()
    return db
}
```

### 3. Закрытие ресурса один раз

```go
type Connection struct {
    conn   net.Conn
    closed sync.Once
}

func (c *Connection) Close() error {
    var err error
    c.closed.Do(func() {
        fmt.Println("Closing connection")
        err = c.conn.Close()
    })
    return err
}

// Можно вызывать Close() много раз - закроется только один раз
conn.Close()
conn.Close() // Не делает ничего
conn.Close() // Не делает ничего
```

### 4. Регистрация один раз

```go
type Service struct {
    registered sync.Once
}

func (s *Service) Register() {
    s.registered.Do(func() {
        fmt.Println("Registering service")
        registerWithDiscovery(s)
    })
}

// Множественные вызовы безопасны
service.Register()
service.Register() // Игнорируется
```

### 5. Once per instance

```go
type Resource struct {
    initialized sync.Once
    data        []byte
}

func (r *Resource) Init() {
    r.initialized.Do(func() {
        fmt.Println("Initializing resource")
        r.data = loadData()
    })
}

// Каждый Resource инициализируется ровно один раз
r1 := &Resource{}
r1.Init() // Инициализирует r1
r1.Init() // Ничего не делает

r2 := &Resource{}
r2.Init() // Инициализирует r2 (отдельный sync.Once)
```

## Комбинирование WaitGroup и Once

```go
type LazyWorkerPool struct {
    once    sync.Once
    workers []*Worker
}

func (p *LazyWorkerPool) Start() {
    p.once.Do(func() {
        var wg sync.WaitGroup
        for i := 0; i < 10; i++ {
            wg.Add(1)
            go func(id int) {
                defer wg.Done()
                worker := startWorker(id)
                p.workers = append(p.workers, worker)
            }(i)
        }
        wg.Wait()
        fmt.Println("All workers started")
    })
}
```

## Типичные ошибки

### WaitGroup ошибки

#### 1. Add внутри горутины

```go
// ❌ Race condition!
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1) // Может вызваться после wg.Wait()!
        defer wg.Done()
        work()
    }()
}
wg.Wait() // Может завершиться до всех Add()

// ✅ Правильно
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        work()
    }()
}
wg.Wait()
```

#### 2. Забыть Done

```go
// ❌ Deadlock - забыли Done
wg.Add(1)
go func() {
    work()
    // Забыли wg.Done()!
}()
wg.Wait() // Ждет вечно!

// ✅ Используйте defer
wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()
wg.Wait()
```

#### 3. Done вызван больше раз чем Add

```go
// ❌ Panic: negative WaitGroup counter
wg.Add(1)
go func() {
    defer wg.Done()
    defer wg.Done() // Второй Done - panic!
    work()
}()
```

#### 4. Копирование WaitGroup

```go
// ❌ Копирование WaitGroup
func worker(wg sync.WaitGroup) {
    defer wg.Done() // Работает с копией!
    work()
}

// ✅ Передача по указателю
func worker(wg *sync.WaitGroup) {
    defer wg.Done()
    work()
}
```

#### 5. Переиспользование WaitGroup

```go
// ❌ Небезопасно переиспользовать до Wait
var wg sync.WaitGroup

wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()

wg.Add(1) // Опасно! Может быть race с Wait внутри горутины
go func() {
    defer wg.Done()
    wg.Wait() // ОЧЕНЬ опасно!
}()

// ✅ Дождитесь Wait перед переиспользованием
var wg sync.WaitGroup

wg.Add(1)
go func() { defer wg.Done(); work() }()
wg.Wait()

// Теперь безопасно переиспользовать
wg.Add(1)
go func() { defer wg.Done(); work() }()
wg.Wait()
```

### Once ошибки

#### 1. Once.Do с panic

```go
// ❌ Panic помечает Once как выполненный!
var once sync.Once

once.Do(func() {
    panic("error") // Once помечается как done!
})

once.Do(func() {
    fmt.Println("Never runs!") // Не выполнится
})

// ✅ Обрабатывайте панику внутри
once.Do(func() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
        }
    }()
    riskyOperation()
})
```

#### 2. Передача замыкания с внешними переменными

```go
// ❌ Захват переменной - может измениться
for i := 0; i < 5; i++ {
    once.Do(func() {
        fmt.Println(i) // Какое значение i?
    })
}

// ✅ Передавайте как параметры или не зависьте от внешних переменных
```

#### 3. Длинная функция в Do

```go
// ❌ Блокирует все конкурентные вызовы Do
once.Do(func() {
    time.Sleep(10 * time.Second) // Все ждут 10 секунд!
    initialize()
})

// ✅ Держите Do функцию короткой
once.Do(func() {
    go longInitialization() // В фоне
    quickSetup()
})
```

## Альтернативы

### Вместо WaitGroup

**Context для отмены:**
```go
// Вместо WaitGroup + done канала
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

go worker(ctx)
go worker(ctx)

// Отмена всех workers
cancel()
```

См. [[Go - Context]]

**Errgroup для обработки ошибок:**
```go
g := new(errgroup.Group)
g.Go(func() error { return work1() })
g.Go(func() error { return work2() })

if err := g.Wait(); err != nil {
    // Обработка первой ошибки
}
```

### Вместо Once

**sync.Map для множественных singleton'ов:**
```go
var instances sync.Map

func GetInstance(key string) *Instance {
    if val, ok := instances.Load(key); ok {
        return val.(*Instance)
    }

    inst := createInstance(key)
    actual, _ := instances.LoadOrStore(key, inst)
    return actual.(*Instance)
}
```

## Best Practices

### WaitGroup

1. ✅ Всегда `Add()` перед `go func()`
2. ✅ Всегда `defer Done()`
3. ✅ Передавайте по указателю
4. ✅ Документируйте ответственность за Done()
5. ❌ Не переиспользуйте до завершения Wait()

### Once

1. ✅ Используйте для lazy initialization
2. ✅ Держите функцию в Do() короткой
3. ✅ Обрабатывайте panic внутри Do() если нужно
4. ❌ Не зависьте от порядка вызовов (первый выигрывает)
5. ❌ Не используйте для cleanup (используйте Close() pattern)

## Производительность

**WaitGroup:**
- `Add(1)`: ~5-10 ns
- `Done()`: ~5-10 ns
- `Wait()` (без ожидания): ~10-20 ns
- `Wait()` (с блокировкой): ~100-500 ns

**Once:**
- `Do()` (уже выполнен): ~1-2 ns (fast path)
- `Do()` (первый вызов): ~20-50 ns

Once **очень** эффективен для повторных вызовов!

## Связанные темы

- [[Go - Пакет sync]] - обзор всех примитивов синхронизации
- [[Go - Горутины (goroutines)]] - конкурентное выполнение
- [[Go - Context]] - управление отменой и таймаутами
- [[Go - Mutex и RWMutex]] - защита shared state
- [[Go - Каналы (channels)]] - альтернатива для координации
- [[Многопоточность vs Параллелизм vs Конкурентность]] - концепции
