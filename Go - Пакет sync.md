# Go - Пакет sync

Пакет `sync` предоставляет примитивы синхронизации для координации горутин: мьютексы, wait groups, условные переменные и другие инструменты для безопасной работы с shared state.

## Когда использовать sync vs каналы

**Используйте каналы [[Go - Каналы (channels)]] когда:**
- Передаете ownership данных
- Распределяете работу между [[Go - Горутины (goroutines)|горутинами]]
- Передаете результаты выполнения

**Используйте sync когда:**
- Защищаете shared state (счетчики, кэши)
- Нужна взаимоисключающая блокировка
- Короткие критические секции
- Ожидаете завершения группы горутин

> "Use channels when goroutines need to communicate. Use mutexes when goroutines need to coordinate access."

## Основные типы

### 1. sync.Mutex

Взаимоисключающая блокировка (mutual exclusion lock).

```go
import "sync"

type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func main() {
    counter := &SafeCounter{}
    var wg sync.WaitGroup

    // 1000 горутин инкрементируют счетчик
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Inc()
        }()
    }

    wg.Wait()
    fmt.Println("Final count:", counter.Value()) // 1000
}
```

**Правила использования Mutex:**
- ✅ Всегда используйте `defer mu.Unlock()` сразу после `mu.Lock()`
- ❌ Никогда не копируйте mutex (используйте указатели)
- ❌ Не блокируйте рекурсивно (deadlock!)
- ✅ Держите критическую секцию максимально короткой

Подробнее: [[Go - Mutex и RWMutex]]

### 2. sync.RWMutex

Read-Write mutex позволяет множественные читатели или одного писателя.

```go
type SafeMap struct {
    mu   sync.RWMutex
    data map[string]int
}

func (m *SafeMap) Get(key string) (int, bool) {
    m.mu.RLock()         // Read lock
    defer m.mu.RUnlock()
    val, ok := m.data[key]
    return val, ok
}

func (m *SafeMap) Set(key string, value int) {
    m.mu.Lock()          // Write lock
    defer m.mu.Unlock()
    m.data[key] = value
}
```

**Когда использовать RWMutex:**
- Частые чтения, редкие записи
- Много читателей могут работать параллельно
- Писатель получает эксклюзивный доступ

**Производительность:**
- Read lock быстрее для параллельных чтений
- Если писателей много, RWMutex может быть медленнее обычного Mutex

### 3. sync.WaitGroup

Ожидание завершения группы [[Go - Горутины (goroutines)|горутин]].

```go
func main() {
    var wg sync.WaitGroup
    urls := []string{"url1", "url2", "url3"}

    for _, url := range urls {
        wg.Add(1) // Добавляем счетчик

        go func(u string) {
            defer wg.Done() // Уменьшаем счетчик
            fetch(u)
        }(url)
    }

    wg.Wait() // Ждем пока счетчик не станет 0
    fmt.Println("All fetches complete")
}
```

**Правила WaitGroup:**
- ✅ `Add()` вызывается до `go func()`
- ✅ `Done()` вызывается через `defer`
- ❌ Не копируйте WaitGroup после первого использования
- ✅ Передавайте WaitGroup по указателю

**Типичная ошибка:**

```go
// ❌ НЕПРАВИЛЬНО - race condition
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1) // Add внутри горутины - race!
        defer wg.Done()
    }()
}

// ✅ ПРАВИЛЬНО
for i := 0; i < 10; i++ {
    wg.Add(1) // Add ДО запуска горутины
    go func() {
        defer wg.Done()
    }()
}
```

Подробнее: [[Go - WaitGroup и Once]]

### 4. sync.Once

Гарантирует выполнение функции ровно один раз, даже при конкурентных вызовах.

```go
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
            _ = GetInstance()
        }()
    }

    wg.Wait()
    // "Creating singleton" напечатается только один раз
}
```

**Типичные use cases:**
- Lazy initialization
- Singleton pattern
- One-time setup (загрузка конфига, подключение к БД)

**Важно**: `once.Do()` блокирует пока первая функция не завершится!

Подробнее: [[Go - WaitGroup и Once]]

### 5. sync.Cond

Условная переменная для сложной координации горутин.

```go
type Queue struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []int
}

func NewQueue() *Queue {
    q := &Queue{items: make([]int, 0)}
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Queue) Enqueue(item int) {
    q.mu.Lock()
    defer q.mu.Unlock()

    q.items = append(q.items, item)
    q.cond.Signal() // Будим одного ожидающего
}

func (q *Queue) Dequeue() int {
    q.mu.Lock()
    defer q.mu.Unlock()

    // Ждем пока очередь не пуста
    for len(q.items) == 0 {
        q.cond.Wait() // Освобождает lock и ждет сигнала
    }

    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```

**Методы Cond:**
- `Wait()` - освобождает lock, ждет сигнала, затем снова захватывает lock
- `Signal()` - будит одну ожидающую горутину
- `Broadcast()` - будит все ожидающие горутины

**Важно**: `Wait()` всегда используется в цикле:

```go
// ✅ Правильно
for !condition {
    cond.Wait()
}

// ❌ Неправильно
if !condition {
    cond.Wait() // Spurious wakeup возможен!
}
```

**В большинстве случаев лучше использовать каналы вместо Cond!**

### 6. sync.Map

Concurrent map безопасная для использования из множества горутин.

```go
var m sync.Map

// Store
m.Store("key", "value")

// Load
value, ok := m.Load("key")
if ok {
    fmt.Println(value)
}

// LoadOrStore - атомарно
actual, loaded := m.LoadOrStore("key", "newvalue")
// loaded = true если ключ существовал

// Delete
m.Delete("key")

// Range - итерация
m.Range(func(key, value interface{}) bool {
    fmt.Printf("%v: %v\n", key, value)
    return true // continue, false = break
})
```

**Когда использовать sync.Map:**
- ✅ Ключи пишутся один раз, читаются много раз (cache)
- ✅ Множество горутин читают/пишут непересекающиеся множества ключей
- ❌ Если нужна типобезопасность (sync.Map использует interface{})

**Когда НЕ использовать:**
- Часто изменяемые ключи
- Нужна типобезопасность
- Простые случаи (лучше `map` с `RWMutex`)

```go
// Альтернатива sync.Map с типобезопасностью
type SafeMap struct {
    mu   sync.RWMutex
    data map[string]int
}
```

### 7. sync.Pool

Пул временных объектов для переиспользования и снижения нагрузки на GC.

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        // Вызывается когда пул пуст
        return new(bytes.Buffer)
    },
}

func processRequest(data []byte) {
    // Получаем буфер из пула
    buf := bufferPool.Get().(*bytes.Buffer)
    defer bufferPool.Put(buf) // Возвращаем в пул
    defer buf.Reset()          // Очищаем буфер

    buf.Write(data)
    // Работа с буфером
}
```

**Особенности Pool:**
- Потокобезопасный
- Объекты могут быть удалены GC в любой момент
- Не гарантирует порядок
- Отлично для временных буферов, билдеров, etc.

**Когда использовать:**
- ✅ Частое создание/уничтожение временных объектов
- ✅ Объекты дорогие в создании
- ✅ Снижение нагрузки на GC

**Типичные use cases:**
- `bytes.Buffer` для форматирования
- HTTP request/response объекты
- Temporary slices

## Паттерны использования

### 1. Resource Pool Pattern

```go
type ResourcePool struct {
    resources chan Resource
}

func NewResourcePool(size int) *ResourcePool {
    pool := &ResourcePool{
        resources: make(chan Resource, size),
    }

    // Инициализируем пул
    for i := 0; i < size; i++ {
        pool.resources <- createResource()
    }

    return pool
}

func (p *ResourcePool) Acquire() Resource {
    return <-p.resources // Блокируется если пул пуст
}

func (p *ResourcePool) Release(r Resource) {
    p.resources <- r
}

// Использование
func doWork(pool *ResourcePool) {
    resource := pool.Acquire()
    defer pool.Release(resource)

    // Работа с ресурсом
}
```

### 2. Worker Pool с sync

```go
type WorkerPool struct {
    wg      sync.WaitGroup
    tasks   chan func()
    workers int
}

func NewWorkerPool(workers int) *WorkerPool {
    pool := &WorkerPool{
        tasks:   make(chan func(), 100),
        workers: workers,
    }

    pool.wg.Add(workers)
    for i := 0; i < workers; i++ {
        go pool.worker()
    }

    return pool
}

func (p *WorkerPool) worker() {
    defer p.wg.Done()
    for task := range p.tasks {
        task()
    }
}

func (p *WorkerPool) Submit(task func()) {
    p.tasks <- task
}

func (p *WorkerPool) Shutdown() {
    close(p.tasks)
    p.wg.Wait()
}
```

### 3. Double-Checked Locking (устарело, используйте sync.Once!)

```go
// ❌ Устаревший паттерн (до Go 1.0)
var (
    instance *Singleton
    mu       sync.Mutex
)

func getInstance() *Singleton {
    if instance == nil { // Первая проверка без lock
        mu.Lock()
        defer mu.Unlock()
        if instance == nil { // Вторая проверка с lock
            instance = &Singleton{}
        }
    }
    return instance
}

// ✅ Современный подход - используйте sync.Once!
var (
    instance *Singleton
    once     sync.Once
)

func getInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}
```

## Типичные ошибки

### 1. Копирование sync типов

```go
// ❌ НЕПРАВИЛЬНО - копирование mutex
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c Counter) Inc() { // Копирует c, включая mutex!
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// ✅ ПРАВИЛЬНО - указатель
func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}
```

Используйте `go vet` для обнаружения копирований!

### 2. Забыть Unlock

```go
// ❌ Забыли unlock при ошибке
mu.Lock()
if err != nil {
    return err // Lock не освобожден! Deadlock!
}
mu.Unlock()

// ✅ Всегда используйте defer
mu.Lock()
defer mu.Unlock()
if err != nil {
    return err // Lock освобождается автоматически
}
```

### 3. WaitGroup Add внутри горутины

```go
// ❌ Race condition
for i := 0; i < 10; i++ {
    go func() {
        wg.Add(1) // Может вызваться после wg.Wait()!
        defer wg.Done()
    }()
}
wg.Wait()

// ✅ Add до запуска горутины
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
    }()
}
wg.Wait()
```

### 4. Deadlock с вложенными locks

```go
// ❌ Deadlock
mu.Lock()
someFunction() // Внутри тоже вызывает mu.Lock()!
mu.Unlock()

// ✅ Избегайте вложенных блокировок или используйте RWMutex
```

### 5. Не сброс sync.Pool объектов

```go
// ❌ Утечка данных между использованиями
buf := bufferPool.Get().(*bytes.Buffer)
defer bufferPool.Put(buf) // Не очищен!

// ✅ Всегда очищайте перед возвратом
buf := bufferPool.Get().(*bytes.Buffer)
defer func() {
    buf.Reset()
    bufferPool.Put(buf)
}()
```

## Производительность

**Стоимость операций (приблизительно):**
- `Mutex.Lock/Unlock` (без состязания): ~20-30 ns
- `Mutex.Lock/Unlock` (с состязанием): ~100-500 ns
- `RWMutex.RLock/RUnlock`: ~30-40 ns
- `atomic` операции: ~5-10 ns
- Канал send/receive: ~50-100 ns

**Рекомендации:**
- Для счетчиков используйте [[Go - Атомарные операции|atomic]]
- Для сложных структур используйте Mutex/RWMutex
- Держите критические секции короткими
- Избегайте блокировок в hot paths

## Best Practices

### 1. Защищайте данные, а не код

```go
// ❌ Неясно, что защищает mutex
type Server struct {
    mu sync.Mutex
    // ...
}

// ✅ Явно группируйте защищаемые данные
type Server struct {
    // Защищено mu
    mu      sync.RWMutex
    clients map[string]*Client
    stats   Stats
    // Защищено другим mutex или не требует защиты
    config Config // read-only после init
}
```

### 2. Документируйте что защищается

```go
type Cache struct {
    mu    sync.RWMutex
    // Все поля ниже защищены mu
    items map[string]*Item
    stats CacheStats
}
```

### 3. Используйте defer для unlock

```go
// ✅ Всегда
mu.Lock()
defer mu.Unlock()
```

### 4. Минимизируйте критическую секцию

```go
// ❌ Долгая критическая секция
mu.Lock()
result := expensiveComputation(data)
process(result)
mu.Unlock()

// ✅ Выносите работу из критической секции
data := getData()
mu.Unlock()

result := expensiveComputation(data)
process(result)
```

### 5. Выбирайте правильный примитив

```go
// Счетчик - используйте atomic
var counter int64
atomic.AddInt64(&counter, 1)

// Сложная структура - используйте Mutex
type Stats struct {
    mu      sync.Mutex
    count   int
    average float64
}

// Передача данных - используйте каналы
results := make(chan Result)
```

## Связанные темы

- [[Go - Mutex и RWMutex]] - детальное рассмотрение мьютексов
- [[Go - WaitGroup и Once]] - синхронизация завершения
- [[Go - Атомарные операции]] - lock-free синхронизация
- [[Go - Горутины (goroutines)]] - конкурентное выполнение
- [[Go - Каналы (channels)]] - альтернатива sync для коммуникации
- [[Go - Context]] - управление отменой и таймаутами
- [[Многопоточность vs Параллелизм vs Конкурентность]] - концептуальные основы
