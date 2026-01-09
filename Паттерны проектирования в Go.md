# Паттерны проектирования в Go

Паттерны проектирования — это проверенные решения типовых задач в программировании. В Go реализация паттернов отличается от классических ООП языков из-за особенностей языка: отсутствия классов и наследования, наличия интерфейсов и композиции.

## Особенности Go

### Композиция вместо наследования

```go
// В классическом ООП используется наследование
// class Dog extends Animal { }

// В Go используется композиция (embedding)
type Animal struct {
    Name string
}

func (a *Animal) Speak() string {
    return "Some sound"
}

type Dog struct {
    Animal // Встраивание (embedding)
    Breed string
}

// Dog автоматически получает метод Speak()
dog := Dog{Animal: Animal{Name: "Бобик"}, Breed: "Овчарка"}
dog.Speak() // "Some sound"
```

### Интерфейсы вместо абстрактных классов

```go
// Интерфейс неявно реализуется
type Speaker interface {
    Speak() string
}

type Dog struct {
    Name string
}

// Dog реализует Speaker просто определив метод Speak
func (d Dog) Speak() string {
    return "Гав!"
}

// Полиморфизм через интерфейсы
func MakeSpeak(s Speaker) {
    fmt.Println(s.Speak())
}

dog := Dog{Name: "Бобик"}
MakeSpeak(dog) // Работает!
```

### Функции как first-class citizens

```go
// Функции можно передавать как параметры
type Handler func(string) string

func Process(data string, handler Handler) string {
    return handler(data)
}

// Использование
result := Process("hello", func(s string) string {
    return strings.ToUpper(s)
})
```

## Категории паттернов

### Порождающие паттерны (Creational)

Управляют процессом создания объектов.

**Основные паттерны:**
- **Singleton** — единственный экземпляр объекта
- **Factory** — создание объектов через функцию
- **Builder** — пошаговое создание сложных объектов
- **Prototype** — клонирование объектов
- **Object Pool** — переиспользование объектов

### Структурные паттерны (Structural)

Описывают композицию объектов и классов.

**Основные паттерны:**
- **Adapter** — адаптация интерфейса
- **Decorator** — добавление функциональности
- **Facade** — упрощенный интерфейс
- **Proxy** — контроль доступа к объекту
- **Composite** — древовидная структура объектов

### Поведенческие паттерны (Behavioral)

Описывают взаимодействие между объектами.

**Основные паттерны:**
- **Strategy** — семейство взаимозаменяемых алгоритмов
- **Observer** — подписка на события
- **Command** — инкапсуляция запроса
- **Iterator** — последовательный доступ к элементам
- **State** — изменение поведения при смене состояния

## Примеры популярных паттернов в Go

### Strategy (Стратегия)

```go
// Интерфейс стратегии
type PaymentStrategy interface {
    Pay(amount float64) error
}

// Конкретные стратегии
type CreditCardPayment struct {
    CardNumber string
}

func (c *CreditCardPayment) Pay(amount float64) error {
    fmt.Printf("Оплата %.2f руб. картой %s\n", amount, c.CardNumber)
    return nil
}

type PayPalPayment struct {
    Email string
}

func (p *PayPalPayment) Pay(amount float64) error {
    fmt.Printf("Оплата %.2f руб. через PayPal %s\n", amount, p.Email)
    return nil
}

// Контекст использует стратегию
type Order struct {
    Amount  float64
    Payment PaymentStrategy
}

func (o *Order) ProcessPayment() error {
    return o.Payment.Pay(o.Amount)
}

// Использование
order := Order{
    Amount:  1000,
    Payment: &CreditCardPayment{CardNumber: "1234-5678"},
}
order.ProcessPayment()

// Можем легко сменить стратегию
order.Payment = &PayPalPayment{Email: "user@example.com"}
order.ProcessPayment()
```

### Decorator (Декоратор)

```go
// Базовый интерфейс
type Coffee interface {
    Cost() float64
    Description() string
}

// Базовая реализация
type SimpleCoffee struct{}

func (s *SimpleCoffee) Cost() float64 {
    return 50
}

func (s *SimpleCoffee) Description() string {
    return "Простой кофе"
}

// Декоратор с молоком
type MilkDecorator struct {
    Coffee Coffee
}

func (m *MilkDecorator) Cost() float64 {
    return m.Coffee.Cost() + 20
}

func (m *MilkDecorator) Description() string {
    return m.Coffee.Description() + " + молоко"
}

// Декоратор с сахаром
type SugarDecorator struct {
    Coffee Coffee
}

func (s *SugarDecorator) Cost() float64 {
    return s.Coffee.Cost() + 5
}

func (s *SugarDecorator) Description() string {
    return s.Coffee.Description() + " + сахар"
}

// Использование
coffee := &SimpleCoffee{}
fmt.Printf("%s: %.2f руб.\n", coffee.Description(), coffee.Cost())
// Простой кофе: 50.00 руб.

coffeeWithMilk := &MilkDecorator{Coffee: coffee}
fmt.Printf("%s: %.2f руб.\n", coffeeWithMilk.Description(), coffeeWithMilk.Cost())
// Простой кофе + молоко: 70.00 руб.

coffeeWithMilkAndSugar := &SugarDecorator{Coffee: coffeeWithMilk}
fmt.Printf("%s: %.2f руб.\n", coffeeWithMilkAndSugar.Description(), coffeeWithMilkAndSugar.Cost())
// Простой кофе + молоко + сахар: 75.00 руб.
```

### Observer (Наблюдатель)

```go
// Наблюдатель
type Observer interface {
    Update(message string)
}

// Издатель (Subject)
type Subject struct {
    observers []Observer
}

func (s *Subject) Attach(o Observer) {
    s.observers = append(s.observers, o)
}

func (s *Subject) Notify(message string) {
    for _, observer := range s.observers {
        observer.Update(message)
    }
}

// Конкретный наблюдатель - Email
type EmailObserver struct {
    Email string
}

func (e *EmailObserver) Update(message string) {
    fmt.Printf("Email на %s: %s\n", e.Email, message)
}

// Конкретный наблюдатель - SMS
type SMSObserver struct {
    Phone string
}

func (s *SMSObserver) Update(message string) {
    fmt.Printf("SMS на %s: %s\n", s.Phone, message)
}

// Использование
subject := &Subject{}

emailObserver := &EmailObserver{Email: "user@example.com"}
smsObserver := &SMSObserver{Phone: "+7 999 123-45-67"}

subject.Attach(emailObserver)
subject.Attach(smsObserver)

subject.Notify("Ваш заказ доставлен!")
// Email на user@example.com: Ваш заказ доставлен!
// SMS на +7 999 123-45-67: Ваш заказ доставлен!
```

### Adapter (Адаптер)

```go
// Целевой интерфейс
type ModernPrinter interface {
    PrintJSON(data interface{}) error
}

// Старый принтер (несовместимый интерфейс)
type LegacyPrinter struct{}

func (l *LegacyPrinter) Print(text string) {
    fmt.Println("Legacy Print:", text)
}

// Адаптер
type PrinterAdapter struct {
    legacy *LegacyPrinter
}

func (a *PrinterAdapter) PrintJSON(data interface{}) error {
    jsonData, err := json.Marshal(data)
    if err != nil {
        return err
    }

    a.legacy.Print(string(jsonData))
    return nil
}

// Использование
func UseModernPrinter(printer ModernPrinter) {
    data := map[string]string{"name": "John", "age": "30"}
    printer.PrintJSON(data)
}

legacy := &LegacyPrinter{}
adapter := &PrinterAdapter{legacy: legacy}
UseModernPrinter(adapter)
// Legacy Print: {"age":"30","name":"John"}
```

### Command (Команда)

```go
// Интерфейс команды
type Command interface {
    Execute() error
    Undo() error
}

// Receiver - объект, выполняющий действия
type Light struct {
    IsOn bool
}

func (l *Light) TurnOn() {
    l.IsOn = true
    fmt.Println("Свет включен")
}

func (l *Light) TurnOff() {
    l.IsOn = false
    fmt.Println("Свет выключен")
}

// Конкретная команда
type LightOnCommand struct {
    Light *Light
}

func (c *LightOnCommand) Execute() error {
    c.Light.TurnOn()
    return nil
}

func (c *LightOnCommand) Undo() error {
    c.Light.TurnOff()
    return nil
}

// Invoker - вызывает команды
type RemoteControl struct {
    Command Command
    History []Command
}

func (r *RemoteControl) PressButton() error {
    if err := r.Command.Execute(); err != nil {
        return err
    }
    r.History = append(r.History, r.Command)
    return nil
}

func (r *RemoteControl) PressUndo() error {
    if len(r.History) == 0 {
        return nil
    }

    lastCommand := r.History[len(r.History)-1]
    r.History = r.History[:len(r.History)-1]

    return lastCommand.Undo()
}

// Использование
light := &Light{}
command := &LightOnCommand{Light: light}
remote := &RemoteControl{Command: command}

remote.PressButton() // Свет включен
remote.PressUndo()   // Свет выключен
```

## Идиоматичные паттерны Go

### Functional Options

```go
// Конфигурация сервера
type Server struct {
    Host    string
    Port    int
    Timeout time.Duration
}

// Option функция
type Option func(*Server)

func WithHost(host string) Option {
    return func(s *Server) {
        s.Host = host
    }
}

func WithPort(port int) Option {
    return func(s *Server) {
        s.Port = port
    }
}

func WithTimeout(timeout time.Duration) Option {
    return func(s *Server) {
        s.Timeout = timeout
    }
}

// Конструктор с опциями
func NewServer(opts ...Option) *Server {
    // Значения по умолчанию
    server := &Server{
        Host:    "localhost",
        Port:    8080,
        Timeout: 30 * time.Second,
    }

    // Применяем опции
    for _, opt := range opts {
        opt(server)
    }

    return server
}

// Использование
server := NewServer(
    WithHost("0.0.0.0"),
    WithPort(3000),
    WithTimeout(60 * time.Second),
)
```

### Pipeline Pattern

```go
// Стадии pipeline
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func filter(in <-chan int, predicate func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            if predicate(n) {
                out <- n
            }
        }
        close(out)
    }()
    return out
}

// Использование pipeline
nums := generator(1, 2, 3, 4, 5)
squared := square(nums)
filtered := filter(squared, func(n int) bool {
    return n > 10
})

for n := range filtered {
    fmt.Println(n) // 16, 25
}
```

## Best Practices

### ✅ Правильно

```go
// Использовать интерфейсы для абстракции
type Storage interface {
    Save(data []byte) error
    Load() ([]byte, error)
}

// Малые интерфейсы (Interface Segregation)
type Reader interface {
    Read() ([]byte, error)
}

type Writer interface {
    Write([]byte) error
}

// Композиция интерфейсов
type ReadWriter interface {
    Reader
    Writer
}

// Dependency injection через интерфейсы
func ProcessData(storage Storage) error {
    data, err := storage.Load()
    if err != nil {
        return err
    }
    // Обработка...
    return storage.Save(data)
}
```

### ❌ Неправильно

```go
// Переусложнение простых вещей паттернами
// Не нужен паттерн Singleton для конфигурации
var Config = &Configuration{} // Просто глобальная переменная

// Не создавать большие интерфейсы
type MegaInterface interface {
    Method1()
    Method2()
    Method3()
    // ... 20 методов
}
```

## Вопросы с собеседований

**Вопрос:** Почему в Go нет классического паттерна Singleton?

**Ответ:** В Go нет конструкторов и статических полей как в классических ООП языках. Вместо этого используется `sync.Once` для ленивой инициализации или просто глобальная переменная с `init()` функцией для eager инициализации.

**Вопрос:** Как реализовать наследование в Go?

**Ответ:** В Go нет наследования. Вместо этого используется композиция через встраивание (embedding) структур и полиморфизм через интерфейсы.

**Вопрос:** Что такое Functional Options pattern?

**Ответ:** Идиоматичный Go паттерн для создания объектов с опциональными параметрами. Использует функции-опции вместо множественных конструкторов или builder'а.

## Связанные темы

- [[Go - Интерфейсы]]
- [[Singleton, Factory, Builder в Go]]
- [[Repository паттерн]]
- [[FSM (Конечные автоматы)]]
- [[SOLID принципы]]
- [[Clean Architecture]]