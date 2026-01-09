# Redis - Key-Value хранилище

Redis (Remote Dictionary Server) - высокопроизводительное in-memory хранилище типа key-value с поддержкой множества структур данных.

## Что такое Redis?

**Key-Value хранилище:**
- Ключ (string) → Значение (различные типы)
- Все данные в RAM (очень быстро!)
- Опциональная персистентность на диск
- Single-threaded, но async I/O

**Скорость:** ~100,000 операций/сек на одном ядре

## Основные команды

### SET / GET - базовые операции

```bash
# Установить значение
SET user:1:name "Alice"
# OK

# Получить значение
GET user:1:name
# "Alice"

# SET с TTL (time to live)
SET session:abc123 "user_data" EX 3600  # Истекает через 3600 сек (1 час)

# SET если не существует (NX - not exists)
SET lock:resource1 "locked" NX EX 10
# OK или (nil) если уже существует

# SET если существует (XX - exists)
SET user:1:name "Bob" XX
# OK
```

### Работа с TTL

```bash
# Установить TTL для ключа
EXPIRE user:1:name 60  # Истекает через 60 секунд

# Проверить TTL
TTL user:1:name
# 59 (секунды до истечения)
# -1 (без TTL)
# -2 (ключ не существует)

# Убрать TTL
PERSIST user:1:name

# SET с TTL в миллисекундах
PSETEX key 5000 "value"  # 5000 мс = 5 сек
```

### EXISTS / DEL - проверка и удаление

```bash
# Проверить существование
EXISTS user:1:name
# 1 (exists)
# 0 (not exists)

# Удалить ключ
DEL user:1:name
# 1 (deleted)
# 0 (not found)

# Удалить несколько ключей
DEL key1 key2 key3
# 3 (количество удаленных)
```

### KEYS / SCAN - поиск ключей

```bash
# Найти ключи по паттерну
KEYS user:*
# ["user:1:name", "user:2:name", "user:3:name"]

# ❌ KEYS блокирует Redis! Не использовать в production

# ✅ SCAN - итеративный поиск (не блокирует)
SCAN 0 MATCH user:* COUNT 100
# ["17", ["user:1:name", "user:2:name"]]
# Первое значение - курсор для следующей итерации
# 0 означает конец
```

## Типы данных

### 1. String (строки)

```bash
# Обычные строки
SET message "Hello, Redis!"
GET message

# Числовые операции
SET counter 10
INCR counter     # 11
INCRBY counter 5 # 16
DECR counter     # 15
DECRBY counter 3 # 12

# Атомарные операции (thread-safe)
INCR views       # Инкремент счетчика просмотров
```

### 2. Hash (хеш-таблицы)

```bash
# Установить поле в hash
HSET user:1 name "Alice"
HSET user:1 age 30
HSET user:1 email "alice@example.com"

# Или сразу несколько
HMSET user:1 name "Alice" age 30 email "alice@example.com"

# Получить поле
HGET user:1 name
# "Alice"

# Получить все поля
HGETALL user:1
# ["name", "Alice", "age", "30", "email", "alice@example.com"]

# Получить несколько полей
HMGET user:1 name email
# ["Alice", "alice@example.com"]

# Инкремент числового поля
HINCRBY user:1 age 1  # 31

# Проверить существование поля
HEXISTS user:1 name   # 1

# Удалить поле
HDEL user:1 email

# Количество полей
HLEN user:1  # 2
```

**Когда использовать:** Объекты с несколькими полями (пользователи, сессии, конфиги).

### 3. List (списки)

```bash
# Добавить в начало (left push)
LPUSH tasks "task1"
LPUSH tasks "task2"  # ["task2", "task1"]

# Добавить в конец (right push)
RPUSH tasks "task3"  # ["task2", "task1", "task3"]

# Получить элемент
LINDEX tasks 0       # "task2"

# Получить диапазон
LRANGE tasks 0 -1    # Все элементы
# ["task2", "task1", "task3"]

# Длина списка
LLEN tasks           # 3

# Извлечь элемент (удалить и вернуть)
LPOP tasks           # "task2" (удален из списка)
RPOP tasks           # "task3"

# Блокирующий pop (ждет если пусто)
BLPOP tasks 10       # Ждет 10 секунд
# ["tasks", "task1"]
```

**Когда использовать:** Очереди, логи, истории действий.

### 4. Set (множества)

```bash
# Добавить элементы
SADD tags:post1 "redis" "database" "nosql"
# 3 (количество добавленных)

# Проверить членство
SISMEMBER tags:post1 "redis"  # 1 (true)
SISMEMBER tags:post1 "mysql"  # 0 (false)

# Все элементы
SMEMBERS tags:post1
# ["redis", "database", "nosql"]

# Количество элементов
SCARD tags:post1  # 3

# Удалить элемент
SREM tags:post1 "nosql"

# Случайный элемент
SRANDMEMBER tags:post1

# Извлечь случайный (удалить)
SPOP tags:post1

# Операции с множествами
SADD set1 "a" "b" "c"
SADD set2 "b" "c" "d"

SINTER set1 set2    # Пересечение: ["b", "c"]
SUNION set1 set2    # Объединение: ["a", "b", "c", "d"]
SDIFF set1 set2     # Разность: ["a"]
```

**Когда использовать:** Уникальные элементы, теги, подписки, друзья.

### 5. Sorted Set (упорядоченные множества)

```bash
# Добавить элементы с баллами (score)
ZADD leaderboard 100 "Alice"
ZADD leaderboard 200 "Bob"
ZADD leaderboard 150 "Charlie"

# Получить топ (по возрастанию)
ZRANGE leaderboard 0 -1
# ["Alice", "Charlie", "Bob"]

# Получить топ (по убыванию)
ZREVRANGE leaderboard 0 -1
# ["Bob", "Charlie", "Alice"]

# С баллами
ZREVRANGE leaderboard 0 -1 WITHSCORES
# ["Bob", "200", "Charlie", "150", "Alice", "100"]

# Получить балл
ZSCORE leaderboard "Bob"  # "200"

# Ранг элемента (позиция)
ZRANK leaderboard "Bob"   # 2 (0-based, по возрастанию)
ZREVRANK leaderboard "Bob" # 0 (по убыванию)

# Инкремент балла
ZINCRBY leaderboard 50 "Alice"  # Теперь 150

# Количество элементов
ZCARD leaderboard  # 3

# Количество в диапазоне баллов
ZCOUNT leaderboard 100 200  # 3

# Удалить элемент
ZREM leaderboard "Alice"

# Удалить по рангу
ZREMRANGEBYRANK leaderboard 0 0  # Удалить первый

# Удалить по баллу
ZREMRANGEBYSCORE leaderboard 0 100  # Удалить с баллами 0-100
```

**Когда использовать:** Рейтинги, лидерборды, очереди с приоритетами.

## Персистентность

Redis хранит данные в RAM, но может сохранять на диск.

### RDB (Redis Database)

Снимок (snapshot) данных в определенный момент.

```bash
# redis.conf
save 900 1      # Сохранить если 1+ изменение за 15 мин
save 300 10     # Сохранить если 10+ изменений за 5 мин
save 60 10000   # Сохранить если 10000+ изменений за 1 мин

# Форсировать сохранение
SAVE          # Синхронно (блокирует)
BGSAVE        # Асинхронно (background)
```

**Плюсы:**
- Компактный файл
- Быстрое восстановление

**Минусы:**
- Можно потерять данные между снимками

### AOF (Append Only File)

Лог всех записывающих операций.

```bash
# redis.conf
appendonly yes
appendfsync everysec   # Синхронизация каждую секунду (рекомендуется)
# appendfsync always   # После каждой команды (медленно)
# appendfsync no       # ОС решает когда (быстро, но риск)
```

**Плюсы:**
- Минимальная потеря данных (до 1 сек)

**Минусы:**
- Файл больше RDB
- Медленнее восстановление

**Рекомендация:** Использовать RDB + AOF вместе.

## Pub/Sub (издатель-подписчик)

```bash
# Подписаться на канал
SUBSCRIBE news
# Ждет сообщений...

# Опубликовать сообщение
PUBLISH news "Hello, subscribers!"
# 1 (количество получателей)

# Подписаться по паттерну
PSUBSCRIBE news:*

# Отписаться
UNSUBSCRIBE news
```

**Применение:** Real-time уведомления, чаты.

## Транзакции

```bash
# Начать транзакцию
MULTI
# OK

SET account:1 100
SET account:2 200
INCR counter

# Выполнить
EXEC
# [OK, OK, 1]

# Или отменить
DISCARD
```

**Особенности:**
- Команды выполняются атомарно
- Нет rollback (если команда ошибочна, остальные выполнятся)
- Не блокирует другие клиенты

### WATCH - оптимистичная блокировка

```bash
WATCH balance:1

# Если balance:1 изменится до EXEC, транзакция отменится
balance = GET balance:1

MULTI
SET balance:1 (balance - 100)
SET balance:2 (balance + 100)
EXEC
# nil (если balance:1 изменился)
```

## Использование в Go

### Подключение

```go
import "github.com/go-redis/redis/v8"

func main() {
    ctx := context.Background()

    // Подключение
    rdb := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "", // Пароль (если есть)
        DB:       0,  // Номер БД (0-15)
    })

    // Проверка подключения
    pong, err := rdb.Ping(ctx).Result()
    if err != nil {
        panic(err)
    }
    fmt.Println(pong) // "PONG"
}
```

### Примеры операций

```go
// SET / GET
err := rdb.Set(ctx, "user:1:name", "Alice", 0).Err()
val, err := rdb.Get(ctx, "user:1:name").Result()
fmt.Println(val) // "Alice"

// SET с TTL
err = rdb.Set(ctx, "session:abc", "data", 1*time.Hour).Err()

// INCR
views, err := rdb.Incr(ctx, "page:views").Result()

// Hash
err = rdb.HSet(ctx, "user:1", "name", "Alice", "age", 30).Err()
name, err := rdb.HGet(ctx, "user:1", "name").Result()
all, err := rdb.HGetAll(ctx, "user:1").Result()
// map[string]string{"name":"Alice", "age":"30"}

// List
err = rdb.LPush(ctx, "tasks", "task1", "task2").Err()
tasks, err := rdb.LRange(ctx, "tasks", 0, -1).Result()
// []string{"task2", "task1"}

// Set
err = rdb.SAdd(ctx, "tags", "redis", "database").Err()
isMember, err := rdb.SIsMember(ctx, "tags", "redis").Result()
members, err := rdb.SMembers(ctx, "tags").Result()

// Sorted Set
err = rdb.ZAdd(ctx, "leaderboard",
    &redis.Z{Score: 100, Member: "Alice"},
    &redis.Z{Score: 200, Member: "Bob"},
).Err()
top, err := rdb.ZRevRange(ctx, "leaderboard", 0, 9).Result()
```

### Pipeline (пакетная обработка)

```go
pipe := rdb.Pipeline()

// Добавить команды
pipe.Set(ctx, "key1", "value1", 0)
pipe.Set(ctx, "key2", "value2", 0)
pipe.Get(ctx, "key1")

// Выполнить все разом
cmds, err := pipe.Exec(ctx)
if err != nil {
    panic(err)
}

// Результат третьей команды (GET)
val := cmds[2].(*redis.StringCmd).Val()
```

### Pub/Sub

```go
// Подписка
pubsub := rdb.Subscribe(ctx, "news")
defer pubsub.Close()

ch := pubsub.Channel()
for msg := range ch {
    fmt.Println(msg.Channel, msg.Payload)
}

// Публикация
err := rdb.Publish(ctx, "news", "Breaking news!").Err()
```

## Паттерны использования

### 1. Кэширование

```go
func GetUser(id int) (*User, error) {
    cacheKey := fmt.Sprintf("user:%d", id)

    // Попытка из кэша
    cached, err := rdb.Get(ctx, cacheKey).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }

    // Из БД
    user, err := db.GetUser(id)
    if err != nil {
        return nil, err
    }

    // Сохранить в кэш
    data, _ := json.Marshal(user)
    rdb.Set(ctx, cacheKey, data, 1*time.Hour)

    return user, nil
}
```

### 2. Распределенная блокировка

```go
func AcquireLock(key string, ttl time.Duration) bool {
    // SET NX EX - атомарная операция
    result, err := rdb.SetNX(ctx, "lock:"+key, "locked", ttl).Result()
    return err == nil && result
}

func ReleaseLock(key string) {
    rdb.Del(ctx, "lock:"+key)
}

// Использование
if AcquireLock("resource1", 10*time.Second) {
    defer ReleaseLock("resource1")
    // Критическая секция
}
```

### 3. Rate Limiting

```go
func IsAllowed(userID string, limit int, window time.Duration) bool {
    key := fmt.Sprintf("ratelimit:%s", userID)

    count, err := rdb.Incr(ctx, key).Result()
    if err != nil {
        return false
    }

    if count == 1 {
        // Первый запрос - установить TTL
        rdb.Expire(ctx, key, window)
    }

    return count <= int64(limit)
}

// Использование: 10 запросов в минуту
if !IsAllowed("user123", 10, 1*time.Minute) {
    return errors.New("rate limit exceeded")
}
```

### 4. Очередь задач

```go
// Producer
func EnqueueTask(task string) {
    rdb.LPush(ctx, "tasks", task)
}

// Consumer
func ProcessTasks() {
    for {
        // Блокирующий pop (ждет если пусто)
        result, err := rdb.BRPop(ctx, 0, "tasks").Result()
        if err != nil {
            continue
        }

        task := result[1]
        processTask(task)
    }
}
```

### 5. Сессии

```go
func CreateSession(userID int) string {
    sessionID := generateID()
    key := fmt.Sprintf("session:%s", sessionID)

    data := map[string]interface{}{
        "user_id": userID,
        "created": time.Now(),
    }

    // Сохранить на 24 часа
    rdb.HSet(ctx, key, data)
    rdb.Expire(ctx, key, 24*time.Hour)

    return sessionID
}

func GetSession(sessionID string) (int, error) {
    key := fmt.Sprintf("session:%s", sessionID)
    userID, err := rdb.HGet(ctx, key, "user_id").Int()
    return userID, err
}
```

## Производительность

### Best Practices

1. ✅ **Pipeline** для множественных команд
2. ✅ **Connection pooling** (go-redis делает автоматически)
3. ✅ **Используйте Hash** вместо множества ключей
4. ❌ Избегайте `KEYS` в production (используйте `SCAN`)
5. ✅ Мониторьте `slowlog`
6. ✅ Настройте `maxmemory` и eviction policy

### Eviction Policies

```bash
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru  # Удалять LRU ключи

# Политики:
# noeviction - ошибка при нехватке памяти
# allkeys-lru - удалять наименее используемые
# volatile-lru - удалять LRU среди ключей с TTL
# allkeys-random - случайное удаление
# volatile-ttl - удалять ближайшие к истечению
```

### Мониторинг

```bash
# Статистика
INFO stats

# Медленные запросы
SLOWLOG GET 10

# Память
INFO memory

# Клиенты
CLIENT LIST
```

## Ограничения

- **Single-threaded:** Одна команда в момент времени
- **Память:** Все данные должны влезать в RAM
- **Нет JOIN:** Не реляционная БД
- **Нет сложных запросов:** Key-value модель

## Связанные темы

- [[Redis - Типы данных и команды]]
- [[Redis - Кэширование стратегии]]
- [[Типы СУБД - SQL vs NoSQL]]
- [[PostgreSQL - Основы]]
- [[Go - Пакет sync]]
