# Go - Структуры (struct)

Структура - композитный тип данных, группирующий несколько полей под одним именем.

## Объявление

### Базовый синтаксис

```go
type Person struct {
    Name string
    Age  int
}

// Использование
var p Person
p.Name = "John"
p.Age = 30
```

### Анонимные структуры

```go
// Без объявления типа
person := struct {
    Name string
    Age  int
}{
    Name: "Alice",
    Age:  25,
}

// Часто используется в тестах
testCases := []struct {
    input    int
    expected int
}{
    {input: 1, expected: 2},
    {input: 2, expected: 4},
}
```

## Инициализация

### Способы создания

```go
type Person struct {
    Name string
    Age  int
    City string
}

// 1. Zero value
var p1 Person // {Name: "", Age: 0, City: ""}

// 2. С указанием полей (рекомендуется)
p2 := Person{
    Name: "Alice",
    Age:  25,
    City: "Moscow",
}

// 3. Позиционная инициализация (не рекомендуется!)
p3 := Person{"Bob", 30, "SPB"}

// 4. Частичная инициализация
p4 := Person{Name: "Charlie"} // Age: 0, City: ""

// 5. С указателем
p5 := &Person{
    Name: "Dave",
    Age:  35,
}
```

**Рекомендуется всегда указывать имена полей:**

```go
// ✅ Хорошо - явно и читабельно
person := Person{
    Name: "Alice",
    Age:  25,
}

// ❌ Плохо - легко ошибиться при изменении структуры
person := Person{"Alice", 25}
```

## Доступ к полям

```go
type Rectangle struct {
    Width  float64
    Height float64
}

rect := Rectangle{Width: 10, Height: 5}

// Чтение
w := rect.Width

// Запись
rect.Height = 7

// Через указатель (автоматическое разыменование)
p := &rect
p.Width = 20 // Эквивалентно (*p).Width = 20
```

## Встраивание (Embedding)

### Простое встраивание

```go
type Address struct {
    City    string
    Country string
}

type Person struct {
    Name    string
    Age     int
    Address // Встраивание
}

// Использование
p := Person{
    Name: "Alice",
    Age:  25,
    Address: Address{
        City:    "Moscow",
        Country: "Russia",
    },
}

// Поля Address продвигаются в Person
fmt.Println(p.City)    // "Moscow" - прямой доступ
fmt.Println(p.Address.City) // Тоже работает
```

### Конфликт имен

```go
type Base struct {
    Name string
}

type Derived struct {
    Base
    Name string // Конфликт!
}

d := Derived{
    Base: Base{Name: "Base Name"},
    Name: "Derived Name",
}

fmt.Println(d.Name)      // "Derived Name" - приоритет у Derived
fmt.Println(d.Base.Name) // "Base Name"
```

### Множественное встраивание

```go
type Person struct {
    Name string
}

type Employee struct {
    EmployeeID int
}

type Manager struct {
    Person   // Встраивание 1
    Employee // Встраивание 2
    Department string
}

m := Manager{
    Person:   Person{Name: "Alice"},
    Employee: Employee{EmployeeID: 123},
    Department: "IT",
}

// Доступ к полям
fmt.Println(m.Name)       // "Alice"
fmt.Println(m.EmployeeID) // 123
```

## Методы

Подробнее: [[Go - Функции и методы]]

```go
type Rectangle struct {
    Width  float64
    Height float64
}

// Value receiver
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Pointer receiver
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

// Использование
rect := Rectangle{Width: 10, Height: 5}
area := rect.Area() // 50

rect.Scale(2)
fmt.Println(rect.Width) // 20
```

## Сравнение структур

### Comparable структуры

```go
type Point struct {
    X, Y int
}

p1 := Point{X: 1, Y: 2}
p2 := Point{X: 1, Y: 2}
p3 := Point{X: 3, Y: 4}

fmt.Println(p1 == p2) // true
fmt.Println(p1 == p3) // false
```

**Структура comparable если все её поля comparable:**
- ✅ int, string, bool, float, array
- ❌ slice, map, function

### Non-comparable структуры

```go
type Data struct {
    Values []int // slice - not comparable
}

d1 := Data{Values: []int{1, 2, 3}}
d2 := Data{Values: []int{1, 2, 3}}

// d1 == d2 // ❌ Ошибка компиляции!

// Ручное сравнение
func (d Data) Equals(other Data) bool {
    if len(d.Values) != len(other.Values) {
        return false
    }
    for i := range d.Values {
        if d.Values[i] != other.Values[i] {
            return false
        }
    }
    return true
}
```

## Теги структур

Метаданные для полей, используются для сериализации и других целей:

```go
type User struct {
    ID        int    `json:"id"`
    Name      string `json:"name"`
    Email     string `json:"email,omitempty"`
    Password  string `json:"-"` // Игнорировать
    CreatedAt time.Time `json:"created_at"`
}

// JSON сериализация
user := User{
    ID:    1,
    Name:  "Alice",
    Email: "alice@example.com",
}

data, _ := json.Marshal(user)
// {"id":1,"name":"Alice","email":"alice@example.com","created_at":"..."}
```

**Популярные теги:**
- `json` - encoding/json
- `xml` - encoding/xml
- `yaml` - gopkg.in/yaml.v3
- `db` - database/sql
- `validate` - валидация

Подробнее: [[Go - Теги структур]]

## Копирование структур

### Value type

```go
// Структуры - value type
type Point struct {
    X, Y int
}

p1 := Point{X: 1, Y: 2}
p2 := p1 // Копия!

p2.X = 99
fmt.Println(p1.X) // 1 (не изменилось)
fmt.Println(p2.X) // 99
```

### Глубокое vs поверхностное копирование

```go
type Person struct {
    Name    string
    Friends []string // Slice - reference type!
}

p1 := Person{
    Name:    "Alice",
    Friends: []string{"Bob", "Charlie"},
}

// Поверхностное копирование
p2 := p1

// Слайс общий!
p2.Friends[0] = "Dave"
fmt.Println(p1.Friends[0]) // "Dave" (изменилось!)

// Глубокое копирование
p3 := Person{
    Name:    p1.Name,
    Friends: make([]string, len(p1.Friends)),
}
copy(p3.Friends, p1.Friends)

p3.Friends[0] = "Eve"
fmt.Println(p1.Friends[0]) // "Dave" (не изменилось)
```

## Пустая структура

`struct{}` - не занимает памяти:

```go
import "unsafe"

fmt.Println(unsafe.Sizeof(struct{}{})) // 0

// Использование как сигнал в канале
done := make(chan struct{})
go func() {
    // работа...
    done <- struct{}{} // Сигнал завершения
}()
<-done

// Set из map
set := make(map[string]struct{})
set["key"] = struct{}{}

if _, exists := set["key"]; exists {
    fmt.Println("Ключ есть")
}
```

## Конструкторы

Go не имеет конструкторов, но есть идиома:

```go
type Server struct {
    host string
    port int
}

// Конструктор (идиома: NewТип)
func NewServer(host string, port int) *Server {
    return &Server{
        host: host,
        port: port,
    }
}

// С валидацией
func NewServer(host string, port int) (*Server, error) {
    if port < 1 || port > 65535 {
        return nil, errors.New("invalid port")
    }
    return &Server{
        host: host,
        port: port,
    }, nil
}
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

### Builder pattern

```go
type QueryBuilder struct {
    table  string
    where  []string
    limit  int
    offset int
}

func NewQueryBuilder(table string) *QueryBuilder {
    return &QueryBuilder{table: table}
}

func (qb *QueryBuilder) Where(condition string) *QueryBuilder {
    qb.where = append(qb.where, condition)
    return qb
}

func (qb *QueryBuilder) Limit(n int) *QueryBuilder {
    qb.limit = n
    return qb
}

func (qb *QueryBuilder) Build() string {
    query := "SELECT * FROM " + qb.table
    if len(qb.where) > 0 {
        query += " WHERE " + strings.Join(qb.where, " AND ")
    }
    if qb.limit > 0 {
        query += fmt.Sprintf(" LIMIT %d", qb.limit)
    }
    return query
}

// Использование
query := NewQueryBuilder("users").
    Where("age > 18").
    Where("active = true").
    Limit(10).
    Build()
```

## Выравнивание полей (alignment)

Порядок полей влияет на размер структуры:

```go
import "unsafe"

// ❌ Неоптимально - 24 bytes
type Bad struct {
    a bool   // 1 byte + 7 padding
    b int64  // 8 bytes
    c bool   // 1 byte + 7 padding
}

fmt.Println(unsafe.Sizeof(Bad{})) // 24

// ✅ Оптимально - 16 bytes
type Good struct {
    b int64  // 8 bytes
    a bool   // 1 byte
    c bool   // 1 byte + 6 padding
}

fmt.Println(unsafe.Sizeof(Good{})) // 16
```

**Правило:** Располагайте поля от большего к меньшему для экономии памяти.

## Приватные и публичные поля

```go
package mypackage

type User struct {
    ID       int    // Публичное (заглавная буква)
    Name     string // Публичное
    password string // Приватное (строчная буква)
}

// Геттер для приватного поля
func (u *User) Password() string {
    return u.password
}

// Сеттер с валидацией
func (u *User) SetPassword(password string) error {
    if len(password) < 8 {
        return errors.New("password too short")
    }
    u.password = password
    return nil
}
```

## Частые ошибки

### 1. Забыть использовать pointer receiver

```go
// ❌ Не изменяет оригинал
type Counter struct {
    count int
}

func (c Counter) Increment() {
    c.count++ // Изменяет копию!
}

c := Counter{}
c.Increment()
fmt.Println(c.count) // 0

// ✅ С pointer receiver
func (c *Counter) Increment() {
    c.count++
}
```

### 2. Неправильное копирование с reference types

```go
// ❌ Shallow copy
type Data struct {
    Items []int
}

d1 := Data{Items: []int{1, 2, 3}}
d2 := d1
d2.Items[0] = 99
fmt.Println(d1.Items[0]) // 99 (изменилось!)
```

### 3. Игнорирование alignment

```go
// ❌ Неэффективно
type Bad struct {
    a bool   // 1 + 7 padding
    b int64  // 8
    c bool   // 1 + 7 padding
    d int64  // 8
} // Итого: 32 bytes

// ✅ Эффективно
type Good struct {
    b int64  // 8
    d int64  // 8
    a bool   // 1
    c bool   // 1 + 6 padding
} // Итого: 24 bytes
```

## Best Practices

1. ✅ Всегда указывайте имена полей при инициализации
2. ✅ Используйте конструкторы (NewТип) для сложной инициализации
3. ✅ Pointer receiver для методов, изменяющих state
4. ✅ Группируйте поля по размеру для оптимизации памяти
5. ✅ Используйте встраивание для композиции
6. ✅ Экспортируйте только необходимые поля
7. ❌ Не используйте позиционную инициализацию

## Связанные темы

- [[Go - Типы данных]]
- [[Go - Функции и методы]]
- [[Go - Интерфейсы]]
- [[Go - Указатели и ссылки]]
- [[Go - Теги структур]]
- [[Паттерны проектирования в Go]]
