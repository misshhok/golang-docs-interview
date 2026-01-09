# Go - Пакет encoding/json

Пакет для работы с JSON - сериализация и десериализация.

## Marshal - сериализация в JSON

```go
import "encoding/json"

type Person struct {
    Name string
    Age  int
    City string
}

p := Person{
    Name: "Alice",
    Age:  25,
    City: "Moscow",
}

// Marshal
data, err := json.Marshal(p)
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(data))
// {"Name":"Alice","Age":25,"City":"Moscow"}

// MarshalIndent - с отступами
data, err = json.MarshalIndent(p, "", "  ")
// {
//   "Name": "Alice",
//   "Age": 25,
//   "City": "Moscow"
// }
```

## Unmarshal - десериализация из JSON

```go
jsonData := []byte(`{"Name":"Bob","Age":30,"City":"SPB"}`)

var p Person
err := json.Unmarshal(jsonData, &p)
if err != nil {
    log.Fatal(err)
}

fmt.Println(p.Name) // Bob
fmt.Println(p.Age)  // 30
```

## Теги структур

```go
type User struct {
    ID        int    `json:"id"`
    Name      string `json:"name"`
    Email     string `json:"email,omitempty"`  // Опустить если пусто
    Password  string `json:"-"`                // Игнорировать
    Age       int    `json:"age,string"`       // Как строка
    IsActive  bool   `json:"is_active"`
    CreatedAt string `json:"created_at,omitempty"`
}

u := User{
    ID:   1,
    Name: "Alice",
    // Email пусто - будет опущено
    Password: "secret", // Не попадет в JSON
}

data, _ := json.Marshal(u)
// {"id":1,"name":"Alice","age":"0","is_active":false}
```

**Опции тегов:**
- `json:"name"` - имя поля в JSON
- `json:"-"` - игнорировать поле
- `json:",omitempty"` - опустить если zero value
- `json:",string"` - число как строка

## Encoder/Decoder - потоковая обработка

### Encoder (запись в io.Writer)

```go
file, _ := os.Create("data.json")
defer file.Close()

encoder := json.NewEncoder(file)
encoder.SetIndent("", "  ")

err := encoder.Encode(person)
```

### Decoder (чтение из io.Reader)

```go
file, _ := os.Open("data.json")
defer file.Close()

decoder := json.NewDecoder(file)

var person Person
err := decoder.Decode(&person)
```

### Множественные JSON объекты

```go
// Чтение нескольких объектов из потока
decoder := json.NewDecoder(file)
for decoder.More() {
    var p Person
    if err := decoder.Decode(&p); err != nil {
        log.Fatal(err)
    }
    fmt.Println(p)
}
```

## RawMessage - отложенная десериализация

```go
type Response struct {
    Type string          `json:"type"`
    Data json.RawMessage `json:"data"` // Сырой JSON
}

jsonData := `{
    "type": "user",
    "data": {"name": "Alice", "age": 25}
}`

var resp Response
json.Unmarshal([]byte(jsonData), &resp)

// Десериализовать data позже
if resp.Type == "user" {
    var user User
    json.Unmarshal(resp.Data, &user)
}
```

## map и interface{}

### Десериализация в map

```go
jsonData := `{"name":"Alice","age":25,"active":true}`

var result map[string]interface{}
json.Unmarshal([]byte(jsonData), &result)

fmt.Println(result["name"])   // Alice
fmt.Println(result["age"])    // 25 (float64!)
fmt.Println(result["active"]) // true
```

**⚠️ Числа всегда десериализуются в float64!**

```go
age := result["age"].(float64) // Type assertion нужна
ageInt := int(age)
```

### Сериализация map

```go
data := map[string]interface{}{
    "name":   "Alice",
    "age":    25,
    "active": true,
}

jsonData, _ := json.Marshal(data)
// {"active":true,"age":25,"name":"Alice"}
```

## Кастомная сериализация

### Marshaler интерфейс

```go
type Marshaler interface {
    MarshalJSON() ([]byte, error)
}

// Пример
type Date struct {
    Year  int
    Month int
    Day   int
}

func (d Date) MarshalJSON() ([]byte, error) {
    s := fmt.Sprintf("\"%d-%02d-%02d\"", d.Year, d.Month, d.Day)
    return []byte(s), nil
}

date := Date{Year: 2023, Month: 12, Day: 31}
data, _ := json.Marshal(date)
// "2023-12-31"
```

### Unmarshaler интерфейс

```go
type Unmarshaler interface {
    UnmarshalJSON([]byte) error
}

func (d *Date) UnmarshalJSON(data []byte) error {
    s := string(data)
    s = strings.Trim(s, "\"")

    _, err := fmt.Sscanf(s, "%d-%d-%d", &d.Year, &d.Month, &d.Day)
    return err
}

var date Date
json.Unmarshal([]byte(`"2023-12-31"`), &date)
// date = {2023, 12, 31}
```

## Валидация JSON

```go
jsonData := []byte(`{"name":"Alice","age":25}`)

// Проверка валидности
if json.Valid(jsonData) {
    fmt.Println("Valid JSON")
}

// Компактная форма (убрать пробелы)
var buf bytes.Buffer
json.Compact(&buf, jsonData)

// Отформатировать с отступами
var formatted bytes.Buffer
json.Indent(&formatted, jsonData, "", "  ")
```

## Слайсы и массивы

```go
// Сериализация
numbers := []int{1, 2, 3, 4, 5}
data, _ := json.Marshal(numbers)
// [1,2,3,4,5]

// Десериализация
jsonData := []byte(`[1,2,3,4,5]`)
var nums []int
json.Unmarshal(jsonData, &nums)
```

## Null значения

```go
type User struct {
    Name  string  `json:"name"`
    Email *string `json:"email"` // Указатель для null
}

// email = null
u := User{Name: "Alice", Email: nil}
data, _ := json.Marshal(u)
// {"name":"Alice","email":null}

// Десериализация null
jsonData := []byte(`{"name":"Bob","email":null}`)
var user User
json.Unmarshal(jsonData, &user)
// user.Email == nil
```

## Типичные ошибки

### 1. Незаэкспортированные поля

```go
// ❌ Не сериализуется - строчная буква
type User struct {
    name string // Не экспортируется!
    age  int
}

// ✅ Экспортированные поля
type User struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}
```

### 2. Забыть передать указатель в Unmarshal

```go
// ❌ Не работает
var user User
json.Unmarshal(data, user) // Нужен указатель!

// ✅ Правильно
json.Unmarshal(data, &user)
```

### 3. Числа как float64 в interface{}

```go
var result map[string]interface{}
json.Unmarshal(data, &result)

// ❌ Ошибка - age это float64, не int
age := result["age"].(int)

// ✅ Правильно
age := int(result["age"].(float64))
```

## HTTP примеры

### Отправка JSON

```go
user := User{Name: "Alice", Age: 25}
data, _ := json.Marshal(user)

resp, err := http.Post(
    "https://api.example.com/users",
    "application/json",
    bytes.NewBuffer(data),
)
```

### Получение JSON

```go
resp, _ := http.Get("https://api.example.com/user/1")
defer resp.Body.Close()

var user User
json.NewDecoder(resp.Body).Decode(&user)
```

### HTTP handler

```go
func createUserHandler(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Обработка user...

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

## Best Practices

1. ✅ Используйте теги для контроля сериализации
2. ✅ Используйте `omitempty` для опциональных полей
3. ✅ Используйте Encoder/Decoder для потоков
4. ✅ Проверяйте ошибки Unmarshal
5. ✅ Используйте указатели для nullable полей
6. ✅ Валидируйте входящий JSON
7. ❌ Не игнорируйте ошибки Marshal/Unmarshal
8. ❌ Не используйте interface{} без необходимости

## Связанные темы

- [[Go - Структуры (struct)]]
- [[Go - Теги структур]]
- [[Go - Пакет net-http]]
- [[REST API - Основы]]
