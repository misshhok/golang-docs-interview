# PostgreSQL - Блокировки и изоляция транзакций

Подробный разбор механизмов блокировок, уровней изоляции и конкурентного доступа в PostgreSQL.

## Зачем нужны блокировки?

Блокировки предотвращают конфликты при одновременной работе нескольких транзакций с одними данными.

**Без блокировок:**
```
T1: SELECT balance FROM accounts WHERE id = 1;  -- 100
T2: SELECT balance FROM accounts WHERE id = 1;  -- 100
T1: UPDATE accounts SET balance = 90 WHERE id = 1;
T2: UPDATE accounts SET balance = 80 WHERE id = 1;
-- Lost update! T1 изменения потеряны
```

**С блокировками:**
```
T1: SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- 100, locked
T2: SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- ждет T1
T1: UPDATE accounts SET balance = 90 WHERE id = 1; COMMIT;
T2: -- теперь видит 90, а не 100
```

## Типы блокировок

### 1. Table-Level Locks (блокировки таблиц)

PostgreSQL поддерживает 8 режимов блокировок таблиц.

```sql
-- ACCESS SHARE (самая слабая)
-- Берется автоматически при SELECT
SELECT * FROM users;

-- ROW SHARE
-- Берется при SELECT ... FOR UPDATE/SHARE
SELECT * FROM users WHERE id = 1 FOR UPDATE;

-- ROW EXCLUSIVE
-- Берется при INSERT, UPDATE, DELETE
UPDATE users SET name = 'Alice' WHERE id = 1;

-- SHARE UPDATE EXCLUSIVE
-- Берется при VACUUM, CREATE INDEX CONCURRENTLY
VACUUM users;

-- SHARE
-- Берется при CREATE INDEX
CREATE INDEX idx_users_email ON users(email);

-- SHARE ROW EXCLUSIVE
-- Редко используется

-- EXCLUSIVE
-- Блокирует все операции кроме SELECT
LOCK TABLE users IN EXCLUSIVE MODE;

-- ACCESS EXCLUSIVE (самая сильная)
-- Блокирует ВСЁ (включая SELECT)
LOCK TABLE users IN ACCESS EXCLUSIVE MODE;
-- Используется при ALTER TABLE, DROP TABLE, TRUNCATE
```

### Матрица конфликтов

|  | ACCESS SHARE | ROW SHARE | ROW EXCL | SHARE | EXCL | ACCESS EXCL |
|--|--------------|-----------|----------|-------|------|-------------|
| ACCESS SHARE | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| ROW SHARE | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| ROW EXCLUSIVE | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| SHARE | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| EXCLUSIVE | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| ACCESS EXCLUSIVE | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

### 2. Row-Level Locks (блокировки строк)

PostgreSQL поддерживает 4 режима блокировки строк.

#### FOR UPDATE (эксклюзивная блокировка)

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Строка заблокирована
-- Другие транзакции НЕ могут:
--   - SELECT ... FOR UPDATE (ждут)
--   - UPDATE (ждут)
--   - DELETE (ждут)
-- Другие транзакции МОГУТ:
--   - SELECT (без FOR UPDATE)

UPDATE accounts SET balance = 50 WHERE id = 1;
COMMIT;
```

**Когда использовать:** Когда планируете изменить строку.

#### FOR NO KEY UPDATE

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE;
-- Блокирует изменения, но допускает FK checks

-- Другая транзакция может проверить FK
SELECT * FROM orders WHERE account_id = 1 FOR KEY SHARE;  -- ✅ OK
COMMIT;
```

**Отличие от FOR UPDATE:** Допускает `FOR KEY SHARE` (для проверки FK).

#### FOR SHARE (разделяемая блокировка)

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR SHARE;
-- Строка заблокирована для чтения
-- Другие транзакции НЕ могут:
--   - UPDATE (ждут)
--   - DELETE (ждут)
--   - FOR UPDATE (ждут)
-- Другие транзакции МОГУТ:
--   - SELECT
--   - FOR SHARE (несколько одновременно)

COMMIT;
```

**Когда использовать:** Когда хотите гарантировать, что строка не изменится, пока вы работаете.

#### FOR KEY SHARE (самая слабая)

```sql
BEGIN;
SELECT * FROM users WHERE id = 1 FOR KEY SHARE;
-- Блокирует только изменение ключевых полей (PK, UNIQUE)

-- Другие транзакции МОГУТ:
--   - UPDATE некоторых полей
--   - FOR KEY SHARE
-- Другие транзакции НЕ могут:
--   - UPDATE ключевых полей
--   - DELETE

COMMIT;
```

**Когда использовать:** При проверке FK. PostgreSQL использует автоматически.

### Сравнение Row-Level Locks

| Блокировка | Блокирует UPDATE | Блокирует DELETE | Блокирует FOR UPDATE | Блокирует FOR SHARE |
|------------|-----------------|------------------|---------------------|-------------------|
| FOR UPDATE | ✅ | ✅ | ✅ | ✅ |
| FOR NO KEY UPDATE | ✅ | ✅ | ✅ | ✅ |
| FOR SHARE | ✅ | ✅ | ✅ | ❌ |
| FOR KEY SHARE | ❌ (некоторые) | ✅ | ✅ | ❌ |

### 3. Advisory Locks (пользовательские блокировки)

Блокировки на уровне приложения (не связаны с данными).

```sql
-- Получить эксклюзивную блокировку
SELECT pg_advisory_lock(123);

-- Попытка получить блокировку (не ждет)
SELECT pg_try_advisory_lock(123);  -- false если занято

-- Освободить блокировку
SELECT pg_advisory_unlock(123);

-- Блокировка на уровне транзакции (автоматически освобождается)
BEGIN;
SELECT pg_advisory_xact_lock(123);
COMMIT;  -- Освобождена
```

**Когда использовать:**
- Координация между процессами приложения
- Предотвращение запуска двух одинаковых задач
- Реализация распределенных блокировок

**Пример: предотвращение дублирования задач**

```go
func ProcessTask(db *sql.DB, taskID int) error {
    // Попробовать получить блокировку
    var acquired bool
    err := db.QueryRow("SELECT pg_try_advisory_lock($1)", taskID).Scan(&acquired)
    if err != nil || !acquired {
        return errors.New("task already running")
    }
    defer db.Exec("SELECT pg_advisory_unlock($1)", taskID)

    // Обработка задачи
    // ...
    return nil
}
```

## MVCC (Multi-Version Concurrency Control)

PostgreSQL использует MVCC для минимизации блокировок.

### Как работает MVCC

**Каждая строка имеет скрытые колонки:**
- `xmin` - ID транзакции, создавшей строку
- `xmax` - ID транзакции, удалившей строку (NULL если активна)
- `cmin`, `cmax` - ID команды внутри транзакции

```sql
-- T1 (xid=1000)
INSERT INTO users (id, name) VALUES (1, 'Alice');
-- Физически: xmin=1000, xmax=NULL

-- T2 (xid=1001)
UPDATE users SET name = 'Bob' WHERE id = 1;
-- Старая версия: xmin=1000, xmax=1001
-- Новая версия: xmin=1001, xmax=NULL

-- T3 (xid=1002)
-- Если T2 committed → видит 'Bob'
-- Если T2 active → видит 'Alice'
SELECT name FROM users WHERE id = 1;
```

**Преимущества MVCC:**
- Читатели НЕ блокируют писателей
- Писатели НЕ блокируют читателей
- Каждая транзакция видит согласованный snapshot

**Недостатки:**
- Накопление старых версий строк (bloat)
- Требуется VACUUM для очистки

### Visibility Rules (правила видимости)

```go
// Псевдокод: когда строка видна транзакции?
func IsVisible(row Row, txn Transaction) bool {
    // Создана после нас?
    if row.xmin > txn.xid {
        return false
    }

    // Создана незавершенной транзакцией?
    if IsActive(row.xmin) && row.xmin != txn.xid {
        return false
    }

    // Создана откаченной транзакцией?
    if IsAborted(row.xmin) {
        return false
    }

    // Удалена до нас?
    if row.xmax < txn.xid && IsCommitted(row.xmax) {
        return false
    }

    return true
}
```

## Уровни изоляции

PostgreSQL поддерживает 3 из 4 стандартных уровней изоляции.

### 1. Read Committed (по умолчанию)

Каждый запрос видит snapshot на момент его начала.

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Или просто BEGIN; (по умолчанию)

-- Транзакция 1
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 100

-- Транзакция 2
BEGIN;
UPDATE accounts SET balance = 50 WHERE id = 1;
COMMIT;

-- Транзакция 1 (продолжение)
SELECT balance FROM accounts WHERE id = 1;  -- 50 (видит изменения!)
COMMIT;
```

**Проблемы:**
- **Non-Repeatable Read:** Значение может измениться между SELECT
- **Phantom Read:** Новые строки могут появиться

**Пример Non-Repeatable Read:**

```sql
-- T1
BEGIN;
SELECT * FROM users WHERE age > 18;  -- 100 строк

-- T2
BEGIN;
UPDATE users SET age = 20 WHERE age = 17;  -- Теперь 18+
COMMIT;

-- T1
SELECT * FROM users WHERE age > 18;  -- 105 строк (больше!)
COMMIT;
```

### 2. Repeatable Read

Транзакция видит snapshot на момент **первого запроса**.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Транзакция 1
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 100 (snapshot зафиксирован)

-- Транзакция 2
BEGIN;
UPDATE accounts SET balance = 50 WHERE id = 1;
COMMIT;

-- Транзакция 1 (продолжение)
SELECT balance FROM accounts WHERE id = 1;  -- 100 (старый snapshot!)
COMMIT;
```

**Нет Phantom Read благодаря MVCC:**

```sql
-- T1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM users WHERE age > 18;  -- 100 строк

-- T2
BEGIN;
INSERT INTO users (age) VALUES (20);
COMMIT;

-- T1
SELECT * FROM users WHERE age > 18;  -- 100 строк (старый snapshot!)
COMMIT;
```

**Проблема: Serialization Failure при UPDATE**

```sql
-- T1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- 100
UPDATE accounts SET balance = 90 WHERE id = 1;

-- T2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- 100 (snapshot)
UPDATE accounts SET balance = 80 WHERE id = 1;
-- ERROR: could not serialize access due to concurrent update

-- T1
COMMIT;  -- ✅ Успешно
```

**Retry логика в приложении:**

```go
func TransferMoney(db *sql.DB, from, to int, amount int) error {
    maxRetries := 3
    for i := 0; i < maxRetries; i++ {
        tx, _ := db.Begin()
        tx.Exec("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")

        err := doTransfer(tx, from, to, amount)
        if err == nil {
            tx.Commit()
            return nil
        }

        tx.Rollback()

        // Если serialization error - повторить
        if isSerializationError(err) {
            continue
        }

        return err
    }
    return errors.New("max retries exceeded")
}
```

### 3. Serializable

Самый строгий уровень. Гарантирует эквивалентность последовательному выполнению.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Обнаруживает аномалии сериализации:**

```sql
-- Аномалия: Write Skew
-- T1 и T2 читают одно, но изменяют разное, нарушая бизнес-правило

-- Правило: на дежурстве должно быть минимум 2 врача

-- T1
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*) FROM doctors WHERE on_duty = true;  -- 2
UPDATE doctors SET on_duty = false WHERE id = 1;  -- Уходит с дежурства

-- T2
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*) FROM doctors WHERE on_duty = true;  -- 2
UPDATE doctors SET on_duty = false WHERE id = 2;  -- Уходит с дежурства
COMMIT;

-- T1
COMMIT;
-- ERROR: could not serialize access due to read/write dependencies
-- PostgreSQL обнаружил, что результат нарушает сериализуемость
```

**Производительность:**
- Медленнее Repeatable Read
- Больше serialization errors
- Требует retry логики

**Когда использовать:**
- Финансовые транзакции
- Критичные бизнес-правила
- Когда нужна абсолютная согласованность

## Deadlock (взаимная блокировка)

Возникает, когда две транзакции ждут друг друга.

### Пример deadlock

```sql
-- Транзакция 1
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 1;  -- Locked id=1
-- Ждем...

-- Транзакция 2
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id = 2;  -- Locked id=2
UPDATE accounts SET balance = balance + 10 WHERE id = 1;  -- Ждет T1...

-- Транзакция 1 (продолжение)
UPDATE accounts SET balance = balance + 10 WHERE id = 2;  -- Ждет T2...

-- DEADLOCK! PostgreSQL обнаруживает и откатывает одну транзакцию
-- ERROR: deadlock detected
```

### Как PostgreSQL обнаруживает deadlock

1. **Deadlock timeout:** По умолчанию 1 секунда
2. **Deadlock detection:** Проверяет граф ожидания блокировок
3. **Выбирает жертву:** Откатывает транзакцию с наименьшей стоимостью

```sql
-- Настройка таймаута
SET deadlock_timeout = '1s';
```

### Как избежать deadlock

#### 1. Блокировать в одном порядке

```sql
-- ✅ Хорошо
BEGIN;
UPDATE accounts SET balance = balance - 10 WHERE id IN (1, 2) ORDER BY id;
COMMIT;

-- ✅ Хорошо (явная блокировка в порядке)
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Теперь безопасно изменять
COMMIT;
```

#### 2. Использовать LOCK TABLE

```sql
BEGIN;
LOCK TABLE accounts IN SHARE ROW EXCLUSIVE MODE;
-- Теперь только эта транзакция может изменять accounts
UPDATE accounts SET balance = balance - 10 WHERE id = 1;
UPDATE accounts SET balance = balance + 10 WHERE id = 2;
COMMIT;
```

#### 3. Минимизировать время блокировки

```sql
-- ❌ Плохо (долгая транзакция)
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- API call (5 секунд)
-- Вычисления (10 секунд)
UPDATE accounts SET balance = result WHERE id = 1;
COMMIT;

-- ✅ Хорошо
-- Вычисления ВНЕ транзакции
result := compute()
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
UPDATE accounts SET balance = result WHERE id = 1;
COMMIT;
```

#### 4. Retry логика

```go
func WithRetry(fn func() error, maxRetries int) error {
    for i := 0; i < maxRetries; i++ {
        err := fn()
        if err == nil {
            return nil
        }

        if isDeadlock(err) || isSerializationError(err) {
            time.Sleep(time.Duration(i*100) * time.Millisecond)
            continue
        }

        return err
    }
    return errors.New("max retries exceeded")
}
```

## Lock Monitoring (мониторинг блокировок)

### Текущие блокировки

```sql
SELECT
    locktype,
    relation::regclass,
    mode,
    granted,
    pid
FROM pg_locks
WHERE NOT granted;  -- Только ожидающие блокировки
```

### Кто кого блокирует

```sql
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS current_blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity
    ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity
    ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### Долгие транзакции

```sql
SELECT
    pid,
    now() - xact_start AS duration,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start IS NOT NULL
ORDER BY xact_start
LIMIT 10;
```

### Убить блокирующую транзакцию

```sql
-- Мягкое завершение
SELECT pg_cancel_backend(12345);  -- pid

-- Жесткое завершение (если cancel не помог)
SELECT pg_terminate_backend(12345);
```

## Оптимизация блокировок

### 1. Используйте правильный уровень блокировки

```sql
-- ❌ Плохо (блокирует всю таблицу)
BEGIN;
LOCK TABLE accounts IN EXCLUSIVE MODE;
UPDATE accounts SET balance = 100 WHERE id = 1;
COMMIT;

-- ✅ Хорошо (блокирует только строку)
BEGIN;
UPDATE accounts SET balance = 100 WHERE id = 1;
COMMIT;
```

### 2. SELECT FOR UPDATE NOWAIT

```sql
-- Не ждать блокировки, вернуть ошибку сразу
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row

-- Пропустить заблокированные строки
SELECT * FROM accounts WHERE id IN (1,2,3) FOR UPDATE SKIP LOCKED;
-- Вернет только незаблокированные строки
COMMIT;
```

**Когда использовать SKIP LOCKED:**
- Очереди задач (каждый worker берет свободную задачу)
- Batch обработка

```go
// Обработка задач несколькими воркерами
func ProcessTasks(db *sql.DB) {
    for {
        tx, _ := db.Begin()

        // Взять одну свободную задачу
        var taskID int
        err := tx.QueryRow(`
            SELECT id FROM tasks
            WHERE status = 'pending'
            ORDER BY created_at
            FOR UPDATE SKIP LOCKED
            LIMIT 1
        `).Scan(&taskID)

        if err == sql.ErrNoRows {
            tx.Rollback()
            time.Sleep(1 * time.Second)
            continue
        }

        // Обработать задачу
        process(tx, taskID)
        tx.Commit()
    }
}
```

### 3. Minimize lock duration

```sql
-- ❌ Плохо
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Тяжелые вычисления (10 сек)
UPDATE accounts SET balance = result WHERE id = 1;
COMMIT;

-- ✅ Хорошо
result := computeOutsideTransaction()
BEGIN;
UPDATE accounts SET balance = result WHERE id = 1;
COMMIT;
```

## Best Practices

1. ✅ Используйте `FOR UPDATE` когда планируете изменить данные
2. ✅ Блокируйте в одном порядке (избежание deadlock)
3. ✅ Минимизируйте время удержания блокировок
4. ✅ Используйте `NOWAIT` или `SKIP LOCKED` где уместно
5. ✅ Выбирайте правильный уровень изоляции
6. ✅ Реализуйте retry логику для serialization errors
7. ❌ Не делайте внешние вызовы (API, sleep) внутри транзакции
8. ❌ Не держите долгие транзакции открытыми
9. ✅ Мониторьте долгие блокировки и deadlocks
10. ✅ Используйте connection pooling с ограничением времени транзакции

## Связанные темы

- [[PostgreSQL - ACID]]
- [[PostgreSQL - Основы]]
- [[PostgreSQL - Типы индексов]]
- [[PostgreSQL - Оптимизация запросов]]
- [[Go - Пакет sync]]
- [[Go - Mutex и RWMutex]]
