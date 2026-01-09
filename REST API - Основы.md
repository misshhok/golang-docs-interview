# REST API - Основы

REST (Representational State Transfer) - архитектурный стиль для построения web API на основе HTTP протокола.

## Ключевые принципы REST

1. **Stateless** - каждый запрос независим, сервер не хранит состояние клиента
2. **Client-Server** - разделение ответственности
3. **Cacheable** - ответы могут кэшироваться
4. **Uniform Interface** - единообразный интерфейс
5. **Layered System** - клиент не знает о промежуточных слоях
6. **Resource-Based** - все взаимодействие через ресурсы

## HTTP Methods

| Метод | Назначение | Идемпотентный | Safe |
|-------|-----------|---------------|------|
| GET | Получение ресурса | ✅ | ✅ |
| POST | Создание ресурса | ❌ | ❌ |
| PUT | Полное обновление | ✅ | ❌ |
| PATCH | Частичное обновление | ❌ | ❌ |
| DELETE | Удаление | ✅ | ❌ |
| HEAD | Получение headers | ✅ | ✅ |
| OPTIONS | Возможные методы | ✅ | ✅ |

**Идемпотентный**: повторные вызовы дают тот же результат
**Safe**: не изменяет данные на сервере

## URL структура

```
https://api.example.com/v1/users/123/orders/456

Scheme: https
Host: api.example.com
Version: v1
Resource: users
ID: 123
Sub-resource: orders
Sub-ID: 456
```

### Best practices для URL

```
✅ /users              - Коллекция (множественное число)
✅ /users/123          - Конкретный ресурс
✅ /users/123/orders   - Вложенный ресурс

❌ /getUser            - Глагол в URL
❌ /user               - Единственное число
❌ /users/delete/123   - Операция в URL
```

## HTTP Status Codes

### 2xx - Success

- **200 OK** - Успешный запрос (GET, PUT, PATCH)
- **201 Created** - Ресурс создан (POST)
- **204 No Content** - Успех без тела ответа (DELETE)

### 3xx - Redirection

- **301 Moved Permanently** - Постоянный редирект
- **304 Not Modified** - Ресурс не изменился (кэш)

### 4xx - Client Error

- **400 Bad Request** - Некорректный запрос
- **401 Unauthorized** - Требуется аутентификация
- **403 Forbidden** - Доступ запрещен
- **404 Not Found** - Ресурс не найден
- **409 Conflict** - Конфликт (например, дубликат email)
- **422 Unprocessable Entity** - Ошибка валидации
- **429 Too Many Requests** - Rate limit

### 5xx - Server Error

- **500 Internal Server Error** - Ошибка сервера
- **502 Bad Gateway** - Ошибка прокси
- **503 Service Unavailable** - Сервис недоступен

## REST API в Go

### Базовый сервер с net/http

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// GET /users
func getUsers(w http.ResponseWriter, r *http.Request) {
    users := []User{
        {ID: 1, Name: "John", Email: "john@example.com"},
        {ID: 2, Name: "Jane", Email: "jane@example.com"},
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

// POST /users
func createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Сохранение в БД...
    user.ID = 123

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func main() {
    http.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
        switch r.Method {
        case http.MethodGet:
            getUsers(w, r)
        case http.MethodPost:
            createUser(w, r)
        default:
            http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        }
    })

    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Подробнее: [[Go - Простой web-сервер]]

### Gorilla Mux

```go
import (
    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()

    // Routes
    r.HandleFunc("/users", getUsers).Methods("GET")
    r.HandleFunc("/users", createUser).Methods("POST")
    r.HandleFunc("/users/{id}", getUser).Methods("GET")
    r.HandleFunc("/users/{id}", updateUser).Methods("PUT")
    r.HandleFunc("/users/{id}", deleteUser).Methods("DELETE")

    // Path parameters
    r.HandleFunc("/users/{id}/orders", getUserOrders).Methods("GET")

    // Query parameters: /users?page=1&limit=10
    r.HandleFunc("/users", getUsersWithPagination).Methods("GET")

    log.Fatal(http.ListenAndServe(":8080", r))
}

func getUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID := vars["id"]

    // Получение пользователя...
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}
```

Подробнее: [[Gorilla Mux]]

### Gin Framework

```go
import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()

    // Routes
    r.GET("/users", getUsers)
    r.POST("/users", createUser)
    r.GET("/users/:id", getUser)
    r.PUT("/users/:id", updateUser)
    r.DELETE("/users/:id", deleteUser)

    r.Run(":8080")
}

func getUser(c *gin.Context) {
    userID := c.Param("id")

    user := User{ID: 1, Name: "John"}

    c.JSON(http.StatusOK, user)
}

func createUser(c *gin.Context) {
    var user User

    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Сохранение...
    c.JSON(http.StatusCreated, user)
}
```

Подробнее: [[Gin - Web фреймворк]]

## CRUD операции

### Create (POST)

```bash
POST /api/v1/users
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john@example.com"
}

# Response: 201 Created
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2026-01-09T10:00:00Z"
}
```

### Read (GET)

```bash
# Список
GET /api/v1/users?page=1&limit=10

# Response: 200 OK
{
  "data": [
    {"id": 1, "name": "John"},
    {"id": 2, "name": "Jane"}
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100
  }
}

# Один ресурс
GET /api/v1/users/123

# Response: 200 OK
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com"
}
```

### Update (PUT / PATCH)

```bash
# PUT - полное обновление
PUT /api/v1/users/123
{
  "name": "John Smith",
  "email": "john.smith@example.com"
}

# PATCH - частичное обновление
PATCH /api/v1/users/123
{
  "email": "newemail@example.com"
}

# Response: 200 OK
{
  "id": 123,
  "name": "John Smith",
  "email": "newemail@example.com"
}
```

### Delete (DELETE)

```bash
DELETE /api/v1/users/123

# Response: 204 No Content
(empty body)
```

## Pagination

```go
type PaginationParams struct {
    Page  int `json:"page"`
    Limit int `json:"limit"`
}

type PaginatedResponse struct {
    Data       []User           `json:"data"`
    Pagination PaginationInfo   `json:"pagination"`
}

type PaginationInfo struct {
    Page       int `json:"page"`
    Limit      int `json:"limit"`
    Total      int `json:"total"`
    TotalPages int `json:"total_pages"`
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    // Parse query params
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))

    if page < 1 {
        page = 1
    }
    if limit < 1 || limit > 100 {
        limit = 10
    }

    offset := (page - 1) * limit

    // Query DB
    users := getUsersFromDB(offset, limit)
    total := getTotalUsers()

    response := PaginatedResponse{
        Data: users,
        Pagination: PaginationInfo{
            Page:       page,
            Limit:      limit,
            Total:      total,
            TotalPages: (total + limit - 1) / limit,
        },
    }

    json.NewEncoder(w).Encode(response)
}
```

## Filtering и Sorting

```bash
# Filtering
GET /api/v1/users?age_min=18&age_max=65&city=Moscow

# Sorting
GET /api/v1/users?sort=name&order=asc
GET /api/v1/users?sort=-created_at  # Descending

# Combined
GET /api/v1/users?age_min=18&sort=name&page=1&limit=20
```

```go
func getUsers(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query()

    filters := map[string]string{
        "age_min": query.Get("age_min"),
        "age_max": query.Get("age_max"),
        "city":    query.Get("city"),
    }

    sort := query.Get("sort")
    order := query.Get("order")

    users := queryUsersWithFilters(filters, sort, order)
    json.NewEncoder(w).Encode(users)
}
```

## Error Responses

### Стандартный формат ошибки

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      },
      {
        "field": "age",
        "message": "Must be at least 18"
      }
    ]
  }
}
```

```go
type ErrorResponse struct {
    Error ErrorDetail `json:"error"`
}

type ErrorDetail struct {
    Code    string            `json:"code"`
    Message string            `json:"message"`
    Details []ValidationError `json:"details,omitempty"`
}

type ValidationError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func writeError(w http.ResponseWriter, status int, code, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)

    response := ErrorResponse{
        Error: ErrorDetail{
            Code:    code,
            Message: message,
        },
    }

    json.NewEncoder(w).Encode(response)
}
```

## Versioning

### URL Versioning

```
GET /api/v1/users
GET /api/v2/users
```

### Header Versioning

```
GET /api/users
Accept: application/vnd.myapi.v1+json
```

### Query Parameter

```
GET /api/users?version=1
```

**Рекомендуется**: URL versioning (проще для клиентов).

## CORS

```go
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/users", getUsers)

    handler := corsMiddleware(mux)
    http.ListenAndServe(":8080", handler)
}
```

## Authentication

### Bearer Token

```bash
GET /api/v1/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

```go
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")

        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Проверка токена...
        if !isValidToken(token) {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

Подробнее: [[JWT (JSON Web Tokens)]], [[OAuth 2.0]]

## Best Practices

### 1. Используйте правильные HTTP методы и статусы

```go
// ✅ Правильно
func deleteUser(w http.ResponseWriter, r *http.Request) {
    // DELETE /users/123
    if err := db.DeleteUser(id); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    w.WriteHeader(http.StatusNoContent) // 204
}

// ❌ Неправильно
func deleteUser(w http.ResponseWriter, r *http.Request) {
    // POST /users/delete/123 - глагол в URL!
    // Возвращает 200 вместо 204
}
```

### 2. Версионируйте API

```go
// ✅ С версией
r.HandleFunc("/api/v1/users", getUsers)

// ❌ Без версии (нельзя изменить без breaking changes)
r.HandleFunc("/api/users", getUsers)
```

### 3. Используйте pagination для больших списков

```go
// ✅ С pagination
GET /api/v1/users?page=1&limit=20

// ❌ Без pagination (может вернуть миллионы записей!)
GET /api/v1/users
```

### 4. Валидация входных данных

```go
func createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        writeError(w, http.StatusBadRequest, "INVALID_JSON", err.Error())
        return
    }

    // Валидация
    if user.Email == "" {
        writeError(w, http.StatusBadRequest, "VALIDATION_ERROR", "Email required")
        return
    }

    if !isValidEmail(user.Email) {
        writeError(w, http.StatusBadRequest, "VALIDATION_ERROR", "Invalid email")
        return
    }

    // Создание...
}
```

### 5. Idempotency для безопасности

```go
// PUT должен быть идемпотентным
func updateUser(w http.ResponseWriter, r *http.Request) {
    // Повторные вызовы с теми же данными дают тот же результат
    user.Name = newName
    db.Save(user)
}

// POST не идемпотентен
func createUser(w http.ResponseWriter, r *http.Request) {
    // Повторные вызовы создают разные ресурсы
    db.Insert(user)
}
```

## Связанные темы

- [[HTTP протокол]]
- [[Go - Простой web-сервер]]
- [[Go - Пакет net-http]]
- [[Gin - Web фреймворк]]
- [[Gorilla Mux]]
- [[JWT (JSON Web Tokens)]]
- [[OAuth 2.0]]
- [[REST API - Дизайн и best practices]]
- [[gRPC]] - альтернатива REST
