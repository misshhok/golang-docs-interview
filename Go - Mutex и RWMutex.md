# Go - Mutex и RWMutex

Mutex (Mutual Exclusion) - примитив синхронизации для защиты shared state от конкурентного доступа. RWMutex - расширение mutex с поддержкой множественных читателей.

## Проблема: Data Race

Без синхронизации конкурентный доступ приводит к гонкам данных:

```go
// ❌ DATA RACE!
var counter int

func main() {
    for i := 0; i < 1000; i++ {
        go func() {
            counter++ // Небезопасно!
        }()
    }

    time.Sleep(time.Second)
    fmt.Println(counter) // Не обязательно 1000!
}
```

Запустите с `-race` для обнаружения:
```bash
go run -race main.go
```

## sync.Mutex

### Базовое использование

```go
import "sync"

var (
    counter int
    mu      sync.Mutex
)

func increment() {
    mu.Lock()   // Захватываем lock
    counter++   // Критическая секция
    mu.Unlock() // Освобождаем lock
}

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            increment()
        }()
    }

    wg.Wait()
    fmt.Println(counter) // Всегда 1000!
}
```

### Всегда используйте defer

```go
// ✅ Правильно - unlock гарантирован
func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()

    if c.count < 0 {
        return // unlock вызовется автоматически
    }
    c.count++
}

// ❌ Неправильно - можем забыть unlock
func (c *Counter) Inc() {
    c.mu.Lock()
    if c.count < 0 {
        return // Забыли unlock! Deadlock!
    }
    c.count++
    c.mu.Unlock()
}
```

### Инкапсуляция с mutex

```go
// ✅ Хороший дизайн - mutex скрыт
type SafeCounter struct {
    mu    sync.Mutex
    count map[string]int
}

func NewSafeCounter() *SafeCounter {
    return &SafeCounter{
        count: make(map[string]int),
    }
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count[key]++
}

func (c *SafeCounter) Value(key string) int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count[key]
}
```

**Не экспортируйте mutex:**
```go
// ❌ Плохо - mutex экспортирован
type Counter struct {
    Mu    sync.Mutex
    Count int
}

// ✅ Хорошо - mutex приватный
type Counter struct {
    mu    sync.Mutex
    count int
}
```

### Критическая секция

Держите критическую секцию **минимальной**:

```go
// ❌ Долгая критическая секция
func (c *Cache) Get(key string) (Value, error) {
    c.mu.Lock()
    defer c.mu.Unlock()

    val, ok := c.data[key]
    if !ok {
        // Долгая операция под lock'ом!
        val = expensiveFetch(key)
        c.data[key] = val
    }
    return val, nil
}

// ✅ Минимальная критическая секция
func (c *Cache) Get(key string) (Value, error) {
    c.mu.Lock()
    val, ok := c.data[key]
    c.mu.Unlock()

    if ok {
        return val, nil
    }

    // Вычисление вне lock'а
    val = expensiveFetch(key)

    c.mu.Lock()
    c.data[key] = val
    c.mu.Unlock()

    return val, nil
}
```

## sync.RWMutex

RWMutex поддерживает два типа блокировок:
- **Read lock** (RLock) - множественные читатели могут держать одновременно
- **Write lock** (Lock) - эксклюзивный доступ

### Базовое использование

```go
type SafeMap struct {
    mu   sync.RWMutex
    data map[string]int
}

// Множественные читатели
func (m *SafeMap) Get(key string) (int, bool) {
    m.mu.RLock()         // Read lock
    defer m.mu.RUnlock()
    val, ok := m.data[key]
    return val, ok
}

// Эксклюзивный писатель
func (m *SafeMap) Set(key string, value int) {
    m.mu.Lock()          // Write lock
    defer m.mu.Unlock()
    m.data[key] = value
}

// Множественные читатели
func (m *SafeMap) Len() int {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return len(m.data)
}
```

### Семантика блокировок

```
Read Lock (RLock):
- ✅ Множественные RLock одновременно
- ❌ Блокируется если есть Lock

Write Lock (Lock):
- ❌ Блокируется если есть RLock
- ❌ Блокируется если есть Lock
- ✅ Эксклюзивный доступ
```

### Когда использовать RWMutex

**Используйте RWMutex когда:**
- ✅ Частые чтения, редкие записи (>10:1 ratio)
- ✅ Критическая секция достаточно длинная (>1 микросекунда)
- ✅ Много читателей конкурируют одновременно

**НЕ используйте RWMutex когда:**
- ❌ Чтения и записи примерно поровну
- ❌ Очень короткие критические секции
- ❌ Мало конкурентных читателей

```go
// Benchmark: Mutex vs RWMutex
// Соотношение read:write = 90:10

// RWMutex: ~100 ns per op
// Mutex:   ~200 ns per op

// Соотношение read:write = 50:50

// RWMutex: ~250 ns per op
// Mutex:   ~200 ns per op
```

## Типичные паттерны

### 1. Cache с double-check

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]Value
}

func (c *Cache) GetOrCompute(key string, compute func() Value) Value {
    // Первая проверка с read lock
    c.mu.RLock()
    val, ok := c.data[key]
    c.mu.RUnlock()

    if ok {
        return val
    }

    // Вычисление вне lock'а
    newVal := compute()

    // Вторая проверка с write lock
    c.mu.Lock()
    defer c.mu.Unlock()

    // Проверяем еще раз - могли добавить пока вычисляли
    if val, ok := c.data[key]; ok {
        return val // Кто-то уже добавил
    }

    c.data[key] = newVal
    return newVal
}
```

### 2. Snapshot с copy

```go
type Stats struct {
    mu     sync.RWMutex
    counts map[string]int64
}

// Возвращаем копию для безопасного чтения
func (s *Stats) Snapshot() map[string]int64 {
    s.mu.RLock()
    defer s.mu.RUnlock()

    // Копируем map
    snapshot := make(map[string]int64, len(s.counts))
    for k, v := range s.counts {
        snapshot[k] = v
    }

    return snapshot // Безопасно использовать без lock'а
}
```

### 3. Защита нескольких полей

```go
type Server struct {
    mu       sync.RWMutex
    // Все поля ниже защищены mu
    clients  map[string]*Client
    requests int64
    errors   int64
}

func (s *Server) AddClient(id string, c *Client) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.clients[id] = c
}

func (s *Server) Stats() (requests, errors int64) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    return s.requests, s.errors
}
```

**Документируйте что защищается:**
```go
type Cache struct {
    // mu защищает items и stats
    mu    sync.RWMutex
    items map[string]Item
    stats CacheStats

    // config - read-only после инициализации
    config Config
}
```

### 4. Try-Lock Pattern (через select)

```go
type Resource struct {
    mu   sync.Mutex
    data string
}

// Неблокирующая попытка захватить lock
func (r *Resource) TryLock() bool {
    locked := make(chan struct{}, 1)

    go func() {
        r.mu.Lock()
        locked <- struct{}{}
    }()

    select {
    case <-locked:
        return true // Успешно захватили
    case <-time.After(100 * time.Millisecond):
        return false // Таймаут
    }
}
```

**Примечание**: Go не имеет встроенного TryLock (доступен с Go 1.18: `mu.TryLock()`).

## Типичные ошибки

### 1. Копирование mutex

```go
// ❌ ОШИБКА - копирование структуры с mutex
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c Counter) Inc() { // Получатель по значению!
    c.mu.Lock()   // Блокируем КОПИЮ mutex
    c.count++
    c.mu.Unlock()
}

// ✅ Правильно - указатель
func (c *Counter) Inc() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}
```

Используйте `go vet` для обнаружения:
```bash
$ go vet .
./main.go:10: Counter.Inc passes lock by value: Counter contains sync.Mutex
```

### 2. Забыть unlock

```go
// ❌ Deadlock
func (c *Cache) Get(key string) Value {
    c.mu.Lock()
    if val, ok := c.data[key]; ok {
        return val // Забыли unlock!
    }
    c.mu.Unlock()
    return nil
}

// ✅ defer гарантирует unlock
func (c *Cache) Get(key string) Value {
    c.mu.Lock()
    defer c.mu.Unlock()
    if val, ok := c.data[key]; ok {
        return val
    }
    return nil
}
```

### 3. Deadlock с вложенными locks

```go
// ❌ Deadlock
func (s *Service) Method1() {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.Method2() // Пытается захватить s.mu снова!
}

func (s *Service) Method2() {
    s.mu.Lock() // Deadlock - уже заблокирован!
    defer s.mu.Unlock()
    // ...
}

// ✅ Разделяйте public и private методы
func (s *Service) Method1() {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.method2Locked() // Не захватывает lock
}

func (s *Service) method2Locked() {
    // Предполагает что lock уже захвачен
}
```

### 4. Lock порядок (lock ordering)

```go
// ❌ Риск deadlock
// Goroutine 1:
mu1.Lock()
mu2.Lock()

// Goroutine 2:
mu2.Lock() // <-- Deadlock!
mu1.Lock()

// ✅ Всегда захватывайте locks в одном порядке
// Goroutine 1 и 2:
mu1.Lock()
mu2.Lock()
```

### 5. RWMutex RLock и RUnlock путаница

```go
// ❌ Mismatched lock/unlock
func (m *SafeMap) Get(key string) int {
    m.mu.RLock()
    defer m.mu.Unlock() // Неправильно! Должно быть RUnlock()
    return m.data[key]
}

// ✅ Правильно
func (m *SafeMap) Get(key string) int {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return m.data[key]
}
```

### 6. Lock с panic

```go
// ❌ Panic оставляет lock захваченным
func (c *Cache) Update(key string) {
    c.mu.Lock()
    val := c.compute(key) // Может вызвать panic!
    c.data[key] = val
    c.mu.Unlock() // Не вызовется при panic!
}

// ✅ defer гарантирует unlock даже при panic
func (c *Cache) Update(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    val := c.compute(key)
    c.data[key] = val
}
```

## Альтернативы mutex

### 1. Каналы

```go
// Mutex approach
type Counter struct {
    mu    sync.Mutex
    count int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// Channel approach
type Counter struct {
    ops chan func(*int)
}

func NewCounter() *Counter {
    c := &Counter{ops: make(chan func(*int))}
    go func() {
        var count int
        for op := range c.ops {
            op(&count)
        }
    }()
    return c
}

func (c *Counter) Inc() {
    c.ops <- func(count *int) { *count++ }
}
```

**Когда использовать каналы [[Go - Каналы (channels)]]:**
- Передача ownership
- Координация между [[Go - Горутины (goroutines)|горутинами]]
- Pipeline обработка

### 2. Atomic операции

```go
// Для простых счетчиков используйте atomic
import "sync/atomic"

var counter int64

func inc() {
    atomic.AddInt64(&counter, 1)
}
```

Подробнее: [[Go - Атомарные операции]]

## Производительность

**Стоимость операций (без состязания):**
- `Mutex.Lock/Unlock`: ~20-30 ns
- `RWMutex.Lock/Unlock`: ~25-35 ns
- `RWMutex.RLock/RUnlock`: ~30-40 ns
- `atomic.AddInt64`: ~5-10 ns
- Channel send/receive: ~50-100 ns

**С состязанием:**
- `Mutex` (contested): ~100-500 ns
- `RWMutex` (contested): ~150-600 ns

**Советы:**
- Минимизируйте время в критической секции
- Избегайте I/O под lock'ом
- Для счетчиков используйте `atomic`
- Для read-heavy loads используйте `RWMutex`

## Best Practices

### 1. Документируйте инварианты

```go
type Queue struct {
    mu    sync.Mutex
    // Инвариант: len(items) == size
    items []Item
    size  int
}
```

### 2. Один mutex на группу связанных полей

```go
// ✅ Один mutex защищает связанные данные
type Stats struct {
    mu      sync.Mutex
    count   int64
    sum     int64
    average float64 // Вычисляется из count и sum
}

// ❌ Слишком много locks
type Stats struct {
    muCount   sync.Mutex
    count     int64
    muSum     sync.Mutex
    sum       int64
    muAverage sync.Mutex
    average   float64
}
```

### 3. Избегайте блокировок в hot paths

```go
// ❌ Lock в hot path
func (s *Server) HandleRequest(req Request) {
    s.mu.Lock()
    s.requestCount++
    s.mu.Unlock()
    // Обработка
}

// ✅ Используйте atomic
func (s *Server) HandleRequest(req Request) {
    atomic.AddInt64(&s.requestCount, 1)
    // Обработка
}
```

### 4. Коротко держите lock

```go
// ✅ Копируем данные под lock'ом, обрабатываем снаружи
func (s *Service) Process() {
    s.mu.RLock()
    data := s.data.Copy()
    s.mu.RUnlock()

    // Долгая обработка вне lock'а
    result := expensiveCompute(data)

    s.mu.Lock()
    s.result = result
    s.mu.Unlock()
}
```

### 5. Тестируйте с -race

```bash
go test -race ./...
go run -race main.go
```

Race detector находит большинство data races в runtime!

## Связанные темы

- [[Go - Пакет sync]] - обзор всех примитивов синхронизации
- [[Go - Атомарные операции]] - lock-free синхронизация
- [[Go - WaitGroup и Once]] - другие sync примитивы
- [[Go - Горутины (goroutines)]] - конкурентное выполнение
- [[Go - Каналы (channels)]] - альтернатива mutex
- [[Многопоточность vs Параллелизм vs Конкурентность]] - концепции
