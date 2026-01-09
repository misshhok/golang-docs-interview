# PostgreSQL - ACID

ACID - четыре ключевых свойства транзакций в реляционных базах данных, которые гарантируют надежность и согласованность данных.

## Что такое ACID?

**ACID** = Atomicity + Consistency + Isolation + Durability

Это гарантии того, что данные останутся корректными даже при сбоях, конкурентном доступе и других проблемах.

## A - Atomicity (Атомарность)

**Принцип:** Транзакция выполняется либо полностью, либо не выполняется вообще. Нет частичных изменений.

### Пример без атомарности

```sql
-- ❌ Опасно без транзакции
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Сбой! 💥
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Деньги пропали!
```

### Пример с атомарностью

```sql
-- ✅ Безопасно с транзакцией
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Если что-то пошло не так
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    -- Ошибка! Constraint violation
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
ROLLBACK;  -- Откатываем ВСЁ
```

### Автоматический ROLLBACK

```sql
BEGIN;
    INSERT INTO orders (user_id, amount) VALUES (1, 100);
    INSERT INTO order_items (order_id, product_id) VALUES (999, 1);  -- Нет order_id=999
    -- PostgreSQL автоматически делает ROLLBACK
COMMIT;  -- Не выполнится, транзакция уже aborted
```

### Savepoints (точки сохранения)

```sql
BEGIN;
    INSERT INTO users (email) VALUES ('user1@example.com');

    SAVEPOINT sp1;
    INSERT INTO users (email) VALUES ('user2@example.com');

    SAVEPOINT sp2;
    INSERT INTO users (email) VALUES ('invalid');  -- Ошибка!

    ROLLBACK TO sp2;  -- Откат до sp2, user2 остался

    INSERT INTO users (email) VALUES ('user3@example.com');
COMMIT;

-- Результат: user1, user2, user3 (invalid откачен)
```

## C - Consistency (Согласованность)

**Принцип:** Транзакция переводит базу из одного корректного состояния в другое корректное. Все ограничения (constraints) соблюдаются.

### Constraints в PostgreSQL

```sql
CREATE TABLE orders (
    id serial PRIMARY KEY,
    user_id int NOT NULL,
    amount decimal(10,2) CHECK (amount > 0),  -- ✅ Constraint
    status varchar(20) CHECK (status IN ('pending', 'paid', 'cancelled')),
    created_at timestamp DEFAULT now()
);

-- ❌ Нарушает constraint
INSERT INTO orders (user_id, amount, status)
VALUES (1, -100, 'pending');
-- ERROR: new row violates check constraint "orders_amount_check"

-- ❌ Нарушает constraint
INSERT INTO orders (user_id, amount, status)
VALUES (1, 100, 'invalid_status');
-- ERROR: new row violates check constraint "orders_status_check"
```

### Foreign Key constraints

```sql
CREATE TABLE order_items (
    id serial PRIMARY KEY,
    order_id int REFERENCES orders(id),  -- ✅ FK constraint
    product_id int REFERENCES products(id),
    quantity int CHECK (quantity > 0)
);

-- ❌ Нарушает FK constraint
INSERT INTO order_items (order_id, product_id, quantity)
VALUES (999, 1, 10);  -- Нет order_id=999
-- ERROR: insert or update on table "order_items" violates foreign key constraint
```

### Triggers для сложной логики

```sql
-- Проверка баланса перед переводом
CREATE OR REPLACE FUNCTION check_balance()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT balance FROM accounts WHERE id = OLD.id) < NEW.withdrawal THEN
        RAISE EXCEPTION 'Insufficient balance';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_check_balance
BEFORE UPDATE ON accounts
FOR EACH ROW
EXECUTE FUNCTION check_balance();
```

### Deferred Constraints

```sql
CREATE TABLE employees (
    id serial PRIMARY KEY,
    name varchar(100),
    manager_id int REFERENCES employees(id) DEFERRABLE INITIALLY DEFERRED
);

BEGIN;
    -- Временно нарушаем constraint (manager_id=2 еще не существует)
    INSERT INTO employees (id, name, manager_id) VALUES (1, 'Alice', 2);
    INSERT INTO employees (id, name, manager_id) VALUES (2, 'Bob', NULL);
COMMIT;  -- Проверка constraints в конце транзакции ✅
```

## I - Isolation (Изолированность)

**Принцип:** Конкурентные транзакции не влияют друг на друга. Каждая видит согласованное состояние.

### Уровни изоляции в PostgreSQL

PostgreSQL поддерживает 3 уровня изоляции из 4 стандарта SQL:

| Уровень | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
|---------|------------|---------------------|--------------|----------------------|
| Read Uncommitted | ❌ (как RC) | ❌ | ❌ | ❌ |
| Read Committed | ✅ | ❌ | ❌ | ❌ |
| Repeatable Read | ✅ | ✅ | ✅ (MVCC) | ❌ |
| Serializable | ✅ | ✅ | ✅ | ✅ |

#### 1. Read Committed (по умолчанию)

```sql
-- Транзакция 1
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 100
-- Ждем...

-- Транзакция 2
BEGIN;
UPDATE accounts SET balance = 50 WHERE id = 1;
COMMIT;

-- Транзакция 1 (продолжение)
SELECT balance FROM accounts WHERE id = 1;  -- 50 (изменилось!)
COMMIT;
```

**Проблемы:**
- Non-Repeatable Read: значение изменилось между SELECT
- Phantom Read: новые строки могут появиться

#### 2. Repeatable Read

```sql
-- Транзакция 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- 100

-- Транзакция 2
BEGIN;
UPDATE accounts SET balance = 50 WHERE id = 1;
COMMIT;

-- Транзакция 1 (продолжение)
SELECT balance FROM accounts WHERE id = 1;  -- Все еще 100! (snapshot)
COMMIT;
```

**PostgreSQL использует MVCC (Multi-Version Concurrency Control):**
- Каждая транзакция видит "снимок" данных на момент начала
- Нет Phantom Read благодаря MVCC
- Возможна ошибка сериализации при UPDATE

```sql
-- Транзакция 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- 100
UPDATE accounts SET balance = 90 WHERE id = 1;

-- Транзакция 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- 100 (старый snapshot)
UPDATE accounts SET balance = 80 WHERE id = 1;
-- ERROR: could not serialize access due to concurrent update
ROLLBACK;

-- Транзакция 1
COMMIT;  -- ✅ Успех
```

#### 3. Serializable

Самый строгий уровень. Гарантирует, что результат эквивалентен последовательному выполнению транзакций.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Обнаруживает аномалии сериализации:**

```sql
-- Транзакция 1
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(balance) FROM accounts WHERE user_id = 1;  -- 1000
INSERT INTO accounts (user_id, balance) VALUES (1, 100);
COMMIT;

-- Транзакция 2
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(balance) FROM accounts WHERE user_id = 1;  -- 1000
INSERT INTO accounts (user_id, balance) VALUES (1, 50);
-- ERROR: could not serialize access due to read/write dependencies
ROLLBACK;
```

### MVCC (Multi-Version Concurrency Control)

PostgreSQL не блокирует читателей! Каждая транзакция видит свою версию данных.

```
Таблица accounts (id=1):

Версия 1: balance=100, xmin=1000, xmax=NULL
Версия 2: balance=50,  xmin=1001, xmax=NULL

Транзакция 1000: видит balance=100
Транзакция 1001: видит balance=50
Транзакция 1002: видит balance=50
```

**xmin** = ID транзакции, которая создала строку
**xmax** = ID транзакции, которая удалила строку (NULL если активна)

### Блокировки (Locks)

```sql
-- SELECT ... FOR UPDATE (эксклюзивная блокировка)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Другие транзакции будут ждать
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- SELECT ... FOR SHARE (разделяемая блокировка)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;
-- Другие могут читать, но не могут изменять
COMMIT;
```

**Типы блокировок:**
- `FOR UPDATE` - эксклюзивная (exclusive)
- `FOR NO KEY UPDATE` - как FOR UPDATE, но допускает FK checks
- `FOR SHARE` - разделяемая (shared)
- `FOR KEY SHARE` - самая слабая (только для FK)

### Deadlock (взаимная блокировка)

```sql
-- Транзакция 1
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;
-- Ждем...

-- Транзакция 2
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 2;
UPDATE accounts SET balance = balance + 10 WHERE id = 1;  -- Ждет T1

-- Транзакция 1 (продолжение)
UPDATE accounts SET balance = balance + 10 WHERE id = 2;  -- Ждет T2
-- DEADLOCK! PostgreSQL обнаруживает и откатывает одну из транзакций
-- ERROR: deadlock detected
```

**Как избежать:**
- Всегда блокировать ресурсы в одном порядке
- Использовать `LOCK TABLE` явно
- Минимизировать время удержания блокировок

## D - Durability (Долговечность)

**Принцип:** После COMMIT изменения сохраняются навсегда, даже при сбое системы.

### WAL (Write-Ahead Logging)

PostgreSQL использует WAL для durability:

1. **Изменения записываются в WAL** (на диск)
2. **Затем в основные файлы данных** (асинхронно)

```
Transaction:
  UPDATE accounts SET balance = 50 WHERE id = 1;
  COMMIT;

1. Запись в WAL (fsync) ✅
2. Возврат клиенту "SUCCESS"
3. Фоновый процесс записывает в data files
```

**Если сбой после шага 2:** При перезапуске PostgreSQL "проиграет" WAL и восстановит данные.

### Настройки durability

```sql
-- Синхронная запись на диск (по умолчанию)
SET synchronous_commit = on;  -- Самый безопасный

-- Асинхронная запись (быстрее, но риск потери последних транзакций)
SET synchronous_commit = off;  -- Риск потери до 1 сек данных

-- Отключить fsync (только для тестов!)
SET fsync = off;  -- ⚠️ ОПАСНО! Данные могут потеряться
```

### Checkpoint

Периодически PostgreSQL записывает "грязные" страницы на диск:

```sql
-- Форсировать checkpoint
CHECKPOINT;
```

**Настройки:**
```sql
-- Максимальное время между checkpoint
checkpoint_timeout = 5min

-- Максимальный размер WAL
max_wal_size = 1GB
```

### Репликация для durability

```sql
-- Синхронная репликация (данные на нескольких серверах)
synchronous_standby_names = 'standby1'

-- После COMMIT данные гарантированно на обоих серверах
```

## Практические примеры

### Банковский перевод

```sql
BEGIN;
    -- Списание с первого счета
    UPDATE accounts
    SET balance = balance - 100
    WHERE id = 1 AND balance >= 100;

    -- Проверка, что UPDATE выполнился
    IF NOT FOUND THEN
        ROLLBACK;
        RAISE EXCEPTION 'Insufficient balance';
    END IF;

    -- Пополнение второго счета
    UPDATE accounts
    SET balance = balance + 100
    WHERE id = 2;

COMMIT;
```

### Резервирование товара

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
    -- Проверить наличие
    SELECT quantity FROM products WHERE id = 1 FOR UPDATE;

    -- Если достаточно, зарезервировать
    UPDATE products
    SET quantity = quantity - 5
    WHERE id = 1 AND quantity >= 5;

    IF NOT FOUND THEN
        ROLLBACK;
        RAISE EXCEPTION 'Not enough stock';
    END IF;

    -- Создать заказ
    INSERT INTO orders (product_id, quantity, status)
    VALUES (1, 5, 'reserved');

COMMIT;
```

### Счетчик с конкурентным доступом

```sql
-- ❌ Плохо (race condition)
BEGIN;
SELECT counter FROM stats WHERE id = 1;  -- Читаем 100
UPDATE stats SET counter = 101 WHERE id = 1;
COMMIT;

-- Другая транзакция тоже прочитала 100 и записала 101!
-- Потеряли одно инкрементирование

-- ✅ Правильно (атомарная операция)
BEGIN;
UPDATE stats SET counter = counter + 1 WHERE id = 1;
COMMIT;
```

## Проверка ACID на практике

### Atomicity

```sql
-- Тест: транзакция откатывается при ошибке
BEGIN;
INSERT INTO users (email) VALUES ('user1@example.com');
INSERT INTO users (email) VALUES (NULL);  -- Ошибка NOT NULL
COMMIT;  -- Не сработает

-- Проверка
SELECT * FROM users WHERE email = 'user1@example.com';
-- Пусто (откат)
```

### Consistency

```sql
-- Тест: constraint не позволяет нарушить целостность
INSERT INTO orders (user_id, amount) VALUES (1, -100);
-- ERROR: constraint violation
```

### Isolation

```sql
-- Terminal 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM accounts WHERE id = 1;

-- Terminal 2
BEGIN;
UPDATE accounts SET balance = 999 WHERE id = 1;
COMMIT;

-- Terminal 1
SELECT * FROM accounts WHERE id = 1;  -- Все еще старое значение
COMMIT;
```

### Durability

```sql
BEGIN;
INSERT INTO logs (message) VALUES ('critical data');
COMMIT;

-- Убить PostgreSQL процесс
-- Перезапустить
SELECT * FROM logs;  -- Данные на месте ✅
```

## Компромиссы

| Свойство | Строгость | Производительность |
|----------|-----------|-------------------|
| Atomicity | Обязательно | Небольшой overhead |
| Consistency | Обязательно | Зависит от constraints |
| Isolation | Настраиваемо | ⬇️ Serializable медленнее |
| Durability | Настраиваемо | ⬇️ Синхронный fsync медленнее |

**Для большинства случаев:**
- Isolation: Read Committed (по умолчанию)
- Durability: synchronous_commit = on (по умолчанию)

**Для критичных данных (финансы):**
- Isolation: Serializable
- Durability: synchronous_commit = on + репликация

**Для аналитики (можно потерять последние секунды):**
- Isolation: Read Committed
- Durability: synchronous_commit = off

## Ошибки и как их избежать

### 1. Lost Update

```sql
-- ❌ Проблема
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 100
-- Другая транзакция тоже прочитала 100
UPDATE accounts SET balance = 100 - 50 WHERE id = 1;
COMMIT;

-- ✅ Решение: атомарный UPDATE
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 1;
COMMIT;
```

### 2. Dirty Read

PostgreSQL **не допускает** Dirty Read даже на Read Uncommitted.

### 3. Deadlock

```sql
-- ✅ Всегда блокировать в одном порядке
BEGIN;
-- Сначала id=1, потом id=2 (всегда!)
UPDATE accounts SET balance = balance - 10 WHERE id = 1;
UPDATE accounts SET balance = balance + 10 WHERE id = 2;
COMMIT;
```

## Best Practices

1. ✅ Используйте транзакции для связанных операций
2. ✅ Минимизируйте время транзакции (быстро COMMIT)
3. ✅ Выбирайте правильный уровень изоляции
4. ✅ Используйте `SELECT ... FOR UPDATE` для критичных данных
5. ✅ Обрабатывайте serialization errors (retry логика)
6. ❌ Не делайте долгих операций в транзакции (API calls, sleep)
7. ❌ Не используйте Serializable без необходимости (медленно)
8. ✅ Мониторьте deadlocks и long-running транзакции

## Мониторинг транзакций

```sql
-- Активные транзакции
SELECT
    pid,
    usename,
    state,
    query_start,
    now() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Блокировки
SELECT
    blocked_locks.pid AS blocked_pid,
    blocking_locks.pid AS blocking_pid,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

## Связанные темы

- [[PostgreSQL - Основы]]
- [[PostgreSQL - Блокировки и изоляция транзакций]]
- [[PostgreSQL - Типы индексов]]
- [[Алгоритмическая сложность (Big O)]]
