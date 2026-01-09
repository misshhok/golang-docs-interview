# Go - Пакет net/http

Стандартный пакет для создания HTTP клиентов и серверов.

## HTTP Server

### Простейший сервер

```go
import "net/http"

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello, World!"))
    })

    http.ListenAndServe(":8080", nil)
}
```

### Handler интерфейс

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// Пример
type HelloHandler struct{}

func (h HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hello!"))
}

http.Handle("/hello", HelloHandler{})
```

### HandlerFunc

```go
// HandlerFunc - адаптер функции к Handler
func greet(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Hi!"))
}

http.HandleFunc("/greet", greet)
// Эквивалентно: http.Handle("/greet", http.HandlerFunc(greet))
```

## Request

### Основные поля

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // HTTP метод
    r.Method // "GET", "POST", "PUT", etc.

    // URL
    r.URL.Path       // "/users/123"
    r.URL.RawQuery   // "page=1&limit=10"

    // Headers
    r.Header.Get("Content-Type")
    r.Header.Get("Authorization")

    // Host
    r.Host // "example.com"

    // Remote address
    r.RemoteAddr // "192.168.1.1:12345"

    // Context
    ctx := r.Context()
}
```

### Query параметры

```go
// URL: /search?q=golang&page=2

q := r.URL.Query().Get("q")           // "golang"
page := r.URL.Query().Get("page")     // "2"

// Все значения для ключа
tags := r.URL.Query()["tag"]          // []string{"go", "web"}

// Проверка наличия
if _, ok := r.URL.Query()["debug"]; ok {
    // debug mode
}
```

### Path параметры (с mux)

```go
// С gorilla/mux или chi
import "github.com/gorilla/mux"

router := mux.NewRouter()
router.HandleFunc("/users/{id}", func(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id := vars["id"] // "123"
})
```

### Чтение Body

```go
// JSON
var user User
if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
    return
}
defer r.Body.Close()

// Весь body в []byte
body, err := io.ReadAll(r.Body)

// Form data
r.ParseForm()
username := r.FormValue("username")
password := r.FormValue("password")
```

### Multipart Form / File Upload

```go
// Парсим multipart form (до 10 MB в памяти)
r.ParseMultipartForm(10 << 20)

// Получаем файл
file, header, err := r.FormFile("upload")
if err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
    return
}
defer file.Close()

// Сохраняем файл
dst, _ := os.Create(header.Filename)
defer dst.Close()
io.Copy(dst, file)
```

## ResponseWriter

### Запись ответа

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Установить заголовки
    w.Header().Set("Content-Type", "application/json")
    w.Header().Set("X-Custom", "value")

    // Установить статус код (должно быть ДО Write!)
    w.WriteHeader(http.StatusOK) // 200

    // Записать body
    w.Write([]byte(`{"message":"success"}`))
}
```

### JSON ответ

```go
func jsonResponse(w http.ResponseWriter, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(data)
}

// Использование
jsonResponse(w, map[string]string{"status": "ok"})
```

### Ошибки

```go
// Простая ошибка
http.Error(w, "Not Found", http.StatusNotFound)

// JSON ошибка
func jsonError(w http.ResponseWriter, message string, code int) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(code)
    json.NewEncoder(w).Encode(map[string]string{
        "error": message,
    })
}
```

### Редирект

```go
http.Redirect(w, r, "/new-url", http.StatusMovedPermanently) // 301
http.Redirect(w, r, "/login", http.StatusSeeOther)           // 303
http.Redirect(w, r, "/temp", http.StatusTemporaryRedirect)   // 307
```

## ServeMux (роутер)

### Базовый роутинг

```go
mux := http.NewServeMux()

// Exact match
mux.HandleFunc("/", homeHandler)
mux.HandleFunc("/about", aboutHandler)

// Prefix match
mux.HandleFunc("/api/", apiHandler) // Все /api/*

http.ListenAndServe(":8080", mux)
```

### Ограничения встроенного mux

- ❌ Нет path параметров (/users/{id})
- ❌ Нет фильтрации по HTTP методу
- ❌ Нет middleware из коробки

**Решение:** используйте gorilla/mux, chi, gin, echo

## HTTP Client

### Простой GET

```go
resp, err := http.Get("https://api.example.com/data")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

body, err := io.ReadAll(resp.Body)
fmt.Println(string(body))
```

### POST с JSON

```go
user := User{Name: "Alice", Age: 25}
jsonData, _ := json.Marshal(user)

resp, err := http.Post(
    "https://api.example.com/users",
    "application/json",
    bytes.NewBuffer(jsonData),
)
defer resp.Body.Close()
```

### Кастомный Request

```go
req, err := http.NewRequest("PUT", url, body)
if err != nil {
    return err
}

// Заголовки
req.Header.Set("Content-Type", "application/json")
req.Header.Set("Authorization", "Bearer "+token)

// Query параметры
q := req.URL.Query()
q.Add("page", "1")
q.Add("limit", "10")
req.URL.RawQuery = q.Encode()

// Выполнение
client := &http.Client{Timeout: 10 * time.Second}
resp, err := client.Do(req)
defer resp.Body.Close()
```

### Client с таймаутом

```go
client := &http.Client{
    Timeout: 10 * time.Second,
}

resp, err := client.Get(url)
```

### Context для отмены

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)

client := &http.Client{}
resp, err := client.Do(req)
if err != nil {
    // Может быть context deadline exceeded
    return err
}
defer resp.Body.Close()
```

## Middleware

### Базовый middleware

```go
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        log.Printf("Started %s %s", r.Method, r.URL.Path)

        next.ServeHTTP(w, r)

        log.Printf("Completed in %v", time.Since(start))
    })
}

// Использование
mux := http.NewServeMux()
mux.HandleFunc("/", homeHandler)

handler := loggingMiddleware(mux)
http.ListenAndServe(":8080", handler)
```

### Цепочка middleware

```go
func middleware1(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Println("Middleware 1 - before")
        next.ServeHTTP(w, r)
        log.Println("Middleware 1 - after")
    })
}

func middleware2(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Println("Middleware 2 - before")
        next.ServeHTTP(w, r)
        log.Println("Middleware 2 - after")
    })
}

// Цепочка
handler := middleware1(middleware2(mux))
```

### Популярные middleware

```go
// Authentication
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}

// CORS
func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}

// Recovery
func recoveryMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic: %v", err)
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

## Graceful Shutdown

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", handler)

    server := &http.Server{
        Addr:    ":8080",
        Handler: mux,
    }

    // Запуск сервера в горутине
    go func() {
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()

    // Ожидание сигнала
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down server...")

    // Graceful shutdown с таймаутом
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    log.Println("Server exited")
}
```

## Cookies

```go
// Установить cookie
http.SetCookie(w, &http.Cookie{
    Name:     "session_id",
    Value:    "abc123",
    Path:     "/",
    MaxAge:   3600,
    HttpOnly: true,
    Secure:   true,
    SameSite: http.SameSiteStrictMode,
})

// Прочитать cookie
cookie, err := r.Cookie("session_id")
if err != nil {
    // Cookie не найдена
}
value := cookie.Value
```

## Status Codes

```go
// 2xx Success
http.StatusOK                  // 200
http.StatusCreated             // 201
http.StatusNoContent           // 204

// 3xx Redirection
http.StatusMovedPermanently    // 301
http.StatusFound               // 302
http.StatusSeeOther            // 303
http.StatusTemporaryRedirect   // 307

// 4xx Client Error
http.StatusBadRequest          // 400
http.StatusUnauthorized        // 401
http.StatusForbidden           // 403
http.StatusNotFound            // 404
http.StatusMethodNotAllowed    // 405

// 5xx Server Error
http.StatusInternalServerError // 500
http.StatusNotImplemented      // 501
http.StatusBadGateway          // 502
http.StatusServiceUnavailable  // 503
```

## Best Practices

1. ✅ Всегда defer Body.Close() для клиента
2. ✅ Используйте context для таймаутов
3. ✅ Устанавливайте Content-Type для ответов
4. ✅ Используйте middleware для общей логики
5. ✅ Graceful shutdown для продакшена
6. ✅ Валидируйте входные данные
7. ✅ Используйте HTTPS в продакшене
8. ❌ Не игнорируйте ошибки
9. ❌ Не забывайте WriteHeader перед Write

## Связанные темы

- [[Go - Простой web-сервер]]
- [[Go - TLS и HTTPS]]
- [[Go - Context]]
- [[Go - Пакет encoding-json]]
- [[REST API - Основы]]
