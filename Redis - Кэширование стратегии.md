# Redis - Кэширование стратегии

Паттерны и стратегии использования Redis для кэширования данных.

## Зачем кэш?

**Проблема:** База данных медленная (10-100ms на запрос)
**Решение:** Кэш в RAM (< 1ms)

**Выигрыш:**
- Скорость: 10-100x быстрее
- Нагрузка на БД: снижается в разы
- Стоимость: меньше серверов БД

## Cache-Aside (Lazy Loading)

Самый популярный паттерн. Приложение управляет кэшем.

### Реализация

```go
func GetUser(id int) (*User, error) {
    cacheKey := fmt.Sprintf("user:%d", id)

    // 1. Попытка из кэша
    cached, err := redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // 2. Cache miss - идем в БД
    user, err := db.QueryUser(id)
    if err != nil {
        return nil, err
    }

    // 3. Сохраняем в кэш
    data, _ := json.Marshal(user)
    redis.Set(ctx, cacheKey, data, 1*time.Hour)

    return user, nil
}
```

**Плюсы:**
- Простота реализации
- Кэшируется только запрашиваемое
- Устойчивость к падению кэша

**Минусы:**
- Cache miss penalty (первый запрос медленный)
- Устаревшие данные при обновлении

### Инвалидация при изменении

```go
func UpdateUser(user *User) error {
    // 1. Обновить в БД
    err := db.Update(user)
    if err != nil {
        return err
    }

    // 2. Удалить из кэша
    cacheKey := fmt.Sprintf("user:%d", user.ID)
    redis.Del(ctx, cacheKey)

    return nil
}
```

## Read-Through Cache

Кэш сам управляет загрузкой данных.

### Концепция

```
Application → Cache → Database
              ↓
         Cache hit/miss
```

**Реализация в Go (обертка):**

```go
type CachedDB struct {
    db    *sql.DB
    cache *redis.Client
}

func (c *CachedDB) Get(key string, loader func() (interface{}, error)) (interface{}, error) {
    // Попытка из кэша
    cached, err := c.cache.Get(ctx, key).Result()
    if err == nil {
        return deserialize(cached), nil
    }

    // Cache miss - загрузить через loader
    data, err := loader()
    if err != nil {
        return nil, err
    }

    // Сохранить в кэш
    serialized := serialize(data)
    c.cache.Set(ctx, key, serialized, 1*time.Hour)

    return data, nil
}

// Использование
user, err := cachedDB.Get("user:1", func() (interface{}, error) {
    return db.QueryUser(1)
})
```

**Плюсы:**
- Единая точка загрузки
- Прозрачность для приложения

**Минусы:**
- Сложнее реализация
- Зависимость от кэша

## Write-Through Cache

Запись в кэш И БД одновременно.

### Реализация

```go
func UpdateUser(user *User) error {
    cacheKey := fmt.Sprintf("user:%d", user.ID)

    // 1. Обновить в БД
    err := db.Update(user)
    if err != nil {
        return err
    }

    // 2. Обновить в кэше
    data, _ := json.Marshal(user)
    redis.Set(ctx, cacheKey, data, 1*time.Hour)

    return nil
}
```

**Плюсы:**
- Кэш всегда актуален
- Нет cache miss на чтение

**Минусы:**
- Дополнительная задержка записи
- Кэшируется всё (даже редко читаемое)

## Write-Behind (Write-Back) Cache

Запись сначала в кэш, затем асинхронно в БД.

### Реализация

```go
func UpdateUser(user *User) error {
    cacheKey := fmt.Sprintf("user:%d", user.ID)

    // 1. Обновить в кэше
    data, _ := json.Marshal(user)
    redis.Set(ctx, cacheKey, data, 1*time.Hour)

    // 2. Добавить в очередь для записи в БД
    redis.LPush(ctx, "write_queue", cacheKey)

    return nil
}

// Background worker
func flushWorker() {
    for {
        result, _ := redis.BRPop(ctx, 0, "write_queue").Result()
        key := result[1]

        // Загрузить из кэша
        data, _ := redis.Get(ctx, key).Result()
        var user User
        json.Unmarshal([]byte(data), &user)

        // Записать в БД
        db.Update(&user)
    }
}
```

**Плюсы:**
- Очень быстрая запись
- Батчинг в БД

**Минусы:**
- Риск потери данных
- Сложность реализации

## Refresh-Ahead Cache

Обновление кэша до истечения TTL.

### Реализация

```go
func GetUser(id int) (*User, error) {
    cacheKey := fmt.Sprintf("user:%d", id)

    // Получить из кэша с TTL
    cached, err := redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)

        // Проверить TTL
        ttl, _ := redis.TTL(ctx, cacheKey).Result()
        if ttl < 5*time.Minute {
            // Скоро истечет - обновить асинхронно
            go func() {
                fresh, _ := db.QueryUser(id)
                data, _ := json.Marshal(fresh)
                redis.Set(ctx, cacheKey, data, 1*time.Hour)
            }()
        }

        return &user, nil
    }

    // Cache miss
    user, err := db.QueryUser(id)
    if err != nil {
        return nil, err
    }

    data, _ := json.Marshal(user)
    redis.Set(ctx, cacheKey, data, 1*time.Hour)

    return user, nil
}
```

**Плюсы:**
- Нет cache miss для популярных ключей
- Всегда свежие данные

**Минусы:**
- Дополнительная нагрузка
- Сложность определения "популярности"

## Cache Stampede (проблема)

Множество запросов одновременно при cache miss.

### Проблема

```
1000 запросов → Cache miss → 1000 запросов в БД!
```

### Решение 1: Mutex Lock

```go
var mu sync.Mutex
var loading = make(map[string]*sync.Mutex)

func GetUser(id int) (*User, error) {
    cacheKey := fmt.Sprintf("user:%d", id)

    // Попытка из кэша
    cached, err := redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // Получить мьютекс для этого ключа
    mu.Lock()
    keyMu, exists := loading[cacheKey]
    if !exists {
        keyMu = &sync.Mutex{}
        loading[cacheKey] = keyMu
    }
    mu.Unlock()

    // Блокировка на загрузку
    keyMu.Lock()
    defer keyMu.Unlock()

    // Проверить еще раз (может уже загрузили)
    cached, err = redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // Загрузить из БД
    user, err := db.QueryUser(id)
    if err != nil {
        return nil, err
    }

    // Сохранить в кэш
    data, _ := json.Marshal(user)
    redis.Set(ctx, cacheKey, data, 1*time.Hour)

    return user, nil
}
```

### Решение 2: Redis Lock

```go
func GetUserWithLock(id int) (*User, error) {
    cacheKey := fmt.Sprintf("user:%d", id)
    lockKey := fmt.Sprintf("lock:user:%d", id)

    // Попытка из кэша
    cached, err := redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // Попытка получить блокировку
    acquired, _ := redis.SetNX(ctx, lockKey, "1", 10*time.Second).Result()
    if acquired {
        defer redis.Del(ctx, lockKey)

        // Загрузить из БД
        user, err := db.QueryUser(id)
        if err != nil {
            return nil, err
        }

        // Сохранить в кэш
        data, _ := json.Marshal(user)
        redis.Set(ctx, cacheKey, data, 1*time.Hour)

        return user, nil
    }

    // Ждем пока другой процесс загрузит
    time.Sleep(100 * time.Millisecond)
    return GetUserWithLock(id)  // Retry
}
```

### Решение 3: Probabilistic Early Expiration

```go
func GetUserProbabilistic(id int) (*User, error) {
    cacheKey := fmt.Sprintf("user:%d", id)

    cached, err := redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)

        // Вероятностное обновление
        ttl, _ := redis.TTL(ctx, cacheKey).Result()
        delta := time.Hour - ttl
        if rand.Float64() < float64(delta)/float64(time.Hour) {
            // Обновить асинхронно
            go refreshUser(id)
        }

        return &user, nil
    }

    // Cache miss
    return loadAndCacheUser(id)
}
```

## TTL стратегии

### Fixed TTL

```go
redis.Set(ctx, key, value, 1*time.Hour)
```

**Применение:** Статичные данные.

### Sliding TTL

```go
// При каждом доступе обновляем TTL
func GetWithSlidingTTL(key string) (string, error) {
    value, err := redis.Get(ctx, key).Result()
    if err != nil {
        return "", err
    }

    // Обновить TTL
    redis.Expire(ctx, key, 1*time.Hour)

    return value, nil
}
```

**Применение:** Сессии, активность пользователя.

### Adaptive TTL

```go
// TTL зависит от популярности
func SetAdaptiveTTL(key string, value string, accessCount int) {
    ttl := time.Duration(accessCount) * time.Minute
    if ttl > 24*time.Hour {
        ttl = 24 * time.Hour
    }
    redis.Set(ctx, key, value, ttl)
}
```

**Применение:** Популярный контент дольше в кэше.

## Многоуровневый кэш

### L1 (Local) + L2 (Redis)

```go
type TieredCache struct {
    local  *sync.Map  // L1: in-memory
    redis  *redis.Client  // L2: Redis
}

func (c *TieredCache) Get(key string) (string, error) {
    // L1: Local cache
    if val, ok := c.local.Load(key); ok {
        return val.(string), nil
    }

    // L2: Redis
    val, err := c.redis.Get(ctx, key).Result()
    if err == nil {
        // Сохранить в L1
        c.local.Store(key, val)
        return val, nil
    }

    // L1 + L2 miss
    return "", err
}

func (c *TieredCache) Set(key, value string, ttl time.Duration) {
    // Сохранить в оба уровня
    c.local.Store(key, value)
    c.redis.Set(ctx, key, value, ttl)
}
```

**Плюсы:**
- Еще быстрее (нет сети)
- Снижает нагрузку на Redis

**Минусы:**
- Инвалидация сложнее
- Использует память приложения

## Cache Warming (прогрев)

Заполнение кэша до получения трафика.

```go
func WarmCache() {
    // Загрузить популярные данные
    popularUsers := db.GetPopularUsers(1000)

    for _, user := range popularUsers {
        key := fmt.Sprintf("user:%d", user.ID)
        data, _ := json.Marshal(user)
        redis.Set(ctx, key, data, 1*time.Hour)
    }

    log.Println("Cache warmed up")
}

// При старте приложения
func main() {
    // ...
    go WarmCache()
    // ...
}
```

## Cache Invalidation

### 1. TTL-based (автоматическая)

```go
redis.Set(ctx, key, value, 1*time.Hour)
// Истекает автоматически
```

**Плюсы:** Просто
**Минусы:** Устаревшие данные до истечения

### 2. Explicit (явная)

```go
func UpdateUser(user *User) error {
    db.Update(user)
    redis.Del(ctx, fmt.Sprintf("user:%d", user.ID))
    return nil
}
```

**Плюсы:** Всегда актуально
**Минусы:** Требует дисциплины

### 3. Tag-based (по тегам)

```go
// Создать связь ключ → теги
redis.SAdd(ctx, "tag:users", "user:1", "user:2", "user:3")
redis.SAdd(ctx, "tag:posts", "post:1", "post:2")

// Инвалидировать все ключи с тегом
func InvalidateTag(tag string) {
    keys, _ := redis.SMembers(ctx, "tag:"+tag).Result()
    for _, key := range keys {
        redis.Del(ctx, key)
    }
    redis.Del(ctx, "tag:"+tag)
}

// Инвалидировать всех пользователей
InvalidateTag("users")
```

### 4. Event-based (по событиям)

```go
// Подписаться на события изменения данных
pubsub := redis.Subscribe(ctx, "user_updated")

for msg := range pubsub.Channel() {
    userID := msg.Payload
    redis.Del(ctx, fmt.Sprintf("user:%s", userID))
}

// При обновлении пользователя
func UpdateUser(user *User) error {
    db.Update(user)
    redis.Publish(ctx, "user_updated", strconv.Itoa(user.ID))
    return nil
}
```

## Cache Patterns

### 1. Counter Cache

```go
// Кэширование счетчиков
func IncrementViews(postID int) {
    key := fmt.Sprintf("views:post:%d", postID)
    redis.Incr(ctx, key)

    // Периодически сбрасывать в БД
    count, _ := redis.Get(ctx, key).Int()
    if count%100 == 0 {
        db.UpdateViews(postID, count)
    }
}
```

### 2. Query Result Cache

```go
// Кэширование результатов запросов
func SearchUsers(query string) ([]User, error) {
    cacheKey := fmt.Sprintf("search:%s", hash(query))

    cached, err := redis.Get(ctx, cacheKey).Result()
    if err == nil {
        var users []User
        json.Unmarshal([]byte(cached), &users)
        return users, nil
    }

    users, err := db.Search(query)
    if err != nil {
        return nil, err
    }

    data, _ := json.Marshal(users)
    redis.Set(ctx, cacheKey, data, 10*time.Minute)

    return users, nil
}
```

### 3. Session Cache

```go
// Сессии пользователей
func CreateSession(userID int) string {
    sessionID := generateID()
    key := fmt.Sprintf("session:%s", sessionID)

    data := map[string]interface{}{
        "user_id": userID,
        "created": time.Now(),
    }

    // Сохранить с TTL (24 часа)
    jsonData, _ := json.Marshal(data)
    redis.Set(ctx, key, jsonData, 24*time.Hour)

    return sessionID
}

func GetSession(sessionID string) (int, error) {
    key := fmt.Sprintf("session:%s", sessionID)

    data, err := redis.Get(ctx, key).Result()
    if err != nil {
        return 0, err
    }

    // Обновить TTL (sliding)
    redis.Expire(ctx, key, 24*time.Hour)

    var session map[string]interface{}
    json.Unmarshal([]byte(data), &session)

    return int(session["user_id"].(float64)), nil
}
```

## Best Practices

1. ✅ **Cache-Aside** для большинства случаев
2. ✅ Явная инвалидация при изменении
3. ✅ TTL для автоматической очистки
4. ✅ Защита от Cache Stampede (locks)
5. ✅ Мониторинг hit/miss rate
6. ✅ Правильный размер кэша (maxmemory)
7. ✅ Eviction policy (allkeys-lru)
8. ❌ Не кэшировать всё подряд
9. ✅ Кэшировать дорогие операции
10. ✅ Документировать стратегию инвалидации

## Метрики кэша

```go
type CacheStats struct {
    Hits   int64
    Misses int64
}

var stats CacheStats

func GetWithStats(key string) (string, error) {
    value, err := redis.Get(ctx, key).Result()
    if err == nil {
        atomic.AddInt64(&stats.Hits, 1)
        return value, nil
    }

    atomic.AddInt64(&stats.Misses, 1)
    return "", err
}

func HitRate() float64 {
    hits := atomic.LoadInt64(&stats.Hits)
    misses := atomic.LoadInt64(&stats.Misses)
    total := hits + misses

    if total == 0 {
        return 0
    }

    return float64(hits) / float64(total)
}

// Целевой hit rate: > 90%
```

## Связанные темы

- [[Redis - Key-Value хранилище]]
- [[Redis - Типы данных и команды]]
- [[PostgreSQL - Оптимизация запросов]]
- [[Типы СУБД - SQL vs NoSQL]]
- [[Задача - LRU Cache]]
