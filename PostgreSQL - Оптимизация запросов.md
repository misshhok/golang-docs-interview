# PostgreSQL - Оптимизация запросов

Практическое руководство по анализу и оптимизации производительности SQL запросов в PostgreSQL.

## EXPLAIN - анализ плана выполнения

### EXPLAIN (основной)

```sql
EXPLAIN
SELECT * FROM users WHERE email = 'user@example.com';
```

**Вывод:**
```
Seq Scan on users  (cost=0.00..18.50 rows=1 width=100)
  Filter: (email = 'user@example.com'::text)
```

**Читаем:**
- `Seq Scan` - последовательное сканирование (плохо для больших таблиц)
- `cost=0.00..18.50` - стоимость (startup..total)
- `rows=1` - оценка количества строк
- `width=100` - средний размер строки в байтах

### EXPLAIN ANALYZE (с выполнением)

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';
```

**Вывод:**
```
Seq Scan on users  (cost=0.00..18.50 rows=1 width=100)
                   (actual time=0.023..0.156 rows=1 loops=1)
  Filter: (email = 'user@example.com'::text)
  Rows Removed by Filter: 999
Planning Time: 0.098 ms
Execution Time: 0.187 ms
```

**Новое:**
- `actual time=0.023..0.156` - реальное время (startup..total) в мс
- `rows=1` - реальное количество строк
- `loops=1` - сколько раз выполнялся узел
- `Rows Removed by Filter` - отфильтровано строк

### EXPLAIN (ANALYZE, BUFFERS)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'user@example.com';
```

**Вывод:**
```
Seq Scan on users  (cost=0.00..18.50 rows=1 width=100)
                   (actual time=0.023..0.156 rows=1 loops=1)
  Filter: (email = 'user@example.com'::text)
  Rows Removed by Filter: 999
  Buffers: shared hit=8
Planning Time: 0.098 ms
Execution Time: 0.187 ms
```

**Buffers:**
- `shared hit=8` - прочитано из shared cache (хорошо!)
- `shared read=0` - прочитано с диска (медленно)
- `temp read/written` - временные файлы (очень медленно)

## Типы сканирования

### 1. Seq Scan (последовательное сканирование)

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE age > 18;
```

```
Seq Scan on users  (cost=0.00..180.00 rows=5000 width=100)
```

**Когда происходит:**
- Нет индекса на колонке
- Выборка большого % таблицы (> 5-10%)
- Таблица маленькая (< 1000 строк)

**Скорость:** O(n) - проход по всей таблице

### 2. Index Scan (индексное сканирование)

```sql
CREATE INDEX idx_users_email ON users(email);

EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';
```

```
Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=100)
  Index Cond: (email = 'user@example.com'::text)
```

**Скорость:** O(log n) для поиска + O(1) для каждой строки

### 3. Index Only Scan (только индекс)

```sql
CREATE INDEX idx_users_email_name ON users(email) INCLUDE (name);

EXPLAIN ANALYZE
SELECT name FROM users WHERE email = 'user@example.com';
```

```
Index Only Scan using idx_users_email_name on users  (cost=0.42..4.44 rows=1 width=32)
  Index Cond: (email = 'user@example.com'::text)
  Heap Fetches: 0
```

**Оптимально:** Не обращается к таблице, только к индексу.

### 4. Bitmap Index Scan

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE age > 18 AND city = 'Moscow';
```

```
Bitmap Heap Scan on users  (cost=12.34..156.78 rows=50 width=100)
  Recheck Cond: ((age > 18) AND (city = 'Moscow'::text))
  ->  BitmapAnd  (cost=12.34..12.34 rows=50 width=0)
        ->  Bitmap Index Scan on idx_users_age
        ->  Bitmap Index Scan on idx_users_city
```

**Когда:** Используется несколько индексов одновременно.

## Типы JOIN

### 1. Nested Loop

```sql
EXPLAIN ANALYZE
SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.id = 1;
```

```
Nested Loop  (cost=0.42..16.46 rows=2 width=200)
  ->  Index Scan using users_pkey on users u
  ->  Index Scan using idx_orders_user_id on orders o
        Index Cond: (user_id = u.id)
```

**Когда:** Маленькие таблицы, есть индексы.
**Сложность:** O(n × m) worst case, O(n log m) с индексом

### 2. Hash Join

```sql
EXPLAIN ANALYZE
SELECT * FROM users u
JOIN orders o ON u.id = o.user_id;
```

```
Hash Join  (cost=45.00..320.00 rows=5000 width=200)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o
  ->  Hash  (cost=30.00..30.00 rows=1000 width=100)
        ->  Seq Scan on users u
```

**Когда:** Средние/большие таблицы, есть память.
**Сложность:** O(n + m)

### 3. Merge Join

```sql
EXPLAIN ANALYZE
SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
ORDER BY u.id;
```

```
Merge Join  (cost=0.84..350.00 rows=5000 width=200)
  Merge Cond: (u.id = o.user_id)
  ->  Index Scan using users_pkey on users u
  ->  Index Scan using idx_orders_user_id on orders o
```

**Когда:** Обе таблицы отсортированы (есть индексы).
**Сложность:** O(n + m)

## Основные проблемы и решения

### Проблема 1: Seq Scan вместо Index Scan

**Проблема:**
```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';
-- Seq Scan (медленно!)
```

**Решение 1: Создать индекс**
```sql
CREATE INDEX idx_users_email ON users(email);
```

**Решение 2: ANALYZE для обновления статистики**
```sql
ANALYZE users;
```

**Решение 3: Увеличить random_page_cost** (если индекс игнорируется)
```sql
SET random_page_cost = 1.1;  -- По умолчанию 4.0
```

### Проблема 2: Функция на колонке убивает индекс

**Проблема:**
```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';
-- Seq Scan (индекс не используется!)
```

**Решение: Expression Index**
```sql
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

### Проблема 3: OR условие

**Проблема:**
```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'a@b.com' OR phone = '123';
-- Seq Scan (индексы не используются!)
```

**Решение 1: UNION**
```sql
SELECT * FROM users WHERE email = 'a@b.com'
UNION
SELECT * FROM users WHERE phone = '123';
-- Использует оба индекса
```

**Решение 2: GIN индекс** (для частых OR)
```sql
CREATE INDEX idx_users_gin ON users USING gin(email, phone);
```

### Проблема 4: LIKE с % в начале

**Проблема:**
```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email LIKE '%@gmail.com';
-- Seq Scan (индекс не помогает!)
```

**Решение 1: pg_trgm для подстроки**
```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_email_trgm ON users USING gin(email gin_trgm_ops);
```

**Решение 2: Полнотекстовый поиск**
```sql
CREATE INDEX idx_users_email_fts ON users USING gin(to_tsvector('simple', email));

SELECT * FROM users WHERE to_tsvector('simple', email) @@ to_tsquery('gmail');
```

### Проблема 5: Неявное приведение типов

**Проблема:**
```sql
-- phone_number - varchar
EXPLAIN ANALYZE
SELECT * FROM users WHERE phone_number = 123456789;
-- Seq Scan (приведение типов!)
```

**Решение: Явное приведение**
```sql
SELECT * FROM users WHERE phone_number = '123456789';
-- Index Scan
```

### Проблема 6: COUNT(*) медленный

**Проблема:**
```sql
SELECT COUNT(*) FROM huge_table;
-- Долго (Seq Scan)
```

**Решение 1: Приблизительный COUNT**
```sql
SELECT reltuples::bigint AS estimate
FROM pg_class
WHERE relname = 'huge_table';
-- Мгновенно, но приблизительно
```

**Решение 2: Материализованное представление**
```sql
CREATE MATERIALIZED VIEW table_count AS
SELECT COUNT(*) FROM huge_table;

-- Быстро
SELECT * FROM table_count;

-- Обновление периодически
REFRESH MATERIALIZED VIEW table_count;
```

### Проблема 7: N+1 запросов

**Проблема:**
```go
// ❌ N+1 запросов
users := getUsers()  // 1 запрос
for _, user := range users {
    orders := getOrdersByUserID(user.ID)  // N запросов
}
```

**Решение: JOIN или IN**
```go
// ✅ 2 запроса
users := getUsers()
userIDs := extractIDs(users)
orders := getOrdersByUserIDs(userIDs)  // WHERE user_id IN (...)
```

### Проблема 8: Неоптимальный ORDER BY + LIMIT

**Проблема:**
```sql
EXPLAIN ANALYZE
SELECT * FROM users ORDER BY created_at DESC LIMIT 10;
-- Sort (медленно если таблица большая)
```

**Решение: Индекс на ORDER BY колонке**
```sql
CREATE INDEX idx_users_created_at ON users(created_at DESC);
-- Теперь Index Scan вместо Sort
```

## Оптимизация JOIN

### 1. Индексы на JOIN колонках

```sql
-- ❌ Медленно
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id;

-- ✅ Быстро
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id ON users(id);  -- Обычно PK уже индексирован
```

### 2. Фильтры в ON vs WHERE

```sql
-- ❌ Медленно (LEFT JOIN теряет смысл)
SELECT * FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.status = 'paid';

-- ✅ Быстро
SELECT * FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.status = 'paid';
```

### 3. Порядок JOIN имеет значение

```sql
-- ❌ Медленнее
SELECT * FROM huge_table h
JOIN small_table s ON h.id = s.id;

-- ✅ Быстрее (маленькая таблица первая)
SELECT * FROM small_table s
JOIN huge_table h ON s.id = h.id;
```

PostgreSQL оптимизатор обычно это делает сам, но явный порядок помогает.

## Параметры конфигурации

### work_mem

Память для сортировки и hash таблиц.

```sql
-- По умолчанию: 4MB (мало!)
SHOW work_mem;

-- Увеличить для сложных запросов
SET work_mem = '256MB';

-- Применить глобально (postgresql.conf)
work_mem = 64MB
```

**Признаки нехватки:** "Sort Method: external merge" в EXPLAIN.

### shared_buffers

Кэш данных в памяти.

```sql
-- postgresql.conf
shared_buffers = 8GB  -- 25% RAM
```

### effective_cache_size

Подсказка оптимизатору о доступной памяти.

```sql
-- postgresql.conf
effective_cache_size = 24GB  -- 50-75% RAM
```

### random_page_cost

Стоимость случайного чтения с диска.

```sql
-- SSD
random_page_cost = 1.1

-- HDD
random_page_cost = 4.0  -- По умолчанию
```

## Статистика и ANALYZE

### ANALYZE - обновление статистики

```sql
-- Одна таблица
ANALYZE users;

-- Вся база
ANALYZE;

-- Автоматически (autovacuum)
-- postgresql.conf
autovacuum = on
autovacuum_analyze_scale_factor = 0.1  -- Анализ после 10% изменений
```

### Статистика колонок

```sql
-- Увеличить детализацию статистики для колонки
ALTER TABLE users ALTER COLUMN email SET STATISTICS 1000;
ANALYZE users;
```

## Monitoring и профилирование

### pg_stat_statements

```sql
-- Установка
CREATE EXTENSION pg_stat_statements;

-- Топ-10 медленных запросов
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Сброс статистики
SELECT pg_stat_statements_reset();
```

### Логирование медленных запросов

```sql
-- postgresql.conf
log_min_duration_statement = 1000  -- Логировать > 1 сек

-- Временно
SET log_min_duration_statement = 100;
```

### auto_explain

```sql
-- postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 1000  -- EXPLAIN для > 1 сек
auto_explain.log_analyze = true
auto_explain.log_buffers = true
```

## Денормализация

Иногда денормализация ускоряет запросы.

### Пример: счетчики

**❌ Медленно:**
```sql
SELECT users.*, COUNT(orders.id) AS order_count
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.id;
```

**✅ Быстро (денормализация):**
```sql
-- Добавить колонку
ALTER TABLE users ADD COLUMN order_count int DEFAULT 0;

-- Обновлять через триггер
CREATE OR REPLACE FUNCTION update_order_count()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE users SET order_count = order_count + 1 WHERE id = NEW.user_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE users SET order_count = order_count - 1 WHERE id = OLD.user_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_order_count
AFTER INSERT OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION update_order_count();

-- Теперь O(1)
SELECT * FROM users WHERE order_count > 10;
```

## Partitioning (секционирование)

Для огромных таблиц (> 100GB).

```sql
-- Партиционирование по дате
CREATE TABLE logs (
    id bigserial,
    created_at timestamp,
    message text
) PARTITION BY RANGE (created_at);

CREATE TABLE logs_2024_01 PARTITION OF logs
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE logs_2024_02 PARTITION OF logs
FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Запросы автоматически используют нужную партицию
SELECT * FROM logs WHERE created_at >= '2024-01-15';
-- Сканирует только logs_2024_01
```

## Vacuum и bloat

### VACUUM

```sql
-- Очистка мертвых строк
VACUUM users;

-- Полная очистка (блокирует таблицу!)
VACUUM FULL users;

-- Автоматически (autovacuum)
-- postgresql.conf
autovacuum = on
```

### Проверка bloat

```sql
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    n_live_tup,
    n_dead_tup,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

## Best Practices

1. ✅ Всегда используйте EXPLAIN ANALYZE для медленных запросов
2. ✅ Создавайте индексы на WHERE, JOIN, ORDER BY колонках
3. ✅ Регулярно запускайте ANALYZE для обновления статистики
4. ✅ Мониторьте pg_stat_statements для поиска узких мест
5. ✅ Используйте connection pooling (PgBouncer)
6. ✅ VACUUM регулярно (или autovacuum)
7. ❌ Не создавайте слишком много индексов (замедляет INSERT/UPDATE)
8. ✅ Материализованные представления для тяжелых агрегаций
9. ✅ Партиционирование для огромных таблиц
10. ✅ Денормализация для критичных запросов

## Чеклист оптимизации

### Быстрые победы

- [ ] EXPLAIN ANALYZE медленного запроса
- [ ] Создать индексы на WHERE/JOIN колонках
- [ ] ANALYZE таблицы
- [ ] Проверить приведение типов
- [ ] Убрать функции на индексированных колонках

### Средние

- [ ] Оптимизировать JOIN (порядок, индексы)
- [ ] INDEX ONLY SCAN через INCLUDE
- [ ] Partial indexes для частых фильтров
- [ ] Увеличить work_mem для сложных запросов
- [ ] N+1 запросы → JOIN или IN

### Продвинутые

- [ ] Партиционирование больших таблиц
- [ ] Материализованные представления
- [ ] Денормализация счетчиков
- [ ] pg_stat_statements для профилирования
- [ ] Connection pooling
- [ ] Read replicas для read-heavy нагрузки

## Связанные темы

- [[PostgreSQL - Типы индексов]]
- [[PostgreSQL - ACID]]
- [[PostgreSQL - Блокировки и изоляция транзакций]]
- [[SQL - JOIN операции]]
- [[SQL - Агрегация и группировка]]
- [[SQL - Оконные функции]]
- [[Алгоритмическая сложность (Big O)]]
