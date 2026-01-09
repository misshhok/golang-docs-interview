# Go - Моки и стабы

Моки (mocks) и стабы (stubs) — это тестовые заглушки, которые заменяют реальные зависимости в unit-тестах. Они позволяют изолировать тестируемый код от внешних систем (базы данных, API, файловая система) и делают тесты быстрыми и предсказуемыми.

## Основные концепции

### Mock vs Stub

**Stub (заглушка):**
- Возвращает заранее заданные значения
- Не проверяет, как именно его вызывали
- Используется для предоставления тестовых данных

**Mock (имитация):**
- Возвращает заданные значения
- Проверяет, что методы были вызваны правильно (количество вызовов, параметры)
- Используется для проверки взаимодействия между компонентами

```go
// Stub - просто возвращает данные
type StubUserRepository struct{}

func (s *StubUserRepository) GetUser(id int) (*User, error) {
    return &User{ID: id, Name: "Test User"}, nil
}

// Mock - проверяет вызовы
type MockUserRepository struct {
    GetUserCalled bool
    GetUserID     int
}

func (m *MockUserRepository) GetUser(id int) (*User, error) {
    m.GetUserCalled = true
    m.GetUserID = id
    return &User{ID: id, Name: "Test User"}, nil
}
```

## Тестируемость через интерфейсы

В Go моки создаются через интерфейсы. Код должен зависеть от интерфейсов, а не от конкретных реализаций.

### Плохой пример (без интерфейсов)

```go
// ❌ Сложно тестировать - жёсткая зависимость
type UserService struct {
    db *sql.DB  // конкретная реализация
}

func (s *UserService) GetUserName(id int) (string, error) {
    var name string
    err := s.db.QueryRow("SELECT name FROM users WHERE id = ?", id).Scan(&name)
    return name, err
}

// Тест потребует реальную БД!
```

### Хороший пример (с интерфейсами)

```go
// ✅ Легко тестировать - зависимость от интерфейса
type UserRepository interface {
    GetUser(id int) (*User, error)
}

type UserService struct {
    repo UserRepository  // интерфейс
}

func (s *UserService) GetUserName(id int) (string, error) {
    user, err := s.repo.GetUser(id)
    if err != nil {
        return "", err
    }
    return user.Name, nil
}

// Тест использует мок вместо реальной БД
```

## Ручные моки

### Пример: Мокирование репозитория

```go
type User struct {
    ID    int
    Name  string
    Email string
}

// Интерфейс репозитория
type UserRepository interface {
    GetUser(id int) (*User, error)
    SaveUser(user *User) error
}

// Реальная реализация (использует БД)
type PostgresUserRepository struct {
    db *sql.DB
}

func (r *PostgresUserRepository) GetUser(id int) (*User, error) {
    // Реальный запрос к PostgreSQL
    // ...
    return nil, nil
}

// Мок для тестов
type MockUserRepository struct {
    GetUserFunc func(id int) (*User, error)
    SaveUserFunc func(user *User) error

    GetUserCalls []int
    SaveUserCalls []*User
}

func (m *MockUserRepository) GetUser(id int) (*User, error) {
    m.GetUserCalls = append(m.GetUserCalls, id)
    if m.GetUserFunc != nil {
        return m.GetUserFunc(id)
    }
    return &User{ID: id, Name: "Mock User"}, nil
}

func (m *MockUserRepository) SaveUser(user *User) error {
    m.SaveUserCalls = append(m.SaveUserCalls, user)
    if m.SaveUserFunc != nil {
        return m.SaveUserFunc(user)
    }
    return nil
}
```

### Использование в тесте

```go
func TestUserService_GetUserName(t *testing.T) {
    // Создаём мок
    mockRepo := &MockUserRepository{
        GetUserFunc: func(id int) (*User, error) {
            return &User{ID: id, Name: "Alice", Email: "alice@example.com"}, nil
        },
    }

    // Создаём сервис с моком
    service := &UserService{repo: mockRepo}

    // Тестируем
    name, err := service.GetUserName(1)

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if name != "Alice" {
        t.Errorf("got %q, want %q", name, "Alice")
    }

    // Проверяем, что метод был вызван
    if len(mockRepo.GetUserCalls) != 1 {
        t.Errorf("GetUser called %d times, want 1", len(mockRepo.GetUserCalls))
    }

    if mockRepo.GetUserCalls[0] != 1 {
        t.Errorf("GetUser called with id %d, want 1", mockRepo.GetUserCalls[0])
    }
}

func TestUserService_GetUserName_Error(t *testing.T) {
    // Мок возвращает ошибку
    mockRepo := &MockUserRepository{
        GetUserFunc: func(id int) (*User, error) {
            return nil, errors.New("user not found")
        },
    }

    service := &UserService{repo: mockRepo}

    _, err := service.GetUserName(999)

    if err == nil {
        t.Error("expected error, got nil")
    }
}
```

## Библиотеки для мокирования

### 1. gomock (официальная библиотека от Google)

Генерирует моки из интерфейсов автоматически.

**Установка:**
```bash
go install github.com/golang/mock/mockgen@latest
```

**Генерация мока:**
```bash
mockgen -source=user_repository.go -destination=mock_user_repository.go -package=mocks
```

**Использование:**
```go
import (
    "testing"
    "github.com/golang/mock/gomock"
)

func TestWithGomock(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    // Создаём мок
    mockRepo := mocks.NewMockUserRepository(ctrl)

    // Настраиваем ожидания
    mockRepo.EXPECT().
        GetUser(1).
        Return(&User{ID: 1, Name: "Bob"}, nil).
        Times(1)  // ожидаем один вызов

    // Используем мок
    service := &UserService{repo: mockRepo}
    name, err := service.GetUserName(1)

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if name != "Bob" {
        t.Errorf("got %q, want %q", name, "Bob")
    }

    // gomock автоматически проверит, что GetUser был вызван 1 раз
}
```

### 2. testify/mock

Популярная библиотека с удобным API.

**Установка:**
```bash
go get github.com/stretchr/testify/mock
```

**Создание мока:**
```go
import "github.com/stretchr/testify/mock"

type MockUserRepository struct {
    mock.Mock
}

func (m *MockUserRepository) GetUser(id int) (*User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func (m *MockUserRepository) SaveUser(user *User) error {
    args := m.Called(user)
    return args.Error(0)
}
```

**Использование:**
```go
func TestWithTestify(t *testing.T) {
    // Создаём мок
    mockRepo := new(MockUserRepository)

    // Настраиваем ожидания
    mockRepo.On("GetUser", 1).Return(&User{ID: 1, Name: "Charlie"}, nil)

    // Используем мок
    service := &UserService{repo: mockRepo}
    name, err := service.GetUserName(1)

    // Проверяем результат
    assert.NoError(t, err)
    assert.Equal(t, "Charlie", name)

    // Проверяем, что ожидания выполнены
    mockRepo.AssertExpectations(t)
}
```

## Dependency Injection

Для возможности подставить мок, нужно передавать зависимости извне.

### Через конструктор (рекомендуется)

```go
type UserService struct {
    repo UserRepository
}

// NewUserService - конструктор с зависимостью
func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

// В продакшене
func main() {
    db := connectToDatabase()
    repo := &PostgresUserRepository{db: db}
    service := NewUserService(repo)
    // ...
}

// В тестах
func TestUserService(t *testing.T) {
    mockRepo := &MockUserRepository{}
    service := NewUserService(mockRepo)  // подставляем мок
    // ...
}
```

### Через метод Set (для опциональных зависимостей)

```go
type NotificationService struct {
    emailSender EmailSender
    smsSender   SMSSender
}

func (s *NotificationService) SetEmailSender(sender EmailSender) {
    s.emailSender = sender
}

func (s *NotificationService) SetSMSSender(sender SMSSender) {
    s.smsSender = sender
}
```

## Практический пример: HTTP клиент

```go
// Интерфейс для HTTP запросов
type HTTPClient interface {
    Get(url string) (*http.Response, error)
}

// Сервис, использующий HTTP клиент
type WeatherService struct {
    client HTTPClient
}

func (s *WeatherService) GetTemperature(city string) (float64, error) {
    url := fmt.Sprintf("https://api.weather.com/city/%s", city)
    resp, err := s.client.Get(url)
    if err != nil {
        return 0, err
    }
    defer resp.Body.Close()

    var result struct {
        Temp float64 `json:"temperature"`
    }
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return 0, err
    }

    return result.Temp, nil
}

// Мок HTTP клиента
type MockHTTPClient struct {
    GetFunc func(url string) (*http.Response, error)
}

func (m *MockHTTPClient) Get(url string) (*http.Response, error) {
    if m.GetFunc != nil {
        return m.GetFunc(url)
    }
    return nil, errors.New("not implemented")
}

// Тест
func TestWeatherService_GetTemperature(t *testing.T) {
    mockClient := &MockHTTPClient{
        GetFunc: func(url string) (*http.Response, error) {
            // Возвращаем тестовый JSON
            json := `{"temperature": 25.5}`
            return &http.Response{
                StatusCode: 200,
                Body:       io.NopCloser(strings.NewReader(json)),
            }, nil
        },
    }

    service := &WeatherService{client: mockClient}
    temp, err := service.GetTemperature("Moscow")

    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }

    if temp != 25.5 {
        t.Errorf("got %v, want 25.5", temp)
    }
}
```

## Best Practices

1. ✅ **Проектируйте код с учётом тестируемости** - используйте интерфейсы
2. ✅ **Передавайте зависимости через конструктор** - делает зависимости явными
3. ✅ **Моки должны быть простыми** - не усложняйте логику моков
4. ✅ **Проверяйте взаимодействие, а не реализацию** - тестируйте контракт, а не детали
5. ✅ **Используйте table-driven тесты с моками** - легче покрыть разные сценарии
6. ❌ **Не мокайте всё подряд** - мокируйте только внешние зависимости

## Типичные ошибки

### Ошибка 1: Зависимость от конкретной реализации

```go
// ❌ Плохо
type Service struct {
    db *sql.DB  // нельзя подставить мок
}

// ✅ Хорошо
type Service struct {
    repo Repository  // можно подставить мок
}
```

### Ошибка 2: Слишком сложные моки

```go
// ❌ Плохо - мок содержит сложную логику
type ComplexMock struct {
    data map[int]*User
}

func (m *ComplexMock) GetUser(id int) (*User, error) {
    if user, ok := m.data[id]; ok {
        if user.Active {
            return user, nil
        }
        return nil, errors.New("user inactive")
    }
    return nil, errors.New("not found")
}

// ✅ Хорошо - мок простой, логика в тесте
type SimpleMock struct {
    GetUserFunc func(int) (*User, error)
}

func (m *SimpleMock) GetUser(id int) (*User, error) {
    return m.GetUserFunc(id)
}
```

## Вопросы с собеседований

**Вопрос:** В чём разница между mock и stub?

**Ответ:** Stub просто возвращает заданные значения и не проверяет, как его вызывали. Mock не только возвращает значения, но и проверяет, что методы были вызваны правильно (параметры, количество вызовов). Stub используется для предоставления данных, mock — для проверки взаимодействия.

**Вопрос:** Как сделать код тестируемым в Go?

**Ответ:** Главное — зависеть от интерфейсов, а не от конкретных реализаций. Используйте dependency injection (через конструктор или параметры методов). Это позволяет подставить мок вместо реальной зависимости в тестах.

**Вопрос:** Стоит ли мокировать всё в тестах?

**Ответ:** Нет. Мокируйте только внешние зависимости (БД, API, файловая система), которые делают тесты медленными или нестабильными. Внутренние компоненты вашего приложения лучше тестировать с реальными реализациями.

## Связанные темы

- [[Go - Интерфейсы]]
- [[Go - Unit тестирование]]
- [[Go - Table-driven тесты]]
- [[E2E тестирование]]