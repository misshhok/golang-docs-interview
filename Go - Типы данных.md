# Go - Типы данных

Обзор встроенных типов данных в Go.

## Базовые типы

### Булевы значения

```go
var b bool = true
var f bool = false

// Zero value
var b bool // false
```

### Целые числа

**Знаковые:**
- `int8` - от -128 до 127
- `int16` - от -32,768 до 32,767
- `int32` (rune) - от -2^31 до 2^31-1
- `int64` - от -2^63 до 2^63-1
- `int` - 32 или 64 бита (зависит от архитектуры)

**Беззнаковые:**
- `uint8` (byte) - от 0 до 255
- `uint16` - от 0 до 65,535
- `uint32` - от 0 до 2^32-1
- `uint64` - от 0 до 2^64-1
- `uint` - 32 или 64 бита
- `uintptr` - для хранения указателей

```go
var i int = 42
var u uint8 = 255
var b byte = 'A' // byte = uint8
var r rune = 'П' // rune = int32 (Unicode code point)
```

**Zero value**: `0`

### Числа с плавающей точкой

```go
var f32 float32 = 3.14
var f64 float64 = 2.718281828

// Научная нотация
var x float64 = 6.02e23
```

**Zero value**: `0.0`

### Комплексные числа

```go
var c64 complex64 = 1 + 2i
var c128 complex128 = complex(3, 4) // 3 + 4i

// Операции
real(c128) // 3
imag(c128) // 4
```

### Строки

```go
var s string = "Hello, World!"

// Многострочные строки (raw string literal)
var multiline string = `Line 1
Line 2
Line 3`

// UTF-8 encoded
var ru string = "Привет"

// Zero value
var empty string // ""

// Длина в байтах (не символах!)
len("Hello") // 5
len("Привет") // 12 (не 6!)
```

Подробнее: [[Go - Строки и UTF-8]]

## Композитные типы

### Массивы

```go
var arr [5]int              // [0 0 0 0 0]
var arr2 = [3]int{1, 2, 3}  // [1 2 3]
var arr3 = [...]int{1, 2, 3} // Размер автоматически

// Фиксированный размер - часть типа!
var a [3]int
var b [4]int
// a и b - разные типы!
```

Подробнее: [[Go - Массивы и слайсы]]

### Слайсы

```go
var slice []int              // nil slice
slice = []int{1, 2, 3}       // [1 2 3]
slice = make([]int, 5)       // [0 0 0 0 0]
slice = make([]int, 5, 10)   // len=5, cap=10

// Динамический размер
slice = append(slice, 6) // Добавление элемента
```

**Zero value**: `nil`

Подробнее: [[Go - Массивы и слайсы]]

### Карты (maps)

```go
var m map[string]int           // nil map
m = make(map[string]int)       // Пустая map
m = map[string]int{
    "one": 1,
    "two": 2,
}

// Операции
m["three"] = 3      // Добавление
val := m["one"]     // Чтение
val, ok := m["one"] // Чтение с проверкой
delete(m, "one")    // Удаление
```

**Zero value**: `nil`

Подробнее: [[Go - Карты (maps)]]

### Структуры

```go
type Person struct {
    Name string
    Age  int
}

var p Person              // Zero value: {"", 0}
p = Person{"John", 30}
p = Person{Name: "Jane", Age: 25}
```

Подробнее: [[Go - Структуры (struct)]]

### Указатели

```go
var p *int           // nil pointer
x := 42
p = &x               // Адрес x
fmt.Println(*p)      // 42 (разыменование)
```

**Zero value**: `nil`

Подробнее: [[Go - Указатели и ссылки]]

## Интерфейсы

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

var r Reader // nil interface
```

**Zero value**: `nil` (nil type, nil value)

Подробнее: [[Go - Интерфейсы]]

## Функции

```go
// Функция - это тип!
func add(a, b int) int {
    return a + b
}

var f func(int, int) int
f = add
f(2, 3) // 5

// Анонимные функции
f = func(a, b int) int {
    return a * b
}
```

Подробнее: [[Go - Функции и методы]]

## Каналы

```go
var ch chan int              // nil channel
ch = make(chan int)          // Небуферизованный
ch = make(chan int, 10)      // Буферизованный

// Направленные каналы
var recv <-chan int          // Только чтение
var send chan<- int          // Только запись
```

**Zero value**: `nil`

Подробнее: [[Go - Каналы (channels)]]

## Псевдонимы типов

```go
type MyInt int
type UserID int

var a MyInt = 10
var b int = 10

// a и b - разные типы!
// a = b // Ошибка компиляции
a = MyInt(b) // ✅ Явное преобразование
```

## Zero Values

Все переменные в Go имеют zero value:

| Тип | Zero Value |
|-----|------------|
| bool | false |
| int, uint, float | 0 |
| string | "" |
| pointer | nil |
| slice | nil |
| map | nil |
| channel | nil |
| interface | nil |
| function | nil |

```go
var b bool    // false
var i int     // 0
var s string  // ""
var p *int    // nil
var sl []int  // nil
```

## Преобразование типов

```go
var i int = 42
var f float64 = float64(i)
var u uint = uint(i)

// Нет неявного преобразования!
// var x int = 10
// var y int32 = x // ❌ Ошибка!
var y int32 = int32(x) // ✅ Явное преобразование
```

## Константы

```go
const Pi = 3.14
const (
    StatusOK = 200
    StatusNotFound = 404
)

// Типизированные константы
const TypedInt int = 42

// Нетипизированные (более гибкие)
const UntypedInt = 42

var i int = UntypedInt
var i64 int64 = UntypedInt // Работает!
```

Подробнее: [[Go - Переменные и константы]]

## Размеры типов

```go
import "unsafe"

unsafe.Sizeof(int8(0))    // 1 byte
unsafe.Sizeof(int16(0))   // 2 bytes
unsafe.Sizeof(int32(0))   // 4 bytes
unsafe.Sizeof(int64(0))   // 8 bytes
unsafe.Sizeof(float32(0)) // 4 bytes
unsafe.Sizeof(float64(0)) // 8 bytes

// Зависит от архитектуры (32-bit vs 64-bit)
unsafe.Sizeof(int(0))     // 4 или 8 bytes
unsafe.Sizeof("hello")    // 16 bytes (string header)
unsafe.Sizeof([]int{})    // 24 bytes (slice header)
```

## Type Assertions

```go
var i interface{} = "hello"

s := i.(string)        // Успешно
fmt.Println(s)         // "hello"

// С проверкой
s, ok := i.(string)
if ok {
    fmt.Println(s)
}

// Без проверки - panic при неправильном типе
// n := i.(int) // panic!
```

## Выбор типа данных

**Используйте:**
- `int` для целых чисел (по умолчанию)
- `float64` для чисел с плавающей точкой (по умолчанию)
- `string` для текста
- `bool` для логических значений
- `[]T` для списков (не массивы!)
- `map[K]V` для словарей

**Избегайте:**
- Конкретные размеры (`int32`, `int64`) без необходимости
- `uint` без явной причины
- Массивы фиксированного размера (используйте слайсы)

## Связанные темы

- [[Go - Переменные и константы]]
- [[Go - Массивы и слайсы]]
- [[Go - Карты (maps)]]
- [[Go - Структуры (struct)]]
- [[Go - Интерфейсы]]
- [[Go - Указатели и ссылки]]
- [[Go - Строки и UTF-8]]
