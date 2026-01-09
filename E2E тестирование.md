# E2E тестирование

End-to-End (E2E) тестирование — это проверка работы всего приложения целиком, от начала до конца, включая все интеграции с внешними системами (базы данных, API, очереди сообщений). В отличие от unit-тестов, E2E тесты проверяют реальные пользовательские сценарии.

## Unit vs Integration vs E2E

```
Unit тесты:
├─ Тестируют отдельные функции
├─ Используют моки для зависимостей
├─ Быстрые (миллисекунды)
└─ Много тестов

Integration тесты:
├─ Тестируют взаимодействие компонентов
├─ Используют реальные зависимости (БД, Redis)
├─ Средние по скорости (секунды)
└─ Среднее количество

E2E тесты:
├─ Тестируют полный пользовательский сценарий
├─ Используют реальное окружение
├─ Медленные (секунды/минуты)
└─ Мало тестов, только ключевые сценарии
```

## Тестирование HTTP API с httptest

Пакет `httptest` позволяет тестировать HTTP handlers без запуска реального сервера.

### Простой пример

```go
package main

import (
    "encoding/json"
    "net/http"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

// HTTP handler
func GetUserHandler(w http.ResponseWriter, r *http.Request) {
    user := User{ID: 1, Name: "Alice"}
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

### Тест с httptest

```go
package main

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestGetUserHandler(t *testing.T) {
    // Создаём тестовый HTTP запрос
    req := httptest.NewRequest(http.MethodGet, "/user/1", nil)

    // Создаём ResponseRecorder для записи ответа
    rr := httptest.NewRecorder()

    // Вызываем handler
    GetUserHandler(rr, req)

    // Проверяем статус код
    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v",
            status, http.StatusOK)
    }

    // Проверяем Content-Type
    expected := "application/json"
    if ct := rr.Header().Get("Content-Type"); ct != expected {
        t.Errorf("handler returned wrong content type: got %v want %v",
            ct, expected)
    }

    // Проверяем тело ответа
    var user User
    if err := json.NewDecoder(rr.Body).Decode(&user); err != nil {
        t.Fatalf("could not decode response: %v", err)
    }

    if user.ID != 1 {
        t.Errorf("expected user ID 1, got %d", user.ID)
    }

    if user.Name != "Alice" {
        t.Errorf("expected user name Alice, got %s", user.Name)
    }
}
```

### Тестирование с реальным сервером

```go
func TestGetUserHandler_WithServer(t *testing.T) {
    // Создаём тестовый HTTP сервер
    server := httptest.NewServer(http.HandlerFunc(GetUserHandler))
    defer server.Close()

    // Делаем реальный HTTP запрос
    resp, err := http.Get(server.URL + "/user/1")
    if err != nil {
        t.Fatalf("could not send GET request: %v", err)
    }
    defer resp.Body.Close()

    // Проверяем ответ
    if resp.StatusCode != http.StatusOK {
        t.Errorf("expected status 200, got %d", resp.StatusCode)
    }

    var user User
    if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
        t.Fatalf("could not decode response: %v", err)
    }

    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }
}
```

## Тестирование с реальной базой данных

Для полноценных E2E тестов нужна реальная база данных. Есть несколько подходов:

### 1. Тестовая БД с fixtures

```go
func setupTestDB(t *testing.T) *sql.DB {
    // Подключаемся к тестовой БД
    db, err := sql.Open("postgres",
        "postgres://user:pass@localhost:5432/testdb?sslmode=disable")
    if err != nil {
        t.Fatalf("could not connect to test db: %v", err)
    }

    // Очищаем таблицы
    _, err = db.Exec("TRUNCATE users, orders CASCADE")
    if err != nil {
        t.Fatalf("could not truncate tables: %v", err)
    }

    // Загружаем тестовые данные (fixtures)
    _, err = db.Exec(`
        INSERT INTO users (id, name, email) VALUES
        (1, 'Alice', 'alice@example.com'),
        (2, 'Bob', 'bob@example.com')
    `)
    if err != nil {
        t.Fatalf("could not insert fixtures: %v", err)
    }

    return db
}

func TestUserService_E2E(t *testing.T) {
    db := setupTestDB(t)
    defer db.Close()

    repo := NewPostgresUserRepository(db)
    service := NewUserService(repo)

    // Тестируем реальный сценарий
    user, err := service.GetUser(1)
    if err != nil {
        t.Fatalf("GetUser failed: %v", err)
    }

    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }
}
```

### 2. Testcontainers (рекомендуется)

Testcontainers автоматически запускает Docker контейнер с БД для тестов.

**Установка:**
```bash
go get github.com/testcontainers/testcontainers-go
go get github.com/testcontainers/testcontainers-go/modules/postgres
```

**Использование:**
```go
import (
    "context"
    "testing"
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/modules/postgres"
    "github.com/testcontainers/testcontainers-go/wait"
)

func setupPostgresContainer(t *testing.T) *sql.DB {
    ctx := context.Background()

    // Запускаем PostgreSQL в Docker контейнере
    postgresContainer, err := postgres.RunContainer(ctx,
        testcontainers.WithImage("postgres:15-alpine"),
        postgres.WithDatabase("testdb"),
        postgres.WithUsername("user"),
        postgres.WithPassword("password"),
        testcontainers.WithWaitStrategy(
            wait.ForLog("database system is ready to accept connections").
                WithOccurrence(2),
        ),
    )
    if err != nil {
        t.Fatalf("failed to start container: %v", err)
    }

    // Очистка после теста
    t.Cleanup(func() {
        if err := postgresContainer.Terminate(ctx); err != nil {
            t.Fatalf("failed to terminate container: %v", err)
        }
    })

    // Получаем connection string
    connStr, err := postgresContainer.ConnectionString(ctx, "sslmode=disable")
    if err != nil {
        t.Fatalf("failed to get connection string: %v", err)
    }

    // Подключаемся
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        t.Fatalf("failed to connect: %v", err)
    }

    // Создаём схему
    _, err = db.Exec(`
        CREATE TABLE users (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100),
            email VARCHAR(100)
        )
    `)
    if err != nil {
        t.Fatalf("failed to create schema: %v", err)
    }

    return db
}

func TestUserRepository_E2E(t *testing.T) {
    db := setupPostgresContainer(t)
    defer db.Close()

    repo := NewPostgresUserRepository(db)

    // Создаём пользователя
    user := &User{Name: "Charlie", Email: "charlie@example.com"}
    err := repo.SaveUser(user)
    if err != nil {
        t.Fatalf("SaveUser failed: %v", err)
    }

    // Получаем пользователя
    retrieved, err := repo.GetUser(user.ID)
    if err != nil {
        t.Fatalf("GetUser failed: %v", err)
    }

    // Проверяем
    if retrieved.Name != "Charlie" {
        t.Errorf("expected Charlie, got %s", retrieved.Name)
    }
}
```

## Полный E2E тест HTTP API + БД

```go
func TestCreateAndGetUser_E2E(t *testing.T) {
    // 1. Запускаем тестовую БД
    db := setupPostgresContainer(t)
    defer db.Close()

    // 2. Создаём приложение
    repo := NewPostgresUserRepository(db)
    service := NewUserService(repo)
    handler := NewUserHandler(service)

    // 3. Запускаем тестовый HTTP сервер
    server := httptest.NewServer(handler.Router())
    defer server.Close()

    // 4. Тестируем создание пользователя (POST)
    createPayload := `{"name": "David", "email": "david@example.com"}`
    resp, err := http.Post(
        server.URL+"/users",
        "application/json",
        strings.NewReader(createPayload),
    )
    if err != nil {
        t.Fatalf("POST request failed: %v", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusCreated {
        t.Fatalf("expected status 201, got %d", resp.StatusCode)
    }

    var created User
    if err := json.NewDecoder(resp.Body).Decode(&created); err != nil {
        t.Fatalf("could not decode response: %v", err)
    }

    // 5. Тестируем получение пользователя (GET)
    resp, err = http.Get(fmt.Sprintf("%s/users/%d", server.URL, created.ID))
    if err != nil {
        t.Fatalf("GET request failed: %v", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        t.Fatalf("expected status 200, got %d", resp.StatusCode)
    }

    var retrieved User
    if err := json.NewDecoder(resp.Body).Decode(&retrieved); err != nil {
        t.Fatalf("could not decode response: %v", err)
    }

    // 6. Проверяем, что данные совпадают
    if retrieved.ID != created.ID {
        t.Errorf("ID mismatch: got %d, want %d", retrieved.ID, created.ID)
    }

    if retrieved.Name != "David" {
        t.Errorf("Name mismatch: got %s, want David", retrieved.Name)
    }
}
```

## Fixtures и Seeds

Fixtures — это предварительно подготовленные тестовые данные.

### Загрузка из файла

```go
func loadFixtures(t *testing.T, db *sql.DB, filename string) {
    data, err := os.ReadFile(filename)
    if err != nil {
        t.Fatalf("could not read fixtures: %v", err)
    }

    _, err = db.Exec(string(data))
    if err != nil {
        t.Fatalf("could not load fixtures: %v", err)
    }
}

func TestWithFixtures(t *testing.T) {
    db := setupTestDB(t)
    defer db.Close()

    // Загружаем тестовые данные из SQL файла
    loadFixtures(t, db, "testdata/users.sql")

    // Тестируем с предзагруженными данными
    // ...
}
```

**testdata/users.sql:**
```sql
INSERT INTO users (id, name, email) VALUES
    (1, 'Alice', 'alice@example.com'),
    (2, 'Bob', 'bob@example.com'),
    (3, 'Charlie', 'charlie@example.com');
```

## Best Practices

1. ✅ **Изолируйте тесты** - каждый тест должен очищать за собой данные
2. ✅ **Используйте testcontainers** - не зависьте от внешней БД
3. ✅ **Fixtures в отдельных файлах** - проще поддерживать
4. ✅ **Тестируйте ключевые сценарии** - E2E тесты медленные, их должно быть немного
5. ✅ **Делайте E2E тесты независимыми** - они должны работать в любом порядке
6. ❌ **Не дублируйте unit-тесты в E2E** - разные уровни тестирования

## Структура тестов (Testing Pyramid)

```
         E2E
        /   \
       /     \
      / Integ \
     /  ration \
    /   Tests   \
   /_____________\
  /               \
 /   Unit Tests    \
/___________________\

Unit: 70% (быстро, много)
Integration: 20% (средне)
E2E: 10% (медленно, мало)
```

## Вопросы с собеседований

**Вопрос:** В чём разница между integration и E2E тестами?

**Ответ:** Integration тесты проверяют взаимодействие нескольких компонентов (например, сервис + БД). E2E тесты проверяют весь пользовательский сценарий от начала до конца, включая все слои приложения (HTTP → сервис → репозиторий → БД). E2E тесты ближе к реальному использованию.

**Вопрос:** Почему не стоит писать много E2E тестов?

**Ответ:** E2E тесты медленные (требуют реальной БД, HTTP запросов), хрупкие (могут ломаться из-за изменений в любой части системы) и сложны в отладке. Большинство багов можно поймать на уровне unit и integration тестов, которые быстрее и стабильнее.

**Вопрос:** Что такое testcontainers и зачем они нужны?

**Ответ:** Testcontainers — это библиотека, которая автоматически запускает Docker контейнеры (БД, Redis, Kafka и т.д.) для тестов. Это позволяет тестировать с реальными зависимостями без необходимости настройки внешней инфраструктуры. Контейнеры автоматически удаляются после тестов.

## Связанные темы

- [[Go - Unit тестирование]]
- [[Go - Моки и стабы]]
- [[Go - Пакет net-http]]
- [[PostgreSQL - Основы]]
- [[Test Coverage]]