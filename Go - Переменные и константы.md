# Go - Переменные и константы

## Переменные

### Объявление с var

```go
// С типом и значением
var name string = "John"
var age int = 30

// Вывод типа
var name = "John"  // string
var age = 30       // int

// Без инициализации (zero value)
var count int      // 0
var message string // ""
var isActive bool  // false
```

### Короткое объявление :=

Доступно **только внутри функций**:

```go
func main() {
    name := "John"
    age := 30
    isActive := true

    // Множественное объявление
    x, y := 10, 20
    name, age := "Jane", 25
}

// ❌ Вне функций не работает
// name := "John" // Ошибка компиляции

// ✅ Вне функций используйте var
var name = "John"
```

### Множественное объявление

```go
// Одного типа
var x, y int = 1, 2
var a, b, c string = "a", "b", "c"

// Разных типов
var (
    name   = "John"
    age    = 30
    height = 180.5
    active = true
)

// С :=
x, y := 1, 2
name, age, height := "John", 30, 180.5
```

### Переобъявление с :=

```go
// Можно переобъявить ОДНУ переменную
x := 10
x, y := 20, 30 // x переприсваивается, y объявляется

// ❌ Нельзя если все переменные уже объявлены
x := 10
x := 20 // Ошибка! "no new variables"

// ✅ Используйте обычное присваивание
x := 10
x = 20
```

### Zero values

Неинициализированные переменные получают zero value:

```go
var i int       // 0
var f float64   // 0.0
var b bool      // false
var s string    // ""
var p *int      // nil
var sl []int    // nil
var m map[string]int // nil
```

### Пустой идентификатор _

Игнорирование ненужных значений:

```go
// Игнорирование возвращаемого значения
_, err := os.Open("file.txt")

// Игнорирование индекса в range
for _, value := range slice {
    fmt.Println(value)
}

// Импорт для side effects
import _ "net/http/pprof"

// Проверка реализации интерфейса
var _ io.Reader = (*MyType)(nil)
```

## Константы

### Объявление

```go
const Pi = 3.14159
const AppName = "MyApp"

// Группа констант
const (
    StatusOK       = 200
    StatusNotFound = 404
    StatusError    = 500
)

// Типизированные константы
const TypedInt int = 42
const TypedString string = "hello"
```

### Нетипизированные константы

```go
const UntypedInt = 42
const UntypedFloat = 3.14

// Более гибкие - неявное преобразование
var i int = UntypedInt
var i32 int32 = UntypedInt
var i64 int64 = UntypedInt
var f float64 = UntypedInt

// ✅ Все работает!

// Типизированные менее гибкие
const TypedInt int = 42
var i int = TypedInt      // ✅
var i64 int64 = TypedInt  // ❌ Ошибка!
var i64 int64 = int64(TypedInt) // ✅ Нужно явное преобразование
```

### iota - автоинкремент

```go
const (
    Sunday = iota    // 0
    Monday           // 1
    Tuesday          // 2
    Wednesday        // 3
    Thursday         // 4
    Friday           // 5
    Saturday         // 6
)

// Начать с другого числа
const (
    One = iota + 1   // 1
    Two              // 2
    Three            // 3
)

// Пропустить значения
const (
    _      = iota    // 0 (игнорируется)
    KB     = 1 << (10 * iota) // 1 << 10 = 1024
    MB                         // 1 << 20 = 1048576
    GB                         // 1 << 30 = 1073741824
)

// Множественные константы на строку
const (
    a, b = iota, iota + 10  // 0, 10
    c, d                    // 1, 11
    e, f                    // 2, 12
)

// Битовые флаги
const (
    FlagNone  = 0
    FlagRead  = 1 << iota  // 1
    FlagWrite              // 2
    FlagExec               // 4
)

// Использование
perms := FlagRead | FlagWrite // 3 (битовое ИЛИ)
```

### Выражения в константах

```go
const (
    SecondsPerMinute = 60
    MinutesPerHour   = 60
    SecondsPerHour   = SecondsPerMinute * MinutesPerHour
    HoursPerDay      = 24
    SecondsPerDay    = SecondsPerHour * HoursPerDay
)

const Greeting = "Hello, " + "World!"

// ❌ Нельзя вызывать функции
// const Length = len("hello") // Ошибка!

// ✅ Только константные выражения
const Pi = 3.14159
const Tau = 2 * Pi
```

## Область видимости

### Уровень пакета

```go
package main

// Экспортируемая (public)
var PublicVar = "visible outside package"
const PublicConst = 42

// Не экспортируемая (private)
var privateVar = "only in this package"
const privateConst = 42

func main() {
    // Обе доступны
}
```

### Уровень функции

```go
func example() {
    // Локальная переменная
    x := 10

    if true {
        // y видна только в этом блоке
        y := 20
        fmt.Println(x, y) // 10 20
    }

    // fmt.Println(y) // ❌ Ошибка! y не видна
    fmt.Println(x) // ✅ 10
}
```

### Shadowing (затенение)

```go
var x = "package"

func main() {
    fmt.Println(x) // "package"

    x := "local" // Новая переменная!
    fmt.Println(x) // "local"

    {
        x := "block"
        fmt.Println(x) // "block"
    }

    fmt.Println(x) // "local"
}
```

**Осторожно с shadowing!** Может привести к ошибкам:

```go
func readFile() error {
    file, err := os.Open("file.txt")
    if err != nil {
        return err
    }
    defer file.Close()

    // ❌ Затеняем err!
    data, err := io.ReadAll(file)
    if err != nil {
        return err
    }

    // file теперь из внутренней области видимости!
    // Может быть не тем, что ожидаем
}

// ✅ Правильно
func readFile() error {
    file, err := os.Open("file.txt")
    if err != nil {
        return err
    }
    defer file.Close()

    data, err2 := io.ReadAll(file)
    if err2 != nil {
        return err2
    }
    // ...
}
```

## Именование

### Правила

1. Начинается с буквы или `_`
2. Состоит из букв, цифр, `_`
3. Регистрозависимые
4. Заглавная буква = экспортируется

```go
// ✅ Валидные имена
var name string
var userName string
var _temp int
var value123 int

// ❌ Невалидные
// var 123value int  // Начинается с цифры
// var user-name int // Дефис не разрешен
// var user name int // Пробел не разрешен
```

### Стиль именования

**camelCase** для приватных:
```go
var firstName string
var userAge int
func getUserName() string
```

**PascalCase** для публичных:
```go
var FirstName string
var UserAge int
func GetUserName() string
```

**Акронимы заглавными:**
```go
// ✅ Правильно
var userID int
var httpServer *http.Server
var urlPath string
type HTTPClient struct{}

// ❌ Неправильно
var userId int
var httpserver *http.Server
var urlpath string
type HttpClient struct{}
```

**Короткие имена для коротких областей видимости:**
```go
// ✅ В цикле
for i := 0; i < 10; i++ {
    // ...
}

// ✅ В короткой функции
func Add(a, b int) int {
    return a + b
}

// ❌ В длинной функции
func processUserData() {
    for i := 0; i < len(data); i++ {
        // 100 строк кода...
        // Что такое i?
    }
}

// ✅ Описательное имя в длинной функции
func processUserData() {
    for userIndex := 0; userIndex < len(users); userIndex++ {
        // 100 строк кода...
        // Понятно что это индекс пользователя
    }
}
```

## Неиспользуемые переменные

```go
func main() {
    x := 10 // ❌ Ошибка: "x declared and not used"

    // ✅ Используйте
    x := 10
    fmt.Println(x)

    // ✅ Или игнорируйте явно
    _ = 10
}

// Неиспользуемые параметры - OK
func handler(w http.ResponseWriter, r *http.Request) {
    // r не используется - не ошибка
    fmt.Fprint(w, "Hello")
}

// Но лучше явно показать намерение
func handler(w http.ResponseWriter, _ *http.Request) {
    fmt.Fprint(w, "Hello")
}
```

## Best Practices

### 1. Предпочитайте := внутри функций

```go
// ✅ Короче и чище
name := "John"
age := 30

// ❌ Избыточно
var name string = "John"
var age int = 30
```

### 2. Группируйте связанные переменные

```go
// ✅ Группа
var (
    host = "localhost"
    port = 8080
    timeout = 30 * time.Second
)

// ❌ Раздельно
var host = "localhost"
var port = 8080
var timeout = 30 * time.Second
```

### 3. Объявляйте переменные ближе к использованию

```go
// ❌ Далеко от использования
func process() {
    var result string
    // 50 строк кода...
    result = compute()
}

// ✅ Рядом с использованием
func process() {
    // 50 строк кода...
    result := compute()
}
```

### 4. Используйте константы для магических чисел

```go
// ❌ Магические числа
if age >= 18 && age <= 65 {
    // ...
}

// ✅ Константы
const (
    MinAdultAge = 18
    RetirementAge = 65
)

if age >= MinAdultAge && age <= RetirementAge {
    // ...
}
```

### 5. Избегайте shadowing важных переменных

```go
// ❌ Затеняет err
if file, err := os.Open("file.txt"); err == nil {
    data, err := io.ReadAll(file) // Новая err!
    if err != nil {
        // ...
    }
}

// ✅ Используйте другое имя
if file, err := os.Open("file.txt"); err == nil {
    data, readErr := io.ReadAll(file)
    if readErr != nil {
        // ...
    }
}
```

## Связанные темы

- [[Go - Типы данных]]
- [[Go - Основы синтаксиса]]
- [[Go - Функции и методы]]
