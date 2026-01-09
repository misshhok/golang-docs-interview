# Go - Основы синтаксиса

Краткий обзор базового синтаксиса языка Go.

## Hello World

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, World!")
}
```

**Запуск:**
```bash
go run main.go
# или
go build main.go && ./main
```

## Пакеты

Каждый Go файл начинается с объявления пакета:

```go
package main // Исполняемая программа (должна иметь func main)
package mylib // Библиотека
```

## Импорты

```go
// Одиночный импорт
import "fmt"
import "time"

// Групповой импорт (предпочтительный)
import (
    "fmt"
    "time"
    "github.com/user/package"
)

// Алиас
import f "fmt"

// Импорт без использования (для side effects)
import _ "github.com/lib/pq"
```

## Переменные

```go
// Явное объявление с типом
var name string = "John"
var age int = 30

// Вывод типа
var name = "John"  // string
var age = 30       // int

// Короткое объявление (только внутри функций)
name := "John"
age := 30

// Множественное объявление
var x, y int = 1, 2
a, b := 10, "hello"

// Блок объявлений
var (
    name = "John"
    age  = 30
    city = "Moscow"
)

// Zero values
var i int     // 0
var f float64 // 0.0
var b bool    // false
var s string  // ""
```

Подробнее: [[Go - Переменные и константы]]

## Константы

```go
const Pi = 3.14
const (
    StatusOK = 200
    StatusNotFound = 404
)

// iota - автоинкремент
const (
    Monday = iota // 0
    Tuesday       // 1
    Wednesday     // 2
)
```

## Функции

```go
// Базовая функция
func add(a int, b int) int {
    return a + b
}

// Сокращенный синтаксис для одинаковых типов
func add(a, b int) int {
    return a + b
}

// Множественный return
func swap(a, b string) (string, string) {
    return b, a
}

// Именованный return
func split(sum int) (x, y int) {
    x = sum / 2
    y = sum - x
    return // Возвращает x и y
}

// Variadic функции
func sum(nums ...int) int {
    total := 0
    for _, num := range nums {
        total += num
    }
    return total
}

sum(1, 2, 3, 4) // 10
```

Подробнее: [[Go - Функции и методы]]

## Управляющие конструкции

### if-else

```go
if x > 0 {
    fmt.Println("positive")
} else if x < 0 {
    fmt.Println("negative")
} else {
    fmt.Println("zero")
}

// if с инициализацией
if err := doSomething(); err != nil {
    return err
}
```

### switch

```go
switch day {
case "Monday":
    fmt.Println("Start of week")
case "Friday":
    fmt.Println("End of week")
default:
    fmt.Println("Middle of week")
}

// switch без выражения (как if-else chain)
switch {
case x < 0:
    fmt.Println("negative")
case x == 0:
    fmt.Println("zero")
default:
    fmt.Println("positive")
}

// Type switch
switch v := x.(type) {
case int:
    fmt.Println("int:", v)
case string:
    fmt.Println("string:", v)
}
```

### for

```go
// Классический for
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// While-подобный
i := 0
for i < 10 {
    fmt.Println(i)
    i++
}

// Бесконечный цикл
for {
    // infinite loop
    break
}

// range (для слайсов, map, каналов)
nums := []int{1, 2, 3}
for i, num := range nums {
    fmt.Printf("%d: %d\n", i, num)
}

// Игнорирование индекса
for _, num := range nums {
    fmt.Println(num)
}
```

## Указатели

```go
var p *int        // Указатель на int
x := 42
p = &x            // & - адрес
fmt.Println(*p)   // * - разыменование, выведет 42

*p = 21           // Изменение через указатель
fmt.Println(x)    // 21
```

Подробнее: [[Go - Указатели и ссылки]]

## Структуры

```go
// Объявление
type Person struct {
    Name string
    Age  int
}

// Создание
p1 := Person{Name: "John", Age: 30}
p2 := Person{"Jane", 25} // По порядку полей

// Доступ к полям
fmt.Println(p1.Name)
p1.Age = 31
```

Подробнее: [[Go - Структуры (struct)]]

## Методы

```go
type Person struct {
    Name string
}

// Метод с value receiver
func (p Person) SayHello() {
    fmt.Printf("Hello, I'm %s\n", p.Name)
}

// Метод с pointer receiver
func (p *Person) SetName(name string) {
    p.Name = name
}

// Использование
p := Person{Name: "John"}
p.SayHello()
p.SetName("Jane")
```

## Интерфейсы

```go
type Speaker interface {
    Speak() string
}

type Dog struct{}

func (d Dog) Speak() string {
    return "Woof!"
}

// Dog реализует Speaker неявно
var s Speaker = Dog{}
fmt.Println(s.Speak())
```

Подробнее: [[Go - Интерфейсы]]

## Обработка ошибок

```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}

result, err := divide(10, 2)
if err != nil {
    log.Fatal(err)
}
fmt.Println(result)
```

Подробнее: [[Go - Обработка ошибок]]

## defer

```go
func readFile(filename string) {
    f, err := os.Open(filename)
    if err != nil {
        return
    }
    defer f.Close() // Выполнится в конце функции

    // Работа с файлом
}
```

Подробнее: [[Go - Defer, Panic, Recover]]

## Горутины и каналы

```go
// Горутина
go func() {
    fmt.Println("Running concurrently")
}()

// Канал
ch := make(chan int)

go func() {
    ch <- 42 // Отправка
}()

value := <-ch // Получение
```

Подробнее: [[Go - Горутины (goroutines)]], [[Go - Каналы (channels)]]

## Комментарии

```go
// Однострочный комментарий

/*
Многострочный
комментарий
*/

// Документирующий комментарий для пакета (в начале файла)
// Package mylib provides utilities for...
package mylib

// Документирующий комментарий для функции
// Add returns the sum of a and b.
func Add(a, b int) int {
    return a + b
}
```

## Именования

**Экспортируемые** (public) имена начинаются с **заглавной буквы**:
```go
// Экспортируется
func PublicFunction() {}
type PublicStruct struct {
    PublicField int
}

// Не экспортируется (private)
func privateFunction() {}
type privateStruct struct {
    privateField int
}
```

**Соглашения:**
- `camelCase` для приватных имен
- `PascalCase` для публичных имен
- Акронимы заглавными: `HTTP`, `URL`, `ID`

```go
// ✅ Правильно
type HTTPServer struct {}
var userID int

// ❌ Неправильно
type HttpServer struct {}
var userId int
```

## Пустой идентификатор _

```go
// Игнорирование значений
_, err := os.Open("file.txt")

// Импорт для side effects
import _ "net/http/pprof"

// Проверка реализации интерфейса
var _ Speaker = (*Dog)(nil)
```

## Связанные темы

- [[Go - Переменные и константы]]
- [[Go - Типы данных]]
- [[Go - Функции и методы]]
- [[Go - Структуры (struct)]]
- [[Go - Интерфейсы]]
- [[Go - Обработка ошибок]]
- [[Go - Горутины (goroutines)]]
- [[Go - Каналы (channels)]]
