# Go - Generics (Дженерики)

Generics (дженерики) — это механизм параметризации типов, добавленный в Go 1.18. Позволяет писать функции и структуры данных, которые работают с разными типами без потери типобезопасности и без дублирования кода.

## Зачем нужны дженерики?

### Проблема до Go 1.18

```go
// ❌ Дублирование кода для каждого типа
func SumInts(nums []int) int {
    var sum int
    for _, n := range nums {
        sum += n
    }
    return sum
}

func SumFloats(nums []float64) float64 {
    var sum float64
    for _, n := range nums {
        sum += n
    }
    return sum
}

// ❌ Или потеря типобезопасности с interface{}
func SumAny(nums []interface{}) interface{} {
    // Нужны type assertions, возможны runtime паники
}
```

### Решение с дженериками

```go
// ✅ Одна функция для всех числовых типов
func Sum[T int | int64 | float64](nums []T) T {
    var sum T
    for _, n := range nums {
        sum += n
    }
    return sum
}

// Использование
ints := []int{1, 2, 3}
floats := []float64{1.1, 2.2, 3.3}

fmt.Println(Sum(ints))    // 6 (тип выводится автоматически)
fmt.Println(Sum(floats))  // 6.6
```

## Синтаксис

### Type Parameters (параметры типа)

```go
// [T constraint] — параметр типа с ограничением
func Print[T any](value T) {
    fmt.Println(value)
}

// Несколько параметров типа
func Map[T, U any](slice []T, f func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}

// Использование
doubled := Map([]int{1, 2, 3}, func(x int) int { return x * 2 })
// [2, 4, 6]

strings := Map([]int{1, 2, 3}, func(x int) string { return fmt.Sprint(x) })
// ["1", "2", "3"]
```

### Type Constraints (ограничения типов)

```go
// any — любой тип (алиас для interface{})
func Print[T any](v T) { fmt.Println(v) }

// comparable — типы, которые можно сравнивать через == и !=
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}

// Union типов — через |
func Add[T int | int64 | float64](a, b T) T {
    return a + b
}

// Интерфейс как constraint
type Stringer interface {
    String() string
}

func PrintString[T Stringer](v T) {
    fmt.Println(v.String())
}
```

### Встроенные constraints (пакет constraints)

```go
import "golang.org/x/exp/constraints"

// constraints.Ordered — типы с операторами <, >, <=, >=
func Max[T constraints.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

// constraints.Integer — все целочисленные типы
// constraints.Float — все float типы
// constraints.Complex — complex64, complex128
// constraints.Signed — знаковые целые
// constraints.Unsigned — беззнаковые целые
```

### Кастомные constraints

```go
// Определение своего constraint
type Number interface {
    int | int8 | int16 | int32 | int64 |
    uint | uint8 | uint16 | uint32 | uint64 |
    float32 | float64
}

func Sum[T Number](nums []T) T {
    var sum T
    for _, n := range nums {
        sum += n
    }
    return sum
}

// Constraint с методами
type Validator interface {
    Validate() error
}

func ValidateAll[T Validator](items []T) error {
    for _, item := range items {
        if err := item.Validate(); err != nil {
            return err
        }
    }
    return nil
}

// Комбинация: тип + методы
type OrderedStringer interface {
    constraints.Ordered
    String() string
}
```

### Тильда (~) — underlying types

```go
// ~ означает "этот тип и все типы с таким underlying type"
type MyInt int

type Integer interface {
    ~int | ~int64  // включает int, int64, MyInt и др.
}

func Double[T Integer](v T) T {
    return v * 2
}

var x MyInt = 5
fmt.Println(Double(x))  // 10 (работает благодаря ~)
```

## Generic типы (структуры)

```go
// Generic стек
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack[T]) Peek() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

// Использование
intStack := Stack[int]{}
intStack.Push(1)
intStack.Push(2)
val, _ := intStack.Pop()  // 2

stringStack := Stack[string]{}
stringStack.Push("hello")
```

### Generic Map (словарь)

```go
type Map[K comparable, V any] struct {
    data map[K]V
}

func NewMap[K comparable, V any]() *Map[K, V] {
    return &Map[K, V]{
        data: make(map[K]V),
    }
}

func (m *Map[K, V]) Set(key K, value V) {
    m.data[key] = value
}

func (m *Map[K, V]) Get(key K) (V, bool) {
    val, ok := m.data[key]
    return val, ok
}

func (m *Map[K, V]) Keys() []K {
    keys := make([]K, 0, len(m.data))
    for k := range m.data {
        keys = append(keys, k)
    }
    return keys
}
```

## Type Inference (вывод типов)

```go
// Компилятор автоматически выводит типы
func Identity[T any](v T) T { return v }

x := Identity(42)        // T = int (выведено)
y := Identity("hello")   // T = string (выведено)

// Явное указание типа (когда нужно)
z := Identity[float64](42)  // T = float64 (явно)

// Вывод не работает для типов без аргументов
type Container[T any] struct{ Value T }

// ❌ Ошибка: не может вывести T
// c := Container{Value: 42}

// ✅ Нужно указать явно
c := Container[int]{Value: 42}
```

## Практические примеры

### Filter, Map, Reduce

```go
// Filter — отфильтровать элементы
func Filter[T any](slice []T, predicate func(T) bool) []T {
    result := make([]T, 0)
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

// Map — преобразовать элементы
func Map[T, U any](slice []T, mapper func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = mapper(v)
    }
    return result
}

// Reduce — свернуть в одно значение
func Reduce[T, U any](slice []T, initial U, reducer func(U, T) U) U {
    result := initial
    for _, v := range slice {
        result = reducer(result, v)
    }
    return result
}

// Использование
nums := []int{1, 2, 3, 4, 5}

evens := Filter(nums, func(n int) bool { return n%2 == 0 })
// [2, 4]

squares := Map(nums, func(n int) int { return n * n })
// [1, 4, 9, 16, 25]

sum := Reduce(nums, 0, func(acc, n int) int { return acc + n })
// 15
```

### Generic Result type

```go
// Result — аналог Either/Result из функциональных языков
type Result[T any] struct {
    value T
    err   error
}

func Ok[T any](value T) Result[T] {
    return Result[T]{value: value}
}

func Err[T any](err error) Result[T] {
    return Result[T]{err: err}
}

func (r Result[T]) IsOk() bool {
    return r.err == nil
}

func (r Result[T]) Unwrap() T {
    if r.err != nil {
        panic(r.err)
    }
    return r.value
}

func (r Result[T]) UnwrapOr(defaultVal T) T {
    if r.err != nil {
        return defaultVal
    }
    return r.value
}

// Использование
func Divide(a, b int) Result[int] {
    if b == 0 {
        return Err[int](errors.New("division by zero"))
    }
    return Ok(a / b)
}

result := Divide(10, 2)
if result.IsOk() {
    fmt.Println(result.Unwrap())  // 5
}
```

### Generic Optional

```go
type Optional[T any] struct {
    value   T
    present bool
}

func Some[T any](value T) Optional[T] {
    return Optional[T]{value: value, present: true}
}

func None[T any]() Optional[T] {
    return Optional[T]{}
}

func (o Optional[T]) IsSome() bool {
    return o.present
}

func (o Optional[T]) Get() (T, bool) {
    return o.value, o.present
}

func (o Optional[T]) OrElse(defaultVal T) T {
    if o.present {
        return o.value
    }
    return defaultVal
}
```

## Когда использовать дженерики?

### ✅ Хорошие случаи

```go
// 1. Контейнеры данных
type Queue[T any] struct { ... }
type Set[T comparable] struct { ... }
type Cache[K comparable, V any] struct { ... }

// 2. Утилитарные функции
func Min[T constraints.Ordered](a, b T) T { ... }
func Keys[K comparable, V any](m map[K]V) []K { ... }
func Unique[T comparable](slice []T) []T { ... }

// 3. Функциональные утилиты
func Map[T, U any](slice []T, f func(T) U) []U { ... }
func Filter[T any](slice []T, pred func(T) bool) []T { ... }
```

### ❌ Плохие случаи

```go
// 1. Когда достаточно interface{}
// Если работаете с разными типами через рефлексию — дженерики не нужны

// 2. Когда нужен только один тип
// ❌ Оверинжиниринг
func ProcessUser[T User](u T) { ... }
// ✅ Просто
func ProcessUser(u User) { ... }

// 3. Когда интерфейс подходит лучше
// Если важно поведение, а не конкретный тип
type Writer interface { Write([]byte) error }
func Save(w Writer, data []byte) error { ... }
```

## Generics vs Interfaces

| Критерий | Generics | Interfaces |
|----------|----------|------------|
| Полиморфизм | Compile-time | Runtime |
| Производительность | Быстрее (нет boxing) | Медленнее (vtable) |
| Типобезопасность | Полная | Полная |
| Гибкость | Ограничена constraints | Любой тип с методами |
| Читаемость | Может быть сложнее | Обычно проще |

```go
// Interface — когда важно поведение
type Saver interface {
    Save() error
}

func SaveAll(items []Saver) error {
    for _, item := range items {
        if err := item.Save(); err != nil {
            return err
        }
    }
    return nil
}

// Generics — когда важен конкретный тип
func ProcessItems[T any](items []T, process func(T) T) []T {
    result := make([]T, len(items))
    for i, item := range items {
        result[i] = process(item)
    }
    return result
}
```

## Ограничения дженериков в Go

```go
// 1. Нельзя создать generic методы (только на типе)
// ❌ Ошибка
func (s Stack) Push[T any](item T) { }

// ✅ Параметр типа должен быть на типе
type Stack[T any] struct { }
func (s *Stack[T]) Push(item T) { }

// 2. Нет специализации
// Нельзя написать разную логику для разных типов
// (в отличие от C++ templates)

// 3. Нельзя использовать type parameters в type assertions
func Convert[T any](v interface{}) T {
    // ❌ Ошибка: cannot use type parameter T
    // return v.(T)

    // ✅ Обходной путь через reflect
    return reflect.ValueOf(v).Interface().(T)
}
```

## Производительность

```go
// Дженерики компилируются через "GC Shape stenciling"
// — компромисс между полной мономорфизацией и dynamic dispatch

// Для pointer types и interface{} — один общий код
// Для value types — отдельный код для каждого "shape"

// Benchmark: generic vs concrete vs interface
func BenchmarkGeneric(b *testing.B) {
    s := make([]int, 1000)
    for i := 0; i < b.N; i++ {
        _ = Sum(s)  // generic
    }
}

func BenchmarkConcrete(b *testing.B) {
    s := make([]int, 1000)
    for i := 0; i < b.N; i++ {
        _ = SumInt(s)  // конкретный тип
    }
}

// Результат: generic ~= concrete >> interface
// Generic почти не уступает конкретной реализации
```

## Вопросы с собеседований

### Вопрос 1: Что такое дженерики и зачем они нужны в Go?

<details>
<summary>Ответ</summary>

Дженерики (generics) — механизм параметризации типов, добавленный в Go 1.18. Позволяют писать функции и типы, работающие с разными типами данных.

**Зачем нужны:**
- Избавляют от дублирования кода (не нужно писать SumInt, SumFloat, SumString)
- Сохраняют типобезопасность (в отличие от interface{})
- Улучшают производительность (нет boxing/unboxing как с interface{})

</details>

### Вопрос 2: В чём разница между `any` и `comparable`?

<details>
<summary>Ответ</summary>

- `any` — алиас для `interface{}`, принимает любой тип
- `comparable` — только типы, которые можно сравнивать через `==` и `!=`

```go
func Print[T any](v T) { }           // любой тип
func Find[T comparable](s []T, v T)  // можно использовать ==
```

`comparable` включает: числа, строки, bool, указатели, каналы, интерфейсы, структуры и массивы из comparable типов. НЕ включает: slices, maps, functions.

</details>

### Вопрос 3: Что делает тильда (~) в constraints?

<details>
<summary>Ответ</summary>

Тильда означает "этот тип и все типы с таким underlying type":

```go
type MyInt int

type Integer interface {
    int  // только int
}

type IntegerWithTilde interface {
    ~int  // int и все типы на его основе (MyInt, etc.)
}

var x MyInt = 5
// С ~int: Double(x) работает
// Без ~: Double(x) — ошибка компиляции
```

</details>

### Вопрос 4: Когда лучше использовать дженерики, а когда интерфейсы?

<details>
<summary>Ответ</summary>

**Дженерики:**
- Контейнеры данных (Stack, Queue, Set)
- Утилитарные функции (Min, Max, Filter, Map)
- Когда важна производительность (нет overhead интерфейсов)
- Когда нужен конкретный тип на выходе

**Интерфейсы:**
- Когда важно поведение, а не тип (io.Reader, fmt.Stringer)
- Dependency injection
- Когда типы заранее неизвестны (плагины)
- Когда нужен runtime полиморфизм

</details>

### Вопрос 5: Почему нельзя создать generic метод?

<details>
<summary>Ответ</summary>

В Go параметры типа можно указать только на уровне типа, не на уровне метода:

```go
// ❌ Нельзя
func (s Stack) Push[T any](item T) { }

// ✅ Можно
type Stack[T any] struct { }
func (s *Stack[T]) Push(item T) { }
```

Это ограничение дизайна языка — упрощает компиляцию и реализацию. Для метода с другим типом нужно создать отдельную функцию.

</details>

## Связанные темы

- [[Go - Интерфейсы]]
- [[Go - Типы данных]]
- [[Go - Рефлексия]]
- [[Go - Escape Analysis]]
