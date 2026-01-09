# Go - Функции и методы

## Функции

### Базовый синтаксис

```go
func functionName(param1 type1, param2 type2) returnType {
    // тело функции
    return value
}

// Пример
func add(a int, b int) int {
    return a + b
}

// Сокращенная запись для одинаковых типов
func add(a, b int) int {
    return a + b
}
```

### Множественный return

```go
func swap(a, b string) (string, string) {
    return b, a
}

x, y := swap("hello", "world")
// x = "world", y = "hello"

// Игнорирование значения
x, _ := swap("hello", "world")
```

### Именованные возвращаемые значения

```go
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = errors.New("division by zero")
        return // Возвращает result=0.0, err
    }

    result = a / b
    return // Возвращает result и err
}

// Или явно
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}
```

**Когда использовать:**
- ✅ Для документации что возвращается
- ✅ В defer для изменения возвращаемых значений
- ❌ В простых функциях (избыточно)

### Variadic функции

```go
// Принимает переменное количество аргументов
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

sum(1, 2, 3)          // 6
sum(1, 2, 3, 4, 5)    // 15

// Передача slice
numbers := []int{1, 2, 3}
sum(numbers...)       // Развертывание slice
```

**Variadic параметр должен быть последним:**
```go
// ✅ Правильно
func printf(format string, args ...interface{})

// ❌ Неправильно
// func invalid(args ...int, format string)
```

### Функции как значения

```go
// Функция - это тип
func add(a, b int) int {
    return a + b
}

// Присваивание функции
var operation func(int, int) int
operation = add
result := operation(2, 3) // 5

// Функция как параметр
func apply(a, b int, op func(int, int) int) int {
    return op(a, b)
}

result := apply(5, 3, add)
```

### Анонимные функции

```go
// Объявление и вызов
func() {
    fmt.Println("Hello")
}()

// Присваивание переменной
greeting := func(name string) {
    fmt.Printf("Hello, %s!\n", name)
}
greeting("John")

// Как параметр
result := apply(5, 3, func(a, b int) int {
    return a * b
})
```

### Замыкания (Closures)

```go
// Функция захватывает внешние переменные
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

c1 := counter()
fmt.Println(c1()) // 1
fmt.Println(c1()) // 2
fmt.Println(c1()) // 3

c2 := counter()
fmt.Println(c2()) // 1 (новый счетчик)
```

**Практический пример:**
```go
func makeMultiplier(factor int) func(int) int {
    return func(n int) int {
        return n * factor
    }
}

double := makeMultiplier(2)
triple := makeMultiplier(3)

fmt.Println(double(5)) // 10
fmt.Println(triple(5)) // 15
```

### defer

Откладывает выполнение до выхода из функции:

```go
func readFile(filename string) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close() // Выполнится в конце

    // Работа с файлом
    return process(file)
}
```

**Порядок выполнения:** LIFO (Last In, First Out)
```go
func example() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
    fmt.Println("Start")
}

// Output:
// Start
// 3
// 2
// 1
```

Подробнее: [[Go - Defer, Panic, Recover]]

## Методы

Функции с receiver - привязаны к типу.

### Value receiver

```go
type Rectangle struct {
    Width  float64
    Height float64
}

// Метод с value receiver
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

rect := Rectangle{Width: 10, Height: 5}
area := rect.Area() // 50
```

**Копия receiver:**
```go
func (r Rectangle) Scale(factor float64) {
    r.Width *= factor   // Изменяется копия!
    r.Height *= factor
}

rect := Rectangle{Width: 10, Height: 5}
rect.Scale(2)
fmt.Println(rect.Width) // 10 (не изменилось!)
```

### Pointer receiver

```go
// Метод с pointer receiver
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor   // Изменяется оригинал
    r.Height *= factor
}

rect := Rectangle{Width: 10, Height: 5}
rect.Scale(2)
fmt.Println(rect.Width) // 20 (изменилось!)

// Go автоматически берет адрес
rect.Scale(2) // Эквивалентно (&rect).Scale(2)
```

### Когда использовать pointer receiver

**Используйте pointer receiver когда:**
1. ✅ Метод изменяет receiver
2. ✅ Receiver - большая структура (избегаем копирования)
3. ✅ Для консистентности (если один метод с pointer, все должны быть)

```go
type Counter struct {
    count int
}

// ✅ Изменяет state - pointer receiver
func (c *Counter) Increment() {
    c.count++
}

// ✅ Не изменяет, но для консистентности - pointer
func (c *Counter) Value() int {
    return c.count
}
```

### Методы на не-структурных типах

```go
type MyInt int

func (m MyInt) Double() MyInt {
    return m * 2
}

var x MyInt = 5
fmt.Println(x.Double()) // 10

// ❌ Нельзя на встроенных типах
// func (i int) Double() int { } // Ошибка!

// ✅ Только на собственных типах
type MyString string
func (s MyString) ToUpper() MyString {
    return MyString(strings.ToUpper(string(s)))
}
```

### Встраивание и продвижение методов

```go
type Person struct {
    Name string
}

func (p Person) Greet() {
    fmt.Printf("Hello, I'm %s\n", p.Name)
}

type Employee struct {
    Person // Встраивание
    JobTitle string
}

// Методы Person продвигаются в Employee
emp := Employee{
    Person: Person{Name: "John"},
    JobTitle: "Developer",
}

emp.Greet() // Hello, I'm John
// Эквивалентно: emp.Person.Greet()
```

## Рекурсия

```go
func factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorial(n-1)
}

factorial(5) // 120

// Fibonacci
func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    return fibonacci(n-1) + fibonacci(n-2)
}
```

**Осторожно:** Глубокая рекурсия может вызвать stack overflow!

## init() функция

Выполняется автоматически при инициализации пакета:

```go
var config Config

func init() {
    // Инициализация при старте программы
    config = loadConfig()
}

func init() {
    // Можно иметь несколько init()
    setupDatabase()
}
```

**Порядок выполнения:**
1. Инициализация констант
2. Инициализация переменных
3. init() функции (в порядке объявления)
4. main() (если есть)

## main() функция

Точка входа в программу:

```go
package main

func main() {
    // Старт программы
    fmt.Println("Hello, World!")
}
```

**Правила:**
- Должна быть в пакете `main`
- Не принимает параметров
- Не возвращает значений
- Для аргументов командной строки: `os.Args`
- Для exit code: `os.Exit(code)`

## Higher-order функции

Функции, которые принимают или возвращают функции:

```go
// Принимает функцию
func filter(nums []int, predicate func(int) bool) []int {
    var result []int
    for _, n := range nums {
        if predicate(n) {
            result = append(result, n)
        }
    }
    return result
}

numbers := []int{1, 2, 3, 4, 5, 6}
evens := filter(numbers, func(n int) bool {
    return n%2 == 0
})
// [2, 4, 6]

// Возвращает функцию
func makeAdder(x int) func(int) int {
    return func(y int) int {
        return x + y
    }
}

add5 := makeAdder(5)
fmt.Println(add5(10)) // 15
```

## Паттерны

### Options pattern

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
}

type Option func(*Server)

func WithHost(host string) Option {
    return func(s *Server) {
        s.host = host
    }
}

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func NewServer(opts ...Option) *Server {
    s := &Server{
        host:    "localhost",
        port:    8080,
        timeout: 30 * time.Second,
    }

    for _, opt := range opts {
        opt(s)
    }

    return s
}

// Использование
server := NewServer(
    WithHost("0.0.0.0"),
    WithPort(9000),
)
```

### Callback pattern

```go
type Handler func(data Data) error

func Process(data Data, handler Handler) error {
    // Обработка
    result := transform(data)

    // Вызов callback
    return handler(result)
}

// Использование
Process(data, func(d Data) error {
    fmt.Println("Processed:", d)
    return nil
})
```

## Типичные ошибки

### 1. Забыть return после проверки ошибки

```go
// ❌ Продолжает выполнение после ошибки
func process() error {
    if err := validate(); err != nil {
        fmt.Println("Error:", err)
        // Забыли return!
    }
    // Выполнится даже при ошибке!
    return doWork()
}

// ✅ Правильно
func process() error {
    if err := validate(); err != nil {
        return err
    }
    return doWork()
}
```

### 2. Неправильное использование defer

```go
// ❌ defer в цикле
for _, file := range files {
    f, _ := os.Open(file)
    defer f.Close() // Закроется только в конце функции!
}

// ✅ Вызов в отдельной функции
for _, file := range files {
    processFile(file)
}

func processFile(filename string) error {
    f, _ := os.Open(filename)
    defer f.Close() // Закроется в конце processFile
    // ...
}
```

### 3. Захват переменной цикла в замыкании

```go
// ❌ Все горутины используют одну переменную
for _, v := range values {
    go func() {
        fmt.Println(v) // Все печатают последнее значение!
    }()
}

// ✅ Передаем как параметр
for _, v := range values {
    go func(val int) {
        fmt.Println(val)
    }(v)
}

// ✅ Копируем переменную
for _, v := range values {
    v := v // Копия для замыкания
    go func() {
        fmt.Println(v)
    }()
}
```

## Best Practices

### 1. Короткие функции

```go
// ✅ Одна ответственность, короткая
func validateEmail(email string) error {
    if !strings.Contains(email, "@") {
        return errors.New("invalid email")
    }
    return nil
}

// ❌ Слишком длинная, много ответственностей
func processUser(user User) error {
    // 100+ строк кода...
}
```

### 2. Именованные результаты только когда нужно

```go
// ✅ Для defer изменений
func Open(name string) (f *File, err error) {
    defer func() {
        if err != nil {
            log.Printf("failed to open %s: %v", name, err)
        }
    }()
    // ...
}

// ❌ Без необходимости
func Add(a, b int) (sum int) {
    sum = a + b
    return
}

// ✅ Проще
func Add(a, b int) int {
    return a + b
}
```

### 3. Ранний возврат

```go
// ✅ Early return - меньше вложенности
func process(input string) error {
    if input == "" {
        return errors.New("empty input")
    }

    if !isValid(input) {
        return errors.New("invalid input")
    }

    return doWork(input)
}

// ❌ Глубокая вложенность
func process(input string) error {
    if input != "" {
        if isValid(input) {
            return doWork(input)
        } else {
            return errors.New("invalid input")
        }
    } else {
        return errors.New("empty input")
    }
}
```

## Связанные темы

- [[Go - Интерфейсы]]
- [[Go - Defer, Panic, Recover]]
- [[Go - Обработка ошибок]]
- [[Go - Горутины (goroutines)]]
