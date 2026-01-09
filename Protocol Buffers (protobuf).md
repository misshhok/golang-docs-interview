# Protocol Buffers (protobuf)

Protocol Buffers (protobuf) — язык описания структуры данных и формат бинарной сериализации от Google. Используется в gRPC, но может применяться независимо.

## Зачем protobuf вместо JSON?

### JSON

```json
{
    "id": 123,
    "name": "Alice",
    "email": "alice@example.com",
    "age": 30,
    "tags": ["developer", "golang"]
}
```

**Размер:** 107 байт (текст)
**Скорость:** Медленная сериализация/десериализация

### Protobuf

```
Бинарное представление:
0x08 0x7B 0x12 0x05 Alice 0x1A ...
```

**Размер:** ~40 байт (бинарный формат)
**Скорость:** В 5-10 раз быстрее JSON

**Преимущества:**
- ✅ Меньше размер (~ 30-50% от JSON)
- ✅ Быстрее (5-10x)
- ✅ Строгая типизация
- ✅ Генерация кода для разных языков
- ✅ Обратная совместимость

**Недостатки:**
- ❌ Не human-readable (бинарный)
- ❌ Нужна компиляция .proto → код
- ❌ Сложнее debugging

## Основы синтаксиса

### Базовый .proto файл

```protobuf
syntax = "proto3"; // Используем proto3

package user;

option go_package = "github.com/myapp/pb/user";

// Сообщение (структура данных)
message User {
    int32 id = 1;           // Поле с номером 1
    string name = 2;        // Поле с номером 2
    string email = 3;
    int32 age = 4;
    repeated string tags = 5; // Массив
}
```

**Важно:** Номера полей (1, 2, 3...) - это идентификаторы в бинарном формате, НЕ значения!

### Скалярные типы

```protobuf
message Example {
    // Целые числа
    int32 i32 = 1;      // 32-bit signed
    int64 i64 = 2;      // 64-bit signed
    uint32 u32 = 3;     // 32-bit unsigned
    uint64 u64 = 4;     // 64-bit unsigned
    sint32 si32 = 5;    // Signed (эффективнее для отрицательных)
    sint64 si64 = 6;

    // Числа с плавающей точкой
    float f = 7;        // 32-bit float
    double d = 8;       // 64-bit double

    // Логический тип
    bool active = 9;

    // Строка (UTF-8)
    string name = 10;

    // Бинарные данные
    bytes data = 11;
}
```

**Маппинг на Go типы:**

| Protobuf | Go |
|----------|---------|
| int32, int64 | int32, int64 |
| uint32, uint64 | uint32, uint64 |
| sint32, sint64 | int32, int64 |
| bool | bool |
| string | string |
| bytes | []byte |
| float | float32 |
| double | float64 |

### Массивы (repeated)

```protobuf
message User {
    repeated string tags = 1;        // []string
    repeated int32 scores = 2;       // []int32
    repeated Address addresses = 3;   // []Address
}
```

**Go код:**

```go
type User struct {
    Tags      []string   `protobuf:"bytes,1,rep,name=tags" json:"tags,omitempty"`
    Scores    []int32    `protobuf:"varint,2,rep,name=scores" json:"scores,omitempty"`
    Addresses []*Address `protobuf:"bytes,3,rep,name=addresses" json:"addresses,omitempty"`
}
```

### Вложенные сообщения

```protobuf
message User {
    int32 id = 1;
    string name = 2;
    Address address = 3;      // Вложенное сообщение
    repeated Order orders = 4; // Массив вложенных сообщений
}

message Address {
    string city = 1;
    string street = 2;
    string zip_code = 3;
}

message Order {
    int32 id = 1;
    float amount = 2;
}
```

**Go код:**

```go
user := &User{
    Id:   123,
    Name: "Alice",
    Address: &Address{
        City:    "Moscow",
        Street:  "Tverskaya 1",
        ZipCode: "101000",
    },
    Orders: []*Order{
        {Id: 1, Amount: 100.50},
        {Id: 2, Amount: 200.75},
    },
}
```

### Maps

```protobuf
message User {
    int32 id = 1;
    string name = 2;
    map<string, string> metadata = 3;  // map[string]string
    map<int32, Address> addresses = 4; // map[int32]*Address
}
```

**Go код:**

```go
user := &User{
    Id:   123,
    Name: "Alice",
    Metadata: map[string]string{
        "role":   "admin",
        "status": "active",
    },
    Addresses: map[int32]*Address{
        1: {City: "Moscow"},
        2: {City: "SPb"},
    },
}
```

### Enums

```protobuf
enum Status {
    STATUS_UNSPECIFIED = 0; // Всегда должен быть UNSPECIFIED = 0
    STATUS_ACTIVE = 1;
    STATUS_INACTIVE = 2;
    STATUS_SUSPENDED = 3;
}

enum OrderStatus {
    ORDER_STATUS_UNSPECIFIED = 0;
    ORDER_STATUS_PENDING = 1;
    ORDER_STATUS_PAID = 2;
    ORDER_STATUS_SHIPPED = 3;
    ORDER_STATUS_DELIVERED = 4;
}

message User {
    int32 id = 1;
    string name = 2;
    Status status = 3;
}
```

**Go код:**

```go
user := &User{
    Id:     123,
    Name:   "Alice",
    Status: Status_STATUS_ACTIVE, // Enum значение
}

// Проверка
if user.Status == Status_STATUS_ACTIVE {
    log.Println("User is active")
}
```

### Oneof (union type)

```protobuf
message SearchRequest {
    string query = 1;

    // Только одно из полей может быть установлено
    oneof filter {
        string category = 2;
        int32 price_range = 3;
        string brand = 4;
    }
}
```

**Go код:**

```go
// Вариант 1: по категории
req1 := &SearchRequest{
    Query: "laptop",
    Filter: &SearchRequest_Category{
        Category: "electronics",
    },
}

// Вариант 2: по цене
req2 := &SearchRequest{
    Query: "laptop",
    Filter: &SearchRequest_PriceRange{
        PriceRange: 1000,
    },
}

// Проверка
switch filter := req1.Filter.(type) {
case *SearchRequest_Category:
    log.Println("Category:", filter.Category)
case *SearchRequest_PriceRange:
    log.Println("Price range:", filter.PriceRange)
case *SearchRequest_Brand:
    log.Println("Brand:", filter.Brand)
}
```

### Default Values

```protobuf
message User {
    int32 id = 1;          // default: 0
    string name = 2;       // default: ""
    bool active = 3;       // default: false
    repeated string tags = 4; // default: []
    Status status = 5;     // default: STATUS_UNSPECIFIED (0)
}
```

**В proto3 нельзя задать custom defaults!**

### Optional Fields (proto3)

```protobuf
syntax = "proto3";

message User {
    int32 id = 1;
    string name = 2;

    // Optional - можно отличить "не установлено" от "значение по умолчанию"
    optional int32 age = 3;
    optional string email = 4;
}
```

**Go код:**

```go
user := &User{
    Id:   123,
    Name: "Alice",
    Age:  proto.Int32(30), // Указатель на int32
}

// Проверка установлено ли поле
if user.Age != nil {
    log.Println("Age:", *user.Age)
} else {
    log.Println("Age not set")
}
```

## Генерация Go кода

### 1. Установка protoc

```bash
# macOS
brew install protobuf

# Linux
apt install -y protobuf-compiler

# Проверка
protoc --version
```

### 2. Установка Go плагинов

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

# PATH должен включать $GOPATH/bin
export PATH="$PATH:$(go env GOPATH)/bin"
```

### 3. Генерация кода

```bash
# Простая генерация
protoc --go_out=. user.proto

# С указанием пути
protoc --go_out=. --go_opt=paths=source_relative user.proto

# Создаст: user.pb.go
```

### 4. Структура проекта

```
myapp/
├── proto/
│   ├── user.proto
│   ├── order.proto
│   └── product.proto
├── pb/                    # Сгенерированный код
│   ├── user/
│   │   └── user.pb.go
│   ├── order/
│   │   └── order.pb.go
│   └── product/
│       └── product.pb.go
├── cmd/
│   └── server/
│       └── main.go
└── go.mod
```

**Команда генерации:**

```bash
protoc --go_out=pb --go_opt=paths=source_relative proto/*.proto
```

## Сериализация и десериализация

### Marshal (сериализация)

```go
import "google.golang.org/protobuf/proto"

user := &pb.User{
    Id:    123,
    Name:  "Alice",
    Email: "alice@example.com",
    Age:   30,
    Tags:  []string{"developer", "golang"},
}

// Сериализация в байты
data, err := proto.Marshal(user)
if err != nil {
    log.Fatal(err)
}

log.Printf("Serialized size: %d bytes", len(data))

// Сохранение в файл
os.WriteFile("user.pb", data, 0644)

// Отправка по сети
conn.Write(data)
```

### Unmarshal (десериализация)

```go
// Чтение из файла
data, err := os.ReadFile("user.pb")
if err != nil {
    log.Fatal(err)
}

// Десериализация
var user pb.User
err = proto.Unmarshal(data, &user)
if err != nil {
    log.Fatal(err)
}

log.Printf("User: id=%d, name=%s, email=%s", user.Id, user.Name, user.Email)
```

## JSON конвертация

```go
import (
    "google.golang.org/protobuf/encoding/protojson"
)

user := &pb.User{
    Id:    123,
    Name:  "Alice",
    Email: "alice@example.com",
}

// Protobuf → JSON
jsonData, err := protojson.Marshal(user)
if err != nil {
    log.Fatal(err)
}
log.Println(string(jsonData))
// {"id":123,"name":"Alice","email":"alice@example.com"}

// JSON → Protobuf
jsonStr := `{"id":456,"name":"Bob","email":"bob@example.com"}`
var user2 pb.User
err = protojson.Unmarshal([]byte(jsonStr), &user2)
if err != nil {
    log.Fatal(err)
}
log.Printf("User: %+v", user2)
```

## Обратная совместимость

### Правила обновления .proto файлов

**✅ Можно (backward compatible):**

```protobuf
// Версия 1
message User {
    int32 id = 1;
    string name = 2;
}

// Версия 2 - ДОБАВИЛИ новое поле
message User {
    int32 id = 1;
    string name = 2;
    string email = 3;        // ✅ Новое поле (старые клиенты просто игнорируют)
    optional int32 age = 4;  // ✅ Optional поле
}
```

**Старый клиент (v1):** Просто игнорирует `email` и `age`
**Новый клиент (v2):** Может читать старые данные (email="" по умолчанию)

**❌ Нельзя (breaking changes):**

```protobuf
// Версия 1
message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
}

// Версия 2
message User {
    int32 id = 1;
    // ❌ УДАЛИЛИ поле name (номер 2)
    string email = 3;
    int32 age = 2;    // ❌ ПЕРЕИСПОЛЬЗОВАЛИ номер 2 (нарушает совместимость!)
}
```

**Правильно:**

```protobuf
// Версия 2
message User {
    int32 id = 1;
    reserved 2;           // Резервируем номер 2 (name удалено)
    reserved "name";      // Резервируем имя
    string email = 3;
    int32 age = 4;        // Используем новый номер
}
```

### Reserved Fields

```protobuf
message User {
    reserved 2, 15, 9 to 11;    // Зарезервированные номера
    reserved "foo", "bar";       // Зарезервированные имена

    int32 id = 1;
    string name = 3;
}
```

## Импорты

```protobuf
// common.proto
syntax = "proto3";
package common;

message Address {
    string city = 1;
    string street = 2;
}

// user.proto
syntax = "proto3";
package user;

import "common.proto";  // Импортируем

message User {
    int32 id = 1;
    string name = 2;
    common.Address address = 3;  // Используем импортированный тип
}
```

**Генерация:**

```bash
protoc --go_out=. --go_opt=paths=source_relative common.proto user.proto
```

## Well-Known Types

Google предоставляет готовые типы:

```protobuf
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/wrappers.proto";

message Event {
    string id = 1;
    string name = 2;

    // Timestamp
    google.protobuf.Timestamp created_at = 3;

    // Duration
    google.protobuf.Duration timeout = 4;

    // Wrapper types (nullable)
    google.protobuf.Int32Value age = 5;
    google.protobuf.StringValue email = 6;
}

message EmptyRequest {
    // Пустой запрос
}

// Или используйте google.protobuf.Empty
service UserService {
    rpc Ping(google.protobuf.Empty) returns (google.protobuf.Empty);
}
```

**Go код:**

```go
import (
    "google.golang.org/protobuf/types/known/timestamppb"
    "google.golang.org/protobuf/types/known/durationpb"
    "google.golang.org/protobuf/types/known/wrapperspb"
)

event := &pb.Event{
    Id:        "evt123",
    Name:      "User Created",
    CreatedAt: timestamppb.Now(),
    Timeout:   durationpb.New(5 * time.Minute),
    Age:       wrapperspb.Int32(30),    // Nullable int32
    Email:     wrapperspb.String("alice@example.com"),
}

// Конвертация Timestamp в time.Time
t := event.CreatedAt.AsTime()

// Конвертация Duration в time.Duration
d := event.Timeout.AsDuration()

// Проверка nullable поля
if event.Age != nil {
    log.Println("Age:", event.Age.Value)
}
```

## Any (динамический тип)

```protobuf
import "google/protobuf/any.proto";

message Event {
    string id = 1;
    google.protobuf.Any payload = 2;  // Любой тип
}
```

**Go код:**

```go
import (
    "google.golang.org/protobuf/types/known/anypb"
)

// Упаковываем User в Any
user := &pb.User{Id: 123, Name: "Alice"}
anyPayload, _ := anypb.New(user)

event := &pb.Event{
    Id:      "evt123",
    Payload: anyPayload,
}

// Распаковываем Any
var extractedUser pb.User
if event.Payload.MessageIs(&extractedUser) {
    err := event.Payload.UnmarshalTo(&extractedUser)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("User: %+v", extractedUser)
}
```

## Валидация

```protobuf
// Установка: go install github.com/bufbuild/buf/cmd/buf@latest

// Используйте buf для валидации и линтинга

// buf.yaml
version: v1
lint:
  use:
    - DEFAULT
breaking:
  use:
    - FILE

// Команды
buf lint proto/
buf breaking --against '.git#branch=main' proto/
```

## Протоколы сериализации: сравнение

| Критерий | Protobuf | JSON | XML | MessagePack |
|----------|----------|------|-----|-------------|
| **Размер** | Маленький | Большой | Очень большой | Маленький |
| **Скорость** | ⚡ Быстро | Медленно | Медленно | Быстро |
| **Human-readable** | ❌ Нет | ✅ Да | ✅ Да | ❌ Нет |
| **Типизация** | ✅ Строгая | ❌ Нет | ❌ Нет | ❌ Нет |
| **Схема** | ✅ Да (.proto) | ❌ Нет | ✅ Да (XSD) | ❌ Нет |
| **Backward compat** | ✅ Да | ❌ Сложно | ❌ Сложно | ❌ Сложно |
| **Генерация кода** | ✅ Да | ❌ Нет | ❌ Нет | ❌ Нет |
| **Browser support** | ❌ Нет | ✅ Да | ✅ Да | ❌ Нет |

## Когда использовать Protobuf

**✅ Используйте Protobuf когда:**

1. **Производительность критична**
   - Высокая нагрузка
   - Микросервисы (internal API)

2. **Типизация важна**
   - Нужен строгий контракт
   - Compile-time проверки

3. **Обратная совместимость**
   - API evolves over time
   - Нужно поддерживать старые клиенты

4. **Межязыковая коммуникация**
   - Go, Python, Java, C++, etc
   - Автоматическая генерация кода

5. **gRPC**
   - Protobuf - нативный формат gRPC

**❌ НЕ используйте Protobuf когда:**

1. **Публичные REST API**
   - JSON понятнее для внешних разработчиков
   - Легче debugging

2. **Human-readable данные**
   - Конфигурационные файлы
   - Логи

3. **Простые проекты**
   - Overhead от protoc не оправдан
   - JSON достаточно

4. **Browser-only applications**
   - JSON нативно поддерживается

## Best Practices

1. ✅ **Всегда указывайте syntax = "proto3"**
2. ✅ **Используйте meaningful имена полей**
3. ✅ **Номера полей: 1-15 занимают 1 байт** (используйте для частых полей)
4. ✅ **Резервируйте удаленные поля** (reserved)
5. ✅ **Используйте enums вместо строк** для фиксированных значений
6. ✅ **Группируйте .proto файлы по доменам**
7. ✅ **Документируйте .proto файлы** (комментарии)
8. ✅ **Используйте buf для линтинга**
9. ❌ **Не меняйте номера полей**
10. ❌ **Не переиспользуйте удаленные номера**

## Связанные темы

- [[gRPC]]
- [[HTTP протокол]]
- [[REST API - Основы]]
- [[Go - Пакет encoding-json]]
- [[Микросервисная архитектура]]
