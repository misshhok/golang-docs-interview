# Chi

Chi — это легковесный, идиоматичный и композиционный HTTP роутер для Go. Он построен на стандартном `net/http` и полностью с ним совместим, предоставляя удобный API для маршрутизации без излишних абстракций.

## Основные возможности

### Установка

```bash
go get -u github.com/go-chi/chi/v5
```

### Простой пример

```go
package main

import (
    "net/http"
    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
)

func main() {
    r := chi.NewRouter()

    // Middleware
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)

    // Routes
    r.Get("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello, World!"))
    })

    r.Get("/users/{userID}", func(w http.ResponseWriter, r *http.Request) {
        userID := chi.URLParam(r, "userID")
        w.Write([]byte("User ID: " + userID))
    })

    http.ListenAndServe(":8080", r)
}
```

## Routing

### HTTP Methods

```go
r := chi.NewRouter()

r.Get("/users", getUsers)         // GET
r.Post("/users", createUser)      // POST
r.Put("/users/{id}", updateUser)  // PUT
r.Delete("/users/{id}", deleteUser) // DELETE
r.Patch("/users/{id}", patchUser) // PATCH

// Несколько методов для одного пути
r.Route("/articles", func(r chi.Router) {
    r.Get("/", listArticles)    // GET /articles
    r.Post("/", createArticle)  // POST /articles
})
```

### URL Parameters

```go
// Простой параметр
r.Get("/users/{userID}", func(w http.ResponseWriter, r *http.Request) {
    userID := chi.URLParam(r, "userID")
    w.Write([]byte("User: " + userID))
})

// Несколько параметров
r.Get("/users/{userID}/posts/{postID}", func(w http.ResponseWriter, r *http.Request) {
    userID := chi.URLParam(r, "userID")
    postID := chi.URLParam(r, "postID")
    // ...
})

// Regex валидация параметров
r.Get("/users/{userID:[0-9]+}", userHandler) // Только цифры
```

### Wildcard и Catch-all

```go
// Wildcard - захватывает один сегмент
r.Get("/files/{filename}", fileHandler)

// Catch-all - захватывает все оставшиеся сегменты
r.Get("/static/*", func(w http.ResponseWriter, r *http.Request) {
    // chi.URLParam(r, "*") вернет оставшийся путь
    filepath := chi.URLParam(r, "*")
    http.ServeFile(w, r, "./static/"+filepath)
})
```

## Группировка маршрутов

### Route Groups

```go
r := chi.NewRouter()

// Группа для API v1
r.Route("/api/v1", func(r chi.Router) {
    r.Get("/users", getUsersV1)
    r.Post("/users", createUserV1)

    // Вложенная группа
    r.Route("/admin", func(r chi.Router) {
        r.Use(AdminOnly) // Middleware только для admin
        r.Get("/stats", getStats)
        r.Delete("/users/{id}", deleteUser)
    })
})

// Группа для API v2
r.Route("/api/v2", func(r chi.Router) {
    r.Get("/users", getUsersV2)
    r.Post("/users", createUserV2)
})
```

### Mount (подключение sub-router)

```go
func main() {
    r := chi.NewRouter()

    // Создаем отдельный роутер для API
    apiRouter := chi.NewRouter()
    apiRouter.Get("/users", getUsers)
    apiRouter.Post("/users", createUser)

    // Монтируем его по префиксу
    r.Mount("/api", apiRouter)

    http.ListenAndServe(":8080", r)
}
```

## Middleware

### Встроенные Middleware

```go
import "github.com/go-chi/chi/v5/middleware"

r := chi.NewRouter()

// Logger - логирование запросов
r.Use(middleware.Logger)

// Recoverer - восстановление после panic
r.Use(middleware.Recoverer)

// Request ID
r.Use(middleware.RequestID)

// RealIP - определение реального IP клиента
r.Use(middleware.RealIP)

// Timeout для всех запросов
r.Use(middleware.Timeout(60 * time.Second))

// CleanPath - нормализация URL
r.Use(middleware.CleanPath)

// StripSlashes - убирает trailing slash
r.Use(middleware.StripSlashes)

// Compress - gzip сжатие
r.Use(middleware.Compress(5))

// Throttle - ограничение одновременных запросов
r.Use(middleware.Throttle(100))

// URLFormat - определение формата ответа по расширению
r.Use(middleware.URLFormat)
```

### Кастомные Middleware

```go
// Простой middleware
func MyMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Код до обработки запроса
        fmt.Println("Before request")

        next.ServeHTTP(w, r)

        // Код после обработки запроса
        fmt.Println("After request")
    })
}

// Применение
r.Use(MyMiddleware)

// Middleware с параметрами
func AuthMiddleware(requiredRole string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := r.Header.Get("Authorization")

            if token == "" {
                http.Error(w, "Unauthorized", http.StatusUnauthorized)
                return
            }

            // Проверка роли...

            next.ServeHTTP(w, r)
        })
    }
}

// Применение с параметром
r.Use(AuthMiddleware("admin"))
```

### Middleware для конкретных маршрутов

```go
// Middleware только для одного маршрута
r.With(AuthMiddleware).Get("/protected", protectedHandler)

// Цепочка middleware
r.With(AuthMiddleware, LoggingMiddleware).Get("/admin", adminHandler)

// Группа с middleware
r.Group(func(r chi.Router) {
    r.Use(AuthMiddleware)
    r.Get("/profile", profileHandler)
    r.Put("/profile", updateProfileHandler)
})
```

## Context и значения

```go
type contextKey string

const UserKey contextKey = "user"

// Middleware добавляет значение в context
func UserCtx(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := chi.URLParam(r, "userID")

        // Получаем пользователя из БД
        user := getUserFromDB(userID)

        // Сохраняем в context
        ctx := context.WithValue(r.Context(), UserKey, user)

        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Использование в хендлере
r.Route("/users/{userID}", func(r chi.Router) {
    r.Use(UserCtx) // Middleware загружает user в context

    r.Get("/", func(w http.ResponseWriter, r *http.Request) {
        user := r.Context().Value(UserKey).(*User)
        json.NewEncoder(w).Encode(user)
    })

    r.Put("/", func(w http.ResponseWriter, r *http.Request) {
        user := r.Context().Value(UserKey).(*User)
        // Обновление user...
    })
})
```

## Работа с подзапросами

### Subrouters с общей логикой

```go
r.Route("/articles/{articleID}", func(r chi.Router) {
    // Middleware загружает статью в context
    r.Use(ArticleCtx)

    r.Get("/", getArticle)           // GET /articles/123
    r.Put("/", updateArticle)        // PUT /articles/123
    r.Delete("/", deleteArticle)     // DELETE /articles/123

    // Вложенные ресурсы
    r.Route("/comments", func(r chi.Router) {
        r.Get("/", listComments)     // GET /articles/123/comments
        r.Post("/", createComment)   // POST /articles/123/comments
    })
})
```

## Полный пример REST API

```go
package main

import (
    "encoding/json"
    "net/http"
    "strconv"
    "github.com/go-chi/chi/v5"
    "github.com/go-chi/chi/v5/middleware"
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
    r := chi.NewRouter()

    // Global middleware
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    r.Use(middleware.RequestID)
    r.Use(middleware.RealIP)

    // Routes
    r.Route("/api/v1", func(r chi.Router) {
        r.Get("/health", healthCheck)

        r.Route("/users", func(r chi.Router) {
            r.Get("/", listUsers)    // GET /api/v1/users
            r.Post("/", createUser)  // POST /api/v1/users

            r.Route("/{userID}", func(r chi.Router) {
                r.Use(UserCtx) // Загружаем user в context

                r.Get("/", getUser)       // GET /api/v1/users/123
                r.Put("/", updateUser)    // PUT /api/v1/users/123
                r.Delete("/", deleteUser) // DELETE /api/v1/users/123
            })
        })
    })

    http.ListenAndServe(":8080", r)
}

func healthCheck(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("OK"))
}

func listUsers(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    user.ID = nextID
    nextID++
    users = append(users, user)

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

func UserCtx(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID, err := strconv.Atoi(chi.URLParam(r, "userID"))
        if err != nil {
            http.Error(w, "Invalid user ID", http.StatusBadRequest)
            return
        }

        for _, user := range users {
            if user.ID == userID {
                ctx := context.WithValue(r.Context(), "user", user)
                next.ServeHTTP(w, r.WithContext(ctx))
                return
            }
        }

        http.Error(w, "User not found", http.StatusNotFound)
    })
}

func getUser(w http.ResponseWriter, r *http.Request) {
    user := r.Context().Value("user").(User)
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func updateUser(w http.ResponseWriter, r *http.Request) {
    user := r.Context().Value("user").(User)

    var updateData User
    if err := json.NewDecoder(r.Body).Decode(&updateData); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    for i, u := range users {
        if u.ID == user.ID {
            users[i].Name = updateData.Name
            users[i].Email = updateData.Email
            w.Header().Set("Content-Type", "application/json")
            json.NewEncoder(w).Encode(users[i])
            return
        }
    }
}

func deleteUser(w http.ResponseWriter, r *http.Request) {
    user := r.Context().Value("user").(User)

    for i, u := range users {
        if u.ID == user.ID {
            users = append(users[:i], users[i+1:]...)
            w.WriteHeader(http.StatusNoContent)
            return
        }
    }
}
```

## Статические файлы

```go
// FileServer для статических файлов
func FileServer(r chi.Router, path string, root http.FileSystem) {
    if strings.ContainsAny(path, "{}*") {
        panic("FileServer does not permit URL parameters.")
    }

    fs := http.StripPrefix(path, http.FileServer(root))

    if path != "/" && path[len(path)-1] != '/' {
        r.Get(path, http.RedirectHandler(path+"/", http.StatusMovedPermanently).ServeHTTP)
        path += "/"
    }
    path += "*"

    r.Get(path, func(w http.ResponseWriter, r *http.Request) {
        fs.ServeHTTP(w, r)
    })
}

// Использование
r := chi.NewRouter()
FileServer(r, "/static", http.Dir("./static"))
```

## Преимущества Chi

### ✅ Преимущества

- Полная совместимость с `net/http`
- Минимальные накладные расходы
- Идиоматичный Go код
- Композиция через middleware
- Поддержка контекста из коробки
- Легковесный (нет лишних зависимостей)
- Sub-routing с сохранением типа Router

### Когда использовать Chi

- Нужна совместимость с `net/http`
- Важна производительность
- Нужен контроль над middleware
- RESTful API с вложенными ресурсами
- Предпочтение стандартной библиотеке

## Best Practices

### ✅ Правильно

```go
// Группировка связанных маршрутов
r.Route("/articles/{articleID}", func(r chi.Router) {
    r.Use(ArticleCtx) // Загружаем статью в context
    r.Get("/", getArticle)
    r.Put("/", updateArticle)
})

// Использование context для передачи данных
ctx := context.WithValue(r.Context(), "user", user)
next.ServeHTTP(w, r.WithContext(ctx))

// Проверка параметров в middleware
userID, err := strconv.Atoi(chi.URLParam(r, "userID"))
if err != nil {
    http.Error(w, "Invalid ID", http.StatusBadRequest)
    return
}
```

### ❌ Неправильно

```go
// Не использовать глобальные переменные для данных запроса
var currentUser User // Плохо!

// Не игнорировать ошибки
chi.URLParam(r, "userID") // Нет проверки на пустую строку

// Не использовать middleware там, где не нужно
r.Use(HeavyMiddleware) // Для всех маршрутов, даже статических
```

## Вопросы с собеседований

**Вопрос:** В чем основное преимущество Chi перед другими роутерами?

**Ответ:** Chi полностью совместим с `net/http` и использует стандартный `http.Handler` интерфейс. Это делает его легковесным, идиоматичным и легко интегрируемым с любым middleware или библиотекой, работающей со стандартной библиотекой.

**Вопрос:** Как Chi обрабатывает параметры URL?

**Ответ:** Chi использует функцию `chi.URLParam(r, "paramName")` для получения параметров из URL. Параметры определяются в маршруте с помощью `{paramName}` и могут иметь regex валидацию.

**Вопрос:** Что такое Route groups в Chi и зачем они нужны?

**Ответ:** Route groups (`r.Route()`) позволяют группировать связанные маршруты с общим префиксом и middleware. Это полезно для организации RESTful ресурсов и применения middleware только к определенной группе маршрутов.

## Связанные темы

- [[Go - Пакет net-http]]
- [[REST API - Дизайн и best practices]]
- [[Gin - Web фреймворк]]
- [[Gorilla Mux]]
- [[Echo]]
- [[Go - Context]]