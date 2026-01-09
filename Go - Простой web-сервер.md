# Go - Простой web-сервер

Практический пример создания HTTP сервера с роутингом, middleware и обработкой ошибок.

## Минимальный сервер

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })

    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", nil)
}
```

## REST API с CRUD

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "strconv"
    "sync"

    "github.com/gorilla/mux"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
    Email string `json:"email"`
}

type Server struct {
    mu    sync.RWMutex
    users map[int]User
    nextID int
}

func NewServer() *Server {
    return &Server{
        users: make(map[int]User),
        nextID: 1,
    }
}

// GET /users - список всех пользователей
func (s *Server) listUsers(w http.ResponseWriter, r *http.Request) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    users := make([]User, 0, len(s.users))
    for _, user := range s.users {
        users = append(users, user)
    }

    respondJSON(w, http.StatusOK, users)
}

// GET /users/{id} - получить пользователя
func (s *Server) getUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "Invalid user ID")
        return
    }

    s.mu.RLock()
    user, exists := s.users[id]
    s.mu.RUnlock()

    if !exists {
        respondError(w, http.StatusNotFound, "User not found")
        return
    }

    respondJSON(w, http.StatusOK, user)
}

// POST /users - создать пользователя
func (s *Server) createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }

    if user.Name == "" || user.Email == "" {
        respondError(w, http.StatusBadRequest, "Name and email are required")
        return
    }

    s.mu.Lock()
    user.ID = s.nextID
    s.users[user.ID] = user
    s.nextID++
    s.mu.Unlock()

    respondJSON(w, http.StatusCreated, user)
}

// PUT /users/{id} - обновить пользователя
func (s *Server) updateUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "Invalid user ID")
        return
    }

    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid request body")
        return
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    if _, exists := s.users[id]; !exists {
        respondError(w, http.StatusNotFound, "User not found")
        return
    }

    user.ID = id
    s.users[id] = user

    respondJSON(w, http.StatusOK, user)
}

// DELETE /users/{id} - удалить пользователя
func (s *Server) deleteUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id, err := strconv.Atoi(vars["id"])
    if err != nil {
        respondError(w, http.StatusBadRequest, "Invalid user ID")
        return
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    if _, exists := s.users[id]; !exists {
        respondError(w, http.StatusNotFound, "User not found")
        return
    }

    delete(s.users, id)
    w.WriteHeader(http.StatusNoContent)
}

// Helper функции
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}

// Роутинг
func (s *Server) routes() http.Handler {
    r := mux.NewRouter()

    // Middleware
    r.Use(loggingMiddleware)
    r.Use(recoveryMiddleware)

    // Routes
    r.HandleFunc("/users", s.listUsers).Methods("GET")
    r.HandleFunc("/users/{id}", s.getUser).Methods("GET")
    r.HandleFunc("/users", s.createUser).Methods("POST")
    r.HandleFunc("/users/{id}", s.updateUser).Methods("PUT")
    r.HandleFunc("/users/{id}", s.deleteUser).Methods("DELETE")

    return r
}

// Middleware
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("%s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

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

func main() {
    server := NewServer()

    log.Println("Server starting on :8080")
    if err := http.ListenAndServe(":8080", server.routes()); err != nil {
        log.Fatal(err)
    }
}
```

## С базой данных (PostgreSQL)

```go
package main

import (
    "database/sql"
    "encoding/json"
    "log"
    "net/http"

    _ "github.com/lib/pq"
    "github.com/gorilla/mux"
)

type Server struct {
    db *sql.DB
}

func NewServer(db *sql.DB) *Server {
    return &Server{db: db}
}

func (s *Server) getUser(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    id := vars["id"]

    var user User
    err := s.db.QueryRow(
        "SELECT id, name, email FROM users WHERE id = $1",
        id,
    ).Scan(&user.ID, &user.Name, &user.Email)

    if err == sql.ErrNoRows {
        respondError(w, http.StatusNotFound, "User not found")
        return
    }
    if err != nil {
        log.Printf("Database error: %v", err)
        respondError(w, http.StatusInternalServerError, "Internal error")
        return
    }

    respondJSON(w, http.StatusOK, user)
}

func (s *Server) createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        respondError(w, http.StatusBadRequest, "Invalid request")
        return
    }

    err := s.db.QueryRow(
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
        user.Name, user.Email,
    ).Scan(&user.ID)

    if err != nil {
        log.Printf("Database error: %v", err)
        respondError(w, http.StatusInternalServerError, "Internal error")
        return
    }

    respondJSON(w, http.StatusCreated, user)
}

func main() {
    // Подключение к БД
    db, err := sql.Open("postgres", "postgres://user:pass@localhost/dbname?sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    if err := db.Ping(); err != nil {
        log.Fatal(err)
    }

    server := NewServer(db)

    r := mux.NewRouter()
    r.HandleFunc("/users/{id}", server.getUser).Methods("GET")
    r.HandleFunc("/users", server.createUser).Methods("POST")

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", r))
}
```

## Статические файлы

```go
func main() {
    // Serve static files из директории ./static
    fs := http.FileServer(http.Dir("./static"))
    http.Handle("/static/", http.StripPrefix("/static/", fs))

    // API routes
    http.HandleFunc("/api/users", usersHandler)

    http.ListenAndServe(":8080", nil)
}
```

## HTML Templates

```go
import "html/template"

func homeHandler(w http.ResponseWriter, r *http.Request) {
    tmpl := template.Must(template.ParseFiles("templates/home.html"))

    data := struct {
        Title string
        Users []User
    }{
        Title: "Users List",
        Users: getUsers(),
    }

    tmpl.Execute(w, data)
}
```

## Конфигурация

```go
type Config struct {
    Port     string
    DBHost   string
    DBPort   string
    DBUser   string
    DBPass   string
    DBName   string
}

func loadConfig() (*Config, error) {
    return &Config{
        Port:   getEnv("PORT", "8080"),
        DBHost: getEnv("DB_HOST", "localhost"),
        DBPort: getEnv("DB_PORT", "5432"),
        DBUser: getEnv("DB_USER", "postgres"),
        DBPass: getEnv("DB_PASS", ""),
        DBName: getEnv("DB_NAME", "mydb"),
    }, nil
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

## Graceful Shutdown

```go
func main() {
    server := NewServer()

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      server.routes(),
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Запуск в горутине
    go func() {
        log.Println("Server starting on :8080")
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server error: %v", err)
        }
    }()

    // Graceful shutdown
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down server...")

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatalf("Server forced to shutdown: %v", err)
    }

    log.Println("Server stopped")
}
```

## Структура проекта

```
myapp/
├── main.go
├── config/
│   └── config.go
├── handlers/
│   ├── users.go
│   └── auth.go
├── middleware/
│   ├── logging.go
│   ├── auth.go
│   └── cors.go
├── models/
│   └── user.go
├── repository/
│   └── user_repository.go
├── static/
│   ├── css/
│   └── js/
└── templates/
    └── home.html
```

## Best Practices

1. ✅ Используйте роутер (gorilla/mux, chi)
2. ✅ Middleware для общей логики
3. ✅ Graceful shutdown
4. ✅ Таймауты для сервера
5. ✅ Структурируйте код (handlers, models, etc.)
6. ✅ Валидация входных данных
7. ✅ Логирование ошибок
8. ✅ Recovery middleware
9. ✅ Конфигурация через переменные окружения
10. ❌ Не игнорируйте ошибки

## Связанные темы

- [[Go - Пакет net-http]]
- [[REST API - Основы]]
- [[Go - Context]]
- [[Go - Пакет encoding-json]]
- [[PostgreSQL - Основы]]
