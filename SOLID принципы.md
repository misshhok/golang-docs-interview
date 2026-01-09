# SOLID принципы

SOLID — это пять принципов объектно-ориентированного программирования, которые помогают создавать поддерживаемый, расширяемый и тестируемый код. В Go эти принципы применяются с учетом особенностей языка: отсутствия классов и использования интерфейсов и композиции.

## S — Single Responsibility Principle

**Принцип единственной ответственности:** каждый модуль/структура должны иметь одну причину для изменения.

### ❌ Неправильно

```go
// User делает слишком много
type User struct {
    ID       int
    Name     string
    Email    string
    Password string
}

// Нарушение SRP: смешивает бизнес-логику, валидацию и работу с БД
func (u *User) Save() error {
    // Валидация
    if u.Name == "" {
        return errors.New("name is required")
    }
    if !strings.Contains(u.Email, "@") {
        return errors.New("invalid email")
    }

    // Хеширование пароля
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(u.Password), bcrypt.DefaultCost)
    if err != nil {
        return err
    }
    u.Password = string(hashedPassword)

    // Сохранение в БД
    db := getDB()
    _, err = db.Exec("INSERT INTO users (name, email, password) VALUES ($1, $2, $3)",
        u.Name, u.Email, u.Password)

    // Отправка email
    sendWelcomeEmail(u.Email)

    return err
}
```

### ✅ Правильно

```go
// User — только данные
type User struct {
    ID       int
    Name     string
    Email    string
    Password string
}

// UserValidator — валидация
type UserValidator struct{}

func (v *UserValidator) Validate(user *User) error {
    if user.Name == "" {
        return errors.New("name is required")
    }
    if !strings.Contains(user.Email, "@") {
        return errors.New("invalid email")
    }
    return nil
}

// PasswordHasher — работа с паролями
type PasswordHasher struct{}

func (h *PasswordHasher) Hash(password string) (string, error) {
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(hashedPassword), err
}

// UserRepository — работа с БД
type UserRepository struct {
    db *sql.DB
}

func (r *UserRepository) Save(user *User) error {
    _, err := r.db.Exec("INSERT INTO users (name, email, password) VALUES ($1, $2, $3)",
        user.Name, user.Email, user.Password)
    return err
}

// EmailService — отправка email
type EmailService struct{}

func (s *EmailService) SendWelcomeEmail(email string) error {
    // Отправка email
    return nil
}

// UserService — координирует все операции
type UserService struct {
    validator  *UserValidator
    hasher     *PasswordHasher
    repository *UserRepository
    email      *EmailService
}

func (s *UserService) RegisterUser(user *User) error {
    // Валидация
    if err := s.validator.Validate(user); err != nil {
        return err
    }

    // Хеширование пароля
    hashedPassword, err := s.hasher.Hash(user.Password)
    if err != nil {
        return err
    }
    user.Password = hashedPassword

    // Сохранение
    if err := s.repository.Save(user); err != nil {
        return err
    }

    // Email
    return s.email.SendWelcomeEmail(user.Email)
}
```

## O — Open/Closed Principle

**Принцип открытости/закрытости:** программные сущности должны быть открыты для расширения, но закрыты для модификации.

### ❌ Неправильно

```go
type PaymentProcessor struct{}

func (p *PaymentProcessor) Process(paymentType string, amount float64) error {
    if paymentType == "credit_card" {
        // Обработка кредитной карты
        return processCreditCard(amount)
    } else if paymentType == "paypal" {
        // Обработка PayPal
        return processPayPal(amount)
    } else if paymentType == "crypto" {
        // Обработка криптовалюты
        return processCrypto(amount)
    }
    // При добавлении нового способа нужно менять этот код!
    return errors.New("unknown payment type")
}
```

### ✅ Правильно

```go
// Интерфейс для платежных методов
type PaymentMethod interface {
    Process(amount float64) error
}

// Конкретные реализации
type CreditCardPayment struct {
    CardNumber string
}

func (c *CreditCardPayment) Process(amount float64) error {
    fmt.Printf("Processing credit card payment: $%.2f\n", amount)
    return nil
}

type PayPalPayment struct {
    Email string
}

func (p *PayPalPayment) Process(amount float64) error {
    fmt.Printf("Processing PayPal payment: $%.2f\n", amount)
    return nil
}

type CryptoPayment struct {
    WalletAddress string
}

func (c *CryptoPayment) Process(amount float64) error {
    fmt.Printf("Processing crypto payment: $%.2f\n", amount)
    return nil
}

// PaymentProcessor не нужно менять при добавлении новых методов
type PaymentProcessor struct {
    method PaymentMethod
}

func (p *PaymentProcessor) Process(amount float64) error {
    return p.method.Process(amount)
}

// Использование
func main() {
    // Легко добавляем новые способы оплаты без изменения кода
    processor := &PaymentProcessor{method: &CreditCardPayment{CardNumber: "1234"}}
    processor.Process(100.0)

    processor.method = &PayPalPayment{Email: "user@example.com"}
    processor.Process(200.0)
}
```

## L — Liskov Substitution Principle

**Принцип подстановки Лисков:** объекты должны быть заменяемы на экземпляры их подтипов без изменения корректности программы.

В Go это означает, что все реализации интерфейса должны вести себя предсказуемо и соответствовать контракту интерфейса.

### ❌ Неправильно

```go
type Bird interface {
    Fly() error
}

type Sparrow struct{}

func (s *Sparrow) Fly() error {
    fmt.Println("Sparrow is flying")
    return nil
}

type Penguin struct{}

func (p *Penguin) Fly() error {
    // Пингвин не может летать!
    return errors.New("penguins can't fly")
}

// Эта функция предполагает что все птицы могут летать
func MakeBirdFly(b Bird) {
    if err := b.Fly(); err != nil {
        // Неожиданная ошибка для птицы
        panic(err)
    }
}
```

### ✅ Правильно

```go
// Разделяем интерфейсы по возможностям
type Bird interface {
    Eat() error
}

type FlyingBird interface {
    Bird
    Fly() error
}

type SwimmingBird interface {
    Bird
    Swim() error
}

type Sparrow struct{}

func (s *Sparrow) Eat() error {
    fmt.Println("Sparrow is eating")
    return nil
}

func (s *Sparrow) Fly() error {
    fmt.Println("Sparrow is flying")
    return nil
}

type Penguin struct{}

func (p *Penguin) Eat() error {
    fmt.Println("Penguin is eating")
    return nil
}

func (p *Penguin) Swim() error {
    fmt.Println("Penguin is swimming")
    return nil
}

// Теперь функции ожидают только те возможности, которые нужны
func MakeFly(b FlyingBird) {
    b.Fly()
}

func MakeSwim(b SwimmingBird) {
    b.Swim()
}
```

## I — Interface Segregation Principle

**Принцип разделения интерфейсов:** клиенты не должны зависеть от интерфейсов, которые они не используют.

### ❌ Неправильно

```go
// Слишком большой интерфейс
type Worker interface {
    Work() error
    Eat() error
    Sleep() error
    GetSalary() float64
    TakeSickLeave() error
    AttendMeeting() error
}

// Robot реализует Worker, но роботы не едят и не спят!
type Robot struct{}

func (r *Robot) Work() error {
    return nil
}

func (r *Robot) Eat() error {
    // Роботу не нужна эта функция
    return errors.New("robots don't eat")
}

func (r *Robot) Sleep() error {
    // Роботу не нужна эта функция
    return errors.New("robots don't sleep")
}

func (r *Robot) GetSalary() float64 {
    return 0 // Роботы не получают зарплату
}

func (r *Robot) TakeSickLeave() error {
    return errors.New("robots don't get sick")
}

func (r *Robot) AttendMeeting() error {
    return nil
}
```

### ✅ Правильно

```go
// Разделяем на маленькие интерфейсы
type Workable interface {
    Work() error
}

type Eatable interface {
    Eat() error
}

type Sleepable interface {
    Sleep() error
}

type Payable interface {
    GetSalary() float64
}

type Attendable interface {
    AttendMeeting() error
}

// Human реализует все
type Human struct {
    Name   string
    Salary float64
}

func (h *Human) Work() error {
    fmt.Printf("%s is working\n", h.Name)
    return nil
}

func (h *Human) Eat() error {
    fmt.Printf("%s is eating\n", h.Name)
    return nil
}

func (h *Human) Sleep() error {
    fmt.Printf("%s is sleeping\n", h.Name)
    return nil
}

func (h *Human) GetSalary() float64 {
    return h.Salary
}

func (h *Human) AttendMeeting() error {
    fmt.Printf("%s is attending meeting\n", h.Name)
    return nil
}

// Robot реализует только то, что ему нужно
type Robot struct {
    Model string
}

func (r *Robot) Work() error {
    fmt.Printf("Robot %s is working\n", r.Model)
    return nil
}

func (r *Robot) AttendMeeting() error {
    fmt.Printf("Robot %s is attending meeting\n", r.Model)
    return nil
}

// Функции принимают только нужные интерфейсы
func DoWork(w Workable) {
    w.Work()
}

func ServeLunch(e Eatable) {
    e.Eat()
}

func ProcessPayroll(p Payable) {
    salary := p.GetSalary()
    fmt.Printf("Processing payroll: $%.2f\n", salary)
}
```

## D — Dependency Inversion Principle

**Принцип инверсии зависимостей:** модули верхнего уровня не должны зависеть от модулей нижнего уровня. Оба должны зависеть от абстракций.

### ❌ Неправильно

```go
// UserService зависит от конкретной реализации БД
type UserService struct {
    db *PostgresDB // Жесткая зависимость!
}

type PostgresDB struct {
    conn *sql.DB
}

func (db *PostgresDB) SaveUser(user *User) error {
    _, err := db.conn.Exec("INSERT INTO users ...")
    return err
}

func (s *UserService) CreateUser(user *User) error {
    // Если захотим сменить БД, придется менять UserService
    return s.db.SaveUser(user)
}
```

### ✅ Правильно

```go
// Определяем интерфейс (абстракцию)
type UserRepository interface {
    Save(user *User) error
    FindByID(id int) (*User, error)
    FindByEmail(email string) (*User, error)
}

// UserService зависит от интерфейса, а не конкретной реализации
type UserService struct {
    repo UserRepository // Зависимость от абстракции
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) CreateUser(user *User) error {
    return s.repo.Save(user)
}

// Конкретные реализации
type PostgresUserRepository struct {
    db *sql.DB
}

func (r *PostgresUserRepository) Save(user *User) error {
    _, err := r.db.Exec("INSERT INTO users ...")
    return err
}

func (r *PostgresUserRepository) FindByID(id int) (*User, error) {
    // ...
    return nil, nil
}

func (r *PostgresUserRepository) FindByEmail(email string) (*User, error) {
    // ...
    return nil, nil
}

// Легко создать альтернативную реализацию
type MongoUserRepository struct {
    client *mongo.Client
}

func (r *MongoUserRepository) Save(user *User) error {
    // Сохранение в MongoDB
    return nil
}

func (r *MongoUserRepository) FindByID(id int) (*User, error) {
    return nil, nil
}

func (r *MongoUserRepository) FindByEmail(email string) (*User, error) {
    return nil, nil
}

// In-memory для тестов
type InMemoryUserRepository struct {
    users map[int]*User
}

func (r *InMemoryUserRepository) Save(user *User) error {
    r.users[user.ID] = user
    return nil
}

func (r *InMemoryUserRepository) FindByID(id int) (*User, error) {
    user, ok := r.users[id]
    if !ok {
        return nil, errors.New("user not found")
    }
    return user, nil
}

func (r *InMemoryUserRepository) FindByEmail(email string) (*User, error) {
    for _, user := range r.users {
        if user.Email == email {
            return user, nil
        }
    }
    return nil, errors.New("user not found")
}

// Использование
func main() {
    // Легко меняем реализацию
    postgresRepo := &PostgresUserRepository{db: getDB()}
    service := NewUserService(postgresRepo)

    // Или используем MongoDB
    mongoRepo := &MongoUserRepository{client: getMongoClient()}
    service = NewUserService(mongoRepo)

    // Или in-memory для тестов
    memoryRepo := &InMemoryUserRepository{users: make(map[int]*User)}
    service = NewUserService(memoryRepo)
}
```

## Полный пример с SOLID

```go
package main

import (
    "fmt"
)

// === Интерфейсы (Dependency Inversion) ===

type UserRepository interface {
    Save(user *User) error
    FindByEmail(email string) (*User, error)
}

type EmailSender interface {
    Send(to, subject, body string) error
}

type PasswordHasher interface {
    Hash(password string) (string, error)
    Verify(password, hash string) bool
}

// === Доменная модель (Single Responsibility) ===

type User struct {
    ID       int
    Name     string
    Email    string
    Password string
}

// === Реализации (Open/Closed, можно добавлять новые) ===

type InMemoryUserRepository struct {
    users map[string]*User
    nextID int
}

func NewInMemoryUserRepository() *InMemoryUserRepository {
    return &InMemoryUserRepository{
        users:  make(map[string]*User),
        nextID: 1,
    }
}

func (r *InMemoryUserRepository) Save(user *User) error {
    if user.ID == 0 {
        user.ID = r.nextID
        r.nextID++
    }
    r.users[user.Email] = user
    return nil
}

func (r *InMemoryUserRepository) FindByEmail(email string) (*User, error) {
    user, ok := r.users[email]
    if !ok {
        return nil, fmt.Errorf("user not found")
    }
    return user, nil
}

type ConsoleEmailSender struct{}

func (s *ConsoleEmailSender) Send(to, subject, body string) error {
    fmt.Printf("Email to %s: %s\n%s\n", to, subject, body)
    return nil
}

type SimplePasswordHasher struct{}

func (h *SimplePasswordHasher) Hash(password string) (string, error) {
    return "hashed_" + password, nil
}

func (h *SimplePasswordHasher) Verify(password, hash string) bool {
    return "hashed_"+password == hash
}

// === Сервисный слой (координирует операции) ===

type UserService struct {
    repo   UserRepository
    email  EmailSender
    hasher PasswordHasher
}

func NewUserService(repo UserRepository, email EmailSender, hasher PasswordHasher) *UserService {
    return &UserService{
        repo:   repo,
        email:  email,
        hasher: hasher,
    }
}

func (s *UserService) RegisterUser(name, email, password string) error {
    // Проверка существования
    existing, _ := s.repo.FindByEmail(email)
    if existing != nil {
        return fmt.Errorf("user already exists")
    }

    // Хеширование пароля
    hashedPassword, err := s.hasher.Hash(password)
    if err != nil {
        return err
    }

    // Создание пользователя
    user := &User{
        Name:     name,
        Email:    email,
        Password: hashedPassword,
    }

    // Сохранение
    if err := s.repo.Save(user); err != nil {
        return err
    }

    // Отправка email
    return s.email.Send(email, "Welcome!", "Welcome to our service!")
}

func main() {
    // Dependency Injection
    repo := NewInMemoryUserRepository()
    emailSender := &ConsoleEmailSender{}
    hasher := &SimplePasswordHasher{}

    service := NewUserService(repo, emailSender, hasher)

    // Использование
    err := service.RegisterUser("Иван", "ivan@example.com", "secret123")
    if err != nil {
        fmt.Println("Error:", err)
    }
}
```

## Best Practices в Go

### ✅ Правильно

```go
// Малые, сфокусированные интерфейсы
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Зависимость от интерфейсов, а не конкретных типов
func ProcessData(r Reader) error

// Композиция интерфейсов
type ReadWriter interface {
    Reader
    Writer
}
```

### ❌ Неправильно

```go
// Большие интерфейсы
type MegaInterface interface {
    Method1()
    Method2()
    // ... 20 методов
}

// Зависимость от конкретных типов
func ProcessData(db *PostgresDB) error
```

## Вопросы с собеседований

**Вопрос:** Какой принцип SOLID нарушается, если структура одновременно валидирует данные, сохраняет их в БД и отправляет email?

**Ответ:** Single Responsibility Principle — структура имеет три причины для изменения (изменение правил валидации, изменение схемы БД, изменение способа отправки email).

**Вопрос:** Как применяется Liskov Substitution Principle в Go?

**Ответ:** Все реализации интерфейса должны вести себя согласно контракту интерфейса. Если какая-то реализация не может выполнить все методы интерфейса или ведет себя неожиданно — лучше создать отдельный интерфейс.

**Вопрос:** Почему в Go часто говорят "accept interfaces, return structs"?

**Ответ:** Это следует принципу Dependency Inversion — функции принимают интерфейсы (абстракции), что делает код гибким и тестируемым, но возвращают конкретные типы, что дает вызывающему коду полный контроль над возвращаемым значением.

## Связанные темы

- [[Go - Интерфейсы]]
- [[Паттерны проектирования в Go]]
- [[Repository паттерн]]
- [[Clean Architecture]]
- [[Go - Моки и стабы]]