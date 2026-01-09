# JWT (JSON Web Tokens)

JWT (JSON Web Token) — это компактный, URL-безопасный способ передачи claims (утверждений) между двумя сторонами в формате JSON, подписанный цифровой подписью.

## Основные концепции

JWT используется для аутентификации и обмена информацией между клиентом и сервером. Токен содержит закодированную информацию, которую можно проверить без обращения к базе данных.

### Структура JWT

JWT состоит из трёх частей, разделённых точками:

```
header.payload.signature
```

**Пример JWT:**
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### 1. Header (Заголовок)

Содержит тип токена и алгоритм подписи.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### 2. Payload (Полезная нагрузка)

Содержит claims — утверждения о пользователе и дополнительные данные.

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622
}
```

### 3. Signature (Подпись)

Создаётся путём кодирования header + payload + secret key.

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

## Алгоритмы подписи

### HS256 (HMAC with SHA-256)

Симметричное шифрование — один секретный ключ для подписи и проверки.

```go
package main

import (
    "fmt"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

// Структура claims
type Claims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    Role     string `json:"role"`
    jwt.RegisteredClaims
}

var jwtSecret = []byte("your-secret-key-keep-it-safe")

// Создание JWT с HS256
func createTokenHS256(userID int, username, role string) (string, error) {
    // Создаём claims
    claims := Claims{
        UserID:   userID,
        Username: username,
        Role:     role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
            Issuer:    "my-app",
            Subject:   fmt.Sprintf("%d", userID),
        },
    }

    // Создаём токен
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)

    // Подписываем токен
    tokenString, err := token.SignedString(jwtSecret)
    if err != nil {
        return "", fmt.Errorf("failed to sign token: %w", err)
    }

    return tokenString, nil
}

// Валидация JWT с HS256
func validateTokenHS256(tokenString string) (*Claims, error) {
    // Парсинг токена
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        // Проверяем алгоритм
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return jwtSecret, nil
    })

    if err != nil {
        return nil, fmt.Errorf("failed to parse token: %w", err)
    }

    // Извлекаем claims
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }

    return nil, fmt.Errorf("invalid token")
}
```

### RS256 (RSA with SHA-256)

Асимметричное шифрование — приватный ключ для подписи, публичный для проверки.

```go
package main

import (
    "crypto/rsa"
    "crypto/x509"
    "encoding/pem"
    "fmt"
    "os"

    "github.com/golang-jwt/jwt/v5"
)

var (
    privateKey *rsa.PrivateKey
    publicKey  *rsa.PublicKey
)

// Загрузка ключей
func loadKeys() error {
    // Загрузка приватного ключа
    privateKeyData, err := os.ReadFile("private_key.pem")
    if err != nil {
        return err
    }

    block, _ := pem.Decode(privateKeyData)
    privateKey, err = x509.ParsePKCS1PrivateKey(block.Bytes)
    if err != nil {
        return err
    }

    // Загрузка публичного ключа
    publicKeyData, err := os.ReadFile("public_key.pem")
    if err != nil {
        return err
    }

    block, _ = pem.Decode(publicKeyData)
    pubInterface, err := x509.ParsePKIXPublicKey(block.Bytes)
    if err != nil {
        return err
    }

    publicKey = pubInterface.(*rsa.PublicKey)
    return nil
}

// Создание JWT с RS256
func createTokenRS256(userID int, username string) (string, error) {
    claims := Claims{
        UserID:   userID,
        Username: username,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodRS256, claims)

    // Подписываем приватным ключом
    tokenString, err := token.SignedString(privateKey)
    if err != nil {
        return "", err
    }

    return tokenString, nil
}

// Валидация JWT с RS256
func validateTokenRS256(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodRSA); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        // Возвращаем публичный ключ для проверки
        return publicKey, nil
    })

    if err != nil {
        return nil, err
    }

    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }

    return nil, fmt.Errorf("invalid token")
}
```

## Standard Claims (Стандартные поля)

### Registered Claims

```go
type RegisteredClaims struct {
    Issuer    string           `json:"iss,omitempty"` // Издатель токена
    Subject   string           `json:"sub,omitempty"` // Субъект (обычно user ID)
    Audience  jwt.ClaimStrings `json:"aud,omitempty"` // Получатель токена
    ExpiresAt *jwt.NumericDate `json:"exp,omitempty"` // Время истечения
    NotBefore *jwt.NumericDate `json:"nbf,omitempty"` // Не использовать до
    IssuedAt  *jwt.NumericDate `json:"iat,omitempty"` // Время создания
    ID        string           `json:"jti,omitempty"` // Уникальный ID токена
}
```

**Объяснение:**
- **iss** (issuer) — кто выдал токен (например, "auth.example.com")
- **sub** (subject) — для кого токен (обычно ID пользователя)
- **aud** (audience) — для какого сервиса токен предназначен
- **exp** (expiration) — когда токен истекает (Unix timestamp)
- **iat** (issued at) — когда токен был создан
- **nbf** (not before) — токен нельзя использовать до этого времени

### Custom Claims

```go
type CustomClaims struct {
    UserID      int      `json:"user_id"`
    Username    string   `json:"username"`
    Email       string   `json:"email"`
    Role        string   `json:"role"`
    Permissions []string `json:"permissions"`
    jwt.RegisteredClaims
}

func createTokenWithCustomClaims() (string, error) {
    claims := CustomClaims{
        UserID:      123,
        Username:    "john_doe",
        Email:       "john@example.com",
        Role:        "admin",
        Permissions: []string{"read:users", "write:users", "delete:users"},
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            Issuer:    "auth-service",
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}
```

## Практический пример: JWT Middleware

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "strings"

    "github.com/golang-jwt/jwt/v5"
)

// Контекстный ключ для claims
type contextKey string

const ClaimsContextKey contextKey = "claims"

// JWT Middleware для защиты роутов
func JWTAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Извлекаем токен из заголовка Authorization
        authHeader := r.Header.Get("Authorization")
        if authHeader == "" {
            http.Error(w, "Authorization header required", http.StatusUnauthorized)
            return
        }

        // Проверяем формат: "Bearer <token>"
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, "Invalid authorization header format", http.StatusUnauthorized)
            return
        }

        tokenString := parts[1]

        // Валидируем токен
        claims, err := validateTokenHS256(tokenString)
        if err != nil {
            http.Error(w, fmt.Sprintf("Invalid token: %v", err), http.StatusUnauthorized)
            return
        }

        // Сохраняем claims в контекст
        ctx := context.WithValue(r.Context(), ClaimsContextKey, claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Middleware для проверки ролей
func RequireRole(allowedRoles ...string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims, ok := r.Context().Value(ClaimsContextKey).(*Claims)
            if !ok {
                http.Error(w, "No claims in context", http.StatusInternalServerError)
                return
            }

            // Проверяем роль пользователя
            hasRole := false
            for _, role := range allowedRoles {
                if claims.Role == role {
                    hasRole = true
                    break
                }
            }

            if !hasRole {
                http.Error(w, "Insufficient permissions", http.StatusForbidden)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

// Пример использования
func main() {
    mux := http.NewServeMux()

    // Публичный роут
    mux.HandleFunc("/login", loginHandler)

    // Защищённый роут (требуется JWT)
    protectedHandler := http.HandlerFunc(protectedResource)
    mux.Handle("/protected", JWTAuthMiddleware(protectedHandler))

    // Защищённый роут (требуется роль admin)
    adminHandler := http.HandlerFunc(adminResource)
    mux.Handle("/admin",
        JWTAuthMiddleware(
            RequireRole("admin")(adminHandler),
        ),
    )

    http.ListenAndServe(":8080", mux)
}

func loginHandler(w http.ResponseWriter, r *http.Request) {
    // Упрощённая логика аутентификации
    username := r.FormValue("username")
    password := r.FormValue("password")

    if username == "admin" && password == "password" {
        token, err := createTokenHS256(1, username, "admin")
        if err != nil {
            http.Error(w, "Failed to create token", http.StatusInternalServerError)
            return
        }

        w.Header().Set("Content-Type", "application/json")
        fmt.Fprintf(w, `{"token": "%s"}`, token)
        return
    }

    http.Error(w, "Invalid credentials", http.StatusUnauthorized)
}

func protectedResource(w http.ResponseWriter, r *http.Request) {
    claims := r.Context().Value(ClaimsContextKey).(*Claims)
    fmt.Fprintf(w, "Hello, %s! UserID: %d", claims.Username, claims.UserID)
}

func adminResource(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Welcome to admin panel")
}
```

## Refresh Token механизм

```go
package main

import (
    "time"
)

type TokenPair struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

// Создание пары токенов
func createTokenPair(userID int, username string) (*TokenPair, error) {
    // Access token — короткий срок жизни
    accessToken, err := createAccessToken(userID, username)
    if err != nil {
        return nil, err
    }

    // Refresh token — длительный срок жизни
    refreshToken, err := createRefreshToken(userID)
    if err != nil {
        return nil, err
    }

    return &TokenPair{
        AccessToken:  accessToken,
        RefreshToken: refreshToken,
    }, nil
}

func createAccessToken(userID int, username string) (string, error) {
    claims := Claims{
        UserID:   userID,
        Username: username,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)), // Короткий срок
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

func createRefreshToken(userID int) (string, error) {
    claims := jwt.RegisteredClaims{
        Subject:   fmt.Sprintf("%d", userID),
        ExpiresAt: jwt.NewNumericDate(time.Now().Add(7 * 24 * time.Hour)), // 7 дней
        IssuedAt:  jwt.NewNumericDate(time.Now()),
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

// Обновление access token по refresh token
func refreshAccessToken(refreshTokenString string) (string, error) {
    // Валидируем refresh token
    token, err := jwt.Parse(refreshTokenString, func(token *jwt.Token) (interface{}, error) {
        return jwtSecret, nil
    })

    if err != nil || !token.Valid {
        return "", fmt.Errorf("invalid refresh token")
    }

    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok {
        return "", fmt.Errorf("invalid claims")
    }

    // Получаем user ID из refresh token
    userID := int(claims["sub"].(float64))

    // В продакшене: проверить, не отозван ли refresh token (проверка в БД/Redis)

    // Создаём новый access token
    return createAccessToken(userID, "username") // Имя можно взять из БД
}
```

## Хранение токенов

### ❌ Вариант 1: localStorage (небезопасно)

```javascript
// ❌ Уязвимо к XSS атакам
localStorage.setItem('jwt', token);
```

### ✅ Вариант 2: httpOnly Cookie (безопасно)

```go
func setTokenCookie(w http.ResponseWriter, token string) {
    cookie := &http.Cookie{
        Name:     "jwt",
        Value:    token,
        Path:     "/",
        HttpOnly: true,  // Недоступно для JavaScript
        Secure:   true,  // Только по HTTPS
        SameSite: http.SameSiteStrictMode, // Защита от CSRF
        MaxAge:   86400, // 24 часа
    }
    http.SetCookie(w, cookie)
}

func getTokenFromCookie(r *http.Request) (string, error) {
    cookie, err := r.Cookie("jwt")
    if err != nil {
        return "", err
    }
    return cookie.Value, nil
}
```

### ✅ Вариант 3: Memory (для SPA)

```javascript
// ✅ Хранение в памяти (переменная)
let accessToken = null;

// После логина
const response = await fetch('/login', { ... });
const data = await response.json();
accessToken = data.access_token; // Хранится в памяти
```

## Типичные ошибки

### ❌ Ошибка 1: Не проверять алгоритм подписи

```go
// ❌ Уязвимость! Атакующий может подменить алгоритм на "none"
token, _ := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
    return jwtSecret, nil
})
```

```go
// ✅ Правильно
token, _ := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
    // Проверяем алгоритм
    if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("unexpected signing method")
    }
    return jwtSecret, nil
})
```

### ❌ Ошибка 2: Хранить секретный ключ в коде

```go
// ❌ Плохо
var jwtSecret = []byte("my-super-secret-key")
```

```go
// ✅ Правильно
var jwtSecret = []byte(os.Getenv("JWT_SECRET"))
```

### ❌ Ошибка 3: Не проверять exp (expiration)

```go
// ✅ Библиотека golang-jwt автоматически проверяет exp при token.Valid
if claims, ok := token.Claims.(*Claims); ok && token.Valid {
    // exp уже проверен
    return claims, nil
}
```

### ❌ Ошибка 4: Слишком большой payload

```go
// ❌ JWT передаётся в каждом запросе, не храните много данных
type BadClaims struct {
    UserID       int
    UserProfile  LargeObject // ❌ Плохо!
    AllPosts     []Post      // ❌ Плохо!
    jwt.RegisteredClaims
}
```

```go
// ✅ Храните только минимум
type GoodClaims struct {
    UserID   int    `json:"user_id"`
    Username string `json:"username"`
    Role     string `json:"role"`
    jwt.RegisteredClaims
}
```

## Best Practices

1. ✅ **Используйте короткое время жизни** для access токенов (15-30 минут)
2. ✅ **Используйте refresh токены** для обновления access токенов
3. ✅ **Храните токены в httpOnly cookies** для веб-приложений
4. ✅ **Всегда проверяйте алгоритм** подписи при валидации
5. ✅ **Используйте HTTPS** для передачи токенов
6. ✅ **Не храните чувствительные данные** в payload (они видны всем)
7. ✅ **Добавляйте jti** (JWT ID) для возможности отзыва токенов
8. ✅ **Используйте strong secret** (минимум 256 бит для HS256)

## Вопросы с собеседований

**Вопрос 1:** В чём разница между JWT и сессиями?

**Ответ:**
- **JWT:** Stateless (без состояния на сервере), токен содержит всю информацию, легко масштабируется горизонтально, но сложно отозвать токен до истечения срока.
- **Сессии:** Stateful (состояние хранится на сервере в БД/Redis), легко отозвать сессию, но требуется хранилище и сложнее масштабировать.

**Вопрос 2:** Как отозвать JWT токен?

**Ответ:** JWT нельзя отозвать напрямую, так как он stateless. Решения:
1. **Короткий срок жизни** + refresh токены (refresh можно отозвать в БД)
2. **Blacklist токенов** в Redis (храним отозванные токены до истечения exp)
3. **Whitelist токенов** (храним активные токены в Redis, проверяем при каждом запросе)

**Вопрос 3:** Почему не стоит хранить JWT в localStorage?

**Ответ:** localStorage доступен для JavaScript, что делает токен уязвимым к XSS (Cross-Site Scripting) атакам. Если злоумышленник внедрит вредоносный скрипт, он сможет украсть токен. Лучше использовать httpOnly cookies, которые недоступны для JavaScript.

**Вопрос 4:** Что такое "none" алгоритм уязвимость?

**Ответ:** Некоторые библиотеки принимают JWT с алгоритмом "none" (без подписи). Атакующий может создать токен, изменить payload (например, role: "admin") и отправить с алг="none". Защита: всегда явно проверять алгоритм подписи при валидации.

## Связанные темы

- [[OAuth 2.0]]
- [[Безопасность API]]
- [[Session-based аутентификация]]
- [[HTTPS и TLS]]
- [[Go - Пакет encoding-json]]
