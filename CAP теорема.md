# CAP теорема

CAP теорема (теорема Брюера) утверждает, что распределенная система не может одновременно гарантировать все три свойства: **Consistency** (согласованность), **Availability** (доступность) и **Partition tolerance** (устойчивость к разделению). Можно выбрать только 2 из 3.

## Три свойства CAP

### C - Consistency (Согласованность)

**Определение:** Все узлы распределенной системы видят одинаковые данные в один и тот же момент времени.

**Пример:**
```
Пользователь записал данные в узел A → данные = "X"
Сразу же читает из узла B → получает данные = "X"

Если получает старые данные или данные отличаются → нарушена согласованность
```

**В реальности:**
- После записи в один узел все остальные узлы немедленно видят новые данные
- Любое чтение возвращает самую последнюю запись
- Читатели никогда не видят устаревшие данные

**Проблемы:**
- Требует синхронизации между узлами → медленнее
- При сетевых сбоях невозможно гарантировать

### A - Availability (Доступность)

**Определение:** Каждый запрос к системе получает ответ (success или failure), даже если некоторые узлы недоступны.

**Пример:**
```
Узел A недоступен
Запрос к узлу B → получаем ответ (возможно, устаревшие данные)

Если система вернула ошибку "нет доступа" → нарушена доступность
```

**В реальности:**
- Система продолжает обрабатывать запросы даже при частичном отказе
- Нет timeout'ов, нет отказов в обслуживании
- Данные могут быть несогласованными, но ответ будет

**Проблемы:**
- Может возвращать устаревшие данные
- Сложно обеспечить при строгой консистентности

### P - Partition Tolerance (Устойчивость к разделению)

**Определение:** Система продолжает работать, несмотря на потерю или задержку произвольного количества сообщений между узлами (сетевое разделение).

**Пример:**
```
Сеть разделилась:
Узлы A, B могут общаться между собой
Узлы C, D могут общаться между собой
Но группы A-B и C-D не могут связаться друг с другом

→ Система продолжает работать
```

**В реальности:**
- Сеть может упасть в любой момент (особенно в дата-центрах)
- Задержки, обрывы связи, потеря пакетов
- Это неизбежно в распределенных системах

**Важно:** В реальных распределенных системах P (Partition tolerance) обязательно, так как сетевые сбои неизбежны. Поэтому реальный выбор — между C и A.

## Почему нельзя все три?

При сетевом разделении (P) возникает конфликт:

**Вариант 1: Выбрать C (Consistency)**
```
Сеть разделилась → узлы не могут синхронизироваться
Чтобы гарантировать согласованность → отказываем в обслуживании
→ Потеря A (Availability)
```

**Вариант 2: Выбрать A (Availability)**
```
Сеть разделилась → узлы не могут синхронизироваться
Чтобы гарантировать доступность → продолжаем работать
→ Узлы возвращают разные данные → потеря C (Consistency)
```

## Типы систем

### CP системы (Consistency + Partition tolerance)

**Жертвуем:** Availability (доступность)

**Стратегия:** При сетевом разделении отказываем в обслуживании, чтобы сохранить согласованность.

**Примеры:**
- **MongoDB** (с strong consistency mode)
- **HBase**
- **Redis Cluster** (по умолчанию)
- **Consul**
- **etcd**
- **ZooKeeper**

**Поведение:**
```
Сеть разделилась
→ Узлы, которые не могут синхронизироваться, возвращают ошибку
→ Клиенты получают: "Service Unavailable" или timeout
→ Но данные всегда согласованы
```

**Когда использовать:**
- Финансовые транзакции (нельзя показывать неверный баланс)
- Критичные операции (нельзя допустить race condition)
- Конфигурация и координация (ZooKeeper, etcd)

**Пример на MongoDB:**
```go
// MongoDB с strong read concern
opts := options.Client().
    ApplyURI("mongodb://localhost:27017").
    SetReadConcern(readconcern.Majority()).
    SetWriteConcern(writeconcern.New(writeconcern.WMajority()))

client, _ := mongo.Connect(ctx, opts)

// При сетевом разделении:
// - Запись завершится с ошибкой, если большинство узлов недоступно
// - Чтение вернет ошибку, если нет согласованных данных
// → Жертвуем доступностью ради согласованности
```

**Плюсы:**
- ✅ Данные всегда корректны
- ✅ Нет риска устаревших данных
- ✅ Простая логика приложения (нет eventual consistency)

**Минусы:**
- ❌ Система может быть недоступна
- ❌ Медленнее (синхронизация обязательна)

### AP системы (Availability + Partition tolerance)

**Жертвуем:** Consistency (согласованность)

**Стратегия:** При сетевом разделении продолжаем обрабатывать запросы, допуская временную несогласованность.

**Примеры:**
- **Cassandra**
- **DynamoDB**
- **Riak**
- **CouchDB**
- **Voldemort**

**Поведение:**
```
Сеть разделилась
→ Каждый узел продолжает обрабатывать запросы независимо
→ Данные расходятся (разные узлы хранят разные версии)
→ После восстановления сети происходит reconciliation (примирение конфликтов)
```

**Eventual Consistency:**
Данные в итоге станут согласованными, но не сразу.

**Когда использовать:**
- Социальные сети (лайки, комментарии — допустимы задержки)
- Логи, метрики (важна доступность записи)
- Кэширование (данные могут быть устаревшими)
- Каталоги продуктов (не критично показать старую цену на секунду)

**Пример на Cassandra:**
```sql
-- Cassandra
CREATE TABLE users (
    id uuid PRIMARY KEY,
    name text,
    email text
);

-- Настройки consistency level:
-- ONE - запись на 1 узел (максимальная доступность)
-- QUORUM - запись на большинство узлов (баланс)
-- ALL - запись на все узлы (максимальная согласованность)

-- Запись с CL = ONE
INSERT INTO users (id, name, email)
VALUES (uuid(), 'Alice', 'alice@example.com')
USING CONSISTENCY ONE;

-- При сетевом разделении:
-- - Запись успешна, если доступен хотя бы 1 узел
-- - Чтение может вернуть устаревшие данные
-- → Жертвуем согласованностью ради доступности
```

**Пример на Go с Cassandra:**
```go
import "github.com/gocql/gocql"

cluster := gocql.NewCluster("127.0.0.1")
cluster.Consistency = gocql.One // Высокая доступность

session, _ := cluster.CreateSession()

// Запись всегда успешна (если доступен хоть 1 узел)
err := session.Query(`
    INSERT INTO users (id, name, email) VALUES (?, ?, ?)
`, gocql.TimeUUID(), "Alice", "alice@example.com").Exec()

// Чтение может вернуть устаревшие данные
var name string
session.Query(`SELECT name FROM users WHERE id = ?`, userID).
    Consistency(gocql.One).
    Scan(&name)
```

**Плюсы:**
- ✅ Всегда доступна (даже при сбоях)
- ✅ Быстрая запись
- ✅ Горизонтальное масштабирование

**Минусы:**
- ❌ Eventual consistency (данные могут быть несогласованными)
- ❌ Конфликты при reconciliation
- ❌ Сложная логика приложения

### CA системы (Consistency + Availability)

**Жертвуем:** Partition tolerance (устойчивость к разделению)

**Стратегия:** Нет распределения — работаем на одном узле или в сети без разделений.

**Примеры:**
- **Традиционные RDBMS** (PostgreSQL, MySQL) на одном сервере
- **SQLite**
- **Single-node Redis**

**Поведение:**
```
Нет сетевого разделения (один узел или надежная сеть)
→ Согласованность + доступность гарантированы
→ Но если сервер упадет или сеть разделится → вся система недоступна
```

**Когда использовать:**
- Малые проекты (один сервер достаточно)
- Локальные приложения (десктоп, мобильные)
- Когда критична ACID, но масштаб небольшой

**Пример:**
```go
// PostgreSQL на одном сервере
db, _ := sql.Open("postgres", "postgresql://localhost/mydb")

// ACID транзакции
tx, _ := db.Begin()
tx.Exec("UPDATE accounts SET balance = balance - 100 WHERE id = 1")
tx.Exec("UPDATE accounts SET balance = balance + 100 WHERE id = 2")
tx.Commit()

// При падении сервера:
// → Вся система недоступна
// → Но данные всегда согласованы и доступность была до сбоя
```

**Плюсы:**
- ✅ ACID гарантии
- ✅ Простая модель данных
- ✅ Сильная согласованность

**Минусы:**
- ❌ Single point of failure
- ❌ Не масштабируется горизонтально
- ❌ Нет отказоустойчивости

**Важно:** В реальном мире CA системы практически не применяются в распределенных системах, так как сетевые разделения неизбежны.

## Сравнительная таблица

| Система | Тип | Consistency | Availability | Partition Tolerance | Примеры |
|---------|-----|-------------|--------------|---------------------|---------|
| **CP** | Жертвуем A | ✅ Строгая | ❌ Может быть недоступна | ✅ Устойчива | MongoDB, HBase, Redis Cluster, ZooKeeper, etcd |
| **AP** | Жертвуем C | ❌ Eventual | ✅ Всегда доступна | ✅ Устойчива | Cassandra, DynamoDB, Riak, CouchDB |
| **CA** | Жертвуем P | ✅ Строгая | ✅ Доступна | ❌ Не устойчива | PostgreSQL (single), MySQL (single), SQLite |

## Реальный мир: PACELC

**PACELC** — расширение CAP для реального мира:
- **P** (Partition): Если сеть разделена → выбираем между **A** и **C**
- **E** (Else): Если сеть работает нормально → выбираем между **L**atency (задержка) и **C**onsistency (согласованность)

**Примеры:**

**PA/EL (Cassandra):**
- При разделении: Availability
- Обычно: Low latency (быстро, но eventual consistency)

**PC/EC (MongoDB):**
- При разделении: Consistency
- Обычно: Consistency (медленнее, но согласованно)

**PA/EC (DynamoDB):**
- При разделении: Availability
- Обычно: Consistency (настраиваемо)

## Практические рекомендации

### 1. Выбор системы

**Для финансов, платежей, критичных операций:**
- ✅ Выбирайте CP (PostgreSQL, MongoDB strong consistency)
- Нельзя допустить несогласованные данные

**Для социальных сетей, метрик, логов:**
- ✅ Выбирайте AP (Cassandra, DynamoDB)
- Доступность важнее согласованности

**Для малых проектов:**
- ✅ Выбирайте CA (PostgreSQL single node)
- Пока не нужно распределение

### 2. Eventual Consistency в Go

```go
type UserService struct {
    primaryDB   *sql.DB       // PostgreSQL (source of truth)
    cacheDB     *redis.Client // Redis (eventual consistency)
}

func (s *UserService) UpdateUser(user *User) error {
    // 1. Запись в primary DB (CP система)
    err := s.primaryDB.Exec("UPDATE users SET name = $1 WHERE id = $2", user.Name, user.ID)
    if err != nil {
        return err
    }

    // 2. Асинхронное обновление кэша (AP система)
    go func() {
        data, _ := json.Marshal(user)
        s.cacheDB.Set(context.Background(), fmt.Sprintf("user:%d", user.ID), data, 1*time.Hour)
    }()

    // Eventual consistency:
    // - PostgreSQL сразу имеет свежие данные
    // - Redis будет обновлен через несколько миллисекунд
    // - Читатели могут увидеть старые данные из кэша
    return nil
}

func (s *UserService) GetUser(id int) (*User, error) {
    // Попытка из кэша (может быть устаревшим)
    cacheKey := fmt.Sprintf("user:%d", id)
    cached, err := s.cacheDB.Get(context.Background(), cacheKey).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil // Возможно устаревшие данные (eventual consistency)
    }

    // Fallback на PostgreSQL (актуальные данные)
    var user User
    err = s.primaryDB.QueryRow("SELECT id, name, email FROM users WHERE id = $1", id).
        Scan(&user.ID, &user.Name, &user.Email)
    return &user, err
}
```

### 3. Tunable Consistency

Некоторые системы (Cassandra, DynamoDB) позволяют настраивать consistency level:

```go
// Cassandra с настраиваемой согласованностью
cluster := gocql.NewCluster("127.0.0.1")

// Consistency level для записи:
// ONE - быстро, но может быть потеряно
// QUORUM - баланс (большинство узлов)
// ALL - медленно, но надежно

// Для критичных данных (финансы):
cluster.Consistency = gocql.Quorum

// Для некритичных данных (логи):
cluster.Consistency = gocql.One

session, _ := cluster.CreateSession()
```

### 4. Полиглот персистентность

Комбинируйте разные системы:

```go
type OrderService struct {
    postgresDB  *sql.DB         // CP: транзакции заказов (ACID критично)
    cassandraDB *gocql.Session  // AP: история просмотров (eventual OK)
    redisCache  *redis.Client   // AP: кэш (eventual OK)
}

func (s *OrderService) CreateOrder(order *Order) error {
    // PostgreSQL (CP) для критичных данных
    tx, _ := s.postgresDB.Begin()
    tx.Exec("INSERT INTO orders (...) VALUES (...)")
    tx.Exec("UPDATE inventory SET quantity = quantity - 1 WHERE product_id = ?", order.ProductID)
    return tx.Commit() // ACID гарантии
}

func (s *OrderService) TrackView(userID, productID int) {
    // Cassandra (AP) для некритичных данных
    s.cassandraDB.Query(`
        INSERT INTO product_views (user_id, product_id, viewed_at)
        VALUES (?, ?, ?)
    `, userID, productID, time.Now()).Exec()
    // Eventual consistency OK - не критично если запись потеряется
}
```

## Типичные заблуждения

❌ **"NoSQL всегда eventual consistency"**
- MongoDB может работать в strong consistency режиме (CP)

❌ **"SQL всегда ACID и CP"**
- PostgreSQL с репликацией может быть настроен как AP (асинхронная репликация)

❌ **"CAP означает выбор одного из трех"**
- На самом деле P (Partition tolerance) обязательно в распределенных системах
- Реальный выбор: C или A при сетевом разделении

❌ **"Eventual consistency — это плохо"**
- Для многих задач это оптимальный выбор (социальные сети, логи, метрики)
- Позволяет достичь высокой доступности и производительности

## Связанные темы

- [[Типы СУБД - SQL vs NoSQL]]
- [[PostgreSQL - ACID]]
- [[PostgreSQL - Блокировки и изоляция транзакций]]
- [[MongoDB - Основы]]
- [[Redis - Key-Value хранилище]]
- [[Микросервисная архитектура]]
- [[System Design - Основы]]
