# Безопасность API

Безопасность API — критически важный аспект разработки современных веб-приложений. Уязвимости в API могут привести к утечке данных, несанкционированному доступу и компрометации всей системы.

## OWASP API Security Top 10

OWASP (Open Web Application Security Project) публикует список наиболее критичных уязвимостей API.

### 1. Broken Object Level Authorization (BOLA)

Отсутствие проверки прав доступа к объектам — пользователь может получить доступ к чужим данным.

**Пример уязвимости:**
```go
// ❌ Уязвимо: нет проверки владельца
func getOrder(w http.ResponseWriter, r *http.Request) {
    orderID := r.URL.Query().Get("id")

    // Получаем заказ из БД без проверки прав
    order, _ := db.GetOrder(orderID)
    json.NewEncoder(w).Encode(order)
}

// Атака: GET /orders?id=123 (может быть чужой заказ!)
```

**✅ Правильно:**
```go
func getOrder(w http.ResponseWriter, r *http.Request) {
    userID := getUserIDFromToken(r) // Из JWT/сессии
    orderID := r.URL.Query().Get("id")

    order, err := db.GetOrder(orderID)
    if err != nil {
        http.Error(w, "Order not found", http.StatusNotFound)
        return
    }

    // Проверяем, что заказ принадлежит пользователю
    if order.UserID != userID {
        http.Error(w, "Forbidden", http.StatusForbidden)
        return
    }

    json.NewEncoder(w).Encode(order)
}
```

### 2. Broken Authentication

Неправильная реализация аутентификации.

**Проблемы:**
- Слабые пароли без проверки сложности
- Нет rate limiting на логин
- Токены без истечения срока
- Передача учётных данных в URL

**✅ Решения:**
```go
package main

import (
    "time"
    "sync"
)

// Rate limiting для защиты от brute-force
type RateLimiter struct {
    mu       sync.Mutex
    attempts map[string][]time.Time
}

func (rl *RateLimiter) AllowLogin(username string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    now := time.Now()
    cutoff := now.Add(-15 * time.Minute)

    // Оставляем только недавние попытки
    attempts := []time.Time{}
    for _, t := range rl.attempts[username] {
        if t.After(cutoff) {
            attempts = append(attempts, t)
        }
    }

    // Максимум 5 попыток за 15 минут
    if len(attempts) >= 5 {
        return false
    }

    attempts = append(attempts, now)
    rl.attempts[username] = attempts
    return true
}

// Валидация пароля
func validatePassword(password string) error {
    if len(password) < 8 {
        return errors.New("password must be at least 8 characters")
    }

    hasUpper := false
    hasLower := false
    hasDigit := false

    for _, char := range password {
        if unicode.IsUpper(char) { hasUpper = true }
        if unicode.IsLower(char) { hasLower = true }
        if unicode.IsDigit(char) { hasDigit = true }
    }

    if !hasUpper || !hasLower || !hasDigit {
        return errors.New("password must contain uppercase, lowercase and digit")
    }

    return nil
}
```

### 3. Excessive Data Exposure

API возвращает больше данных, чем нужно клиенту.

**Пример уязвимости:**
```go
// ❌ Возвращаем весь объект User (с паролем!)
type User struct {
    ID           int    `json:"id"`
    Username     string `json:"username"`
    Email        string `json:"email"`
    PasswordHash string `json:"password_hash"` // ❌
    SSN          string `json:"ssn"`          // ❌ Sensitive
}

func getUser(w http.ResponseWriter, r *http.Request) {
    user, _ := db.GetUser(123)
    json.NewEncoder(w).Encode(user) // Возвращаем ВСЁ
}
```

**✅ Правильно:**
```go
// Отдельная структура для API response
type UserResponse struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

func getUser(w http.ResponseWriter, r *http.Request) {
    user, _ := db.GetUser(123)

    // Возвращаем только необходимые поля
    response := UserResponse{
        ID:       user.ID,
        Username: user.Username,
        Email:    user.Email,
    }

    json.NewEncoder(w).Encode(response)
}
```

### 4. SQL Injection

Внедрение SQL кода через пользовательский ввод.

**Пример уязвимости:**
```go
// ❌ ОЧЕНЬ ОПАСНО!
func searchUsers(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query().Get("q")

    // SQL инъекция!
    sql := fmt.Sprintf("SELECT * FROM users WHERE username = '%s'", query)
    rows, _ := db.Query(sql)

    // Атака: ?q=' OR '1'='1 -- (вернёт всех пользователей)
}
```

**✅ Правильно:**
```go
// Используем параметризованные запросы
func searchUsers(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query().Get("q")

    // Безопасно: параметры экранируются
    rows, err := db.Query("SELECT * FROM users WHERE username = $1", query)
    if err != nil {
        http.Error(w, "Database error", http.StatusInternalServerError)
        return
    }
    defer rows.Close()

    // Обработка результатов...
}
```

### 5. Rate Limiting

Ограничение количества запросов для защиты от DoS и brute-force.

**Реализация:**
```go
package main

import (
    "net/http"
    "sync"
    "time"

    "golang.org/x/time/rate"
)

// Rate limiter на основе Token Bucket
type APIRateLimiter struct {
    limiters map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewAPIRateLimiter(r rate.Limit, b int) *APIRateLimiter {
    return &APIRateLimiter{
        limiters: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    b,
    }
}

func (rl *APIRateLimiter) getLimiter(key string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    limiter, exists := rl.limiters[key]
    if !exists {
        limiter = rate.NewLimiter(rl.rate, rl.burst)
        rl.limiters[key] = limiter
    }

    return limiter
}

// Middleware
func (rl *APIRateLimiter) Middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Используем IP как ключ
        key := r.RemoteAddr

        limiter := rl.getLimiter(key)

        if !limiter.Allow() {
            http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}

func main() {
    // 10 запросов в секунду, burst до 20
    limiter := NewAPIRateLimiter(rate.Limit(10), 20)

    mux := http.NewServeMux()
    mux.HandleFunc("/api/data", dataHandler)

    http.ListenAndServe(":8080", limiter.Middleware(mux))
}
```

## Input Validation

Валидация всех входных данных критически важна.

### Основные принципы

```go
package main

import (
    "errors"
    "net/mail"
    "regexp"
)

// Валидация email
func validateEmail(email string) error {
    _, err := mail.ParseAddress(email)
    if err != nil {
        return errors.New("invalid email format")
    }
    return nil
}

// Валидация username (только буквы, цифры, подчеркивание)
func validateUsername(username string) error {
    if len(username) < 3 || len(username) > 20 {
        return errors.New("username must be 3-20 characters")
    }

    matched, _ := regexp.MatchString("^[a-zA-Z0-9_]+$", username)
    if !matched {
        return errors.New("username can only contain letters, numbers and underscore")
    }

    return nil
}

// Валидация URL
func validateURL(urlString string) error {
    u, err := url.Parse(urlString)
    if err != nil {
        return errors.New("invalid URL")
    }

    // Только http/https
    if u.Scheme != "http" && u.Scheme != "https" {
        return errors.New("only http/https URLs allowed")
    }

    return nil
}

// Whitelist валидация (предпочтительнее blacklist)
func validateRole(role string) error {
    allowedRoles := map[string]bool{
        "user":  true,
        "admin": true,
        "moderator": true,
    }

    if !allowedRoles[role] {
        return errors.New("invalid role")
    }

    return nil
}

// Middleware для валидации JSON
func validateJSONMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        contentType := r.Header.Get("Content-Type")

        if r.Method == "POST" || r.Method == "PUT" {
            if contentType != "application/json" {
                http.Error(w, "Content-Type must be application/json", http.StatusBadRequest)
                return
            }
        }

        next.ServeHTTP(w, r)
    })
}
```

### Sanitization (очистка данных)

```go
package main

import (
    "html"
    "strings"
)

// Очистка от HTML тегов (XSS protection)
func sanitizeHTML(input string) string {
    return html.EscapeString(input)
}

// Очистка строк
func sanitizeString(input string) string {
    // Убираем пробелы по краям
    input = strings.TrimSpace(input)

    // Убираем null bytes
    input = strings.ReplaceAll(input, "\x00", "")

    // Ограничиваем длину
    if len(input) > 1000 {
        input = input[:1000]
    }

    return input
}
```

## CORS (Cross-Origin Resource Sharing)

Управление доступом к API с других доменов.

```go
package main

import (
    "net/http"
)

func corsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        origin := r.Header.Get("Origin")

        // Whitelist доменов
        allowedOrigins := map[string]bool{
            "https://myapp.com":     true,
            "https://app.myapp.com": true,
        }

        if allowedOrigins[origin] {
            w.Header().Set("Access-Control-Allow-Origin", origin)
        }

        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization")
        w.Header().Set("Access-Control-Max-Age", "3600")
        w.Header().Set("Access-Control-Allow-Credentials", "true")

        // Preflight request
        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}

// Разрешить все домены (только для публичных API!)
func corsAllowAll(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Access-Control-Allow-Origin", "*")
        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Content-Type")

        if r.Method == "OPTIONS" {
            w.WriteHeader(http.StatusOK)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

## Security Headers

Важные HTTP заголовки для безопасности.

```go
func securityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // HSTS: принудительный HTTPS
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")

        // XSS Protection
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "1; mode=block")

        // CSP: Content Security Policy
        w.Header().Set("Content-Security-Policy", "default-src 'self'")

        // Referrer Policy
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")

        // Permissions Policy
        w.Header().Set("Permissions-Policy", "geolocation=(), microphone=(), camera=()")

        next.ServeHTTP(w, r)
    })
}
```

## Logging и Monitoring

Логирование для безопасности и мониторинга.

```go
package main

import (
    "log"
    "net/http"
    "time"
)

type SecurityLogger struct {
    logger *log.Logger
}

func (sl *SecurityLogger) LogSecurityEvent(event, details string, req *http.Request) {
    sl.logger.Printf(
        "SECURITY_EVENT: %s | IP: %s | UserAgent: %s | Path: %s | Details: %s",
        event,
        req.RemoteAddr,
        req.UserAgent(),
        req.URL.Path,
        details,
    )
}

// Middleware для логирования подозрительной активности
func securityLoggingMiddleware(logger *SecurityLogger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()

            // Подозрительные паттерны в URL
            suspiciousPatterns := []string{
                "../", "..\\",  // Path traversal
                "<script>",     // XSS attempts
                "' OR '1'='1",  // SQL injection
                "UNION SELECT", // SQL injection
            }

            for _, pattern := range suspiciousPatterns {
                if strings.Contains(r.URL.String(), pattern) {
                    logger.LogSecurityEvent(
                        "SUSPICIOUS_REQUEST",
                        fmt.Sprintf("Pattern detected: %s", pattern),
                        r,
                    )
                }
            }

            // Создаём custom ResponseWriter для перехвата статус кода
            rw := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

            next.ServeHTTP(rw, r)

            // Логируем 401/403
            if rw.statusCode == http.StatusUnauthorized || rw.statusCode == http.StatusForbidden {
                logger.LogSecurityEvent(
                    "UNAUTHORIZED_ACCESS",
                    fmt.Sprintf("Status: %d", rw.statusCode),
                    r,
                )
            }

            duration := time.Since(start)

            // Логируем медленные запросы (возможна DoS атака)
            if duration > 5*time.Second {
                logger.LogSecurityEvent(
                    "SLOW_REQUEST",
                    fmt.Sprintf("Duration: %v", duration),
                    r,
                )
            }
        })
    }
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}
```

## API Versioning

Версионирование для безопасной эволюции API.

```go
// URL versioning
// GET /api/v1/users
// GET /api/v2/users

func setupRoutes() {
    mux := http.NewServeMux()

    // v1 - старая версия
    mux.HandleFunc("/api/v1/users", getUsersV1)

    // v2 - новая версия
    mux.HandleFunc("/api/v2/users", getUsersV2)
}

// Header versioning
func versionMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        version := r.Header.Get("API-Version")

        if version == "" {
            version = "v1" // default
        }

        ctx := context.WithValue(r.Context(), "api-version", version)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Защита от распространённых атак

### XSS (Cross-Site Scripting)

```go
// Всегда экранировать пользовательский ввод
import "html"

func renderUser(username string) string {
    return html.EscapeString(username)
}

// В JSON API: encoding/json автоматически экранирует
```

### CSRF (Cross-Site Request Forgery)

```go
// CSRF tokens для форм
func generateCSRFToken() string {
    bytes := make([]byte, 32)
    rand.Read(bytes)
    return base64.URLEncoding.EncodeToString(bytes)
}

// SameSite cookie attribute
http.SetCookie(w, &http.Cookie{
    Name:     "session",
    Value:    sessionID,
    SameSite: http.SameSiteStrictMode, // Защита от CSRF
})
```

### Path Traversal

```go
// ❌ Уязвимо
func serveFile(w http.ResponseWriter, r *http.Request) {
    filename := r.URL.Query().Get("file")
    data, _ := ioutil.ReadFile(filename)
    w.Write(data)
    // Атака: ?file=../../../../etc/passwd
}

// ✅ Правильно
func serveFile(w http.ResponseWriter, r *http.Request) {
    filename := filepath.Base(r.URL.Query().Get("file")) // Только имя файла
    safePath := filepath.Join("/safe/directory", filename)

    // Проверяем, что путь внутри безопасной директории
    if !strings.HasPrefix(safePath, "/safe/directory") {
        http.Error(w, "Invalid path", http.StatusBadRequest)
        return
    }

    data, err := ioutil.ReadFile(safePath)
    if err != nil {
        http.Error(w, "File not found", http.StatusNotFound)
        return
    }

    w.Write(data)
}
```

## Best Practices

1. ✅ **Используйте HTTPS везде**
2. ✅ **Валидируйте все входные данные** (whitelist подход)
3. ✅ **Используйте параметризованные SQL запросы**
4. ✅ **Реализуйте rate limiting** на всех endpoint'ах
5. ✅ **Проверяйте авторизацию** на каждом запросе
6. ✅ **Возвращайте минимум данных** (не всю модель)
7. ✅ **Логируйте security events**
8. ✅ **Используйте security headers**
9. ✅ **Храните секреты в env variables**, не в коде
10. ✅ **Регулярно обновляйте зависимости**

## Инструменты для тестирования

### Статический анализ

```bash
# gosec - анализатор безопасности
go install github.com/securego/gosec/v2/cmd/gosec@latest
gosec ./...

# govulncheck - проверка уязвимостей в зависимостях
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

### Динамическое тестирование

- **OWASP ZAP** — автоматическое сканирование уязвимостей
- **Burp Suite** — перехват и анализ HTTP запросов
- **Postman** — тестирование API

## Вопросы с собеседований

**Вопрос 1:** Что такое OWASP Top 10 и почему это важно?

**Ответ:** OWASP Top 10 — это список десяти наиболее критичных угроз безопасности веб-приложений, обновляемый раз в несколько лет. Это важно, потому что даёт приоритизацию: эти уязвимости наиболее распространены и опасны. Зная Top 10, разработчик может сфокусироваться на защите от самых частых атак.

**Вопрос 2:** Как защититься от SQL injection?

**Ответ:**
1. **Параметризованные запросы** (prepared statements) — основной метод
2. **ORM** — дополнительная защита (GORM, sqlx)
3. **Валидация входных данных** — whitelist подход
4. **Principle of Least Privilege** — ограничить права БД пользователя
5. **WAF** (Web Application Firewall) — дополнительный слой защиты

**Вопрос 3:** В чём разница между аутентификации и авторизации?

**Ответ:**
- **Аутентификация** (Authentication) — проверка личности ("кто ты?"). Логин с паролем, JWT токен.
- **Авторизация** (Authorization) — проверка прав ("что ты можешь делать?"). Проверка ролей, permissions.

Пример: JWT токен подтверждает, что ты пользователь #123 (аутентификация), но только проверка роли скажет, можешь ли ты удалить пост (авторизация).

**Вопрос 4:** Что такое Rate Limiting и почему это важно?

**Ответ:** Rate Limiting — ограничение количества запросов от одного клиента за период времени. Защищает от:
- **DoS/DDoS атак** — перегрузка сервера
- **Brute-force** — подбор паролей
- **Scraping** — массовый сбор данных
- **Abuse** — злоупотребление API

Реализуется через Token Bucket, Leaky Bucket или Fixed Window алгоритмы.

## Связанные темы

- [[REST API - Дизайн и best practices]]
- [[JWT (JSON Web Tokens)]]
- [[OAuth 2.0]]
- [[Basic Authentication]]
- [[Session-based аутентификация]]
- [[HTTPS и TLS]]
- [[PostgreSQL - Основы]]
- [[Go - Пакет net-http]]
- [[HTTP протокол]]
