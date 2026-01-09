# Session-based аутентификация

Session-based аутентификация — это метод, при котором сервер создаёт и хранит сессию пользователя после успешной авторизации, а клиенту передаёт уникальный Session ID в cookie.

## Основные концепции

После успешного входа сервер создаёт сессию с уникальным ID и сохраняет её в хранилище (память, Redis, БД). Клиент получает Session ID в cookie и отправляет его в каждом запросе. Сервер проверяет Session ID и извлекает информацию о пользователе.

### Жизненный цикл сессии

```
1. Вход (Login)
   User -> [username, password] -> Server
   Server -> Создаёт Session ID -> Сохраняет в хранилище
   Server -> Отправляет Session ID в cookie -> Client

2. Защищённый запрос
   Client -> [Session ID в cookie] -> Server
   Server -> Проверяет Session ID в хранилище
   Server -> Извлекает данные пользователя
   Server -> Возвращает ответ

3. Выход (Logout)
   Client -> [Session ID] -> Server
   Server -> Удаляет сессию из хранилища
   Server -> Очищает cookie
```

## Реализация в Go

### Простая сессия в памяти

```go
package main

import (
    "crypto/rand"
    "encoding/base64"
    "fmt"
    "net/http"
    "sync"
    "time"
)

// Данные сессии
type Session struct {
    UserID    int
    Username  string
    CreatedAt time.Time
    ExpiresAt time.Time
}

// Хранилище сессий в памяти
type SessionStore struct {
    mu       sync.RWMutex
    sessions map[string]*Session
}

var store = &SessionStore{
    sessions: make(map[string]*Session),
}

// Генерация случайного Session ID
func generateSessionID() (string, error) {
    bytes := make([]byte, 32)
    if _, err := rand.Read(bytes); err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(bytes), nil
}

// Создание новой сессии
func (s *SessionStore) Create(userID int, username string) (string, error) {
    sessionID, err := generateSessionID()
    if err != nil {
        return "", err
    }

    session := &Session{
        UserID:    userID,
        Username:  username,
        CreatedAt: time.Now(),
        ExpiresAt: time.Now().Add(24 * time.Hour),
    }

    s.mu.Lock()
    s.sessions[sessionID] = session
    s.mu.Unlock()

    return sessionID, nil
}

// Получение сессии
func (s *SessionStore) Get(sessionID string) (*Session, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()

    session, exists := s.sessions[sessionID]
    if !exists {
        return nil, false
    }

    // Проверяем истечение
    if time.Now().After(session.ExpiresAt) {
        return nil, false
    }

    return session, true
}

// Удаление сессии
func (s *SessionStore) Delete(sessionID string) {
    s.mu.Lock()
    delete(s.sessions, sessionID)
    s.mu.Unlock()
}

// Очистка истёкших сессий
func (s *SessionStore) CleanExpired() {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now()
    for id, session := range s.sessions {
        if now.After(session.ExpiresAt) {
            delete(s.sessions, id)
        }
    }
}

// Middleware для проверки сессии
func SessionAuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Получаем Session ID из cookie
        cookie, err := r.Cookie("session_id")
        if err != nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // Проверяем сессию
        session, exists := store.Get(cookie.Value)
        if !exists {
            http.Error(w, "Invalid or expired session", http.StatusUnauthorized)
            return
        }

        // Сохраняем данные сессии в контекст
        ctx := context.WithValue(r.Context(), "session", session)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Handler для логина
func loginHandler(w http.ResponseWriter, r *http.Request) {
    username := r.FormValue("username")
    password := r.FormValue("password")

    // Упрощённая проверка (в продакшене использовать БД + bcrypt)
    if username != "admin" || password != "password" {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }

    // Создаём сессию
    sessionID, err := store.Create(1, username)
    if err != nil {
        http.Error(w, "Failed to create session", http.StatusInternalServerError)
        return
    }

    // Устанавливаем cookie
    http.SetCookie(w, &http.Cookie{
        Name:     "session_id",
        Value:    sessionID,
        Path:     "/",
        HttpOnly: true,
        Secure:   true, // Только для HTTPS
        SameSite: http.SameSiteStrictMode,
        MaxAge:   86400, // 24 часа
    })

    fmt.Fprintf(w, "Logged in successfully")
}

// Handler для логаута
func logoutHandler(w http.ResponseWriter, r *http.Request) {
    cookie, err := r.Cookie("session_id")
    if err == nil {
        // Удаляем сессию из хранилища
        store.Delete(cookie.Value)
    }

    // Очищаем cookie
    http.SetCookie(w, &http.Cookie{
        Name:     "session_id",
        Value:    "",
        Path:     "/",
        HttpOnly: true,
        MaxAge:   -1, // Удаляет cookie
    })

    fmt.Fprintf(w, "Logged out successfully")
}

// Защищённый ресурс
func protectedHandler(w http.ResponseWriter, r *http.Request) {
    session := r.Context().Value("session").(*Session)
    fmt.Fprintf(w, "Hello, %s! UserID: %d", session.Username, session.UserID)
}

func main() {
    // Запускаем фоновую очистку истёкших сессий
    go func() {
        ticker := time.NewTicker(1 * time.Hour)
        defer ticker.Stop()
        for range ticker.C {
            store.CleanExpired()
        }
    }()

    mux := http.NewServeMux()
    mux.HandleFunc("/login", loginHandler)
    mux.HandleFunc("/logout", logoutHandler)

    protectedResource := http.HandlerFunc(protectedHandler)
    mux.Handle("/protected", SessionAuthMiddleware(protectedResource))

    http.ListenAndServe(":8080", mux)
}
```

### Сессии с Redis

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/go-redis/redis/v8"
)

type RedisSessionStore struct {
    client *redis.Client
}

func NewRedisSessionStore(addr string) *RedisSessionStore {
    client := redis.NewClient(&redis.Options{
        Addr:     addr,
        Password: "",
        DB:       0,
    })

    return &RedisSessionStore{client: client}
}

// Создание сессии в Redis
func (s *RedisSessionStore) Create(ctx context.Context, userID int, username string) (string, error) {
    sessionID, err := generateSessionID()
    if err != nil {
        return "", err
    }

    session := Session{
        UserID:    userID,
        Username:  username,
        CreatedAt: time.Now(),
        ExpiresAt: time.Now().Add(24 * time.Hour),
    }

    // Сериализуем сессию в JSON
    data, err := json.Marshal(session)
    if err != nil {
        return "", err
    }

    // Сохраняем в Redis с TTL
    key := fmt.Sprintf("session:%s", sessionID)
    err = s.client.Set(ctx, key, data, 24*time.Hour).Err()
    if err != nil {
        return "", err
    }

    return sessionID, nil
}

// Получение сессии из Redis
func (s *RedisSessionStore) Get(ctx context.Context, sessionID string) (*Session, error) {
    key := fmt.Sprintf("session:%s", sessionID)

    data, err := s.client.Get(ctx, key).Bytes()
    if err == redis.Nil {
        return nil, fmt.Errorf("session not found")
    }
    if err != nil {
        return nil, err
    }

    var session Session
    if err := json.Unmarshal(data, &session); err != nil {
        return nil, err
    }

    // Проверяем истечение
    if time.Now().After(session.ExpiresAt) {
        s.Delete(ctx, sessionID)
        return nil, fmt.Errorf("session expired")
    }

    return &session, nil
}

// Удаление сессии из Redis
func (s *RedisSessionStore) Delete(ctx context.Context, sessionID string) error {
    key := fmt.Sprintf("session:%s", sessionID)
    return s.client.Del(ctx, key).Err()
}

// Обновление времени жизни сессии
func (s *RedisSessionStore) Refresh(ctx context.Context, sessionID string) error {
    key := fmt.Sprintf("session:%s", sessionID)
    return s.client.Expire(ctx, key, 24*time.Hour).Err()
}
```

### Сессии с PostgreSQL

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "time"

    _ "github.com/lib/pq"
)

type DBSessionStore struct {
    db *sql.DB
}

// SQL для создания таблицы
const createTableSQL = `
CREATE TABLE IF NOT EXISTS sessions (
    session_id VARCHAR(255) PRIMARY KEY,
    user_id INTEGER NOT NULL,
    username VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_sessions_expires_at ON sessions(expires_at);
`

func NewDBSessionStore(connString string) (*DBSessionStore, error) {
    db, err := sql.Open("postgres", connString)
    if err != nil {
        return nil, err
    }

    // Создаём таблицу
    if _, err := db.Exec(createTableSQL); err != nil {
        return nil, err
    }

    return &DBSessionStore{db: db}, nil
}

// Создание сессии в БД
func (s *DBSessionStore) Create(ctx context.Context, userID int, username string) (string, error) {
    sessionID, err := generateSessionID()
    if err != nil {
        return "", err
    }

    query := `
        INSERT INTO sessions (session_id, user_id, username, created_at, expires_at)
        VALUES ($1, $2, $3, $4, $5)
    `

    now := time.Now()
    expiresAt := now.Add(24 * time.Hour)

    _, err = s.db.ExecContext(ctx, query, sessionID, userID, username, now, expiresAt)
    if err != nil {
        return "", err
    }

    return sessionID, nil
}

// Получение сессии из БД
func (s *DBSessionStore) Get(ctx context.Context, sessionID string) (*Session, error) {
    query := `
        SELECT user_id, username, created_at, expires_at
        FROM sessions
        WHERE session_id = $1 AND expires_at > NOW()
    `

    var session Session
    err := s.db.QueryRowContext(ctx, query, sessionID).Scan(
        &session.UserID,
        &session.Username,
        &session.CreatedAt,
        &session.ExpiresAt,
    )

    if err == sql.ErrNoRows {
        return nil, fmt.Errorf("session not found or expired")
    }
    if err != nil {
        return nil, err
    }

    return &session, nil
}

// Удаление сессии из БД
func (s *DBSessionStore) Delete(ctx context.Context, sessionID string) error {
    query := "DELETE FROM sessions WHERE session_id = $1"
    _, err := s.db.ExecContext(ctx, query, sessionID)
    return err
}

// Очистка истёкших сессий
func (s *DBSessionStore) CleanExpired(ctx context.Context) error {
    query := "DELETE FROM sessions WHERE expires_at < NOW()"
    _, err := s.db.ExecContext(ctx, query)
    return err
}
```

## Расширенные возможности

### Remember Me функциональность

```go
package main

import (
    "time"
)

// Создание долгоживущей сессии (remember me)
func createRememberMeSession(userID int, username string, rememberMe bool) (string, error) {
    sessionID, err := generateSessionID()
    if err != nil {
        return "", err
    }

    // Время жизни зависит от remember me
    var expiresAt time.Time
    if rememberMe {
        expiresAt = time.Now().Add(30 * 24 * time.Hour) // 30 дней
    } else {
        expiresAt = time.Now().Add(24 * time.Hour) // 1 день
    }

    session := &Session{
        UserID:    userID,
        Username:  username,
        CreatedAt: time.Now(),
        ExpiresAt: expiresAt,
    }

    store.sessions[sessionID] = session
    return sessionID, nil
}

func loginWithRememberMe(w http.ResponseWriter, r *http.Request) {
    username := r.FormValue("username")
    password := r.FormValue("password")
    rememberMe := r.FormValue("remember_me") == "true"

    // Проверка учётных данных...

    sessionID, _ := createRememberMeSession(1, username, rememberMe)

    maxAge := 86400 // 1 день
    if rememberMe {
        maxAge = 30 * 86400 // 30 дней
    }

    http.SetCookie(w, &http.Cookie{
        Name:     "session_id",
        Value:    sessionID,
        MaxAge:   maxAge,
        HttpOnly: true,
        Secure:   true,
        SameSite: http.SameSiteStrictMode,
    })
}
```

### Хранение дополнительных данных в сессии

```go
type ExtendedSession struct {
    UserID      int
    Username    string
    Role        string
    Permissions []string
    LastActive  time.Time
    IPAddress   string
    UserAgent   string
    CreatedAt   time.Time
    ExpiresAt   time.Time
}

// Обновление времени последней активности
func (s *SessionStore) UpdateLastActive(sessionID string) {
    s.mu.Lock()
    defer s.mu.Unlock()

    if session, exists := s.sessions[sessionID]; exists {
        session.LastActive = time.Now()
    }
}

// Middleware для отслеживания активности
func ActivityTrackingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        cookie, err := r.Cookie("session_id")
        if err == nil {
            store.UpdateLastActive(cookie.Value)
        }
        next.ServeHTTP(w, r)
    })
}
```

## Сравнение: Session vs JWT

| Параметр | Session-based | JWT |
|----------|---------------|-----|
| **Хранение** | На сервере (Redis/БД) | На клиенте |
| **Stateful/Stateless** | Stateful | Stateless |
| **Масштабируемость** | Требует shared storage | Легко масштабируется |
| **Отзыв токена** | ✅ Легко (удалить из БД) | ❌ Сложно (нужен blacklist) |
| **Размер** | Маленький (ID в cookie) | Большой (весь токен) |
| **Безопасность** | ✅ Можно отозвать мгновенно | ⚠️ Действует до истечения |
| **Нагрузка на сервер** | ⚠️ Проверка в БД на каждом запросе | ✅ Только валидация подписи |
| **Проверка данных** | ✅ Всегда актуальные (из БД) | ⚠️ Данные могут устареть |

## Типичные ошибки

### ❌ Ошибка 1: Слабый Session ID

```go
// ❌ Предсказуемый ID
sessionID := fmt.Sprintf("%d", time.Now().Unix())
```

```go
// ✅ Криптографически стойкий ID
bytes := make([]byte, 32)
rand.Read(bytes)
sessionID := base64.URLEncoding.EncodeToString(bytes)
```

### ❌ Ошибка 2: Не удалять сессию при логауте

```go
// ❌ Просто очищаем cookie
http.SetCookie(w, &http.Cookie{Name: "session_id", MaxAge: -1})
```

```go
// ✅ Удаляем сессию из хранилища
store.Delete(sessionID)
http.SetCookie(w, &http.Cookie{Name: "session_id", MaxAge: -1})
```

### ❌ Ошибка 3: Не чистить истёкшие сессии

```go
// ✅ Фоновая очистка
go func() {
    ticker := time.NewTicker(1 * time.Hour)
    for range ticker.C {
        store.CleanExpired()
    }
}()
```

### ❌ Ошибка 4: Session Fixation атака

```go
// ❌ Уязвимо: используем Session ID из запроса
sessionID := r.URL.Query().Get("session")
```

```go
// ✅ Генерируем новый Session ID при логине
sessionID, _ := generateSessionID()
// Всегда создаём новую сессию после успешной авторизации
```

## Best Practices

1. ✅ **Генерируйте криптографически стойкие Session ID** (crypto/rand)
2. ✅ **Используйте httpOnly и Secure флаги** для cookies
3. ✅ **Устанавливайте разумное время жизни** (24 часа для обычных, 30 дней для remember me)
4. ✅ **Удаляйте сессию при логауте** из хранилища
5. ✅ **Очищайте истёкшие сессии** фоновым процессом
6. ✅ **Обновляйте Session ID** после успешного логина (против fixation)
7. ✅ **Храните минимум данных** в сессии (только ID и критичные поля)
8. ✅ **Используйте Redis** для production (быстрее чем БД)
9. ✅ **Отслеживайте активность** пользователя (last_active)

## Защита от атак

### CSRF Protection

```go
// Генерация CSRF токена
func generateCSRFToken() string {
    bytes := make([]byte, 32)
    rand.Read(bytes)
    return base64.URLEncoding.EncodeToString(bytes)
}

// Middleware для проверки CSRF
func CSRFMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.Method == "POST" || r.Method == "PUT" || r.Method == "DELETE" {
            token := r.Header.Get("X-CSRF-Token")
            session := r.Context().Value("session").(*Session)

            if token != session.CSRFToken {
                http.Error(w, "Invalid CSRF token", http.StatusForbidden)
                return
            }
        }
        next.ServeHTTP(w, r)
    })
}
```

### Session Hijacking Prevention

```go
// Привязка сессии к User-Agent и IP
type SecureSession struct {
    Session
    UserAgent string
    IPAddress string
}

func validateSessionSecurity(r *http.Request, session *SecureSession) bool {
    currentUA := r.Header.Get("User-Agent")
    currentIP := r.RemoteAddr

    // Проверяем соответствие
    return currentUA == session.UserAgent && currentIP == session.IPAddress
}
```

## Вопросы с собеседований

**Вопрос 1:** Как масштабировать приложение с session-based аутентификацией?

**Ответ:** Для горизонтального масштабирования нужно использовать централизованное хранилище сессий:
1. **Redis** — рекомендуется, быстрая работа, встроенный TTL
2. **БД** — PostgreSQL, MySQL (медленнее Redis)
3. **Sticky Sessions** — привязка пользователя к одному серверу (не рекомендуется)

**Вопрос 2:** В чём разница между httpOnly и Secure флагами cookie?

**Ответ:**
- **httpOnly** — cookie недоступен для JavaScript (защита от XSS атак)
- **Secure** — cookie передаётся только по HTTPS (защита от перехвата)
- **SameSite** — защита от CSRF атак (Strict/Lax/None)

**Вопрос 3:** Что такое Session Fixation атака и как защититься?

**Ответ:** Session Fixation — атака, когда злоумышленник устанавливает жертве известный ему Session ID до авторизации. После успешного входа жертвы, атакующий получает доступ.

**Защита:** Всегда генерировать новый Session ID после успешной авторизации.

**Вопрос 4:** Почему Redis лучше БД для хранения сессий?

**Ответ:**
- **Скорость** — in-memory операции быстрее дисковых
- **TTL** — встроенное автоматическое удаление истёкших ключей
- **Простота** — не нужны сложные индексы и очистка
- **Масштабируемость** — легко добавить Redis Cluster

## Связанные темы

- [[Redis - Key-Value хранилище]]
- [[JWT (JSON Web Tokens)]]
- [[OAuth 2.0]]
- [[Basic Authentication]]
- [[HTTPS и TLS]]
- [[Безопасность API]]
- [[PostgreSQL - Основы]]
- [[Go - Пакет net-http]]
