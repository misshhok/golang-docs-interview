# Echo

Echo — это высокопроизводительный, минималистичный веб-фреймворк для Go. Он конкурирует с Gin по скорости и удобству, предоставляя богатый набор возможностей для разработки RESTful API и веб-приложений.

## Основные возможности

### Установка

```bash
go get -u github.com/labstack/echo/v4
```

### Простой пример

```go
package main

import (
    "net/http"
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

func main() {
    // Создаем экземпляр Echo
    e := echo.New()

    // Middleware
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())

    // Routes
    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, World!")
    })

    e.GET("/users/:id", func(c echo.Context) error {
        id := c.Param("id")
        return c.String(http.StatusOK, "User ID: "+id)
    })

    // Запуск сервера
    e.Logger.Fatal(e.Start(":8080"))
}
```

## Routing

### Path Parameters

```go
// Простой параметр
e.GET("/users/:id", func(c echo.Context) error {
    id := c.Param("id")
    return c.JSON(http.StatusOK, map[string]string{"id": id})
})

// Несколько параметров
e.GET("/users/:uid/posts/:pid", func(c echo.Context) error {
    uid := c.Param("uid")
    pid := c.Param("pid")
    return c.JSON(http.StatusOK, map[string]string{
        "user_id": uid,
        "post_id": pid,
    })
})

// Wildcard параметр (захватывает все)
e.GET("/files/*", func(c echo.Context) error {
    filepath := c.Param("*")
    return c.String(http.StatusOK, filepath)
})
```

### Query Parameters

```go
e.GET("/search", func(c echo.Context) error {
    // Получение query параметров
    query := c.QueryParam("q")                  // Одно значение
    page := c.QueryParam("page")
    sort := c.QueryParams()["sort"]             // Массив значений

    return c.JSON(http.StatusOK, map[string]interface{}{
        "query": query,
        "page":  page,
        "sort":  sort,
    })
})
```

### HTTP Methods

```go
e.GET("/users", getUsers)
e.POST("/users", createUser)
e.PUT("/users/:id", updateUser)
e.DELETE("/users/:id", deleteUser)
e.PATCH("/users/:id", patchUser)
e.OPTIONS("/users", optionsUsers)
e.HEAD("/users", headUsers)
```

## Context (контекст запроса)

```go
func handler(c echo.Context) error {
    // Request
    req := c.Request()
    method := req.Method
    path := req.URL.Path

    // Response
    res := c.Response()

    // Path параметры
    id := c.Param("id")

    // Query параметры
    name := c.QueryParam("name")

    // Form данные
    email := c.FormValue("email")

    // Headers
    userAgent := c.Request().Header.Get("User-Agent")
    c.Response().Header().Set("X-Custom", "value")

    // Cookies
    cookie, _ := c.Cookie("session")

    // Сохранение данных в контексте
    c.Set("user", "John")
    user := c.Get("user").(string)

    return c.String(http.StatusOK, "OK")
}
```

## Middleware

### Встроенные Middleware

```go
e := echo.New()

// Logger - логирование запросов
e.Use(middleware.Logger())

// Recover - восстановление после panic
e.Use(middleware.Recover())

// CORS
e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
    AllowOrigins: []string{"https://example.com"},
    AllowMethods: []string{http.MethodGet, http.MethodPost},
}))

// JWT аутентификация
e.Use(middleware.JWTWithConfig(middleware.JWTConfig{
    SigningKey: []byte("secret"),
}))

// Rate Limiting
e.Use(middleware.RateLimiter(middleware.NewRateLimiterMemoryStore(20)))

// Gzip сжатие
e.Use(middleware.Gzip())

// Request ID
e.Use(middleware.RequestID())

// Secure headers
e.Use(middleware.SecureWithConfig(middleware.SecureConfig{
    XSSProtection:         "1; mode=block",
    ContentTypeNosniff:    "nosniff",
    XFrameOptions:         "SAMEORIGIN",
}))
```

### Кастомные Middleware

```go
// Middleware для аутентификации
func AuthMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        token := c.Request().Header.Get("Authorization")

        if token == "" {
            return c.JSON(http.StatusUnauthorized, map[string]string{
                "error": "unauthorized",
            })
        }

        // Проверка токена...

        c.Set("user_id", "123")
        return next(c)
    }
}

// Применение middleware
e.Use(AuthMiddleware)

// Или только для определенных маршрутов
e.GET("/admin/dashboard", dashboardHandler, AuthMiddleware)
```

### Middleware с конфигурацией

```go
type CustomMiddlewareConfig struct {
    Timeout time.Duration
    MaxRequests int
}

func CustomMiddleware(config CustomMiddlewareConfig) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            // Логика middleware с использованием config
            fmt.Printf("Timeout: %v, MaxRequests: %d\n", config.Timeout, config.MaxRequests)
            return next(c)
        }
    }
}

// Использование
e.Use(CustomMiddleware(CustomMiddlewareConfig{
    Timeout: 30 * time.Second,
    MaxRequests: 100,
}))
```

## Data Binding и Validation

### JSON Binding

```go
type User struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"required,email"`
    Age   int    `json:"age" validate:"gte=0,lte=130"`
}

e.POST("/users", func(c echo.Context) error {
    user := new(User)

    // Bind JSON
    if err := c.Bind(user); err != nil {
        return c.JSON(http.StatusBadRequest, map[string]string{
            "error": err.Error(),
        })
    }

    // Валидация
    if err := c.Validate(user); err != nil {
        return c.JSON(http.StatusBadRequest, map[string]string{
            "error": err.Error(),
        })
    }

    return c.JSON(http.StatusCreated, user)
})
```

### Кастомный валидатор

```go
import "github.com/go-playground/validator/v10"

type CustomValidator struct {
    validator *validator.Validate
}

func (cv *CustomValidator) Validate(i interface{}) error {
    return cv.validator.Struct(i)
}

func main() {
    e := echo.New()
    e.Validator = &CustomValidator{validator: validator.New()}
}
```

### Form Binding

```go
type LoginForm struct {
    Username string `form:"username" validate:"required"`
    Password string `form:"password" validate:"required,min=6"`
}

e.POST("/login", func(c echo.Context) error {
    form := new(LoginForm)

    if err := c.Bind(form); err != nil {
        return err
    }

    if err := c.Validate(form); err != nil {
        return err
    }

    return c.JSON(http.StatusOK, form)
})
```

## Группировка маршрутов

```go
e := echo.New()

// API v1
v1 := e.Group("/api/v1")
v1.GET("/users", getUsersV1)
v1.POST("/users", createUserV1)

// API v2
v2 := e.Group("/api/v2")
v2.GET("/users", getUsersV2)
v2.POST("/users", createUserV2)

// Admin группа с middleware
admin := e.Group("/admin")
admin.Use(middleware.BasicAuth(func(username, password string, c echo.Context) (bool, error) {
    if username == "admin" && password == "secret" {
        return true, nil
    }
    return false, nil
}))
admin.GET("/dashboard", dashboardHandler)
```

## Responses

### JSON Response

```go
e.GET("/user", func(c echo.Context) error {
    user := map[string]string{
        "name": "John",
        "email": "john@example.com",
    }
    return c.JSON(http.StatusOK, user)
})

// Красивый JSON (с отступами)
e.GET("/user/pretty", func(c echo.Context) error {
    return c.JSONPretty(http.StatusOK, user, "  ")
})
```

### String Response

```go
e.GET("/hello", func(c echo.Context) error {
    return c.String(http.StatusOK, "Hello, World!")
})

// HTML Response
e.GET("/html", func(c echo.Context) error {
    return c.HTML(http.StatusOK, "<h1>Hello</h1>")
})
```

### File Response

```go
// Отправка файла
e.GET("/download", func(c echo.Context) error {
    return c.File("file.pdf")
})

// Attachment (скачивание)
e.GET("/download-attachment", func(c echo.Context) error {
    return c.Attachment("file.pdf", "downloaded.pdf")
})

// Inline (открытие в браузере)
e.GET("/view", func(c echo.Context) error {
    return c.Inline("file.pdf", "document.pdf")
})
```

### Redirect

```go
e.GET("/old", func(c echo.Context) error {
    return c.Redirect(http.StatusMovedPermanently, "/new")
})
```

## Error Handling

### Кастомный Error Handler

```go
func customHTTPErrorHandler(err error, c echo.Context) {
    code := http.StatusInternalServerError
    message := "Internal Server Error"

    if he, ok := err.(*echo.HTTPError); ok {
        code = he.Code
        message = he.Message.(string)
    }

    c.Logger().Error(err)

    if !c.Response().Committed {
        c.JSON(code, map[string]string{
            "error": message,
        })
    }
}

e := echo.New()
e.HTTPErrorHandler = customHTTPErrorHandler
```

### Возврат ошибок

```go
e.GET("/user/:id", func(c echo.Context) error {
    id := c.Param("id")

    if id == "" {
        return echo.NewHTTPError(http.StatusBadRequest, "id is required")
    }

    user, err := getUserFromDB(id)
    if err != nil {
        return echo.NewHTTPError(http.StatusNotFound, "user not found")
    }

    return c.JSON(http.StatusOK, user)
})
```

## Полный пример CRUD API

```go
package main

import (
    "net/http"
    "strconv"
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

type Book struct {
    ID     int    `json:"id"`
    Title  string `json:"title" validate:"required"`
    Author string `json:"author" validate:"required"`
}

var books = []Book{
    {ID: 1, Title: "Go в действии", Author: "Уильям Кеннеди"},
    {ID: 2, Title: "Язык программирования Go", Author: "Алан Донован"},
}
var nextID = 3

func main() {
    e := echo.New()

    // Middleware
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())
    e.Use(middleware.CORS())

    // Routes
    e.GET("/books", getBooks)
    e.GET("/books/:id", getBook)
    e.POST("/books", createBook)
    e.PUT("/books/:id", updateBook)
    e.DELETE("/books/:id", deleteBook)

    e.Logger.Fatal(e.Start(":8080"))
}

func getBooks(c echo.Context) error {
    return c.JSON(http.StatusOK, books)
}

func getBook(c echo.Context) error {
    id, _ := strconv.Atoi(c.Param("id"))

    for _, book := range books {
        if book.ID == id {
            return c.JSON(http.StatusOK, book)
        }
    }

    return echo.NewHTTPError(http.StatusNotFound, "book not found")
}

func createBook(c echo.Context) error {
    book := new(Book)

    if err := c.Bind(book); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }

    book.ID = nextID
    nextID++
    books = append(books, *book)

    return c.JSON(http.StatusCreated, book)
}

func updateBook(c echo.Context) error {
    id, _ := strconv.Atoi(c.Param("id"))

    book := new(Book)
    if err := c.Bind(book); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }

    for i, b := range books {
        if b.ID == id {
            books[i].Title = book.Title
            books[i].Author = book.Author
            return c.JSON(http.StatusOK, books[i])
        }
    }

    return echo.NewHTTPError(http.StatusNotFound, "book not found")
}

func deleteBook(c echo.Context) error {
    id, _ := strconv.Atoi(c.Param("id"))

    for i, book := range books {
        if book.ID == id {
            books = append(books[:i], books[i+1:]...)
            return c.JSON(http.StatusOK, map[string]string{
                "message": "book deleted",
            })
        }
    }

    return echo.NewHTTPError(http.StatusNotFound, "book not found")
}
```

## Сравнение с Gin

### Echo

✅ **Преимущества:**
- Более чистый API
- Лучшая организация middleware
- Встроенная поддержка WebSocket
- Более гибкая обработка ошибок
- Auto TLS (Let's Encrypt)

### Gin

✅ **Преимущества:**
- Немного быстрее
- Более популярен (больше community)
- Встроенная валидация из коробки
- gin.H для удобного создания JSON

## Best Practices

### ✅ Правильно

```go
// Структурированная обработка ошибок
func handler(c echo.Context) error {
    data, err := fetchData()
    if err != nil {
        return echo.NewHTTPError(http.StatusInternalServerError, err.Error())
    }
    return c.JSON(http.StatusOK, data)
}

// Использование групп для версионирования API
v1 := e.Group("/api/v1")
v2 := e.Group("/api/v2")

// Валидация входных данных
type Request struct {
    Name string `json:"name" validate:"required"`
}
```

### ❌ Неправильно

```go
// Игнорирование ошибок
func handler(c echo.Context) error {
    c.String(200, "OK") // Без проверки ошибок
    return nil
}

// Хардкод статус кодов
c.JSON(200, data) // Использовать http.StatusOK
```

## Вопросы с собеседований

**Вопрос:** В чем главное отличие Echo от Gin?

**Ответ:** Echo предлагает более чистый API с лучшей организацией middleware и встроенной поддержкой WebSocket. Gin немного быстрее и популярнее, но оба фреймворка отлично подходят для создания REST API.

**Вопрос:** Как работает middleware в Echo?

**Ответ:** Middleware в Echo — это функция, которая принимает `echo.HandlerFunc` и возвращает новую `echo.HandlerFunc`. Middleware выполняется в порядке регистрации и может изменять запрос/ответ или прерывать цепочку обработки.

**Вопрос:** Как в Echo обрабатываются ошибки?

**Ответ:** Echo имеет централизованный error handler. Хендлеры возвращают `error`, который обрабатывается глобальным `HTTPErrorHandler`. Можно создать кастомный error handler для специфичной обработки ошибок.

## Связанные темы

- [[Go - Пакет net-http]]
- [[REST API - Дизайн и best practices]]
- [[Gin - Web фреймворк]]
- [[Gorilla Mux]]
- [[Chi]]