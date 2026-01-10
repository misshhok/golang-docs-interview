# Go - Escape Analysis

Escape Analysis (анализ убегания) — это оптимизация компилятора Go, которая определяет, где разместить переменную: на стеке или в куче. Понимание escape analysis критически важно для написания производительного кода.

## Stack vs Heap

```
┌─────────────────────────────────────────────────────────────┐
│                         ПАМЯТЬ                               │
├─────────────────────────────┬───────────────────────────────┤
│           STACK             │            HEAP               │
├─────────────────────────────┼───────────────────────────────┤
│ • Быстрое выделение         │ • Медленное выделение         │
│ • Автоматическое освобожд.  │ • Требует GC                  │
│ • Локальные переменные      │ • Динамические данные         │
│ • Фиксированный размер      │ • Растёт по необходимости     │
│ • LIFO порядок              │ • Произвольный доступ         │
│ • ~2KB на горутину          │ • Общий для всех горутин      │
└─────────────────────────────┴───────────────────────────────┘
```

| Характеристика | Stack | Heap |
|----------------|-------|------|
| Скорость выделения | ~1-2 ns | ~25-50 ns |
| Освобождение | Автоматически (при выходе из функции) | Garbage Collector |
| Фрагментация | Нет | Возможна |
| Размер | Ограничен (~1GB max) | Ограничен RAM |

## Что такое Escape Analysis?

Escape Analysis — процесс, при котором **компилятор** (не runtime!) решает:
- Переменная используется только внутри функции → **Stack**
- Переменная "убегает" за пределы функции → **Heap**

```go
// Stack allocation — переменная не убегает
func stackExample() int {
    x := 42        // x на стеке
    return x       // возвращается копия значения
}

// Heap allocation — переменная убегает
func heapExample() *int {
    x := 42        // x на куче!
    return &x      // возвращается указатель — x должен жить после return
}
```

## Причины escape на heap

### 1. Возврат указателя из функции

```go
// ❌ Убегает на heap
func createUser() *User {
    u := User{Name: "John"}  // u на heap
    return &u                 // указатель убегает
}

// ✅ Остаётся на stack (возврат по значению)
func createUserValue() User {
    u := User{Name: "John"}  // u на stack
    return u                  // копия возвращается
}
```

### 2. Присваивание указателю вне функции

```go
var global *int

func escape() {
    x := 42
    global = &x  // x убегает — присваивается глобальной переменной
}
```

### 3. Отправка указателя в канал

```go
func sendToChannel(ch chan *int) {
    x := 42
    ch <- &x  // x убегает — будет использоваться в другой горутине
}
```

### 4. Захват переменной в замыкании

```go
func closure() func() int {
    x := 42
    return func() int {
        return x  // x убегает — захвачена замыканием
    }
}
```

### 5. Передача в горутину

```go
func goroutineEscape() {
    x := 42
    go func() {
        fmt.Println(x)  // x убегает — горутина может пережить функцию
    }()
}
```

### 6. Слишком большой объект для стека

```go
func bigArray() {
    // Большие массивы часто убегают на heap
    arr := [1000000]int{}  // ~8MB — слишком много для стека
    _ = arr
}
```

### 7. Slice с неизвестным размером на compile-time

```go
func dynamicSlice(n int) {
    s := make([]int, n)  // размер известен только в runtime → heap
    _ = s
}

func staticSlice() {
    s := make([]int, 100)  // размер известен → может быть на stack
    _ = s
}
```

### 8. Interface conversion

```go
func interfaceEscape() {
    x := 42
    var i interface{} = x  // x убегает при конверсии в interface{}
    _ = i
}
```

## Как проверить Escape Analysis

### Флаг -gcflags '-m'

```bash
# Базовый анализ
go build -gcflags '-m' main.go

# Более детальный (-m -m)
go build -gcflags '-m -m' main.go

# Только для конкретного пакета
go build -gcflags '-m' ./pkg/...
```

### Пример вывода

```go
// main.go
package main

func noEscape() int {
    x := 42
    return x
}

func escape() *int {
    x := 42
    return &x
}

func main() {
    _ = noEscape()
    _ = escape()
}
```

```bash
$ go build -gcflags '-m' main.go

./main.go:9:2: moved to heap: x    # escape() — x убегает
./main.go:4:2: x does not escape   # noEscape() — x остаётся на стеке
```

### Интерпретация сообщений

| Сообщение | Значение |
|-----------|----------|
| `does not escape` | Переменная на стеке |
| `moved to heap` | Переменная убежала на heap |
| `leaking param` | Параметр функции утекает |
| `escapes to heap` | Значение убегает |

## Оптимизация: предотвращение escape

### 1. Возвращайте значения, не указатели (когда возможно)

```go
// ❌ Heap allocation
func NewUser() *User {
    return &User{Name: "John"}
}

// ✅ Stack allocation (если структура небольшая)
func NewUser() User {
    return User{Name: "John"}
}
```

### 2. Передавайте указатели внутрь, не наружу

```go
// ❌ Создаём внутри и возвращаем указатель
func Process() *Result {
    r := Result{}
    // ... заполняем r
    return &r  // r убегает
}

// ✅ Принимаем указатель снаружи
func Process(r *Result) {
    // ... заполняем r
    // r был создан вызывающим кодом
}

// Использование
func main() {
    var r Result     // r на стеке main
    Process(&r)      // передаём указатель внутрь
}
```

### 3. Используйте sync.Pool для частых аллокаций

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

func process() {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf)

    // используем buf
    // buf возвращается в pool, не создаётся заново
}
```

### 4. Избегайте interface{} где возможно

```go
// ❌ Конверсия в interface{} вызывает escape
func process(v interface{}) {
    // ...
}

// ✅ Используйте конкретный тип или generics
func process[T any](v T) {
    // ...
}
```

### 5. Предвыделяйте slice с известной capacity

```go
// ❌ Множественные реаллокации
func collect(n int) []int {
    var result []int
    for i := 0; i < n; i++ {
        result = append(result, i)  // может вызвать reallocate
    }
    return result
}

// ✅ Одна аллокация
func collect(n int) []int {
    result := make([]int, 0, n)  // capacity известна
    for i := 0; i < n; i++ {
        result = append(result, i)
    }
    return result
}
```

## Практические примеры

### Пример 1: String concatenation

```go
// ❌ Каждая конкатенация создаёт новую строку на heap
func buildString(parts []string) string {
    result := ""
    for _, p := range parts {
        result += p  // новая строка каждый раз
    }
    return result
}

// ✅ strings.Builder — одна аллокация
func buildString(parts []string) string {
    var b strings.Builder
    for _, p := range parts {
        b.WriteString(p)
    }
    return b.String()
}
```

### Пример 2: Slice of pointers vs values

```go
// ❌ Каждый элемент — отдельная аллокация
type Item struct {
    ID   int
    Name string
}

func createItems(n int) []*Item {
    items := make([]*Item, n)
    for i := 0; i < n; i++ {
        items[i] = &Item{ID: i}  // n heap allocations
    }
    return items
}

// ✅ Один непрерывный блок памяти
func createItems(n int) []Item {
    items := make([]Item, n)
    for i := 0; i < n; i++ {
        items[i] = Item{ID: i}  // 1 heap allocation
    }
    return items
}
```

### Пример 3: Map vs struct для options

```go
// ❌ Map — всегда heap
func process(opts map[string]interface{}) {
    // ...
}

// ✅ Struct — может быть на stack
type Options struct {
    Timeout int
    Retries int
}

func process(opts Options) {
    // ...
}
```

## Инструменты анализа

### 1. go build -gcflags

```bash
# Escape analysis
go build -gcflags '-m' .

# Детальный анализ
go build -gcflags '-m -m' .

# Все оптимизации
go build -gcflags '-m -m -l' .
```

### 2. Benchmarks с memory stats

```go
func BenchmarkStack(b *testing.B) {
    b.ReportAllocs()  // показывать аллокации
    for i := 0; i < b.N; i++ {
        _ = stackFunc()
    }
}

func BenchmarkHeap(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        _ = heapFunc()
    }
}
```

```bash
$ go test -bench=. -benchmem

BenchmarkStack-8   1000000000   0.5 ns/op   0 B/op   0 allocs/op
BenchmarkHeap-8    50000000    25 ns/op    8 B/op   1 allocs/op
```

### 3. pprof для heap профилирования

```go
import _ "net/http/pprof"

func main() {
    go http.ListenAndServe(":6060", nil)
    // ...
}
```

```bash
go tool pprof http://localhost:6060/debug/pprof/heap
```

## Мифы и заблуждения

### Миф 1: "Указатели всегда быстрее"

```go
// ❌ Миф: передача указателя быстрее
func process(u *User) { }

// Реальность: для маленьких структур копирование быстрее
// (меньше indirection, лучше cache locality)
func process(u User) { }  // если User < ~64 bytes — часто быстрее
```

### Миф 2: "Stack всегда лучше heap"

```go
// Stack ограничен (~1GB)
// Большие данные ДОЛЖНЫ быть на heap

// Также heap нужен для:
// - Данных, переживающих функцию
// - Shared данных между горутинами
```

### Миф 3: "Я могу контролировать allocation"

```go
// Нет гарантий! Компилятор решает сам.
// Даже если сейчас на stack — в следующей версии Go может измениться.
// Оптимизируйте только после профилирования.
```

## Вопросы с собеседований

### Вопрос 1: Что такое Escape Analysis?

<details>
<summary>Ответ</summary>

Escape Analysis — оптимизация компилятора Go, определяющая, где разместить переменную: на стеке или в куче.

- Если переменная используется только внутри функции → **стек** (быстро, автоматическое освобождение)
- Если переменная "убегает" за пределы функции → **куча** (требует GC)

Проверить можно через `go build -gcflags '-m'`.

</details>

### Вопрос 2: Назовите причины escape на heap

<details>
<summary>Ответ</summary>

1. Возврат указателя из функции
2. Присваивание глобальной переменной
3. Отправка в канал
4. Захват замыканием
5. Передача в горутину
6. Слишком большой объект
7. Slice с динамическим размером
8. Конверсия в interface{}

</details>

### Вопрос 3: Как оптимизировать код, чтобы избежать лишних аллокаций?

<details>
<summary>Ответ</summary>

1. **Возвращать значения, не указатели** (для небольших структур)
2. **Передавать указатели внутрь, а не наружу** — пусть вызывающий код владеет памятью
3. **Использовать sync.Pool** для частых аллокаций
4. **Избегать interface{}** — использовать конкретные типы или generics
5. **Предвыделять capacity** для slice и map
6. **Использовать strings.Builder** вместо конкатенации
7. **Профилировать** через `go build -gcflags '-m'` и benchmarks с `-benchmem`

</details>

### Вопрос 4: В чём разница между стеком и кучей в Go?

<details>
<summary>Ответ</summary>

| Stack | Heap |
|-------|------|
| Быстрое выделение (~1ns) | Медленнее (~25-50ns) |
| Автоматическое освобождение | Требует GC |
| ~2KB на горутину (растёт до 1GB) | Общий для всех горутин |
| Для локальных переменных | Для динамических данных |
| Нет фрагментации | Возможна фрагментация |

</details>

### Вопрос 5: Всегда ли указатели быстрее передачи по значению?

<details>
<summary>Ответ</summary>

Нет! Для маленьких структур (<64 bytes) передача по значению часто быстрее:

- Меньше indirection (прямой доступ vs разыменование)
- Лучше cache locality (данные рядом)
- Нет escape на heap

Указатели быстрее для больших структур (>64-128 bytes) и когда нужно модифицировать оригинал.

</details>

## Связанные темы

- [[Go - Управление памятью]]
- [[Go - Garbage Collector]]
- [[Go - Профилирование (pprof)]]
- [[Go - Бенчмаркинг]]
- [[Go - Указатели и ссылки]]
- [[Go - Внутреннее устройство slice]]
