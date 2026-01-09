# Gin - Web фреймворк

Gin — это высокопроизводительный HTTP веб-фреймворк для Go, построенный на основе httprouter. Он предоставляет удобный API для создания REST API и веб-приложений с минимальными накладными расходами.

## Основные возможности

### Установка

```bash
go get -u github.com/gin-gonic/gin
```

### Простой пример

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func main() {
    // Создаем роутер с дефолтными middleware (Logger и Recovery)
    r := gin.Default()

    // Определяем простой GET эндпоинт
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "message": "pong",
        })
    })

    // Запускаем сервер на порту 8080
    r.Run(":8080")
}
```

## Routing

### Path Parameters

```go
// Параметр в пути
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(http.StatusOK, gin.H{
        "user_id": id,
    })
})

// Wildcard параметр (захватывает все после /files/)
r.GET("/files/*filepath", func(c *gin.Context) {
    filepath := c.Param("filepath")
    c.String(http.StatusOK, filepath)
})
```

### Query Parameters

```go
r.GET("/search", func(c *gin.Context) {
    // Query параметры: /search?query=golang&page=1
    query := c.Query("query")              // Получить значение или ""
    page := c.DefaultQuery("page", "1")    // Значение или дефолт

    c.JSON(http.StatusOK, gin.H{
        "query": query,
        "page":  page,
    })
})
```

### HTTP Methods

```go
r.GET("/users", getUsers)       // Получение списка
r.POST("/users", createUser)    // Создание
r.PUT("/users/:id", updateUser) // Обновление
r.DELETE("/users/:id", deleteUser) // Удаление
r.PATCH("/users/:id", patchUser)   // Частичное обновление
```

## Middleware

### Встроенные Middleware

```go
// Logger и Recovery middleware
r := gin.Default()

// Или создать без middleware и добавить вручную
r := gin.New()
r.Use(gin.Logger())
r.Use(gin.Recovery())
```

### Кастомные Middleware

```go
// Middleware для аутентификации
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")

        if token == "" {
            c.JSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
            c.Abort() // Останавливаем цепочку обработчиков
            return
        }

        // Проверка токена...

        c.Set("user_id", "123") // Сохраняем данные в контексте
        c.Next() // Передаем управление следующему обработчику
    }
}

// Применение middleware
r.Use(AuthMiddleware())

// Или только для определенных маршрутов
authorized := r.Group("/api")
authorized.Use(AuthMiddleware())
{
    authorized.GET("/profile", getProfile)
    authorized.POST("/logout", logout)
}
```

### CORS Middleware

```go
func CORSMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
        c.Writer.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
        c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(http.StatusOK)
            return
        }

        c.Next()
    }
}
```

## Binding и Валидация

### JSON Binding

```go
type User struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"gte=18,lte=100"`
}

r.POST("/users", func(c *gin.Context) {
    var user User

    // Автоматический парсинг и валидация JSON
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // user заполнен и провалидирован
    c.JSON(http.StatusCreated, user)
})
```

### Form Binding

```go
type LoginForm struct {
    Username string `form:"username" binding:"required"`
    Password string `form:"password" binding:"required,min=6"`
}

r.POST("/login", func(c *gin.Context) {
    var form LoginForm

    if err := c.ShouldBind(&form); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Обработка логина
})
```

### Query String Binding

```go
type Pagination struct {
    Page     int    `form:"page" binding:"required,min=1"`
    PageSize int    `form:"page_size" binding:"required,min=1,max=100"`
    Sort     string `form:"sort"`
}

r.GET("/items", func(c *gin.Context) {
    var pagination Pagination

    if err := c.ShouldBindQuery(&pagination); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // Используем pagination
})
```

## Группировка маршрутов

```go
func main() {
    r := gin.Default()

    // API v1
    v1 := r.Group("/api/v1")
    {
        v1.GET("/users", getUsersV1)
        v1.POST("/users", createUserV1)

        // Вложенная группа с middleware
        admin := v1.Group("/admin")
        admin.Use(AdminAuthMiddleware())
        {
            admin.GET("/stats", getStats)
            admin.DELETE("/users/:id", deleteUser)
        }
    }

    // API v2
    v2 := r.Group("/api/v2")
    {
        v2.GET("/users", getUsersV2)
        v2.POST("/users", createUserV2)
    }

    r.Run(":8080")
}
```

## Полный пример CRUD API

```go
package main

import (
    "net/http"
    "strconv"
    "github.com/gin-gonic/gin"
)

type Book struct {
    ID     int    `json:"id"`
    Title  string `json:"title" binding:"required"`
    Author string `json:"author" binding:"required"`
}

var books = []Book{
    {ID: 1, Title: "Go в действии", Author: "Уильям Кеннеди"},
    {ID: 2, Title: "Язык программирования Go", Author: "Алан Донован"},
}
var nextID = 3

func main() {
    r := gin.Default()

    api := r.Group("/api/v1")
    {
        // Получить все книги
        api.GET("/books", getBooks)

        // Получить книгу по ID
        api.GET("/books/:id", getBook)

        // Создать книгу
        api.POST("/books", createBook)

        // Обновить книгу
        api.PUT("/books/:id", updateBook)

        // Удалить книгу
        api.DELETE("/books/:id", deleteBook)
    }

    r.Run(":8080")
}

func getBooks(c *gin.Context) {
    c.JSON(http.StatusOK, books)
}

func getBook(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
        return
    }

    for _, book := range books {
        if book.ID == id {
            c.JSON(http.StatusOK, book)
            return
        }
    }

    c.JSON(http.StatusNotFound, gin.H{"error": "book not found"})
}

func createBook(c *gin.Context) {
    var newBook Book

    if err := c.ShouldBindJSON(&newBook); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    newBook.ID = nextID
    nextID++
    books = append(books, newBook)

    c.JSON(http.StatusCreated, newBook)
}

func updateBook(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
        return
    }

    var updatedBook Book
    if err := c.ShouldBindJSON(&updatedBook); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    for i, book := range books {
        if book.ID == id {
            books[i].Title = updatedBook.Title
            books[i].Author = updatedBook.Author
            c.JSON(http.StatusOK, books[i])
            return
        }
    }

    c.JSON(http.StatusNotFound, gin.H{"error": "book not found"})
}

func deleteBook(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
        return
    }

    for i, book := range books {
        if book.ID == id {
            books = append(books[:i], books[i+1:]...)
            c.JSON(http.StatusOK, gin.H{"message": "book deleted"})
            return
        }
    }

    c.JSON(http.StatusNotFound, gin.H{"error": "book not found"})
}
```

## Best Practices

### ✅ Правильно

```go
// Использовать структуры для request/response
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required"`
    Email string `json:"email" binding:"required,email"`
}

type UserResponse struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// Выносить бизнес-логику из хендлеров
func createUserHandler(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    user, err := userService.Create(req.Name, req.Email)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusCreated, UserResponse{
        ID:    user.ID,
        Name:  user.Name,
        Email: user.Email,
    })
}
```

### ❌ Неправильно

```go
// Не смешивать бизнес-логику с HTTP логикой
func createUser(c *gin.Context) {
    var data map[string]interface{} // Не использовать map
    c.BindJSON(&data)

    // Вся бизнес-логика в хендлере - плохо
    db.Exec("INSERT INTO users ...")

    c.JSON(200, data) // Не указывать хардкод статусы
}
```

## Вопросы с собеседований

**Вопрос:** В чем преимущества Gin перед стандартным net/http?

**Ответ:**
- Быстрый роутер на основе radix tree (httprouter)
- Встроенные middleware (Logger, Recovery)
- Удобный API для работы с JSON, формами, query параметрами
- Встроенная валидация через validator/v10
- Группировка маршрутов
- Более удобная работа с контекстом запроса

**Вопрос:** Как в Gin работает метод c.Abort()?

**Ответ:** `c.Abort()` останавливает выполнение цепочки middleware и хендлеров. Все middleware/хендлеры, которые были вызваны до Abort(), завершат свою работу (включая defer'ы), но следующие в цепочке вызваны не будут.

**Вопрос:** Разница между c.BindJSON() и c.ShouldBindJSON()?

**Ответ:**
- `c.BindJSON()` — при ошибке автоматически отправляет 400 Bad Request клиенту
- `c.ShouldBindJSON()` — возвращает ошибку, позволяя обработать её самостоятельно (предпочтительный вариант)

## Связанные темы

- [[Go - Пакет net-http]]
- [[REST API - Дизайн и best practices]]
- [[Gorilla Mux]]
- [[Echo]]
- [[Chi]]