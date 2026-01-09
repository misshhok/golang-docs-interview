# REST API - Дизайн и best practices

Лучшие практики проектирования REST API: именование, версионирование, обработка ошибок, пагинация, фильтрация и другие рекомендации.

## Именование ресурсов

### 1. Используйте существительные, не глаголы

```
❌ ПЛОХО:
GET  /getUsers
POST /createUser
PUT  /updateUser
DELETE /deleteUser

✅ ХОРОШО:
GET    /users       # Получить список пользователей
POST   /users       # Создать пользователя
GET    /users/{id}  # Получить пользователя
PUT    /users/{id}  # Обновить пользователя
DELETE /users/{id}  # Удалить пользователя
```

### 2. Используйте множественное число

```
❌ ПЛОХО:
/user
/product
/order

✅ ХОРОШО:
/users
/products
/orders
```

### 3. Вложенные ресурсы для отношений

```
GET  /users/{userId}/orders              # Заказы пользователя
GET  /users/{userId}/orders/{orderId}    # Конкретный заказ
POST /users/{userId}/orders              # Создать заказ для пользователя
GET  /orders/{orderId}/items             # Товары в заказе
```

### 4. Используйте kebab-case для URL

```
❌ ПЛОХО:
/userOrders
/user_orders

✅ ХОРОШО:
/user-orders
/order-items
```

## HTTP методы

### GET - Получение данных

```go
// Список с пагинацией
// GET /users?page=1&limit=20
func GetUsers(w http.ResponseWriter, r *http.Request) {
    page := r.URL.Query().Get("page")
    limit := r.URL.Query().Get("limit")

    users, err := userService.GetUsers(page, limit)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    json.NewEncoder(w).Encode(users)
}

// Один ресурс
// GET /users/123
func GetUser(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]

    user, err := userService.GetUser(id)
    if err != nil {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }

    json.NewEncoder(w).Encode(user)
}
```

**Свойства GET:**
- ✅ Идемпотентный (можно вызывать многократно)
- ✅ Безопасный (не изменяет данные)
- ✅ Кэшируемый

### POST - Создание ресурса

```go
// POST /users
func CreateUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    // Валидация
    if err := validateUser(&user); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Создание
    createdUser, err := userService.CreateUser(&user)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Возвращаем 201 Created + Location header
    w.Header().Set("Location", fmt.Sprintf("/users/%d", createdUser.ID))
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(createdUser)
}
```

**Свойства POST:**
- ❌ НЕ идемпотентный (каждый вызов создает новый ресурс)
- ❌ НЕ безопасный (изменяет данные)

### PUT - Полное обновление

```go
// PUT /users/123 - заменяет ВСЕ поля
func UpdateUser(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]

    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    user.ID = id

    // Обновляем ВСЕ поля
    updatedUser, err := userService.UpdateUser(&user)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    json.NewEncoder(w).Encode(updatedUser)
}
```

**Свойства PUT:**
- ✅ Идемпотентный (повторный вызов с теми же данными дает тот же результат)
- ❌ НЕ безопасный

### PATCH - Частичное обновление

```go
// PATCH /users/123 - обновляет только указанные поля
func PatchUser(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]

    // Принимаем map для частичного обновления
    var updates map[string]interface{}
    if err := json.NewDecoder(r.Body).Decode(&updates); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    // Обновляем только указанные поля
    updatedUser, err := userService.PatchUser(id, updates)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    json.NewEncoder(w).Encode(updatedUser)
}
```

**Пример запроса:**

```bash
# PUT - заменяет ВСЕ поля
curl -X PUT /users/123 -d '{
  "name": "Alice",
  "email": "alice@example.com",
  "age": 30,
  "address": "Moscow"
}'

# PATCH - обновляет только name
curl -X PATCH /users/123 -d '{
  "name": "Alice Updated"
}'
# Остальные поля (email, age, address) остаются неизменными
```

### DELETE - Удаление

```go
// DELETE /users/123
func DeleteUser(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]

    err := userService.DeleteUser(id)
    if err != nil {
        http.Error(w, err.Error(), http.StatusNotFound)
        return
    }

    // 204 No Content - успешное удаление без тела ответа
    w.WriteHeader(http.StatusNoContent)
}
```

**Свойства DELETE:**
- ✅ Идемпотентный (повторное удаление того же ресурса безопасно)
- ❌ НЕ безопасный

## Коды ответов (Status Codes)

### Успешные (2xx)

```go
// 200 OK - стандартный успешный ответ
w.WriteHeader(http.StatusOK)
json.NewEncoder(w).Encode(user)

// 201 Created - ресурс создан
w.Header().Set("Location", "/users/123")
w.WriteHeader(http.StatusCreated)
json.NewEncoder(w).Encode(user)

// 202 Accepted - запрос принят, но обработка еще не завершена (асинхронные операции)
w.WriteHeader(http.StatusAccepted)
json.NewEncoder(w).Encode(map[string]string{"job_id": "abc123"})

// 204 No Content - успех без тела ответа (обычно DELETE)
w.WriteHeader(http.StatusNoContent)
```

### Клиентские ошибки (4xx)

```go
// 400 Bad Request - невалидный запрос
if err := validateUser(&user); err != nil {
    w.WriteHeader(http.StatusBadRequest)
    json.NewEncoder(w).Encode(ErrorResponse{
        Error: "Validation failed",
        Details: err.Error(),
    })
    return
}

// 401 Unauthorized - не авторизован (нет/невалидный токен)
token := r.Header.Get("Authorization")
if token == "" {
    w.WriteHeader(http.StatusUnauthorized)
    json.NewEncoder(w).Encode(ErrorResponse{Error: "Authentication required"})
    return
}

// 403 Forbidden - авторизован, но нет прав
if !canAccessResource(user, resourceID) {
    w.WriteHeader(http.StatusForbidden)
    json.NewEncoder(w).Encode(ErrorResponse{Error: "Access denied"})
    return
}

// 404 Not Found - ресурс не найден
user, err := userService.GetUser(id)
if err != nil {
    w.WriteHeader(http.StatusNotFound)
    json.NewEncoder(w).Encode(ErrorResponse{Error: "User not found"})
    return
}

// 409 Conflict - конфликт (например, email уже существует)
if userService.EmailExists(user.Email) {
    w.WriteHeader(http.StatusConflict)
    json.NewEncoder(w).Encode(ErrorResponse{Error: "Email already registered"})
    return
}

// 422 Unprocessable Entity - запрос понятен, но невалидные данные
if user.Age < 0 || user.Age > 150 {
    w.WriteHeader(http.StatusUnprocessableEntity)
    json.NewEncoder(w).Encode(ErrorResponse{Error: "Invalid age"})
    return
}

// 429 Too Many Requests - rate limit exceeded
w.WriteHeader(http.StatusTooManyRequests)
w.Header().Set("Retry-After", "60")
json.NewEncoder(w).Encode(ErrorResponse{Error: "Rate limit exceeded"})
```

### Серверные ошибки (5xx)

```go
// 500 Internal Server Error - общая ошибка сервера
if err != nil {
    log.Error(err)
    w.WriteHeader(http.StatusInternalServerError)
    json.NewEncoder(w).Encode(ErrorResponse{Error: "Internal server error"})
    return
}

// 503 Service Unavailable - сервис временно недоступен
w.WriteHeader(http.StatusServiceUnavailable)
w.Header().Set("Retry-After", "120")
json.NewEncoder(w).Encode(ErrorResponse{Error: "Service under maintenance"})
```

## Обработка ошибок

### Стандартизированный формат ошибок

```go
type ErrorResponse struct {
    Error   string            `json:"error"`           // Краткое описание
    Message string            `json:"message"`         // Подробное сообщение
    Code    string            `json:"code,omitempty"`  // Код ошибки (опционально)
    Details map[string]string `json:"details,omitempty"` // Детали (опционально)
}

// Пример ответа
{
    "error": "Validation failed",
    "message": "The request contains invalid data",
    "code": "VALIDATION_ERROR",
    "details": {
        "email": "Email is required",
        "age": "Age must be between 0 and 150"
    }
}
```

### Middleware для обработки ошибок

```go
type AppError struct {
    StatusCode int
    Error      error
    Message    string
}

type AppHandler func(w http.ResponseWriter, r *http.Request) *AppError

func (fn AppHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := fn(w, r); err != nil {
        w.WriteHeader(err.StatusCode)
        json.NewEncoder(w).Encode(ErrorResponse{
            Error:   http.StatusText(err.StatusCode),
            Message: err.Message,
        })
        log.Error(err.Error)
    }
}

// Использование
func GetUser(w http.ResponseWriter, r *http.Request) *AppError {
    id := mux.Vars(r)["id"]

    user, err := userService.GetUser(id)
    if err != nil {
        return &AppError{
            StatusCode: http.StatusNotFound,
            Error:      err,
            Message:    "User not found",
        }
    }

    json.NewEncoder(w).Encode(user)
    return nil
}

// Регистрация
http.Handle("/users/{id}", AppHandler(GetUser))
```

## Пагинация

### Cursor-based Pagination (рекомендуется)

```go
// GET /users?cursor=abc123&limit=20

type PaginatedResponse struct {
    Data       []User  `json:"data"`
    NextCursor *string `json:"next_cursor"`
    HasMore    bool    `json:"has_more"`
}

func GetUsers(w http.ResponseWriter, r *http.Request) {
    cursor := r.URL.Query().Get("cursor")
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    if limit == 0 {
        limit = 20
    }

    users, nextCursor, hasMore := userService.GetUsers(cursor, limit)

    response := PaginatedResponse{
        Data:       users,
        NextCursor: nextCursor,
        HasMore:    hasMore,
    }

    json.NewEncoder(w).Encode(response)
}
```

**Пример ответа:**

```json
{
    "data": [
        {"id": 1, "name": "Alice"},
        {"id": 2, "name": "Bob"}
    ],
    "next_cursor": "eyJpZCI6Mn0=",
    "has_more": true
}
```

**Следующий запрос:**

```
GET /users?cursor=eyJpZCI6Mn0=&limit=20
```

### Offset-based Pagination (для простых случаев)

```go
// GET /users?page=2&limit=20

type PaginatedResponse struct {
    Data       []User `json:"data"`
    Page       int    `json:"page"`
    Limit      int    `json:"limit"`
    TotalCount int    `json:"total_count"`
    TotalPages int    `json:"total_pages"`
}

func GetUsers(w http.ResponseWriter, r *http.Request) {
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    if page == 0 {
        page = 1
    }

    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    if limit == 0 {
        limit = 20
    }

    offset := (page - 1) * limit

    users, totalCount := userService.GetUsers(offset, limit)

    response := PaginatedResponse{
        Data:       users,
        Page:       page,
        Limit:      limit,
        TotalCount: totalCount,
        TotalPages: (totalCount + limit - 1) / limit,
    }

    json.NewEncoder(w).Encode(response)
}
```

## Фильтрация и поиск

```go
// GET /users?name=Alice&age_min=18&age_max=65&city=Moscow&sort=created_at:desc

func GetUsers(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query()

    filters := UserFilters{
        Name:    query.Get("name"),
        AgeMin:  parseIntOrZero(query.Get("age_min")),
        AgeMax:  parseIntOrZero(query.Get("age_max")),
        City:    query.Get("city"),
        Sort:    query.Get("sort"), // "created_at:desc"
    }

    users := userService.GetUsersWithFilters(filters)

    json.NewEncoder(w).Encode(users)
}
```

**Поиск (search):**

```go
// GET /users/search?q=alice

func SearchUsers(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query().Get("q")

    if query == "" {
        http.Error(w, "Query parameter 'q' is required", http.StatusBadRequest)
        return
    }

    users := userService.SearchUsers(query)

    json.NewEncoder(w).Encode(users)
}
```

## Версионирование API

### 1. URL Versioning (рекомендуется)

```go
// v1
router.HandleFunc("/v1/users", GetUsersV1)

// v2 - новый формат ответа
router.HandleFunc("/v2/users", GetUsersV2)

func GetUsersV1(w http.ResponseWriter, r *http.Request) {
    users := userService.GetUsers()
    // Старый формат
    json.NewEncoder(w).Encode(users)
}

func GetUsersV2(w http.ResponseWriter, r *http.Request) {
    users := userService.GetUsers()
    // Новый формат с дополнительными полями
    response := transformToV2(users)
    json.NewEncoder(w).Encode(response)
}
```

### 2. Header Versioning

```go
func VersionMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        version := r.Header.Get("API-Version")

        if version == "" {
            version = "1" // Default
        }

        ctx := context.WithValue(r.Context(), "api-version", version)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func GetUsers(w http.ResponseWriter, r *http.Request) {
    version := r.Context().Value("api-version").(string)

    users := userService.GetUsers()

    if version == "2" {
        response := transformToV2(users)
        json.NewEncoder(w).Encode(response)
    } else {
        json.NewEncoder(w).Encode(users)
    }
}
```

## HATEOAS (Hypermedia as the Engine of Application State)

```go
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
    Links Links  `json:"_links"`
}

type Links struct {
    Self    Link `json:"self"`
    Orders  Link `json:"orders"`
    Profile Link `json:"profile"`
}

type Link struct {
    Href string `json:"href"`
}

func GetUser(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]

    user := userService.GetUser(id)

    // Добавляем links
    user.Links = Links{
        Self:    Link{Href: fmt.Sprintf("/users/%d", user.ID)},
        Orders:  Link{Href: fmt.Sprintf("/users/%d/orders", user.ID)},
        Profile: Link{Href: fmt.Sprintf("/users/%d/profile", user.ID)},
    }

    json.NewEncoder(w).Encode(user)
}
```

**Ответ:**

```json
{
    "id": 123,
    "name": "Alice",
    "email": "alice@example.com",
    "_links": {
        "self": {"href": "/users/123"},
        "orders": {"href": "/users/123/orders"},
        "profile": {"href": "/users/123/profile"}
    }
}
```

## Rate Limiting

```go
import "golang.org/x/time/rate"

var limiters = make(map[string]*rate.Limiter)
var mu sync.Mutex

func getLimiter(ip string) *rate.Limiter {
    mu.Lock()
    defer mu.Unlock()

    limiter, exists := limiters[ip]
    if !exists {
        // 10 запросов в секунду, burst 20
        limiter = rate.NewLimiter(10, 20)
        limiters[ip] = limiter
    }

    return limiter
}

func RateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ip := r.RemoteAddr
        limiter := getLimiter(ip)

        if !limiter.Allow() {
            w.Header().Set("Retry-After", "1")
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

## Идемпотентность

### Идемпотентные операции с Idempotency Key

```go
var processedKeys = make(map[string]*Response)
var mu sync.Mutex

func CreateOrder(w http.ResponseWriter, r *http.Request) {
    idempotencyKey := r.Header.Get("Idempotency-Key")

    if idempotencyKey != "" {
        mu.Lock()
        if cachedResponse, exists := processedKeys[idempotencyKey]; exists {
            mu.Unlock()
            // Возвращаем кэшированный ответ
            json.NewEncoder(w).Encode(cachedResponse)
            return
        }
        mu.Unlock()
    }

    var order Order
    json.NewDecoder(r.Body).Decode(&order)

    createdOrder, err := orderService.CreateOrder(&order)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    response := &Response{Order: createdOrder}

    // Кэшируем ответ
    if idempotencyKey != "" {
        mu.Lock()
        processedKeys[idempotencyKey] = response
        mu.Unlock()
    }

    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(response)
}
```

**Запрос:**

```bash
curl -X POST /orders \
  -H "Idempotency-Key: abc123" \
  -d '{"product_id": 1, "quantity": 2}'

# Повторный запрос с тем же ключом
# → вернет тот же результат, не создаст дубликат
```

## Content Negotiation

```go
func GetUser(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]
    user := userService.GetUser(id)

    accept := r.Header.Get("Accept")

    switch accept {
    case "application/xml":
        w.Header().Set("Content-Type", "application/xml")
        xml.NewEncoder(w).Encode(user)
    case "application/json":
        fallthrough
    default:
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(user)
    }
}
```

## Bulk Operations

```go
// POST /users/bulk
func BulkCreateUsers(w http.ResponseWriter, r *http.Request) {
    var users []User
    if err := json.NewDecoder(r.Body).Decode(&users); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    results := make([]BulkResult, len(users))

    for i, user := range users {
        createdUser, err := userService.CreateUser(&user)
        if err != nil {
            results[i] = BulkResult{
                Success: false,
                Error:   err.Error(),
            }
        } else {
            results[i] = BulkResult{
                Success: true,
                Data:    createdUser,
            }
        }
    }

    json.NewEncoder(w).Encode(results)
}

type BulkResult struct {
    Success bool        `json:"success"`
    Data    interface{} `json:"data,omitempty"`
    Error   string      `json:"error,omitempty"`
}
```

## Async Operations (Long-Running Tasks)

```go
// POST /reports - создать отчет (long-running)
func CreateReport(w http.ResponseWriter, r *http.Request) {
    var request ReportRequest
    json.NewDecoder(r.Body).Decode(&request)

    // Создаем задачу
    jobID := uuid.New().String()

    // Запускаем асинхронно
    go func() {
        report, err := reportService.Generate(request)
        if err != nil {
            jobStore.SetStatus(jobID, "failed", err.Error())
        } else {
            jobStore.SetStatus(jobID, "completed", report)
        }
    }()

    // Возвращаем 202 Accepted + job ID
    w.Header().Set("Location", fmt.Sprintf("/jobs/%s", jobID))
    w.WriteHeader(http.StatusAccepted)
    json.NewEncoder(w).Encode(map[string]string{
        "job_id": jobID,
        "status": "processing",
    })
}

// GET /jobs/{jobID} - проверить статус
func GetJobStatus(w http.ResponseWriter, r *http.Request) {
    jobID := mux.Vars(r)["jobID"]

    status, result := jobStore.GetStatus(jobID)

    response := map[string]interface{}{
        "job_id": jobID,
        "status": status, // "processing", "completed", "failed"
    }

    if status == "completed" {
        response["result"] = result
    } else if status == "failed" {
        response["error"] = result
    }

    json.NewEncoder(w).Encode(response)
}
```

## Best Practices Summary

### ✅ DO (Делайте)

1. **Используйте правильные HTTP методы и коды ответов**
2. **Версионируйте API** (например, /v1/, /v2/)
3. **Всегда возвращайте JSON** (Content-Type: application/json)
4. **Стандартизируйте формат ошибок**
5. **Используйте пагинацию** для списков
6. **Документируйте API** (Swagger/OpenAPI)
7. **Используйте HTTPS** в продакшене
8. **Валидируйте входные данные**
9. **Логируйте все запросы** (request ID, latency)
10. **Используйте rate limiting**

### ❌ DON'T (Не делайте)

1. ❌ **Не используйте глаголы в URL** (/getUsers, /createUser)
2. ❌ **Не возвращайте массив в корне** (используйте `{"data": [...]}`)
3. ❌ **Не используйте разные форматы ошибок**
4. ❌ **Не игнорируйте HTTP методы** (все через POST)
5. ❌ **Не возвращайте внутренние ошибки** клиенту (stack traces)
6. ❌ **Не используйте GET для изменения данных**
7. ❌ **Не забывайте про CORS** headers
8. ❌ **Не возвращайте все поля** (используйте field selection)
9. ❌ **Не делайте breaking changes** без новой версии API
10. ❌ **Не используйте синхронные операции для long-running tasks**

## Документирование API (OpenAPI/Swagger)

```go
// @Summary Get user by ID
// @Description Get user details by user ID
// @Tags users
// @Accept json
// @Produce json
// @Param id path int true "User ID"
// @Success 200 {object} User
// @Failure 404 {object} ErrorResponse
// @Router /users/{id} [get]
func GetUser(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

Генерация документации:

```bash
# Установка swag
go install github.com/swaggo/swag/cmd/swag@latest

# Генерация документации
swag init

# Документация доступна на /swagger/index.html
```

## Связанные темы

- [[REST API - Основы]]
- [[HTTP протокол]]
- [[gRPC]]
- [[GraphQL]]
- [[Микросервисная архитектура]]
- [[OAuth 2.0]]
- [[JWT (JSON Web Tokens)]]
- [[Безопасность API]]
