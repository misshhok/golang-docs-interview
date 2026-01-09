# Go - Теги структур

Метаданные полей структур для управления сериализацией, валидацией и другими аспектами.

## Базовый синтаксис

```go
type User struct {
    ID    int    `json:"id" xml:"id" db:"user_id"`
    Name  string `json:"name" xml:"name" db:"name"`
    Email string `json:"email,omitempty" xml:"email,omitempty"`
}
```

**Формат:** `` `key1:"value1" key2:"value2"` ``

## JSON теги

### Основные опции

```go
type User struct {
    ID        int    `json:"id"`                    // Имя поля в JSON
    Name      string `json:"name"`
    Email     string `json:"email,omitempty"`       // Опустить если пусто
    Password  string `json:"-"`                     // Игнорировать
    Age       int    `json:"age,string"`            // Как строка "25"
    IsActive  bool   `json:"is_active"`
    CreatedAt string `json:"created_at,omitempty"`
}
```

**Опции:**
- `json:"name"` - имя в JSON
- `json:"-"` - игнорировать поле
- `json:",omitempty"` - опустить zero value
- `json:",string"` - число как строка
- `json:"name,omitempty"` - комбинация

### Примеры

```go
user := User{
    ID:   1,
    Name: "Alice",
    // Email пустой - будет опущен (omitempty)
    Password: "secret", // Не попадет в JSON (-)
}

data, _ := json.Marshal(user)
// {"id":1,"name":"Alice","is_active":false}
```

## XML теги

```go
type Book struct {
    XMLName xml.Name `xml:"book"`
    Title   string   `xml:"title"`
    Author  string   `xml:"author,attr"` // Как атрибут
    Year    int      `xml:"year"`
    Pages   int      `xml:"pages,omitempty"`
}

// <book author="John Doe">
//   <title>Go Programming</title>
//   <year>2024</year>
// </book>
```

## Database теги

### database/sql

```go
type User struct {
    ID        int       `db:"id"`
    Name      string    `db:"name"`
    Email     string    `db:"email"`
    CreatedAt time.Time `db:"created_at"`
}

// С sqlx
var users []User
db.Select(&users, "SELECT * FROM users")
```

### GORM

```go
type User struct {
    ID        uint      `gorm:"primaryKey"`
    Name      string    `gorm:"size:100;not null"`
    Email     string    `gorm:"uniqueIndex;not null"`
    Age       int       `gorm:"default:18"`
    CreatedAt time.Time `gorm:"autoCreateTime"`
    UpdatedAt time.Time `gorm:"autoUpdateTime"`
}
```

## Валидация

### validator/v10

```go
import "github.com/go-playground/validator/v10"

type User struct {
    Name  string `validate:"required,min=3,max=50"`
    Email string `validate:"required,email"`
    Age   int    `validate:"gte=18,lte=100"`
    Role  string `validate:"oneof=admin user guest"`
}

validate := validator.New()
err := validate.Struct(user)
```

**Популярные валидаторы:**
- `required` - обязательное
- `email` - валидный email
- `min=N`, `max=N` - диапазон
- `gte=N`, `lte=N` - >= и <=
- `oneof=val1 val2` - одно из значений
- `url` - валидный URL
- `uuid` - валидный UUID

## YAML теги

```go
import "gopkg.in/yaml.v3"

type Config struct {
    Host     string `yaml:"host"`
    Port     int    `yaml:"port"`
    Database struct {
        Host string `yaml:"host"`
        Name string `yaml:"name"`
    } `yaml:"database"`
}
```

## Кастомные теги

### Чтение тегов

```go
import "reflect"

type User struct {
    Name string `myTag:"username" required:"true"`
}

func readTags() {
    t := reflect.TypeOf(User{})
    field, _ := t.FieldByName("Name")

    // Получить тег
    myTag := field.Tag.Get("myTag")       // "username"
    required := field.Tag.Get("required")  // "true"

    // Lookup - с проверкой существования
    value, ok := field.Tag.Lookup("myTag")
    if ok {
        fmt.Println(value)
    }
}
```

### Обработка тегов

```go
func processStruct(v interface{}) {
    t := reflect.TypeOf(v)
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        tag := field.Tag.Get("json")

        if tag == "-" {
            fmt.Printf("%s ignored\n", field.Name)
        } else {
            fmt.Printf("%s -> %s\n", field.Name, tag)
        }
    }
}
```

## Комбинирование тегов

```go
type User struct {
    ID    int    `json:"id" db:"user_id" validate:"required"`
    Email string `json:"email,omitempty" db:"email" validate:"required,email"`
    Name  string `json:"name" db:"name" validate:"required,min=3"`
}

// Используется для:
// - JSON сериализации (json)
// - Маппинга БД (db)
// - Валидации (validate)
```

## Генерация кода по тегам

### go generate с tags

```go
//go:generate stringer -type=Status

type Status int

const (
    Active Status = iota
    Inactive
    Pending
)
```

## Практические примеры

### REST API модель

```go
type CreateUserRequest struct {
    Name     string `json:"name" validate:"required,min=3,max=50"`
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
}

type UserResponse struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`
    // Password не включаем в ответ!
}
```

### Database модель с GORM

```go
type Product struct {
    ID          uint      `gorm:"primaryKey" json:"id"`
    Name        string    `gorm:"size:200;not null" json:"name" validate:"required"`
    Description string    `gorm:"type:text" json:"description"`
    Price       float64   `gorm:"not null" json:"price" validate:"required,gt=0"`
    Stock       int       `gorm:"default:0" json:"stock" validate:"gte=0"`
    CategoryID  uint      `gorm:"not null" json:"category_id"`
    CreatedAt   time.Time `gorm:"autoCreateTime" json:"created_at"`
    UpdatedAt   time.Time `gorm:"autoUpdateTime" json:"updated_at"`
}
```

### Конфигурация

```go
type Config struct {
    Server struct {
        Host string `yaml:"host" env:"SERVER_HOST" default:"localhost"`
        Port int    `yaml:"port" env:"SERVER_PORT" default:"8080"`
    } `yaml:"server"`

    Database struct {
        Host     string `yaml:"host" env:"DB_HOST" required:"true"`
        Port     int    `yaml:"port" env:"DB_PORT" default:"5432"`
        User     string `yaml:"user" env:"DB_USER" required:"true"`
        Password string `yaml:"password" env:"DB_PASSWORD" required:"true"`
        DBName   string `yaml:"dbname" env:"DB_NAME" required:"true"`
    } `yaml:"database"`
}
```

## Best Practices

1. ✅ Используйте понятные имена в тегах
2. ✅ Группируйте связанные теги логически
3. ✅ Используйте `omitempty` для опциональных полей
4. ✅ Используйте `-` для исключения полей (пароли, etc.)
5. ✅ Документируйте кастомные теги
6. ✅ Валидируйте входные данные с помощью тегов
7. ❌ Не дублируйте логику в разных местах
8. ❌ Не забывайте экспортировать поля (заглавная буква)

## Ошибки

### 1. Поле не экспортировано

```go
type User struct {
    name string `json:"name"` // ❌ не работает - строчная буква!
}

type User struct {
    Name string `json:"name"` // ✅
}
```

### 2. Неправильный синтаксис

```go
// ❌ Нет пробела после запятой
`json:"name,omitempty"`

// ✅ С пробелом после ключа
`json:"name,omitempty" db:"name"`
```

### 3. Забыли обработать ошибку валидации

```go
// ❌ Игнорируем ошибку
validate.Struct(user)

// ✅ Обрабатываем
if err := validate.Struct(user); err != nil {
    // Обработка ошибок валидации
}
```

## Связанные темы

- [[Go - Структуры (struct)]]
- [[Go - Пакет encoding-json]]
- [[Go - Рефлексия]]
- [[REST API - Основы]]
