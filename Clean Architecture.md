# Clean Architecture

Clean Architecture — это архитектурный подход Роберта Мартина (Uncle Bob), который фокусируется на разделении ответственности и независимости от внешних фреймворков, БД и UI. Главная идея — бизнес-логика не должна зависеть от деталей реализации.

## Основные принципы

### Dependency Rule (Правило зависимостей)

Зависимости направлены только внутрь — внешние слои могут зависеть от внутренних, но не наоборот.

```
Внешние слои → Внутренние слои
(Frameworks)  →  (Use Cases)  →  (Entities)
```

### Независимость от:

✅ **Фреймворков** — бизнес-логика не зависит от Gin, Echo, GORM
✅ **UI** — можно заменить REST API на gRPC без изменения логики
✅ **Базы данных** — можно сменить Postgres на MongoDB
✅ **Внешних сервисов** — бизнес-логика не знает о деталях интеграций

## Слои Clean Architecture

### 1. Entities (Сущности) — Центр

Бизнес-объекты и правила, которые не зависят от приложения.

```go
package domain

import "time"

// User - доменная сущность
type User struct {
    ID        int
    Name      string
    Email     string
    Password  string
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Бизнес-правила в доменной модели
func (u *User) IsEmailValid() bool {
    return len(u.Email) > 0 && contains(u.Email, "@")
}

func (u *User) CanBeSuspended() bool {
    // Бизнес-правило: пользователь должен существовать больше 30 дней
    return time.Since(u.CreatedAt) > 30*24*time.Hour
}

// Order - доменная сущность
type Order struct {
    ID          int
    UserID      int
    Items       []OrderItem
    TotalAmount float64
    Status      OrderStatus
    CreatedAt   time.Time
}

type OrderItem struct {
    ProductID int
    Quantity  int
    Price     float64
}

type OrderStatus string

const (
    OrderPending   OrderStatus = "pending"
    OrderConfirmed OrderStatus = "confirmed"
    OrderShipped   OrderStatus = "shipped"
    OrderDelivered OrderStatus = "delivered"
)

// Бизнес-правило
func (o *Order) CalculateTotal() float64 {
    var total float64
    for _, item := range o.Items {
        total += item.Price * float64(item.Quantity)
    }
    return total
}

func (o *Order) CanBeCancelled() bool {
    return o.Status == OrderPending || o.Status == OrderConfirmed
}
```

### 2. Use Cases (Варианты использования)

Бизнес-логика приложения. Оркестрирует поток данных между Entities и внешним миром.

```go
package usecase

import (
    "context"
    "errors"
    "myapp/domain"
)

// Интерфейсы для внешних зависимостей
type UserRepository interface {
    Create(ctx context.Context, user *domain.User) error
    FindByID(ctx context.Context, id int) (*domain.User, error)
    FindByEmail(ctx context.Context, email string) (*domain.User, error)
    Update(ctx context.Context, user *domain.User) error
    Delete(ctx context.Context, id int) error
}

type PasswordHasher interface {
    Hash(password string) (string, error)
    Compare(password, hash string) bool
}

type EmailSender interface {
    SendWelcomeEmail(ctx context.Context, email string) error
}

// UserUseCase - бизнес-логика работы с пользователями
type UserUseCase struct {
    userRepo UserRepository
    hasher   PasswordHasher
    email    EmailSender
}

func NewUserUseCase(
    userRepo UserRepository,
    hasher PasswordHasher,
    email EmailSender,
) *UserUseCase {
    return &UserUseCase{
        userRepo: userRepo,
        hasher:   hasher,
        email:    email,
    }
}

// RegisterUser - регистрация пользователя
func (uc *UserUseCase) RegisterUser(ctx context.Context, name, email, password string) (*domain.User, error) {
    // Проверка существования
    existing, _ := uc.userRepo.FindByEmail(ctx, email)
    if existing != nil {
        return nil, errors.New("email already exists")
    }

    // Валидация
    if len(password) < 8 {
        return nil, errors.New("password too short")
    }

    // Хеширование пароля
    hashedPassword, err := uc.hasher.Hash(password)
    if err != nil {
        return nil, err
    }

    // Создание пользователя
    user := &domain.User{
        Name:     name,
        Email:    email,
        Password: hashedPassword,
    }

    // Валидация доменной модели
    if !user.IsEmailValid() {
        return nil, errors.New("invalid email format")
    }

    // Сохранение
    if err := uc.userRepo.Create(ctx, user); err != nil {
        return nil, err
    }

    // Отправка welcome email
    if err := uc.email.SendWelcomeEmail(ctx, email); err != nil {
        // Логируем, но не возвращаем ошибку
        // (email - не критичная операция)
    }

    return user, nil
}

// GetUser - получение пользователя
func (uc *UserUseCase) GetUser(ctx context.Context, id int) (*domain.User, error) {
    return uc.userRepo.FindByID(ctx, id)
}

// UpdateUser - обновление пользователя
func (uc *UserUseCase) UpdateUser(ctx context.Context, id int, name, email string) (*domain.User, error) {
    user, err := uc.userRepo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }

    user.Name = name
    user.Email = email

    if !user.IsEmailValid() {
        return nil, errors.New("invalid email format")
    }

    if err := uc.userRepo.Update(ctx, user); err != nil {
        return nil, err
    }

    return user, nil
}
```

### 3. Interface Adapters (Адаптеры интерфейсов)

Преобразование данных между форматами Use Cases и внешними системами.

#### Repository (адаптер для БД)

```go
package repository

import (
    "context"
    "database/sql"
    "myapp/domain"
)

// PostgresUserRepository - адаптер для PostgreSQL
type PostgresUserRepository struct {
    db *sql.DB
}

func NewPostgresUserRepository(db *sql.DB) *PostgresUserRepository {
    return &PostgresUserRepository{db: db}
}

func (r *PostgresUserRepository) Create(ctx context.Context, user *domain.User) error {
    query := `
        INSERT INTO users (name, email, password, created_at, updated_at)
        VALUES ($1, $2, $3, NOW(), NOW())
        RETURNING id, created_at, updated_at
    `

    err := r.db.QueryRowContext(ctx, query, user.Name, user.Email, user.Password).
        Scan(&user.ID, &user.CreatedAt, &user.UpdatedAt)

    return err
}

func (r *PostgresUserRepository) FindByID(ctx context.Context, id int) (*domain.User, error) {
    query := `
        SELECT id, name, email, password, created_at, updated_at
        FROM users
        WHERE id = $1
    `

    user := &domain.User{}
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.Password,
        &user.CreatedAt,
        &user.UpdatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, errors.New("user not found")
    }

    return user, err
}

func (r *PostgresUserRepository) FindByEmail(ctx context.Context, email string) (*domain.User, error) {
    query := `
        SELECT id, name, email, password, created_at, updated_at
        FROM users
        WHERE email = $1
    `

    user := &domain.User{}
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.Password,
        &user.CreatedAt,
        &user.UpdatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, nil // Не найден - не ошибка
    }

    return user, err
}

func (r *PostgresUserRepository) Update(ctx context.Context, user *domain.User) error {
    query := `
        UPDATE users
        SET name = $1, email = $2, updated_at = NOW()
        WHERE id = $3
    `

    _, err := r.db.ExecContext(ctx, query, user.Name, user.Email, user.ID)
    return err
}

func (r *PostgresUserRepository) Delete(ctx context.Context, id int) error {
    query := `DELETE FROM users WHERE id = $1`
    _, err := r.db.ExecContext(ctx, query, id)
    return err
}
```

#### HTTP Handler (адаптер для API)

```go
package handler

import (
    "encoding/json"
    "net/http"
    "strconv"
    "myapp/usecase"
    "github.com/gorilla/mux"
)

// UserHandler - HTTP адаптер
type UserHandler struct {
    userUseCase *usecase.UserUseCase
}

func NewUserHandler(userUseCase *usecase.UserUseCase) *UserHandler {
    return &UserHandler{userUseCase: userUseCase}
}

// DTO для HTTP
type RegisterUserRequest struct {
    Name     string `json:"name"`
    Email    string `json:"email"`
    Password string `json:"password"`
}

type UserResponse struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// RegisterUser - HTTP handler
func (h *UserHandler) RegisterUser(w http.ResponseWriter, r *http.Request) {
    var req RegisterUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    user, err := h.userUseCase.RegisterUser(r.Context(), req.Name, req.Email, req.Password)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Преобразование domain.User в UserResponse
    response := UserResponse{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(response)
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        http.Error(w, "invalid id", http.StatusBadRequest)
        return
    }

    user, err := h.userUseCase.GetUser(r.Context(), id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    response := UserResponse{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

### 4. Frameworks & Drivers (Внешний слой)

Конкретные технологии и инструменты.

```go
package main

import (
    "database/sql"
    "log"
    "net/http"

    "myapp/handler"
    "myapp/repository"
    "myapp/usecase"
    "myapp/infrastructure"

    "github.com/gorilla/mux"
    _ "github.com/lib/pq"
)

func main() {
    // Подключение к БД
    db, err := sql.Open("postgres", "postgres://user:pass@localhost/mydb?sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Создание зависимостей (DI)
    userRepo := repository.NewPostgresUserRepository(db)
    passwordHasher := infrastructure.NewBcryptHasher()
    emailSender := infrastructure.NewSMTPEmailSender()

    // Use Case
    userUseCase := usecase.NewUserUseCase(userRepo, passwordHasher, emailSender)

    // HTTP Handler
    userHandler := handler.NewUserHandler(userUseCase)

    // Роутер
    r := mux.NewRouter()
    r.HandleFunc("/users", userHandler.RegisterUser).Methods("POST")
    r.HandleFunc("/users/{id}", userHandler.GetUser).Methods("GET")

    // Запуск сервера
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", r))
}
```

## Структура проекта

```
myapp/
├── cmd/
│   └── server/
│       └── main.go              # Точка входа
├── internal/
│   ├── domain/                  # Entities (слой 1)
│   │   ├── user.go
│   │   └── order.go
│   ├── usecase/                 # Use Cases (слой 2)
│   │   ├── user_usecase.go
│   │   └── order_usecase.go
│   ├── repository/              # Adapters (слой 3)
│   │   ├── user_repository.go
│   │   └── order_repository.go
│   ├── handler/                 # Adapters (слой 3)
│   │   ├── user_handler.go
│   │   └── order_handler.go
│   └── infrastructure/          # Frameworks (слой 4)
│       ├── password_hasher.go
│       └── email_sender.go
├── pkg/                         # Публичные библиотеки
└── go.mod
```

## Преимущества Clean Architecture

### ✅ Тестируемость

```go
// Mock для тестирования
type MockUserRepository struct {
    users map[int]*domain.User
}

func (m *MockUserRepository) Create(ctx context.Context, user *domain.User) error {
    user.ID = len(m.users) + 1
    m.users[user.ID] = user
    return nil
}

// Тест Use Case без реальной БД
func TestUserUseCase_RegisterUser(t *testing.T) {
    mockRepo := &MockUserRepository{users: make(map[int]*domain.User)}
    mockHasher := &MockPasswordHasher{}
    mockEmail := &MockEmailSender{}

    useCase := usecase.NewUserUseCase(mockRepo, mockHasher, mockEmail)

    user, err := useCase.RegisterUser(context.Background(), "Иван", "ivan@example.com", "password123")

    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }

    if user.Name != "Иван" {
        t.Errorf("expected name Иван, got %s", user.Name)
    }
}
```

### ✅ Независимость от фреймворков

```go
// Легко меняем Gorilla Mux на Chi
r := chi.NewRouter()
r.Post("/users", userHandler.RegisterUser)
r.Get("/users/{id}", userHandler.GetUser)

// Или на Gin
r := gin.Default()
r.POST("/users", ginAdapter(userHandler.RegisterUser))
r.GET("/users/:id", ginAdapter(userHandler.GetUser))
```

### ✅ Независимость от БД

```go
// Меняем Postgres на MongoDB
mongoRepo := repository.NewMongoUserRepository(mongoClient)
userUseCase := usecase.NewUserUseCase(mongoRepo, hasher, emailSender)
// Use Case остается неизменным!
```

## Best Practices

### ✅ Правильно

```go
// Интерфейсы в use case слое
type UserRepository interface {
    Create(ctx context.Context, user *domain.User) error
}

// Зависимости передаются через конструктор
func NewUserUseCase(repo UserRepository) *UserUseCase

// Доменные сущности содержат бизнес-правила
func (u *User) CanBeSuspended() bool {
    return time.Since(u.CreatedAt) > 30*24*time.Hour
}
```

### ❌ Неправильно

```go
// Use Case зависит от конкретной реализации
type UserUseCase struct {
    db *sql.DB // Нарушение Dependency Rule!
}

// Бизнес-логика в handler
func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    // Валидация, хеширование, бизнес-правила в handler - плохо!
}

// Доменная модель зависит от БД
type User struct {
    gorm.Model // Нарушение! Entities не должны знать о БД
    Name string
}
```

## Вопросы с собеседований

**Вопрос:** В чем главное отличие Clean Architecture от обычной трехслойной архитектуры?

**Ответ:** В Clean Architecture зависимости направлены внутрь (к бизнес-логике), а в трехслойной — сверху вниз (UI→BL→DB). Clean Architecture делает бизнес-логику независимой от внешних деталей через Dependency Inversion.

**Вопрос:** Где должны находиться интерфейсы Repository в Clean Architecture?

**Ответ:** Интерфейсы Repository должны находиться в слое Use Cases (внутренний слой), а их реализации — во внешнем слое (Adapters). Это следует Dependency Inversion Principle.

**Вопрос:** Можно ли использовать GORM в Domain layer?

**Ответ:** Нет. Domain layer не должен зависеть от фреймворков и библиотек. GORM используется только в Repository (adapter layer), а Domain содержит чистые Go структуры.

## Связанные темы

- [[SOLID принципы]]
- [[Repository паттерн]]
- [[Паттерны проектирования в Go]]
- [[Go - Интерфейсы]]
- [[Микросервисная архитектура]]