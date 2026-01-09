# Go - Атомарные операции

Пакет `sync/atomic` предоставляет low-level атомарные операции для lock-free синхронизации. Атомарные операции выполняются как единое неделимое действие без возможности прерывания.

## Зачем нужны атомарные операции

**Проблема с обычными операциями:**
```go
// ❌ НЕ атомарно - data race!
var counter int64

func increment() {
    counter++ // Три операции: read, modify, write
}
```

Операция `counter++` компилируется в:
1. Загрузить `counter` в регистр (READ)
2. Увеличить значение (MODIFY)
3. Записать обратно (WRITE)

Между этими шагами может произойти context switch!

**Решение 1: Mutex**
```go
var (
    counter int64
    mu      sync.Mutex
)

func increment() {
    mu.Lock()
    counter++
    mu.Unlock()
}
```

**Решение 2: Atomic (быстрее!)**
```go
var counter int64

func increment() {
    atomic.AddInt64(&counter, 1) // Одна неделимая операция
}
```

## Поддерживаемые типы

Атомарные операции поддерживают:
- `int32`, `int64`
- `uint32`, `uint64`
- `uintptr`
- `unsafe.Pointer`
- Go 1.19+: `atomic.Bool`, `atomic.Int32`, `atomic.Int64`, etc.

## Основные операции

### 1. Load - Атомарное чтение

```go
import "sync/atomic"

var counter int64

// Атомарное чтение
value := atomic.LoadInt64(&counter)
fmt.Println(value)

// Для других типов
var flag uint32
val := atomic.LoadUint32(&flag)

var ptr unsafe.Pointer
p := atomic.LoadPointer(&ptr)
```

### 2. Store - Атомарная запись

```go
var counter int64

// Атомарная запись
atomic.StoreInt64(&counter, 42)

// Неправильно!
// counter = 42 // ❌ НЕ атомарно!

// Правильно
atomic.StoreInt64(&counter, 42) // ✅ Атомарно
```

### 3. Add - Атомарное увеличение/уменьшение

```go
var counter int64

// Увеличить на 1
atomic.AddInt64(&counter, 1)

// Уменьшить на 1 (через отрицательное число)
atomic.AddInt64(&counter, -1)

// Увеличить на N
atomic.AddInt64(&counter, 10)
```

**Примечание**: Нет `Sub` операции, используйте отрицательное число в `Add`.

### 4. Swap - Атомарный обмен

```go
var value int64 = 10

// Атомарно заменить значение и вернуть старое
old := atomic.SwapInt64(&value, 20)
fmt.Println(old)   // 10
fmt.Println(value) // 20
```

### 5. CompareAndSwap (CAS) - Условная атомарная замена

Самая мощная атомарная операция!

```go
var value int64 = 10

// Заменить только если текущее значение == old
swapped := atomic.CompareAndSwapInt64(&value, 10, 20)
fmt.Println(swapped) // true (замена произошла)
fmt.Println(value)   // 20

// Попытка с неправильным old
swapped = atomic.CompareAndSwapInt64(&value, 10, 30)
fmt.Println(swapped) // false (замена НЕ произошла)
fmt.Println(value)   // 20 (не изменилось)
```

**CAS - основа для lock-free алгоритмов!**

## Практические примеры

### 1. Атомарный счетчик

```go
type Counter struct {
    value int64
}

func (c *Counter) Inc() int64 {
    return atomic.AddInt64(&c.value, 1)
}

func (c *Counter) Dec() int64 {
    return atomic.AddInt64(&c.value, -1)
}

func (c *Counter) Get() int64 {
    return atomic.LoadInt64(&c.value)
}

func (c *Counter) Set(n int64) {
    atomic.StoreInt64(&c.value, n)
}

// Использование
func main() {
    var counter Counter
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
    fmt.Println(counter.Get()) // Всегда 1000!
}
```

### 2. Атомарный флаг (bool)

```go
// До Go 1.19
type AtomicBool struct {
    flag uint32
}

func (b *AtomicBool) Set(value bool) {
    var i uint32 = 0
    if value {
        i = 1
    }
    atomic.StoreUint32(&b.flag, i)
}

func (b *AtomicBool) Get() bool {
    return atomic.LoadUint32(&b.flag) != 0
}

// Go 1.19+
var flag atomic.Bool

flag.Store(true)
if flag.Load() {
    fmt.Println("Flag is set")
}
```

### 3. Once-like behavior с CAS

```go
type Once struct {
    done uint32
}

func (o *Once) Do(f func()) {
    // Быстрая проверка без lock'а
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    // Пытаемся захватить через CAS
    if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
        f() // Выполняем только если CAS успешен
    }
}

// Использование
var once Once
once.Do(func() {
    fmt.Println("Executed once")
})
```

### 4. Spinlock (простая lock-free блокировка)

```go
type SpinLock struct {
    state uint32
}

func (s *SpinLock) Lock() {
    // Крутимся пока не захватим lock
    for !atomic.CompareAndSwapUint32(&s.state, 0, 1) {
        runtime.Gosched() // Отдаем CPU другим горутинам
    }
}

func (s *SpinLock) Unlock() {
    atomic.StoreUint32(&s.state, 0)
}

// ⚠️ Spinlock подходит только для очень коротких критических секций!
```

### 5. Lock-free Stack

```go
type Node struct {
    value int
    next  *Node
}

type Stack struct {
    head unsafe.Pointer // *Node
}

func (s *Stack) Push(value int) {
    node := &Node{value: value}

    for {
        old := atomic.LoadPointer(&s.head)
        node.next = (*Node)(old)

        if atomic.CompareAndSwapPointer(&s.head, old, unsafe.Pointer(node)) {
            return
        }
        // CAS failed, retry
    }
}

func (s *Stack) Pop() (int, bool) {
    for {
        old := atomic.LoadPointer(&s.head)
        if old == nil {
            return 0, false // Empty stack
        }

        node := (*Node)(old)
        if atomic.CompareAndSwapPointer(&s.head, old, unsafe.Pointer(node.next)) {
            return node.value, true
        }
        // CAS failed, retry
    }
}
```

### 6. Graceful shutdown флаг

```go
type Server struct {
    shutdown atomic.Bool
}

func (s *Server) Serve() {
    for !s.shutdown.Load() {
        // Обрабатываем запросы
        handleRequest()
    }
    fmt.Println("Server stopped")
}

func (s *Server) Shutdown() {
    s.shutdown.Store(true)
}

func main() {
    server := &Server{}

    go server.Serve()

    time.Sleep(5 * time.Second)
    server.Shutdown()
}
```

### 7. Reference counting

```go
type RefCounted struct {
    refCount int32
    data     interface{}
}

func (r *RefCounted) AddRef() {
    atomic.AddInt32(&r.refCount, 1)
}

func (r *RefCounted) Release() {
    if atomic.AddInt32(&r.refCount, -1) == 0 {
        // Последняя ссылка, освобождаем ресурс
        r.cleanup()
    }
}

func (r *RefCounted) cleanup() {
    fmt.Println("Cleaning up resource")
    r.data = nil
}
```

## atomic.Value - Для любых типов

`atomic.Value` позволяет атомарно загружать и сохранять значения **любого типа**.

```go
var config atomic.Value

// Хранит Config struct
type Config struct {
    Host string
    Port int
}

func loadConfig() {
    cfg := Config{Host: "localhost", Port: 8080}
    config.Store(cfg)
}

func getConfig() Config {
    return config.Load().(Config)
}

// Hot reload конфигурации
func watchConfig() {
    for {
        newCfg := loadConfigFromFile()
        config.Store(newCfg) // Атомарное обновление!
        time.Sleep(10 * time.Second)
    }
}

func handler(w http.ResponseWriter, r *http.Request) {
    cfg := getConfig() // Всегда получаем консистентный Config
    connectTo(cfg.Host, cfg.Port)
}
```

**Ограничения atomic.Value:**
- Первый `Store()` определяет тип - нельзя менять!
- Нельзя хранить `nil` (используйте указатель)
- Нельзя хранить `interface{}` с разными конкретными типами

```go
var v atomic.Value

v.Store(42)      // Тип int
v.Store("hello") // ❌ Panic! Другой тип!

v.Store(nil)     // ❌ Panic! Нельзя nil

// ✅ Используйте указатель для nil
v.Store((*Config)(nil))
```

## Go 1.19+ Typed Atomic Types

С Go 1.19 появились типизированные атомарные типы:

```go
// Вместо старого подхода
var counter int64
atomic.AddInt64(&counter, 1)

// Новый подход (Go 1.19+)
var counter atomic.Int64
counter.Add(1)

// Доступные типы
var (
    b   atomic.Bool
    i32 atomic.Int32
    i64 atomic.Int64
    u32 atomic.Uint32
    u64 atomic.Uint64
    ptr atomic.Pointer[T]
)

// Методы
b.Store(true)
b.Load()
b.Swap(false)
b.CompareAndSwap(old, new)

i64.Add(1)
i64.Load()
i64.Store(42)
i64.Swap(100)
i64.CompareAndSwap(old, new)
```

**Преимущества:**
- ✅ Не нужно передавать указатель
- ✅ Типобезопасность
- ✅ Чище и понятнее код

## Когда использовать atomic

### ✅ Используйте atomic когда:

1. **Простые счетчики**
```go
var requestCount atomic.Int64
requestCount.Add(1)
```

2. **Флаги и состояния**
```go
var shutdown atomic.Bool
if shutdown.Load() { return }
```

3. **Lock-free алгоритмы**
```go
// CAS loops для lock-free структур данных
```

4. **Read-heavy конфигурация**
```go
var config atomic.Value
cfg := config.Load().(Config)
```

### ❌ НЕ используйте atomic когда:

1. **Сложные структуры данных**
```go
// ❌ Используйте Mutex
type Cache struct {
    items map[string]Item
    mu    sync.RWMutex
}
```

2. **Множественные связанные поля**
```go
// ❌ Atomic не гарантирует консистентность между полями
var count atomic.Int64
var sum atomic.Int64

// ✅ Используйте Mutex для связанных данных
type Stats struct {
    mu    sync.Mutex
    count int64
    sum   int64
}
```

3. **Нужен порядок операций**
```go
// Atomic не гарантирует ordering между разными переменными!
// Используйте Mutex или каналы
```

## Memory Ordering

Атомарные операции обеспечивают **happens-before** гарантии:

```go
var data int
var flag atomic.Bool

// Goroutine 1
data = 42           // (1)
flag.Store(true)    // (2)

// Goroutine 2
if flag.Load() {    // (3)
    fmt.Println(data) // (4) Всегда увидит 42!
}
```

(1) happens-before (2) happens-before (3) happens-before (4)

**Без atomic:**
```go
var data int
var flag bool

// Goroutine 1
data = 42    // (1)
flag = true  // (2) Может быть переупорядочено!

// Goroutine 2
if flag {           // (3)
    fmt.Println(data) // (4) Может увидеть 0! Data race!
}
```

## Производительность

**Сравнение времени операций:**

```
Операция                      Время
----------------------------------------
Обычный инкремент            ~1 ns
atomic.AddInt64              ~5-10 ns
sync.Mutex Lock/Unlock       ~20-30 ns
Channel send/receive         ~50-100 ns
```

**Atomic в 2-3 раза быстрее Mutex!**

Но atomic работают только для простых типов.

**Benchmark:**
```go
// Atomic
BenchmarkAtomicAdd      200000000    7.5 ns/op

// Mutex
BenchmarkMutexInc       50000000     25 ns/op

// Channel
BenchmarkChannelInc     20000000     75 ns/op
```

## Типичные ошибки

### 1. Забыть взять адрес

```go
// ❌ Compile error
var counter int64
atomic.AddInt64(counter, 1) // Нужен указатель!

// ✅ Правильно
atomic.AddInt64(&counter, 1)

// ✅ С Go 1.19+
var counter atomic.Int64
counter.Add(1) // Не нужен указатель
```

### 2. Смешивать атомарные и обычные операции

```go
// ❌ Data race!
var counter int64

go func() {
    atomic.AddInt64(&counter, 1) // Атомарно
}()

go func() {
    counter++ // НЕ атомарно! Data race!
}()

// ✅ Все операции должны быть атомарными
```

### 3. Использовать для сложных структур

```go
// ❌ Не защищает связанные поля
var count atomic.Int64
var sum atomic.Int64

// Между этими операциями могут вклиниться другие горутины!
count.Add(1)
sum.Add(value) // Нет атомарности между count и sum!

// ✅ Используйте Mutex
type Stats struct {
    mu    sync.Mutex
    count int64
    sum   int64
}
```

### 4. Ожидать ordering между переменными

```go
// ❌ Ordering не гарантирован!
var x atomic.Int64
var y atomic.Int64

// Goroutine 1
x.Store(1)
y.Store(1)

// Goroutine 2 может увидеть y=1, x=0!
```

### 5. Неправильный тип в atomic.Value

```go
// ❌ Panic - разные типы!
var v atomic.Value
v.Store(42)
v.Store("hello") // Panic!

// ✅ Всегда один тип
var v atomic.Value
v.Store(&Config{...})
v.Store(&Config{...}) // OK
```

## Best Practices

1. ✅ Используйте atomic для простых счетчиков и флагов
2. ✅ Для сложных структур используйте [[Go - Mutex и RWMutex]]
3. ✅ Всегда документируйте атомарные поля
4. ✅ С Go 1.19+ используйте typed atomic types
5. ✅ Не смешивайте atomic и non-atomic операции
6. ✅ Используйте `go run -race` для проверки

```go
// ✅ Документируйте!
type Server struct {
    // requestCount is accessed atomically
    requestCount atomic.Int64
}
```

## Связанные темы

- [[Go - Пакет sync]] - другие примитивы синхронизации
- [[Go - Mutex и RWMutex]] - альтернатива для сложных случаев
- [[Go - Горутины (goroutines)]] - конкурентное выполнение
- [[Go - Каналы (channels)]] - другой способ синхронизации
- [[Многопоточность vs Параллелизм vs Конкурентность]] - концепции
