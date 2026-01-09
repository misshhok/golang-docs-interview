# Go - Управление памятью

Обзор того, как Go управляет памятью: стек, куча, аллокации, escape analysis.

## Стек vs Куча

### Stack (Стек)

Быстрое выделение памяти для локальных переменных:

```go
func sum(a, b int) int {
    result := a + b // result на стеке
    return result
} // стек очищается автоматически
```

**Характеристики:**
- ✅ Очень быстрое выделение (bump pointer)
- ✅ Автоматическое освобождение при выходе из функции
- ✅ Не требует GC
- ❌ Ограниченный размер
- ❌ Данные живут только в рамках функции

### Heap (Куча)

Память для долгоживущих объектов:

```go
func createUser() *User {
    user := &User{Name: "Alice"} // user на куче!
    return user
} // user продолжает жить
```

**Характеристики:**
- ✅ Большой размер
- ✅ Данные живут пока есть ссылки
- ❌ Медленнее выделение
- ❌ Требует GC для очистки

## Escape Analysis

Компилятор решает: стек или куча?

### Переменная остается на стеке

```go
func local() {
    x := 42 // Остается на стеке
    fmt.Println(x)
}
```

### Переменная "убегает" на кучу

```go
// 1. Возврат указателя
func escape1() *int {
    x := 42
    return &x // x убегает на кучу!
}

// 2. Присваивание в поле структуры
type Data struct {
    Value *int
}

func escape2() Data {
    x := 42
    return Data{Value: &x} // x убегает!
}

// 3. Отправка в канал
func escape3() {
    x := 42
    ch := make(chan *int)
    ch <- &x // x убегает!
}

// 4. Закрытие (closure) с указателем
func escape4() func() int {
    x := 42
    return func() int {
        return x // x убегает!
    }
}

// 5. Слишком большой размер
func escape5() {
    // Большие объекты идут на кучу
    data := make([]byte, 10*1024*1024) // 10 MB на куче
    _ = data
}
```

### Проверка escape analysis

```bash
# Анализ компилятором
go build -gcflags="-m" main.go

# Вывод:
# ./main.go:5:2: moved to heap: x
# ./main.go:6:9: &x escapes to heap
```

### Оптимизация: избегайте escape

```go
// ❌ Аллокация на куче
func bad() *Result {
    result := &Result{Value: 42}
    return result
}

// ✅ Возврат по значению
func good() Result {
    result := Result{Value: 42}
    return result // Может остаться на стеке
}
```

## Размер аллокаций

### Маленькие объекты (< 32 KB)

Выделяются из thread-local кэшей (быстро):

```go
// Быстро
x := make([]byte, 1024) // 1 KB
```

### Большие объекты (>= 32 KB)

Выделяются напрямую из heap (медленнее):

```go
// Медленнее
x := make([]byte, 64*1024) // 64 KB
```

## Измерение аллокаций

### runtime.MemStats

```go
var m1, m2 runtime.MemStats
runtime.ReadMemStats(&m1)

// Ваш код
doWork()

runtime.ReadMemStats(&m2)
fmt.Printf("Allocs: %d\n", m2.Mallocs-m1.Mallocs)
fmt.Printf("Bytes: %d\n", m2.TotalAlloc-m1.TotalAlloc)
```

### Benchmark аллокаций

```go
func BenchmarkSomething(b *testing.B) {
    b.ReportAllocs() // Показать аллокации

    for i := 0; i < b.N; i++ {
        doSomething()
    }
}

// Вывод:
// BenchmarkSomething-8    1000000    1043 ns/op    240 B/op    5 allocs/op
//                                                   ^^^         ^^^
//                                                   байт        количество
```

### pprof

```go
import _ "net/http/pprof"

go func() {
    http.ListenAndServe("localhost:6060", nil)
}()

// go tool pprof http://localhost:6060/debug/pprof/allocs
```

## Оптимизация аллокаций

### 1. Sync.Pool для переиспользования

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process() {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()

    // Используем buf
}
```

### 2. Pre-allocation

```go
// ❌ Многократное перевыделение
s := []int{}
for i := 0; i < 10000; i++ {
    s = append(s, i) // Много аллокаций
}

// ✅ Одна аллокация
s := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    s = append(s, i)
}
```

### 3. Strings.Builder вместо конкатенации

```go
// ❌ Много аллокаций
var s string
for i := 0; i < 100; i++ {
    s += "text" // Каждая конкатенация = новая аллокация
}

// ✅ Одна аллокация
var builder strings.Builder
builder.Grow(400) // Pre-allocate
for i := 0; i < 100; i++ {
    builder.WriteString("text")
}
s := builder.String()
```

### 4. Возврат по значению вместо указателя

```go
// ❌ Аллокация на куче
func NewPoint() *Point {
    return &Point{X: 1, Y: 2}
}

// ✅ Может остаться на стеке
func NewPoint() Point {
    return Point{X: 1, Y: 2}
}
```

### 5. Переиспользование массивов

```go
// ❌ Новая аллокация каждый раз
func process(data []int) []int {
    result := make([]int, len(data))
    // обработка...
    return result
}

// ✅ Переиспользуем существующий слайс
func process(data []int, result []int) {
    if cap(result) < len(data) {
        result = make([]int, len(data))
    }
    result = result[:len(data)]
    // обработка...
}
```

## Утечки памяти

### 1. Slice держит ссылку на большой массив

```go
// ❌ Утечка
func leak() []byte {
    bigData := make([]byte, 10*1024*1024) // 10 MB
    return bigData[:100] // Возвращаем 100 байт, держим 10 MB!
}

// ✅ Копируем нужную часть
func noLeak() []byte {
    bigData := make([]byte, 10*1024*1024)
    result := make([]byte, 100)
    copy(result, bigData[:100])
    return result
}
```

### 2. Забытые горутины

```go
// ❌ Горутина висит навсегда
func leak() {
    ch := make(chan int)
    go func() {
        <-ch // Блокируется навсегда, утечка!
    }()
}

// ✅ С таймаутом или контекстом
func noLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case <-ch:
        case <-ctx.Done():
            return // Горутина завершится
        }
    }()
}
```

### 3. Незакрытые ресурсы

```go
// ❌ Утечка
func leak() {
    file, _ := os.Open("file.txt")
    // Забыли закрыть!
}

// ✅ С defer
func noLeak() {
    file, err := os.Open("file.txt")
    if err != nil {
        return
    }
    defer file.Close()
    // ...
}
```

### 4. Map/slice с указателями

```go
// ❌ Удаление ключа не освобождает объект
cache := make(map[string]*BigObject)
cache["key"] = &BigObject{...}
delete(cache, "key") // Объект все еще может быть в памяти!

// ✅ Явно обнуляем
cache["key"] = nil
delete(cache, "key")
```

## Выравнивание (Alignment)

Структуры выравниваются для эффективного доступа:

```go
// ❌ Неоптимально: 24 bytes
type Bad struct {
    a bool   // 1 byte + 7 padding
    b int64  // 8 bytes
    c bool   // 1 byte + 7 padding
}

// ✅ Оптимально: 16 bytes
type Good struct {
    b int64  // 8 bytes
    a bool   // 1 byte
    c bool   // 1 byte + 6 padding
}

fmt.Println(unsafe.Sizeof(Bad{}))  // 24
fmt.Println(unsafe.Sizeof(Good{})) // 16
```

**Правило:** Располагайте поля от большего к меньшему.

## Zero-copy оптимизации

### String to []byte без копирования

```go
// ⚠️ Unsafe, но быстро
func stringToBytes(s string) []byte {
    return *(*[]byte)(unsafe.Pointer(&s))
}

// Обычно так делать НЕ нужно!
// Используйте только если профилирование показывает необходимость
```

### mmap для больших файлов

```go
// Вместо чтения всего файла
data, err := ioutil.ReadFile("huge.txt") // Копирование в память

// Используйте mmap (через внешние библиотеки)
// Файл отображается в память без копирования
```

## Размер базовых типов

```go
import "unsafe"

// 64-bit система:
unsafe.Sizeof(int(0))        // 8
unsafe.Sizeof(int8(0))       // 1
unsafe.Sizeof(int32(0))      // 4
unsafe.Sizeof(int64(0))      // 8
unsafe.Sizeof(float64(0))    // 8
unsafe.Sizeof(true)          // 1
unsafe.Sizeof("hello")       // 16 (string header: ptr + len)
unsafe.Sizeof([]int{})       // 24 (slice header: ptr + len + cap)
unsafe.Sizeof(map[string]int{}) // 8 (pointer)
unsafe.Sizeof(interface{}(nil)) // 16 (interface: type + value)
```

## Best Practices

1. ✅ Измеряйте перед оптимизацией (benchmarks, pprof)
2. ✅ Pre-allocate когда знаете размер
3. ✅ Используйте sync.Pool для горячих путей
4. ✅ Избегайте escape на кучу (возврат по значению)
5. ✅ Используйте strings.Builder для конкатенации
6. ✅ Закрывайте ресурсы с defer
7. ✅ Копируйте подслайсы больших массивов
8. ❌ Не оптимизируйте преждевременно
9. ❌ Не используйте unsafe без веских причин

## Инструменты диагностики

```bash
# Escape analysis
go build -gcflags="-m" main.go

# Benchmark с аллокациями
go test -bench=. -benchmem

# Memory profiling
go test -memprofile=mem.prof
go tool pprof mem.prof

# Trace
go test -trace=trace.out
go tool trace trace.out

# Runtime stats
GODEBUG=gctrace=1 ./myapp
```

## Связанные темы

- [[Go - Garbage Collector]]
- [[Go - Профилирование (pprof)]]
- [[Go - Бенчмаркинг]]
- [[Go - Структуры (struct)]]
- [[Go - Массивы и слайсы]]
