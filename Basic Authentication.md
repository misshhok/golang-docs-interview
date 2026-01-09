# Basic Authentication

Basic Authentication — это простейший метод HTTP-аутентификации, при котором клиент отправляет учётные данные (логин и пароль) в заголовке Authorization в формате Base64.

## Основные концепции

Basic Auth передаёт имя пользователя и пароль в каждом HTTP-запросе. Несмотря на простоту, этот метод требует HTTPS для безопасной передачи данных.

### Формат заголовка

```
Authorization: Basic <base64(username:password)>
```

**Пример:**
```
Username: admin
Password: password123

Строка для кодирования: admin:password123
Base64: YWRtaW46cGFzc3dvcmQxMjM=

Заголовок:
Authorization: Basic YWRtaW46cGFzc3dvcmQxMjM=
```

## Реализация в Go

### Сервер с Basic Authentication

```go
package main

import (
    "crypto/subtle"
    "encoding/base64"
    "fmt"
    "log"
    "net/http"
    "strings"
)

// Учётные данные (в продакшене хранить в БД с хешированием)
var users = map[string]string{
    "admin":     "admin123",
    "user":      "password",
    "developer": "dev123",
}

// Middleware для Basic Authentication
func BasicAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Получаем заголовок Authorization
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            requireAuth(w)
            return
        }

        // Проверяем префикс "Basic "
        if !strings.HasPrefix(authHeader, "Basic ") {
            http.Error(w, "Invalid authorization header", http.StatusUnauthorized)
            return
        }

        // Декодируем Base64
        encodedCredentials := strings.TrimPrefix(authHeader, "Basic ")
        decodedBytes, err := base64.StdEncoding.DecodeString(encodedCredentials)
        if err != nil {
            http.Error(w, "Invalid base64 encoding", http.StatusUnauthorized)
            return
        }

        // Разделяем на username:password
        credentials := string(decodedBytes)
        parts := strings.SplitN(credentials, ":", 2)
        if len(parts) != 2 {
            http.Error(w, "Invalid credentials format", http.StatusUnauthorized)
            return
        }

        username := parts[0]
        password := parts[1]

        // Проверяем учётные данные
        if !validateCredentials(username, password) {
            requireAuth(w)
            return
        }

        // Сохраняем username в контекст для использования в handlers
        ctx := context.WithValue(r.Context(), "username", username)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Проверка учётных данных
func validateCredentials(username, password string) bool {
    expectedPassword, exists := users[username]
    if !exists {
        return false
    }

    // Используем constant-time comparison для защиты от timing attacks
    return subtle.ConstantTimeCompare([]byte(password), []byte(expectedPassword)) == 1
}

// Требовать аутентификацию (возвращает 401)
func requireAuth(w http.ResponseWriter) {
    w.Header().Set("WWW-Authenticate", `Basic realm="Restricted"`)
    http.Error(w, "Unauthorized", http.StatusUnauthorized)
}

// Защищённый ресурс
func protectedHandler(w http.ResponseWriter, r *http.Request) {
    username := r.Context().Value("username").(string)
    fmt.Fprintf(w, "Hello, %s! Welcome to protected resource.", username)
}

func main() {
    mux := http.NewServeMux()

    // Публичный endpoint
    mux.HandleFunc("/public", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "This is public")
    })

    // Защищённый endpoint
    protectedResource := http.HandlerFunc(protectedHandler)
    mux.Handle("/protected", BasicAuthMiddleware(protectedResource))

    log.Println("Server started at :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### Клиент с Basic Authentication

```go
package main

import (
    "encoding/base64"
    "fmt"
    "io"
    "net/http"
)

// Функция для добавления Basic Auth к запросу
func addBasicAuth(req *http.Request, username, password string) {
    credentials := username + ":" + password
    encoded := base64.StdEncoding.EncodeToString([]byte(credentials))
    req.Header.Set("Authorization", "Basic "+encoded)
}

// Вариант 1: Вручную добавляем заголовок
func requestWithManualAuth() {
    req, err := http.NewRequest("GET", "http://localhost:8080/protected", nil)
    if err != nil {
        log.Fatal(err)
    }

    // Добавляем Basic Auth
    addBasicAuth(req, "admin", "admin123")

    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Printf("Status: %s\nBody: %s\n", resp.Status, body)
}

// Вариант 2: Используем встроенный метод SetBasicAuth
func requestWithBuiltinAuth() {
    req, err := http.NewRequest("GET", "http://localhost:8080/protected", nil)
    if err != nil {
        log.Fatal(err)
    }

    // Встроенный метод Go для Basic Auth
    req.SetBasicAuth("admin", "admin123")

    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Printf("Status: %s\nBody: %s\n", resp.Status, body)
}

func main() {
    // Оба варианта работают одинаково
    requestWithManualAuth()
    requestWithBuiltinAuth()
}
```

## Практический пример: Basic Auth с базой данных

```go
package main

import (
    "context"
    "database/sql"
    "net/http"

    "golang.org/x/crypto/bcrypt"
)

type AuthService struct {
    db *sql.DB
}

// Проверка учётных данных с хешированием паролей
func (s *AuthService) ValidateUser(ctx context.Context, username, password string) (bool, error) {
    var hashedPassword string

    // Получаем хешированный пароль из БД
    query := "SELECT password_hash FROM users WHERE username = $1 AND is_active = true"
    err := s.db.QueryRowContext(ctx, query, username).Scan(&hashedPassword)

    if err == sql.ErrNoRows {
        return false, nil // Пользователь не найден
    }
    if err != nil {
        return false, err
    }

    // Сравниваем пароли с использованием bcrypt
    err = bcrypt.CompareHashAndPassword([]byte(hashedPassword), []byte(password))
    if err != nil {
        return false, nil // Неверный пароль
    }

    return true, nil
}

// Middleware с использованием БД
func (s *AuthService) BasicAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        username, password, ok := r.BasicAuth()
        if !ok {
            requireAuth(w)
            return
        }

        // Проверяем в БД
        valid, err := s.ValidateUser(r.Context(), username, password)
        if err != nil {
            http.Error(w, "Internal server error", http.StatusInternalServerError)
            return
        }

        if !valid {
            requireAuth(w)
            return
        }

        ctx := context.WithValue(r.Context(), "username", username)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

## Использование с curl

```bash
# Вариант 1: Флаг -u
curl -u admin:admin123 http://localhost:8080/protected

# Вариант 2: Заголовок вручную
curl -H "Authorization: Basic YWRtaW46YWRtaW4xMjM=" http://localhost:8080/protected

# Генерация Base64 для Basic Auth
echo -n "admin:admin123" | base64
# Результат: YWRtaW46YWRtaW4xMjM=
```

## Когда использовать Basic Authentication

### ✅ Подходит для:

1. **Внутренние API** (не публичные)
2. **Тестовые окружения**
3. **Простые административные панели**
4. **API с дополнительной защитой** (IP whitelist, VPN)
5. **Временные решения** во время разработки

### ❌ Не подходит для:

1. **Публичные API**
2. **Мобильные приложения** (учётные данные легко извлечь)
3. **Single Page Applications**
4. **Высоконагруженные системы** (проверка на каждом запросе)

## Проблемы безопасности

### ❌ Проблема 1: Передача в открытом виде

```go
// ❌ HTTP — пароль можно перехватить
http://api.example.com/data
Authorization: Basic YWRtaW46cGFzc3dvcmQ=
```

**✅ Решение:** Всегда используйте HTTPS

```go
// ✅ HTTPS — защищённая передача
https://api.example.com/data
```

### ❌ Проблема 2: Хранение паролей в открытом виде

```go
// ❌ Плохо
var users = map[string]string{
    "admin": "password123",
}
```

**✅ Решение:** Хешировать пароли с помощью bcrypt

```go
// ✅ Правильно
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

func checkPassword(hashedPassword, password string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hashedPassword), []byte(password))
    return err == nil
}
```

### ❌ Проблема 3: Timing attacks

```go
// ❌ Уязвимо к timing attack
if password == expectedPassword {
    return true
}
```

**✅ Решение:** Используйте constant-time comparison

```go
// ✅ Защита от timing attacks
import "crypto/subtle"

if subtle.ConstantTimeCompare([]byte(password), []byte(expectedPassword)) == 1 {
    return true
}
```

### ❌ Проблема 4: Нет механизма выхода (logout)

Basic Auth не имеет встроенного механизма для "выхода". Браузер кеширует учётные данные и отправляет их автоматически.

**✅ Решение:** Использовать token-based аутентификацию (JWT, OAuth)

## Улучшение безопасности Basic Auth

### 1. Rate Limiting

```go
package main

import (
    "net/http"
    "sync"
    "time"
)

type RateLimiter struct {
    mu       sync.Mutex
    attempts map[string][]time.Time
    limit    int
    window   time.Duration
}

func NewRateLimiter(limit int, window time.Duration) *RateLimiter {
    return &RateLimiter{
        attempts: make(map[string][]time.Time),
        limit:    limit,
        window:   window,
    }
}

func (rl *RateLimiter) Allow(username string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    now := time.Now()
    cutoff := now.Add(-rl.window)

    // Удаляем старые попытки
    attempts := rl.attempts[username]
    validAttempts := []time.Time{}
    for _, t := range attempts {
        if t.After(cutoff) {
            validAttempts = append(validAttempts, t)
        }
    }

    // Проверяем лимит
    if len(validAttempts) >= rl.limit {
        return false
    }

    // Добавляем новую попытку
    validAttempts = append(validAttempts, now)
    rl.attempts[username] = validAttempts

    return true
}

// Middleware с rate limiting
func BasicAuthWithRateLimit(limiter *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            username, password, ok := r.BasicAuth()

            // Проверяем rate limit перед валидацией
            if !limiter.Allow(r.RemoteAddr) {
                http.Error(w, "Too many requests", http.StatusTooManyRequests)
                return
            }

            if !ok || !validateCredentials(username, password) {
                requireAuth(w)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

### 2. IP Whitelist

```go
func IPWhitelistMiddleware(allowedIPs []string) func(http.Handler) http.Handler {
    whitelist := make(map[string]bool)
    for _, ip := range allowedIPs {
        whitelist[ip] = true
    }

    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            clientIP := r.RemoteAddr
            // Убираем порт
            if idx := strings.LastIndex(clientIP, ":"); idx != -1 {
                clientIP = clientIP[:idx]
            }

            if !whitelist[clientIP] {
                http.Error(w, "Forbidden", http.StatusForbidden)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

## Сравнение с другими методами

| Метод | Простота | Безопасность | Stateless | Logout |
|-------|----------|--------------|-----------|--------|
| Basic Auth | ✅ Очень просто | ⚠️ Требует HTTPS | ✅ Да | ❌ Нет |
| Session-based | ⚠️ Средне | ✅ Хорошая | ❌ Нет | ✅ Да |
| JWT | ⚠️ Средне | ✅ Хорошая | ✅ Да | ⚠️ Сложно |
| OAuth 2.0 | ❌ Сложно | ✅ Отличная | ✅ Да | ✅ Да |

## Типичные ошибки

### ❌ Ошибка 1: Использование HTTP вместо HTTPS

```go
// ❌ Небезопасно!
http.ListenAndServe(":8080", handler)
```

```go
// ✅ Правильно
http.ListenAndServeTLS(":8443", "cert.pem", "key.pem", handler)
```

### ❌ Ошибка 2: Не использовать bcrypt для паролей

```go
// ❌ Пароли в открытом виде
users["admin"] = "password123"
```

```go
// ✅ Хешированные пароли
hashedPassword, _ := bcrypt.GenerateFromPassword([]byte("password123"), bcrypt.DefaultCost)
users["admin"] = string(hashedPassword)
```

### ❌ Ошибка 3: Игнорировать rate limiting

```go
// ❌ Уязвимо к brute-force
if validateCredentials(username, password) {
    // ...
}
```

```go
// ✅ С rate limiting
if !rateLimiter.Allow(username) {
    return errors.New("too many attempts")
}
if validateCredentials(username, password) {
    // ...
}
```

## Best Practices

1. ✅ **Всегда используйте HTTPS** для защиты учётных данных в транзите
2. ✅ **Хешируйте пароли** с использованием bcrypt или argon2
3. ✅ **Используйте constant-time comparison** для защиты от timing attacks
4. ✅ **Добавьте rate limiting** для защиты от brute-force атак
5. ✅ **Логируйте неудачные попытки** входа для мониторинга
6. ✅ **Используйте IP whitelist** для дополнительной защиты
7. ✅ **Не используйте** для публичных API или клиентских приложений

## Вопросы с собеседований

**Вопрос 1:** Почему Basic Authentication небезопасно без HTTPS?

**Ответ:** Basic Auth передаёт учётные данные в формате Base64, который легко декодируется (это кодирование, не шифрование). Без HTTPS злоумышленник может перехватить заголовок Authorization и получить логин и пароль пользователя.

**Вопрос 2:** Как защититься от brute-force атак при использовании Basic Auth?

**Ответ:** Несколько способов:
1. **Rate limiting** — ограничить количество попыток с одного IP/для одного пользователя
2. **Account lockout** — блокировать аккаунт после N неудачных попыток
3. **CAPTCHA** — после нескольких неудачных попыток
4. **IP whitelist** — разрешить доступ только с определённых IP
5. **Мониторинг** — отслеживать подозрительную активность

**Вопрос 3:** В чём разница между Base64 кодированием и шифрованием?

**Ответ:** Base64 — это **кодирование** (encoding), а не шифрование. Его цель — представить бинарные данные в текстовом формате для безопасной передачи. Любой может декодировать Base64 без ключа. Шифрование же требует ключ для расшифровки и обеспечивает конфиденциальность.

## Связанные темы

- [[HTTP протокол]]
- [[Безопасность API]]
- [[JWT (JSON Web Tokens)]]
- [[OAuth 2.0]]
- [[Session-based аутентификация]]
- [[HTTPS и TLS]]
- [[Go - Пакет net-http]]
