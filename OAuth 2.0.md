# OAuth 2.0

OAuth 2.0 — это протокол авторизации, который позволяет приложениям получать ограниченный доступ к ресурсам пользователя без передачи его учётных данных.

## Основные концепции

OAuth 2.0 решает задачу делегирования доступа: пользователь может разрешить стороннему приложению получить доступ к его данным, не передавая ему логин и пароль.

### Роли в OAuth 2.0

**1. Resource Owner (Владелец ресурса)**
- Пользователь, владеющий защищенными данными
- Может предоставить доступ к своим ресурсам

**2. Client (Клиент)**
- Приложение, которое хочет получить доступ к ресурсам
- Например: мобильное приложение, веб-сайт

**3. Authorization Server (Сервер авторизации)**
- Выдаёт токены после успешной аутентификации
- Проверяет права пользователя

**4. Resource Server (Сервер ресурсов)**
- Хранит защищенные данные
- Проверяет токены и отдаёт данные

```go
// Пример структур для OAuth 2.0
type OAuthConfig struct {
    ClientID     string
    ClientSecret string
    RedirectURL  string
    AuthURL      string // URL сервера авторизации
    TokenURL     string // URL для получения токена
    Scopes       []string
}

type OAuthToken struct {
    AccessToken  string    `json:"access_token"`
    TokenType    string    `json:"token_type"`
    ExpiresIn    int       `json:"expires_in"`
    RefreshToken string    `json:"refresh_token,omitempty"`
    Scope        string    `json:"scope,omitempty"`
    CreatedAt    time.Time
}
```

## Grant Types (Типы предоставления доступа)

### 1. Authorization Code Flow

Самый безопасный способ, используется для серверных приложений.

**Процесс:**
1. Клиент перенаправляет пользователя на сервер авторизации
2. Пользователь авторизуется и даёт согласие
3. Сервер возвращает код авторизации
4. Клиент обменивает код на токен

```go
package main

import (
    "context"
    "fmt"
    "log"
    "net/http"

    "golang.org/x/oauth2"
    "golang.org/x/oauth2/google"
)

func main() {
    // Настройка OAuth конфигурации
    config := &oauth2.Config{
        ClientID:     "your-client-id",
        ClientSecret: "your-client-secret",
        RedirectURL:  "http://localhost:8080/callback",
        Scopes: []string{
            "https://www.googleapis.com/auth/userinfo.email",
        },
        Endpoint: google.Endpoint,
    }

    // 1. Получаем URL для авторизации
    http.HandleFunc("/login", func(w http.ResponseWriter, r *http.Request) {
        // state защищает от CSRF атак
        state := generateRandomState()
        url := config.AuthCodeURL(state, oauth2.AccessTypeOffline)

        http.Redirect(w, r, url, http.StatusTemporaryRedirect)
    })

    // 2. Обработка callback после авторизации
    http.HandleFunc("/callback", func(w http.ResponseWriter, r *http.Request) {
        // Проверяем state
        state := r.FormValue("state")
        if !validateState(state) {
            http.Error(w, "Invalid state", http.StatusBadRequest)
            return
        }

        // Получаем код авторизации
        code := r.FormValue("code")

        // 3. Обмениваем код на токен
        token, err := config.Exchange(context.Background(), code)
        if err != nil {
            http.Error(w, "Failed to exchange token", http.StatusInternalServerError)
            return
        }

        fmt.Fprintf(w, "Access Token: %s\n", token.AccessToken)
        fmt.Fprintf(w, "Refresh Token: %s\n", token.RefreshToken)

        // 4. Используем токен для доступа к API
        client := config.Client(context.Background(), token)
        resp, err := client.Get("https://www.googleapis.com/oauth2/v2/userinfo")
        if err != nil {
            log.Printf("Failed to get user info: %v", err)
            return
        }
        defer resp.Body.Close()
    })

    log.Println("Server started at :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

func generateRandomState() string {
    // Генерация случайной строки для защиты от CSRF
    return "random-state-string"
}

func validateState(state string) bool {
    // Проверка валидности state
    return state == "random-state-string"
}
```

### 2. Client Credentials Flow

Используется для server-to-server взаимодействия без участия пользователя.

```go
package main

import (
    "context"
    "fmt"
    "log"

    "golang.org/x/oauth2/clientcredentials"
)

func clientCredentialsExample() {
    config := &clientcredentials.Config{
        ClientID:     "service-client-id",
        ClientSecret: "service-client-secret",
        TokenURL:     "https://auth.example.com/oauth/token",
        Scopes:       []string{"read:data", "write:data"},
    }

    // Автоматическое получение и обновление токена
    client := config.Client(context.Background())

    // Использование клиента для API запросов
    resp, err := client.Get("https://api.example.com/data")
    if err != nil {
        log.Fatal(err)
    }
    defer resp.Body.Close()

    fmt.Println("Status:", resp.Status)
}
```

### 3. Password Grant (устаревший)

Пользователь передаёт логин и пароль напрямую клиенту. **Не рекомендуется** использовать, кроме legacy систем.

### 4. Implicit Flow (устаревший)

Токен возвращается сразу в URL. **Не рекомендуется** для новых приложений из-за проблем безопасности.

## Access Token и Refresh Token

### Access Token
- Короткоживущий токен (15 минут - 1 час)
- Используется для доступа к ресурсам
- Передаётся в заголовке Authorization

### Refresh Token
- Долгоживущий токен (дни, месяцы)
- Используется для получения новых access токенов
- Хранится в безопасном месте

```go
// Пример обновления токена
func refreshAccessToken(config *oauth2.Config, refreshToken string) (*oauth2.Token, error) {
    token := &oauth2.Token{
        RefreshToken: refreshToken,
    }

    // TokenSource автоматически обновляет токен при необходимости
    tokenSource := config.TokenSource(context.Background(), token)

    // Получение нового токена
    newToken, err := tokenSource.Token()
    if err != nil {
        return nil, fmt.Errorf("failed to refresh token: %w", err)
    }

    return newToken, nil
}
```

## Scopes (Области доступа)

Scopes определяют уровень доступа к ресурсам.

```go
// Примеры scopes
const (
    ScopeReadUser   = "read:user"
    ScopeWriteUser  = "write:user"
    ScopeReadEmail  = "read:email"
    ScopeAdmin      = "admin"
)

// Проверка наличия scope в токене
func hasScope(token *oauth2.Token, requiredScope string) bool {
    // Scopes обычно возвращаются как строка, разделённая пробелами
    scopes := strings.Split(token.Extra("scope").(string), " ")

    for _, scope := range scopes {
        if scope == requiredScope {
            return true
        }
    }
    return false
}

// Middleware для проверки scopes
func requireScope(requiredScope string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            token := extractTokenFromRequest(r)

            if !hasScope(token, requiredScope) {
                http.Error(w, "Insufficient permissions", http.StatusForbidden)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}

func extractTokenFromRequest(r *http.Request) *oauth2.Token {
    // Извлечение и валидация токена из заголовка Authorization
    return nil // Упрощённый пример
}
```

## Практический пример: OAuth 2.0 сервер

```go
package main

import (
    "crypto/rand"
    "encoding/base64"
    "encoding/json"
    "net/http"
    "sync"
    "time"
)

// Хранилище токенов (в продакшене использовать Redis/БД)
type TokenStore struct {
    mu            sync.RWMutex
    authCodes     map[string]AuthCodeData
    accessTokens  map[string]TokenData
    refreshTokens map[string]string // refreshToken -> clientID
}

type AuthCodeData struct {
    ClientID    string
    RedirectURI string
    Scope       string
    ExpiresAt   time.Time
}

type TokenData struct {
    ClientID  string
    Scope     string
    ExpiresAt time.Time
}

var store = &TokenStore{
    authCodes:     make(map[string]AuthCodeData),
    accessTokens:  make(map[string]TokenData),
    refreshTokens: make(map[string]string),
}

// 1. Authorization endpoint
func authorizationHandler(w http.ResponseWriter, r *http.Request) {
    clientID := r.URL.Query().Get("client_id")
    redirectURI := r.URL.Query().Get("redirect_uri")
    scope := r.URL.Query().Get("scope")
    state := r.URL.Query().Get("state")

    // Валидация клиента (упрощено)
    if clientID != "trusted-client" {
        http.Error(w, "Invalid client", http.StatusBadRequest)
        return
    }

    // Генерация кода авторизации
    code := generateRandomString(32)

    store.mu.Lock()
    store.authCodes[code] = AuthCodeData{
        ClientID:    clientID,
        RedirectURI: redirectURI,
        Scope:       scope,
        ExpiresAt:   time.Now().Add(10 * time.Minute),
    }
    store.mu.Unlock()

    // Перенаправление обратно на клиент
    redirectURL := fmt.Sprintf("%s?code=%s&state=%s", redirectURI, code, state)
    http.Redirect(w, r, redirectURL, http.StatusFound)
}

// 2. Token endpoint
func tokenHandler(w http.ResponseWriter, r *http.Request) {
    grantType := r.FormValue("grant_type")

    switch grantType {
    case "authorization_code":
        handleAuthorizationCodeGrant(w, r)
    case "refresh_token":
        handleRefreshTokenGrant(w, r)
    default:
        http.Error(w, "Unsupported grant type", http.StatusBadRequest)
    }
}

func handleAuthorizationCodeGrant(w http.ResponseWriter, r *http.Request) {
    code := r.FormValue("code")
    clientID := r.FormValue("client_id")
    clientSecret := r.FormValue("client_secret")

    // Проверка client credentials
    if !validateClient(clientID, clientSecret) {
        http.Error(w, "Invalid client credentials", http.StatusUnauthorized)
        return
    }

    // Проверка authorization code
    store.mu.Lock()
    authData, exists := store.authCodes[code]
    if !exists || time.Now().After(authData.ExpiresAt) {
        store.mu.Unlock()
        http.Error(w, "Invalid or expired code", http.StatusBadRequest)
        return
    }
    delete(store.authCodes, code) // Код используется один раз
    store.mu.Unlock()

    // Генерация токенов
    accessToken := generateRandomString(32)
    refreshToken := generateRandomString(32)

    store.mu.Lock()
    store.accessTokens[accessToken] = TokenData{
        ClientID:  clientID,
        Scope:     authData.Scope,
        ExpiresAt: time.Now().Add(1 * time.Hour),
    }
    store.refreshTokens[refreshToken] = clientID
    store.mu.Unlock()

    // Ответ с токенами
    response := map[string]interface{}{
        "access_token":  accessToken,
        "token_type":    "Bearer",
        "expires_in":    3600,
        "refresh_token": refreshToken,
        "scope":         authData.Scope,
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

func handleRefreshTokenGrant(w http.ResponseWriter, r *http.Request) {
    refreshToken := r.FormValue("refresh_token")

    store.mu.Lock()
    clientID, exists := store.refreshTokens[refreshToken]
    if !exists {
        store.mu.Unlock()
        http.Error(w, "Invalid refresh token", http.StatusBadRequest)
        return
    }
    store.mu.Unlock()

    // Генерация нового access token
    accessToken := generateRandomString(32)

    store.mu.Lock()
    store.accessTokens[accessToken] = TokenData{
        ClientID:  clientID,
        ExpiresAt: time.Now().Add(1 * time.Hour),
    }
    store.mu.Unlock()

    response := map[string]interface{}{
        "access_token": accessToken,
        "token_type":   "Bearer",
        "expires_in":   3600,
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}

func validateClient(clientID, clientSecret string) bool {
    // В продакшене проверять по БД
    return clientID == "trusted-client" && clientSecret == "secret"
}

func generateRandomString(length int) string {
    bytes := make([]byte, length)
    rand.Read(bytes)
    return base64.URLEncoding.EncodeToString(bytes)[:length]
}
```

## Типичные ошибки

### ❌ Ошибка 1: Хранение токенов в localStorage (SPA)

```javascript
// ❌ Небезопасно - доступно для XSS атак
localStorage.setItem('access_token', token);
```

**✅ Правильно:** Использовать httpOnly cookies или хранить в памяти приложения.

### ❌ Ошибка 2: Не проверять state параметр

```go
// ❌ CSRF уязвимость
code := r.FormValue("code")
token, _ := config.Exchange(context.Background(), code)
```

```go
// ✅ Правильно
state := r.FormValue("state")
if !validateState(state) {
    return errors.New("invalid state")
}
code := r.FormValue("code")
token, _ := config.Exchange(context.Background(), code)
```

### ❌ Ошибка 3: Использование implicit flow

```go
// ❌ Устаревший и небезопасный способ
// Токен возвращается в URL фрагменте
```

**✅ Правильно:** Использовать Authorization Code Flow с PKCE для SPA.

## Best Practices

1. ✅ **Всегда используйте HTTPS** для OAuth потоков
2. ✅ **Используйте state параметр** для защиты от CSRF
3. ✅ **Проверяйте redirect_uri** на соответствие зарегистрированным
4. ✅ **Используйте короткое время жизни** для access токенов
5. ✅ **Храните refresh токены безопасно** (зашифрованными в БД)
6. ✅ **Используйте PKCE** для публичных клиентов (SPA, мобильные приложения)
7. ✅ **Ограничивайте scopes** до минимально необходимых

## Вопросы с собеседований

**Вопрос 1:** В чём разница между OAuth 2.0 и OpenID Connect?

**Ответ:** OAuth 2.0 — это протокол **авторизации** (что пользователь может делать), а OpenID Connect (OIDC) — это протокол **аутентификации** (кто пользователь), построенный поверх OAuth 2.0. OIDC добавляет ID Token (JWT) с информацией о пользователе.

**Вопрос 2:** Почему не стоит использовать Password Grant?

**Ответ:** Потому что пользователь передаёт свои учётные данные стороннему приложению, что противоречит идее OAuth. Приложение получает полный доступ и может злоупотреблять им. Безопаснее использовать Authorization Code Flow.

**Вопрос 3:** Что такое PKCE и зачем он нужен?

**Ответ:** PKCE (Proof Key for Code Exchange) — расширение OAuth 2.0 для защиты Authorization Code Flow в публичных клиентах (где нельзя безопасно хранить client_secret). Клиент генерирует code_verifier и отправляет code_challenge, что предотвращает перехват кода авторизации.

## Связанные темы

- [[JWT (JSON Web Tokens)]]
- [[Безопасность API]]
- [[Session-based аутентификация]]
- [[HTTPS и TLS]]
- [[HTTP протокол]]
