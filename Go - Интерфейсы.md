# Go - Интерфейсы

Интерфейсы в Go - это набор сигнатур методов. Тип реализует интерфейс **неявно** (implicit), просто определяя все методы интерфейса.

## Определение интерфейса

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}

type Reader interface {
    Read(p []byte) (n int, err error)
}
```

## Реализация интерфейса

```go
type File struct {
    name string
}

// File реализует Writer неявно
func (f *File) Write(p []byte) (int, error) {
    fmt.Printf("Writing %s to %s\n", string(p), f.name)
    return len(p), nil
}

// Использование
var w Writer = &File{name: "test.txt"}
w.Write([]byte("Hello"))
```

**Нет ключевого слова `implements`!** Реализация неявная.

## Пустой интерфейс

`interface{}` или `any` (Go 1.18+) может хранить значение любого типа:

```go
func printAnything(v interface{}) {
    fmt.Println(v)
}

printAnything(42)
printAnything("hello")
printAnything([]int{1, 2, 3})

// Go 1.18+
func printAnything(v any) {
    fmt.Println(v)
}
```

## Type Assertion

Проверка конкретного типа:

```go
var i interface{} = "hello"

// Type assertion
s := i.(string)
fmt.Println(s) // hello

// Type assertion с проверкой
s, ok := i.(string)
if ok {
    fmt.Println(s)
} else {
    fmt.Println("not a string")
}

// Без проверки - panic!
n := i.(int) // panic: interface conversion
```

## Type Switch

Проверка нескольких типов:

```go
func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Int: %d\n", v)
    case string:
        fmt.Printf("String: %s\n", v)
    case bool:
        fmt.Printf("Bool: %t\n", v)
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}

describe(42)      // Int: 42
describe("hello") // String: hello
describe(true)    // Bool: true
```

## Композиция интерфейсов

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Композиция
type ReadWriter interface {
    Reader
    Writer
}

// Эквивалентно:
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```

## Стандартные интерфейсы

### io.Reader и io.Writer

```go
// Из пакета io
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Примеры реализаций:
// - os.File
// - bytes.Buffer
// - strings.Reader
// - http.Request.Body
```

### fmt.Stringer

```go
type Stringer interface {
    String() string
}

type Person struct {
    Name string
    Age  int
}

func (p Person) String() string {
    return fmt.Sprintf("%s (%d years)", p.Name, p.Age)
}

p := Person{"John", 30}
fmt.Println(p) // John (30 years)
```

### error

```go
type error interface {
    Error() string
}

// Кастомная ошибка
type MyError struct {
    Msg  string
    Code int
}

func (e MyError) Error() string {
    return fmt.Sprintf("error %d: %s", e.Code, e.Msg)
}

func doSomething() error {
    return MyError{Msg: "something went wrong", Code: 500}
}
```

Подробнее: [[Go - Обработка ошибок]]

## Принятие интерфейсов, возврат структур

> "Accept interfaces, return structs"
> — Go proverb

```go
// ✅ Хорошо - принимает интерфейс
func SaveUser(w io.Writer, user User) error {
    data, _ := json.Marshal(user)
    _, err := w.Write(data)
    return err
}

// Работает с любым Writer:
SaveUser(os.Stdout, user)
SaveUser(&bytes.Buffer{}, user)
SaveUser(file, user)

// ❌ Плохо - принимает конкретный тип
func SaveUser(file *os.File, user User) error {
    // Работает только с файлами!
}
```

## Интерфейсы с одним методом

Предпочитайте маленькие интерфейсы (особенно с одним методом):

```go
// ✅ Маленькие интерфейсы
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// Комбинируйте при необходимости
type ReadCloser interface {
    Reader
    Closer
}

// ❌ Большие интерфейсы (Go - не Java!)
type DataManager interface {
    Create()
    Read()
    Update()
    Delete()
    List()
    Find()
    // ...
}
```

## Nil интерфейсы

```go
var w io.Writer // nil интерфейс

// nil интерфейс имеет nil тип и nil значение
if w == nil {
    fmt.Println("w is nil") // Печатается
}

// Интерфейс со значением nil НЕ является nil интерфейсом!
var p *File = nil
w = p // Теперь w != nil! (тип *File, значение nil)

if w == nil {
    fmt.Println("w is nil") // НЕ печатается!
}
```

**Ловушка:**
```go
func returnsError() error {
    var err *MyError = nil
    return err // error интерфейс != nil!
}

if err := returnsError(); err != nil {
    fmt.Println("error occurred") // Печатается!
}

// ✅ Правильно
func returnsError() error {
    return nil
}
```

## Проверка реализации интерфейса

```go
type Speaker interface {
    Speak() string
}

type Dog struct{}

func (d Dog) Speak() string {
    return "Woof"
}

// Compile-time проверка что Dog реализует Speaker
var _ Speaker = (*Dog)(nil)

// Если Dog не реализует Speaker - compile error
```

## Паттерны с интерфейсами

### Dependency Injection

```go
type UserRepository interface {
    GetUser(id int) (*User, error)
}

type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) ProcessUser(id int) error {
    user, err := s.repo.GetUser(id)
    // ...
}

// Легко тестировать с mock'ом
type MockRepo struct{}
func (m MockRepo) GetUser(id int) (*User, error) {
    return &User{ID: id}, nil
}

service := NewUserService(MockRepo{})
```

### Strategy Pattern

```go
type Sorter interface {
    Sort([]int)
}

type BubbleSort struct{}
func (b BubbleSort) Sort(data []int) {
    // Bubble sort
}

type QuickSort struct{}
func (q QuickSort) Sort(data []int) {
    // Quick sort
}

type DataProcessor struct {
    sorter Sorter
}

func (d *DataProcessor) Process(data []int) {
    d.sorter.Sort(data) // Используем переданную стратегию
}

// Использование
processor := DataProcessor{sorter: QuickSort{}}
processor.Process(data)
```

## Связанные темы

- [[Go - Структуры (struct)]]
- [[Go - Функции и методы]]
- [[Go - Обработка ошибок]]
- [[SOLID принципы]]
- [[Go - Моки и стабы]]
- [[Паттерны проектирования в Go]]
