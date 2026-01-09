# Gorilla Mux

Gorilla Mux — это мощный HTTP роутер и URL matcher для Go, входящий в состав проекта Gorilla web toolkit. В отличие от стандартного `net/http`, Mux предоставляет продвинутые возможности маршрутизации при сохранении совместимости со стандартной библиотекой.

## Основные возможности

### Установка

```bash
go get -u github.com/gorilla/mux
```

### Простой пример

```go
package main

import (
    "fmt"
    "net/http"
    "github.com/gorilla/mux"
)

func main() {
    r := mux.NewRouter()

    r.HandleFunc("/", homeHandler)
    r.HandleFunc("/users/{id}", userHandler)

    http.ListenAndServe(":8080", r)
}

func homeHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Главная страница")
}

func userHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id := vars["id"]
    fmt.Fprintf(w, "User ID: %s", id)
}
```

## Path Variables (переменные в пути)

### Базовые переменные

```go
r := mux.NewRouter()

// Простая переменная
r.HandleFunc("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id := vars["id"]
    fmt.Fprintf(w, "User ID: %s", id)
})

// Несколько переменных
r.HandleFunc("/users/{userID}/posts/{postID}", func(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userID := vars["userID"]
    postID := vars["postID"]
    fmt.Fprintf(w, "User: %s, Post: %s", userID, postID)
})
```

### Регулярные выражения для переменных

```go
// ID должен быть числом
r.HandleFunc("/users/{id:[0-9]+}", userHandler)

// Username только буквы
r.HandleFunc("/profile/{username:[a-zA-Z]+}", profileHandler)

// Email формат
r.HandleFunc("/email/{email:.+@.+\\..+}", emailHandler)

// UUID формат
r.HandleFunc("/articles/{id:[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}}",
    articleHandler)
```

## HTTP Methods

```go
r := mux.NewRouter()

// Ограничение по HTTP методу
r.HandleFunc("/users", getUsers).Methods("GET")
r.HandleFunc("/users", createUser).Methods("POST")
r.HandleFunc("/users/{id}", updateUser).Methods("PUT")
r.HandleFunc("/users/{id}", deleteUser).Methods("DELETE")

// Несколько методов для одного хендлера
r.HandleFunc("/users/{id}", getUserOrUpdate).Methods("GET", "PUT")
```

## Subrouters (подмаршрутизаторы)

```go
r := mux.NewRouter()

// API v1
apiV1 := r.PathPrefix("/api/v1").Subrouter()
apiV1.HandleFunc("/users", getUsersV1).Methods("GET")
apiV1.HandleFunc("/users/{id}", getUserV1).Methods("GET")

// API v2
apiV2 := r.PathPrefix("/api/v2").Subrouter()
apiV2.HandleFunc("/users", getUsersV2).Methods("GET")
apiV2.HandleFunc("/users/{id}", getUserV2).Methods("GET")

// Admin routes с middleware
admin := r.PathPrefix("/admin").Subrouter()
admin.Use(authMiddleware)
admin.HandleFunc("/dashboard", dashboardHandler)
admin.HandleFunc("/settings", settingsHandler)
```

## Middleware

### Создание middleware

```go
// Middleware для логирования
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Printf("[%s] %s %s\n", time.Now().Format("2006-01-02 15:04:05"),
            r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

// Middleware для аутентификации
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")

        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Проверка токена...

        next.ServeHTTP(w, r)
    })
}
```

### Применение middleware

```go
r := mux.NewRouter()

// Глобальный middleware для всех маршрутов
r.Use(loggingMiddleware)

// Middleware для конкретного subrouter
api := r.PathPrefix("/api").Subrouter()
api.Use(authMiddleware)

// Цепочка middleware
r.Use(loggingMiddleware, recoveryMiddleware, corsMiddleware)
```

## Query Parameters

```go
r.HandleFunc("/search", func(w http.ResponseWriter, r *http.Request) {
    // Query параметры из стандартной библиотеки
    query := r.URL.Query()

    search := query.Get("q")           // Одно значение
    page := query.Get("page")
    tags := query["tags"]              // Множественные значения: ?tags=go&tags=web

    fmt.Fprintf(w, "Search: %s, Page: %s, Tags: %v", search, page, tags)
})
```

## Матчеры (дополнительные условия)

### Host Matcher

```go
r := mux.NewRouter()

// Разные обработчики для разных хостов
r.Host("www.example.com").HandlerFunc(mainSiteHandler)
r.Host("api.example.com").HandlerFunc(apiHandler)
r.Host("{subdomain:[a-z]+}.example.com").HandlerFunc(subdomainHandler)
```

### Scheme Matcher

```go
// Только HTTPS
r.HandleFunc("/secure", secureHandler).Schemes("https")

// HTTP и HTTPS
r.HandleFunc("/public", publicHandler).Schemes("http", "https")
```

### Headers Matcher

```go
// Проверка заголовка
r.HandleFunc("/api/data", dataHandler).
    Headers("Content-Type", "application/json")

// Несколько заголовков
r.HandleFunc("/api/users", usersHandler).
    Headers("X-API-Version", "v1", "Content-Type", "application/json")
```

### Query Matcher

```go
// Маршрут активен только если есть определенный query параметр
r.HandleFunc("/search", searchHandler).
    Queries("q", "{query}")

// Несколько query параметров
r.HandleFunc("/filter", filterHandler).
    Queries("category", "{category}", "min_price", "{price:[0-9]+}")
```

## Обратные маршруты (Reverse Routing)

```go
r := mux.NewRouter()

// Именованные маршруты
r.HandleFunc("/users/{id:[0-9]+}", userHandler).Name("user")
r.HandleFunc("/articles/{category}/{id:[0-9]+}", articleHandler).Name("article")

// Генерация URL
url, err := r.Get("user").URL("id", "123")
// url.Path = "/users/123"

url, err = r.Get("article").URL("category", "tech", "id", "456")
// url.Path = "/articles/tech/456"

// Использование в хендлере
func someHandler(w http.ResponseWriter, r *http.Request) {
    url, _ := mux.CurrentRoute(r).URL("id", "42")
    fmt.Fprintf(w, "Generated URL: %s", url.Path)
}
```

## Статические файлы

```go
r := mux.NewRouter()

// Сервер статических файлов
r.PathPrefix("/static/").Handler(http.StripPrefix("/static/",
    http.FileServer(http.Dir("./static"))))

// Или с использованием HandleFunc для большего контроля
r.PathPrefix("/uploads/{rest:.*}").HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    filepath := vars["rest"]
    http.ServeFile(w, r, "./uploads/"+filepath)
})
```

## CORS Middleware

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

r.Use(corsMiddleware)
```

## Полный пример REST API

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "github.com/gorilla/mux"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

var users = []User{
    {ID: 1, Name: "Иван", Email: "ivan@example.com"},
    {ID: 2, Name: "Мария", Email: "maria@example.com"},
}
var nextID = 3

func main() {
    r := mux.NewRouter()

    // Middleware
    r.Use(loggingMiddleware)
    r.Use(corsMiddleware)

    // API routes
    api := r.PathPrefix("/api/v1").Subrouter()
    api.HandleFunc("/users", getUsers).Methods("GET")
    api.HandleFunc("/users/{id:[0-9]+}", getUser).Methods("GET")
    api.HandleFunc("/users", createUser).Methods("POST")
    api.HandleFunc("/users/{id:[0-9]+}", updateUser).Methods("PUT")
    api.HandleFunc("/users/{id:[0-9]+}", deleteUser).Methods("DELETE")

    fmt.Println("Сервер запущен на :8080")
    http.ListenAndServe(":8080", r)
}

func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Printf("[%s] %s\n", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

func getUsers(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func getUser(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    for _, user := range users {
        if user.ID == id {
            json.NewEncoder(w).Encode(user)
            return
        }
    }

    w.WriteHeader(http.StatusNotFound)
    json.NewEncoder(w).Encode(map[string]string{"error": "user not found"})
}

func createUser(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    var user User

    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
        return
    }

    user.ID = nextID
    nextID++
    users = append(users, user)

    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func updateUser(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    var updatedUser User
    if err := json.NewDecoder(r.Body).Decode(&updatedUser); err != nil {
        w.WriteHeader(http.StatusBadRequest)
        json.NewEncoder(w).Encode(map[string]string{"error": err.Error()})
        return
    }

    for i, user := range users {
        if user.ID == id {
            users[i].Name = updatedUser.Name
            users[i].Email = updatedUser.Email
            json.NewEncoder(w).Encode(users[i])
            return
        }
    }

    w.WriteHeader(http.StatusNotFound)
    json.NewEncoder(w).Encode(map[string]string{"error": "user not found"})
}

func deleteUser(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    vars := mux.Vars(r)
    id, _ := strconv.Atoi(vars["id"])

    for i, user := range users {
        if user.ID == id {
            users = append(users[:i], users[i+1:]...)
            json.NewEncoder(w).Encode(map[string]string{"message": "user deleted"})
            return
        }
    }

    w.WriteHeader(http.StatusNotFound)
    json.NewEncoder(w).Encode(map[string]string{"error": "user not found"})
}
```

## Сравнение с net/http

### Gorilla Mux

✅ **Преимущества:**
- Переменные в пути с поддержкой regex
- Subrouters для организации кода
- Множество матчеров (host, scheme, headers, query)
- Обратная генерация URL
- Встроенная поддержка middleware

❌ **Недостатки:**
- Дополнительная зависимость
- Немного медленнее чем httprouter

### net/http

✅ **Преимущества:**
- Стандартная библиотека
- Никаких зависимостей
- Достаточно для простых случаев

❌ **Недостатки:**
- Нет переменных в пути
- Примитивная маршрутизация
- Нужно писать много кода вручную

## Best Practices

### ✅ Правильно

```go
// Использовать именованные маршруты для генерации URL
r.HandleFunc("/users/{id}", userHandler).Name("user")

// Группировать связанные маршруты через subrouters
api := r.PathPrefix("/api/v1").Subrouter()

// Использовать regex для валидации параметров
r.HandleFunc("/users/{id:[0-9]+}", userHandler)

// Применять middleware на нужном уровне
adminRouter := r.PathPrefix("/admin").Subrouter()
adminRouter.Use(authMiddleware)
```

### ❌ Неправильно

```go
// Не использовать слишком сложные regex
r.HandleFunc("/users/{id:[0-9]{1,10}[a-f]{2}[A-Z]*}", handler) // Слишком сложно

// Не дублировать middleware
r.Use(loggingMiddleware)
r.HandleFunc("/api", handler).Use(loggingMiddleware) // Дублирование
```

## Вопросы с собеседований

**Вопрос:** В чем основное отличие Gorilla Mux от стандартного net/http?

**Ответ:** Gorilla Mux предоставляет продвинутую маршрутизацию с переменными в пути, regex валидацией, subrouters и различными матчерами, при этом сохраняя полную совместимость с `http.Handler` из стандартной библиотеки.

**Вопрос:** Как получить переменные из пути в Gorilla Mux?

**Ответ:** Используя функцию `mux.Vars(r)`, которая возвращает `map[string]string` со всеми переменными из пути.

**Вопрос:** Что такое subrouter и когда его использовать?

**Ответ:** Subrouter — это дочерний роутер с общим префиксом пути. Используется для группировки связанных маршрутов (например, `/api/v1/...`) и применения middleware только к определенной группе маршрутов.

## Связанные темы

- [[Go - Пакет net-http]]
- [[REST API - Дизайн и best practices]]
- [[Gin - Web фреймворк]]
- [[Echo]]
- [[Chi]]