# Singleton, Factory, Builder в Go

Три классических порождающих паттерна, адаптированных под особенности Go. Эти паттерны решают задачу создания объектов различными способами в зависимости от требований.

## Singleton (Одиночка)

Singleton гарантирует, что у класса есть только один экземпляр, и предоставляет глобальную точку доступа к нему.

### Реализация через sync.Once

```go
package database

import (
    "database/sql"
    "sync"
)

// Приватная структура
type database struct {
    connection *sql.DB
}

var (
    instance *database
    once     sync.Once
)

// GetInstance - потокобезопасное получение экземпляра
func GetInstance() *database {
    once.Do(func() {
        // Инициализация происходит только один раз
        db, err := sql.Open("postgres", "connection_string")
        if err != nil {
            panic(err)
        }
        instance = &database{connection: db}
    })
    return instance
}

// Методы singleton
func (d *database) Query(query string) {
    // Выполнение запроса
    d.connection.Query(query)
}
```

### Использование

```go
func main() {
    db1 := database.GetInstance()
    db2 := database.GetInstance()

    // db1 и db2 указывают на один и тот же объект
    fmt.Println(db1 == db2) // true
}
```

### Eager Initialization (через init)

```go
package config

type Configuration struct {
    ServerPort string
    DBHost     string
}

var instance *Configuration

// init вызывается автоматически при загрузке пакета
func init() {
    instance = &Configuration{
        ServerPort: "8080",
        DBHost:     "localhost",
    }
}

func GetConfig() *Configuration {
    return instance
}
```

### Когда использовать Singleton

✅ **Используйте когда:**
- Подключение к базе данных
- Конфигурация приложения
- Logger
- Кэш

❌ **Не используйте когда:**
- Нужно несколько экземпляров
- Требуется тестирование с моками
- Объект не имеет состояния

### Альтернатива: Dependency Injection

```go
// Вместо Singleton используйте DI
type Service struct {
    db     *Database
    config *Config
    logger *Logger
}

func NewService(db *Database, config *Config, logger *Logger) *Service {
    return &Service{
        db:     db,
        config: config,
        logger: logger,
    }
}

// В main создаем зависимости и передаем их
func main() {
    db := NewDatabase()
    config := NewConfig()
    logger := NewLogger()

    service := NewService(db, config, logger)
}
```

## Factory (Фабрика)

Factory предоставляет интерфейс для создания объектов, позволяя подклассам решать, какой класс инстанцировать.

### Simple Factory

```go
// Интерфейс продукта
type Transport interface {
    Deliver() string
}

// Конкретные продукты
type Truck struct{}

func (t *Truck) Deliver() string {
    return "Доставка грузовиком по земле"
}

type Ship struct{}

func (s *Ship) Deliver() string {
    return "Доставка кораблем по морю"
}

type Plane struct{}

func (p *Plane) Deliver() string {
    return "Доставка самолетом по воздуху"
}

// Factory функция
func NewTransport(transportType string) (Transport, error) {
    switch transportType {
    case "truck":
        return &Truck{}, nil
    case "ship":
        return &Ship{}, nil
    case "plane":
        return &Plane{}, nil
    default:
        return nil, fmt.Errorf("unknown transport type: %s", transportType)
    }
}

// Использование
func main() {
    transport, err := NewTransport("truck")
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(transport.Deliver())
}
```

### Factory Method Pattern

```go
// Абстрактная фабрика
type LoggerFactory interface {
    CreateLogger() Logger
}

// Интерфейс продукта
type Logger interface {
    Log(message string)
}

// Конкретные продукты
type FileLogger struct {
    FilePath string
}

func (f *FileLogger) Log(message string) {
    fmt.Printf("Logging to file %s: %s\n", f.FilePath, message)
}

type ConsoleLogger struct{}

func (c *ConsoleLogger) Log(message string) {
    fmt.Printf("Console: %s\n", message)
}

// Конкретные фабрики
type FileLoggerFactory struct {
    FilePath string
}

func (f *FileLoggerFactory) CreateLogger() Logger {
    return &FileLogger{FilePath: f.FilePath}
}

type ConsoleLoggerFactory struct{}

func (c *ConsoleLoggerFactory) CreateLogger() Logger {
    return &ConsoleLogger{}
}

// Использование
func main() {
    var factory LoggerFactory

    // Используем файловый logger
    factory = &FileLoggerFactory{FilePath: "/var/log/app.log"}
    logger := factory.CreateLogger()
    logger.Log("Application started")

    // Переключаемся на консольный logger
    factory = &ConsoleLoggerFactory{}
    logger = factory.CreateLogger()
    logger.Log("Debug message")
}
```

### Функциональный подход (идиоматично для Go)

```go
// Constructor function pattern
type User struct {
    ID        int
    Name      string
    Email     string
    CreatedAt time.Time
}

// NewUser - базовый конструктор
func NewUser(name, email string) *User {
    return &User{
        ID:        generateID(),
        Name:      name,
        Email:     email,
        CreatedAt: time.Now(),
    }
}

// NewUserFromDB - конструктор для создания из БД
func NewUserFromDB(id int, name, email string, createdAt time.Time) *User {
    return &User{
        ID:        id,
        Name:      name,
        Email:     email,
        CreatedAt: createdAt,
    }
}

// NewGuestUser - конструктор для гостя
func NewGuestUser() *User {
    return &User{
        ID:        0,
        Name:      "Guest",
        Email:     "",
        CreatedAt: time.Now(),
    }
}
```

## Builder (Строитель)

Builder отделяет конструирование сложного объекта от его представления, позволяя создавать различные представления.

### Классический Builder

```go
// Продукт
type House struct {
    Windows  int
    Doors    int
    Floors   int
    HasGarage bool
    HasPool   bool
}

// Builder интерфейс
type HouseBuilder interface {
    SetWindows(count int) HouseBuilder
    SetDoors(count int) HouseBuilder
    SetFloors(count int) HouseBuilder
    SetGarage(has bool) HouseBuilder
    SetPool(has bool) HouseBuilder
    Build() *House
}

// Конкретный Builder
type ConcreteHouseBuilder struct {
    house *House
}

func NewHouseBuilder() *ConcreteHouseBuilder {
    return &ConcreteHouseBuilder{
        house: &House{},
    }
}

func (b *ConcreteHouseBuilder) SetWindows(count int) HouseBuilder {
    b.house.Windows = count
    return b
}

func (b *ConcreteHouseBuilder) SetDoors(count int) HouseBuilder {
    b.house.Doors = count
    return b
}

func (b *ConcreteHouseBuilder) SetFloors(count int) HouseBuilder {
    b.house.Floors = count
    return b
}

func (b *ConcreteHouseBuilder) SetGarage(has bool) HouseBuilder {
    b.house.HasGarage = has
    return b
}

func (b *ConcreteHouseBuilder) SetPool(has bool) HouseBuilder {
    b.house.HasPool = has
    return b
}

func (b *ConcreteHouseBuilder) Build() *House {
    return b.house
}

// Использование
func main() {
    house := NewHouseBuilder().
        SetWindows(10).
        SetDoors(2).
        SetFloors(2).
        SetGarage(true).
        SetPool(false).
        Build()

    fmt.Printf("House: %+v\n", house)
}
```

### Functional Options Pattern (идиоматично для Go)

```go
// Продукт
type Server struct {
    Host         string
    Port         int
    Timeout      time.Duration
    MaxConnections int
    TLS          *tls.Config
}

// Option функция
type ServerOption func(*Server)

// Функции-опции
func WithHost(host string) ServerOption {
    return func(s *Server) {
        s.Host = host
    }
}

func WithPort(port int) ServerOption {
    return func(s *Server) {
        s.Port = port
    }
}

func WithTimeout(timeout time.Duration) ServerOption {
    return func(s *Server) {
        s.Timeout = timeout
    }
}

func WithMaxConnections(max int) ServerOption {
    return func(s *Server) {
        s.MaxConnections = max
    }
}

func WithTLS(config *tls.Config) ServerOption {
    return func(s *Server) {
        s.TLS = config
    }
}

// Конструктор с опциями
func NewServer(opts ...ServerOption) *Server {
    // Значения по умолчанию
    server := &Server{
        Host:           "localhost",
        Port:           8080,
        Timeout:        30 * time.Second,
        MaxConnections: 100,
    }

    // Применяем опции
    for _, opt := range opts {
        opt(server)
    }

    return server
}

// Использование
func main() {
    // Сервер с настройками по умолчанию
    defaultServer := NewServer()

    // Сервер с кастомными настройками
    customServer := NewServer(
        WithHost("0.0.0.0"),
        WithPort(3000),
        WithTimeout(60 * time.Second),
        WithMaxConnections(200),
    )

    // Можно передавать только нужные опции
    simpleServer := NewServer(
        WithPort(9000),
    )
}
```

### Builder с Director (управляющий)

```go
// Director - управляет процессом построения
type HouseDirector struct {
    builder HouseBuilder
}

func NewHouseDirector(builder HouseBuilder) *HouseDirector {
    return &HouseDirector{builder: builder}
}

// Предустановленные конфигурации
func (d *HouseDirector) BuildSmallHouse() *House {
    return d.builder.
        SetWindows(4).
        SetDoors(1).
        SetFloors(1).
        SetGarage(false).
        SetPool(false).
        Build()
}

func (d *HouseDirector) BuildMansion() *House {
    return d.builder.
        SetWindows(20).
        SetDoors(4).
        SetFloors(3).
        SetGarage(true).
        SetPool(true).
        Build()
}

// Использование
func main() {
    builder := NewHouseBuilder()
    director := NewHouseDirector(builder)

    smallHouse := director.BuildSmallHouse()
    mansion := director.BuildMansion()
}
```

## Сравнение паттернов

| Паттерн | Назначение | Когда использовать |
|---------|------------|-------------------|
| **Singleton** | Один экземпляр объекта | DB, Config, Logger |
| **Factory** | Создание объектов через функцию | Разные типы объектов одного интерфейса |
| **Builder** | Пошаговое создание сложного объекта | Много опциональных параметров |

## Best Practices в Go

### ✅ Правильно

```go
// Functional Options для гибкости
func NewServer(opts ...ServerOption) *Server

// Constructor functions для простых случаев
func NewUser(name, email string) *User

// sync.Once для thread-safe singleton
var once sync.Once
once.Do(func() { /* init */ })

// Factory через функции (не через типы)
func NewLogger(loggerType string) Logger
```

### ❌ Неправильно

```go
// Не использовать Singleton везде
var GlobalDB *sql.DB // Плохо для тестирования

// Не делать builder для простых структур
type User struct {
    Name string
}
// Не нужен builder, достаточно NewUser(name string)

// Не использовать сложные иерархии фабрик
type AbstractFactory interface {
    CreateAbstractProduct() AbstractProduct
}
// В Go это излишне сложно
```

## Вопросы с собеседований

**Вопрос:** Как реализовать потокобезопасный Singleton в Go?

**Ответ:** Использовать `sync.Once`, который гарантирует, что код выполнится только один раз даже при конкурентном доступе:
```go
var instance *Database
var once sync.Once

func GetInstance() *Database {
    once.Do(func() {
        instance = &Database{}
    })
    return instance
}
```

**Вопрос:** Что такое Functional Options pattern?

**Ответ:** Идиоматичный Go паттерн для создания объектов с опциональными параметрами. Использует функции-опции вместо множественных конструкторов или традиционного Builder'а. Позволяет задавать значения по умолчанию и переопределять только нужные параметры.

**Вопрос:** В чем разница между Factory Method и Abstract Factory?

**Ответ:**
- **Factory Method** — один метод создания одного типа объектов
- **Abstract Factory** — семейство методов для создания семейства связанных объектов

В Go обычно достаточно простой factory функции без сложных иерархий.

## Связанные темы

- [[Go - WaitGroup и Once]]
- [[Паттерны проектирования в Go]]
- [[Go - Интерфейсы]]
- [[SOLID принципы]]