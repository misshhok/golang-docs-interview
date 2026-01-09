# Repository паттерн

Repository паттерн — это абстракция для работы с данными, которая инкапсулирует логику доступа к источнику данных (БД, API, файлы). Он отделяет бизнес-логику от деталей хранения данных.

## Основная идея

Repository действует как коллекция объектов в памяти, скрывая детали работы с базой данных, SQL запросами или любым другим источником данных.

### Преимущества

✅ **Разделение ответственности** — бизнес-логика не знает о деталях БД
✅ **Легкое тестирование** — можно заменить на mock репозиторий
✅ **Переиспользование** — общие запросы в одном месте
✅ **Смена источника данных** — изменения только в репозитории

## Базовая реализация

### Интерфейс Repository

```go
package repository

import "context"

// User - доменная модель
type User struct {
    ID        int
    Name      string
    Email     string
    CreatedAt time.Time
}

// UserRepository - интерфейс для работы с пользователями
type UserRepository interface {
    // CRUD операции
    Create(ctx context.Context, user *User) error
    GetByID(ctx context.Context, id int) (*User, error)
    GetByEmail(ctx context.Context, email string) (*User, error)
    GetAll(ctx context.Context) ([]*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id int) error

    // Дополнительные методы
    FindByName(ctx context.Context, name string) ([]*User, error)
    Count(ctx context.Context) (int, error)
}
```

### Реализация с PostgreSQL

```go
package repository

import (
    "context"
    "database/sql"
    "fmt"
)

// PostgresUserRepository - реализация для PostgreSQL
type PostgresUserRepository struct {
    db *sql.DB
}

// NewPostgresUserRepository - конструктор
func NewPostgresUserRepository(db *sql.DB) UserRepository {
    return &PostgresUserRepository{db: db}
}

func (r *PostgresUserRepository) Create(ctx context.Context, user *User) error {
    query := `
        INSERT INTO users (name, email, created_at)
        VALUES ($1, $2, $3)
        RETURNING id
    `

    err := r.db.QueryRowContext(ctx, query, user.Name, user.Email, time.Now()).
        Scan(&user.ID)

    if err != nil {
        return fmt.Errorf("failed to create user: %w", err)
    }

    return nil
}

func (r *PostgresUserRepository) GetByID(ctx context.Context, id int) (*User, error) {
    query := `
        SELECT id, name, email, created_at
        FROM users
        WHERE id = $1
    `

    user := &User{}
    err := r.db.QueryRowContext(ctx, query, id).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.CreatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("user not found")
    }

    if err != nil {
        return nil, fmt.Errorf("failed to get user: %w", err)
    }

    return user, nil
}

func (r *PostgresUserRepository) GetByEmail(ctx context.Context, email string) (*User, error) {
    query := `
        SELECT id, name, email, created_at
        FROM users
        WHERE email = $1
    `

    user := &User{}
    err := r.db.QueryRowContext(ctx, query, email).Scan(
        &user.ID,
        &user.Name,
        &user.Email,
        &user.CreatedAt,
    )

    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("user not found")
    }

    if err != nil {
        return nil, fmt.Errorf("failed to get user: %w", err)
    }

    return user, nil
}

func (r *PostgresUserRepository) GetAll(ctx context.Context) ([]*User, error) {
    query := `
        SELECT id, name, email, created_at
        FROM users
        ORDER BY created_at DESC
    `

    rows, err := r.db.QueryContext(ctx, query)
    if err != nil {
        return nil, fmt.Errorf("failed to query users: %w", err)
    }
    defer rows.Close()

    var users []*User
    for rows.Next() {
        user := &User{}
        err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
        if err != nil {
            return nil, fmt.Errorf("failed to scan user: %w", err)
        }
        users = append(users, user)
    }

    return users, nil
}

func (r *PostgresUserRepository) Update(ctx context.Context, user *User) error {
    query := `
        UPDATE users
        SET name = $1, email = $2
        WHERE id = $3
    `

    result, err := r.db.ExecContext(ctx, query, user.Name, user.Email, user.ID)
    if err != nil {
        return fmt.Errorf("failed to update user: %w", err)
    }

    rows, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("failed to get rows affected: %w", err)
    }

    if rows == 0 {
        return fmt.Errorf("user not found")
    }

    return nil
}

func (r *PostgresUserRepository) Delete(ctx context.Context, id int) error {
    query := `DELETE FROM users WHERE id = $1`

    result, err := r.db.ExecContext(ctx, query, id)
    if err != nil {
        return fmt.Errorf("failed to delete user: %w", err)
    }

    rows, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("failed to get rows affected: %w", err)
    }

    if rows == 0 {
        return fmt.Errorf("user not found")
    }

    return nil
}

func (r *PostgresUserRepository) FindByName(ctx context.Context, name string) ([]*User, error) {
    query := `
        SELECT id, name, email, created_at
        FROM users
        WHERE name ILIKE $1
    `

    rows, err := r.db.QueryContext(ctx, query, "%"+name+"%")
    if err != nil {
        return nil, fmt.Errorf("failed to query users: %w", err)
    }
    defer rows.Close()

    var users []*User
    for rows.Next() {
        user := &User{}
        err := rows.Scan(&user.ID, &user.Name, &user.Email, &user.CreatedAt)
        if err != nil {
            return nil, fmt.Errorf("failed to scan user: %w", err)
        }
        users = append(users, user)
    }

    return users, nil
}

func (r *PostgresUserRepository) Count(ctx context.Context) (int, error) {
    query := `SELECT COUNT(*) FROM users`

    var count int
    err := r.db.QueryRowContext(ctx, query).Scan(&count)
    if err != nil {
        return 0, fmt.Errorf("failed to count users: %w", err)
    }

    return count, nil
}
```

## Использование Repository

### В сервисном слое

```go
package service

import (
    "context"
    "fmt"
    "myapp/repository"
)

// UserService - бизнес-логика для работы с пользователями
type UserService struct {
    repo repository.UserRepository
}

func NewUserService(repo repository.UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) RegisterUser(ctx context.Context, name, email string) (*repository.User, error) {
    // Проверка на существование email
    existing, _ := s.repo.GetByEmail(ctx, email)
    if existing != nil {
        return nil, fmt.Errorf("email already exists")
    }

    // Бизнес-логика валидации
    if len(name) < 2 {
        return nil, fmt.Errorf("name too short")
    }

    // Создание пользователя
    user := &repository.User{
        Name:  name,
        Email: email,
    }

    err := s.repo.Create(ctx, user)
    if err != nil {
        return nil, err
    }

    // Отправка welcome email...

    return user, nil
}

func (s *UserService) GetUserProfile(ctx context.Context, id int) (*repository.User, error) {
    return s.repo.GetByID(ctx, id)
}

func (s *UserService) UpdateUserProfile(ctx context.Context, id int, name, email string) error {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return err
    }

    user.Name = name
    user.Email = email

    return s.repo.Update(ctx, user)
}
```

### В HTTP хендлере

```go
package handler

import (
    "encoding/json"
    "net/http"
    "myapp/service"
    "strconv"
)

type UserHandler struct {
    service *service.UserService
}

func NewUserHandler(service *service.UserService) *UserHandler {
    return &UserHandler{service: service}
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }

    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    user, err := h.service.RegisterUser(r.Context(), req.Name, req.Email)
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteStatus(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    idStr := r.URL.Query().Get("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        http.Error(w, "invalid id", http.StatusBadRequest)
        return
    }

    user, err := h.service.GetUserProfile(r.Context(), id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

## Mock Repository для тестирования

```go
package repository

import "context"

// MockUserRepository - mock для тестирования
type MockUserRepository struct {
    users map[int]*User
    nextID int
}

func NewMockUserRepository() *MockUserRepository {
    return &MockUserRepository{
        users:  make(map[int]*User),
        nextID: 1,
    }
}

func (m *MockUserRepository) Create(ctx context.Context, user *User) error {
    user.ID = m.nextID
    user.CreatedAt = time.Now()
    m.users[user.ID] = user
    m.nextID++
    return nil
}

func (m *MockUserRepository) GetByID(ctx context.Context, id int) (*User, error) {
    user, ok := m.users[id]
    if !ok {
        return nil, fmt.Errorf("user not found")
    }
    return user, nil
}

func (m *MockUserRepository) GetByEmail(ctx context.Context, email string) (*User, error) {
    for _, user := range m.users {
        if user.Email == email {
            return user, nil
        }
    }
    return nil, fmt.Errorf("user not found")
}

// ... остальные методы
```

### Использование mock в тестах

```go
package service_test

import (
    "context"
    "testing"
    "myapp/repository"
    "myapp/service"
)

func TestRegisterUser(t *testing.T) {
    // Создаем mock repository
    mockRepo := repository.NewMockUserRepository()
    userService := service.NewUserService(mockRepo)

    ctx := context.Background()

    // Тест успешной регистрации
    user, err := userService.RegisterUser(ctx, "Иван", "ivan@example.com")
    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }

    if user.Name != "Иван" {
        t.Errorf("expected name Иван, got %s", user.Name)
    }

    // Тест дубликата email
    _, err = userService.RegisterUser(ctx, "Петр", "ivan@example.com")
    if err == nil {
        t.Error("expected error for duplicate email")
    }

    // Тест короткого имени
    _, err = userService.RegisterUser(ctx, "A", "a@example.com")
    if err == nil {
        t.Error("expected error for short name")
    }
}
```

## Реализация с GORM

```go
package repository

import (
    "context"
    "fmt"
    "gorm.io/gorm"
)

type GORMUserRepository struct {
    db *gorm.DB
}

func NewGORMUserRepository(db *gorm.DB) UserRepository {
    return &GORMUserRepository{db: db}
}

func (r *GORMUserRepository) Create(ctx context.Context, user *User) error {
    result := r.db.WithContext(ctx).Create(user)
    return result.Error
}

func (r *GORMUserRepository) GetByID(ctx context.Context, id int) (*User, error) {
    var user User
    result := r.db.WithContext(ctx).First(&user, id)

    if result.Error == gorm.ErrRecordNotFound {
        return nil, fmt.Errorf("user not found")
    }

    return &user, result.Error
}

func (r *GORMUserRepository) GetByEmail(ctx context.Context, email string) (*User, error) {
    var user User
    result := r.db.WithContext(ctx).Where("email = ?", email).First(&user)

    if result.Error == gorm.ErrRecordNotFound {
        return nil, fmt.Errorf("user not found")
    }

    return &user, result.Error
}

func (r *GORMUserRepository) GetAll(ctx context.Context) ([]*User, error) {
    var users []*User
    result := r.db.WithContext(ctx).Order("created_at DESC").Find(&users)
    return users, result.Error
}

func (r *GORMUserRepository) Update(ctx context.Context, user *User) error {
    result := r.db.WithContext(ctx).Save(user)
    return result.Error
}

func (r *GORMUserRepository) Delete(ctx context.Context, id int) error {
    result := r.db.WithContext(ctx).Delete(&User{}, id)

    if result.RowsAffected == 0 {
        return fmt.Errorf("user not found")
    }

    return result.Error
}
```

## Расширенные возможности

### Specification pattern для фильтрации

```go
// Specification для динамических фильтров
type UserSpecification func(*gorm.DB) *gorm.DB

func WithName(name string) UserSpecification {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("name ILIKE ?", "%"+name+"%")
    }
}

func WithMinAge(age int) UserSpecification {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("age >= ?", age)
    }
}

func (r *GORMUserRepository) FindBySpec(ctx context.Context, specs ...UserSpecification) ([]*User, error) {
    query := r.db.WithContext(ctx)

    for _, spec := range specs {
        query = spec(query)
    }

    var users []*User
    result := query.Find(&users)
    return users, result.Error
}

// Использование
users, err := repo.FindBySpec(ctx,
    WithName("Иван"),
    WithMinAge(18),
)
```

## Best Practices

### ✅ Правильно

```go
// Интерфейс в том же пакете, где используется (service)
// Реализация в отдельном пакете (repository)

// Использовать context для отмены операций
func (r *Repository) GetByID(ctx context.Context, id int) (*User, error)

// Возвращать доменные ошибки, а не sql.ErrNoRows
if err == sql.ErrNoRows {
    return nil, ErrUserNotFound
}

// Использовать транзакции через интерфейс
type Repository interface {
    WithTx(tx *sql.Tx) Repository
    // ...
}
```

### ❌ Неправильно

```go
// Не возвращать *sql.Row, *gorm.DB из репозитория
func (r *Repository) GetUser(id int) *sql.Row // Плохо!

// Не делать репозиторий зависимым от HTTP слоя
func (r *Repository) CreateUser(r *http.Request) // Плохо!

// Не смешивать бизнес-логику с доступом к данным
func (r *Repository) RegisterUserAndSendEmail() // Плохо!
```

## Вопросы с собеседований

**Вопрос:** В чем разница между Repository и DAO (Data Access Object)?

**Ответ:** Repository работает с коллекцией объектов и использует язык предметной области, DAO — более низкоуровневый паттерн, который просто предоставляет CRUD операции. Repository обычно включает более сложную логику фильтрации и может работать с агрегатами.

**Вопрос:** Где должен находиться интерфейс Repository?

**Ответ:** В Go принято размещать интерфейс в том пакете, где он используется (обычно в service layer), а реализацию — в отдельном пакете (repository). Это следует принципу Dependency Inversion.

**Вопрос:** Как тестировать код, использующий Repository?

**Ответ:** Создать mock реализацию интерфейса Repository (in-memory или через gomock/testify), заменить реальный репозиторий на mock в тестах. Это позволяет тестировать бизнес-логику без доступа к реальной БД.

## Связанные темы

- [[Go - Интерфейсы]]
- [[GORM - ORM для Go]]
- [[sqlx]]
- [[SOLID принципы]]
- [[Clean Architecture]]
- [[Go - Моки и стабы]]