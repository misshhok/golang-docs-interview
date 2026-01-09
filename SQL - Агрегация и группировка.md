# SQL - Агрегация и группировка

Агрегатные функции обрабатывают наборы строк и возвращают одно значение. GROUP BY группирует строки для агрегации.

## Агрегатные функции

### COUNT - подсчет строк

```sql
-- Количество всех строк (включая NULL)
SELECT COUNT(*) FROM orders;

-- Количество NOT NULL значений
SELECT COUNT(user_id) FROM orders;

-- Количество уникальных значений
SELECT COUNT(DISTINCT user_id) FROM orders;
```

**Примеры:**

```sql
CREATE TABLE orders (
    id int,
    user_id int,
    amount decimal(10,2),
    status varchar(20)
);

INSERT INTO orders VALUES
    (1, 101, 50.00, 'paid'),
    (2, 101, 75.00, 'paid'),
    (3, 102, 100.00, 'pending'),
    (4, NULL, 25.00, 'cancelled'),
    (5, 102, 150.00, 'paid');

SELECT COUNT(*) FROM orders;              -- 5
SELECT COUNT(user_id) FROM orders;        -- 4 (NULL не считается)
SELECT COUNT(DISTINCT user_id) FROM orders;  -- 2 (101, 102)
```

### SUM - сумма

```sql
SELECT SUM(amount) FROM orders;  -- 400.00

-- С условием
SELECT SUM(amount) FROM orders WHERE status = 'paid';  -- 275.00

-- Игнорирует NULL
SELECT SUM(amount) FROM orders WHERE user_id IS NULL;  -- 25.00
```

### AVG - среднее

```sql
SELECT AVG(amount) FROM orders;  -- 80.00

-- Округление
SELECT ROUND(AVG(amount), 2) FROM orders;  -- 80.00

-- AVG игнорирует NULL
SELECT AVG(amount) FROM orders;  -- 80.00, а не 66.67 (не делит на 5)
```

### MIN / MAX - минимум / максимум

```sql
SELECT MIN(amount) FROM orders;  -- 25.00
SELECT MAX(amount) FROM orders;  -- 150.00

-- Работает с датами
SELECT MIN(created_at), MAX(created_at) FROM orders;

-- Работает со строками (лексикографический порядок)
SELECT MIN(status), MAX(status) FROM orders;  -- 'cancelled', 'pending'
```

### STRING_AGG - конкатенация строк

```sql
-- Объединить все статусы через запятую
SELECT STRING_AGG(status, ', ') FROM orders;
-- 'paid, paid, pending, cancelled, paid'

-- С ORDER BY
SELECT STRING_AGG(status, ', ' ORDER BY status) FROM orders;
-- 'cancelled, paid, paid, paid, pending'

-- DISTINCT
SELECT STRING_AGG(DISTINCT status, ', ' ORDER BY status) FROM orders;
-- 'cancelled, paid, pending'
```

### ARRAY_AGG - массив значений

```sql
-- Собрать все amounts в массив
SELECT ARRAY_AGG(amount) FROM orders;
-- {50.00,75.00,100.00,25.00,150.00}

-- С ORDER BY
SELECT ARRAY_AGG(amount ORDER BY amount DESC) FROM orders;
-- {150.00,100.00,75.00,50.00,25.00}
```

## GROUP BY - группировка

GROUP BY разбивает результат на группы и применяет агрегатные функции к каждой группе.

### Базовый пример

```sql
-- Сумма по каждому пользователю
SELECT
    user_id,
    SUM(amount) AS total
FROM orders
WHERE user_id IS NOT NULL
GROUP BY user_id;
```

**Результат:**
```
+---------+--------+
| user_id | total  |
+---------+--------+
| 101     | 125.00 |
| 102     | 250.00 |
+---------+--------+
```

### Множественные агрегации

```sql
SELECT
    user_id,
    COUNT(*) AS order_count,
    SUM(amount) AS total_spent,
    AVG(amount) AS avg_order,
    MIN(amount) AS min_order,
    MAX(amount) AS max_order
FROM orders
WHERE user_id IS NOT NULL
GROUP BY user_id;
```

**Результат:**
```
+---------+-------------+-------------+-----------+-----------+-----------+
| user_id | order_count | total_spent | avg_order | min_order | max_order |
+---------+-------------+-------------+-----------+-----------+-----------+
| 101     | 2           | 125.00      | 62.50     | 50.00     | 75.00     |
| 102     | 2           | 250.00      | 125.00    | 100.00    | 150.00    |
+---------+-------------+-------------+-----------+-----------+-----------+
```

### GROUP BY с несколькими колонками

```sql
-- Статистика по пользователям и статусам
SELECT
    user_id,
    status,
    COUNT(*) AS count,
    SUM(amount) AS total
FROM orders
WHERE user_id IS NOT NULL
GROUP BY user_id, status
ORDER BY user_id, status;
```

**Результат:**
```
+---------+---------+-------+--------+
| user_id | status  | count | total  |
+---------+---------+-------+--------+
| 101     | paid    | 2     | 125.00 |
| 102     | paid    | 1     | 150.00 |
| 102     | pending | 1     | 100.00 |
+---------+---------+-------+--------+
```

### Правило GROUP BY

**Все колонки в SELECT (кроме агрегатных) должны быть в GROUP BY!**

```sql
-- ❌ Ошибка
SELECT user_id, status, SUM(amount)
FROM orders
GROUP BY user_id;
-- ERROR: column "status" must appear in GROUP BY clause

-- ✅ Правильно
SELECT user_id, status, SUM(amount)
FROM orders
GROUP BY user_id, status;
```

## HAVING - фильтрация групп

WHERE фильтрует строки **до** группировки.
HAVING фильтрует группы **после** агрегации.

```sql
-- Пользователи, потратившие больше 200
SELECT
    user_id,
    SUM(amount) AS total
FROM orders
WHERE user_id IS NOT NULL
GROUP BY user_id
HAVING SUM(amount) > 200;
```

**Результат:**
```
+---------+--------+
| user_id | total  |
+---------+--------+
| 102     | 250.00 |
+---------+--------+
```

### WHERE vs HAVING

```sql
-- WHERE: фильтр ДО группировки
SELECT user_id, COUNT(*)
FROM orders
WHERE status = 'paid'  -- Сначала фильтруем строки
GROUP BY user_id;

-- HAVING: фильтр ПОСЛЕ группировки
SELECT user_id, COUNT(*)
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 1;  -- Затем фильтруем группы

-- Можно использовать оба
SELECT user_id, SUM(amount)
FROM orders
WHERE status = 'paid'       -- Только оплаченные заказы
GROUP BY user_id
HAVING SUM(amount) > 100;   -- Потратили больше 100
```

**Порядок выполнения:**
```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

## DISTINCT vs GROUP BY

```sql
-- Уникальные user_id
SELECT DISTINCT user_id FROM orders WHERE user_id IS NOT NULL;

-- Эквивалентно
SELECT user_id FROM orders WHERE user_id IS NOT NULL GROUP BY user_id;
```

**DISTINCT** - для уникальных значений без агрегации.
**GROUP BY** - когда нужна агрегация.

## Группировка с JOIN

```sql
CREATE TABLE users (
    id int PRIMARY KEY,
    name varchar(50)
);

INSERT INTO users VALUES (101, 'Alice'), (102, 'Bob'), (103, 'Charlie');

-- Заказы по пользователям (включая тех, у кого нет заказов)
SELECT
    users.name,
    COUNT(orders.id) AS order_count,
    COALESCE(SUM(orders.amount), 0) AS total_spent
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.id, users.name
ORDER BY total_spent DESC;
```

**Результат:**
```
+---------+-------------+-------------+
| name    | order_count | total_spent |
+---------+-------------+-------------+
| Bob     | 2           | 250.00      |
| Alice   | 2           | 125.00      |
| Charlie | 0           | 0.00        |
+---------+-------------+-------------+
```

**ВАЖНО:** `COUNT(orders.id)` vs `COUNT(*)`
- `COUNT(orders.id)` - не считает NULL (правильно для LEFT JOIN)
- `COUNT(*)` - считает все строки (неправильно для LEFT JOIN)

## FILTER - условная агрегация

PostgreSQL поддерживает FILTER для условной агрегации.

```sql
SELECT
    user_id,
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE status = 'paid') AS paid_orders,
    COUNT(*) FILTER (WHERE status = 'pending') AS pending_orders,
    SUM(amount) FILTER (WHERE status = 'paid') AS paid_amount
FROM orders
WHERE user_id IS NOT NULL
GROUP BY user_id;
```

**Результат:**
```
+---------+--------------+-------------+----------------+-------------+
| user_id | total_orders | paid_orders | pending_orders | paid_amount |
+---------+--------------+-------------+----------------+-------------+
| 101     | 2            | 2           | 0              | 125.00      |
| 102     | 2            | 1           | 1              | 150.00      |
+---------+--------------+-------------+----------------+-------------+
```

### Альтернатива FILTER - CASE

```sql
SELECT
    user_id,
    COUNT(*) AS total_orders,
    COUNT(CASE WHEN status = 'paid' THEN 1 END) AS paid_orders,
    SUM(CASE WHEN status = 'paid' THEN amount ELSE 0 END) AS paid_amount
FROM orders
WHERE user_id IS NOT NULL
GROUP BY user_id;
```

## GROUPING SETS - множественная группировка

```sql
-- Группировка по (user_id, status) + (user_id) + общий итог
SELECT
    user_id,
    status,
    SUM(amount) AS total
FROM orders
WHERE user_id IS NOT NULL
GROUP BY GROUPING SETS (
    (user_id, status),  -- По пользователю и статусу
    (user_id),          -- Только по пользователю
    ()                  -- Общий итог
)
ORDER BY user_id NULLS LAST, status NULLS LAST;
```

**Результат:**
```
+---------+---------+--------+
| user_id | status  | total  |
+---------+---------+--------+
| 101     | paid    | 125.00 | -- user_id + status
| 101     | NULL    | 125.00 | -- только user_id
| 102     | paid    | 150.00 |
| 102     | pending | 100.00 |
| 102     | NULL    | 250.00 |
| NULL    | NULL    | 375.00 | -- общий итог
+---------+---------+--------+
```

### ROLLUP - иерархическая группировка

```sql
-- Группировка от детального к общему
SELECT
    user_id,
    status,
    SUM(amount) AS total
FROM orders
WHERE user_id IS NOT NULL
GROUP BY ROLLUP (user_id, status)
ORDER BY user_id NULLS LAST, status NULLS LAST;
```

**Эквивалентно:**
```sql
GROUPING SETS (
    (user_id, status),
    (user_id),
    ()
)
```

### CUBE - все комбинации

```sql
-- Все возможные комбинации группировок
SELECT
    user_id,
    status,
    SUM(amount) AS total
FROM orders
WHERE user_id IS NOT NULL
GROUP BY CUBE (user_id, status);
```

**Эквивалентно:**
```sql
GROUPING SETS (
    (user_id, status),
    (user_id),
    (status),
    ()
)
```

## Сложные примеры

### Top N по группам

```sql
-- Топ-2 заказа по каждому пользователю
SELECT *
FROM (
    SELECT
        user_id,
        amount,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rank
    FROM orders
    WHERE user_id IS NOT NULL
) ranked
WHERE rank <= 2;
```

**Результат:**
```
+---------+--------+------+
| user_id | amount | rank |
+---------+--------+------+
| 101     | 75.00  | 1    |
| 101     | 50.00  | 2    |
| 102     | 150.00 | 1    |
| 102     | 100.00 | 2    |
+---------+--------+------+
```

### Процент от общего

```sql
SELECT
    user_id,
    SUM(amount) AS user_total,
    ROUND(
        100.0 * SUM(amount) / (SELECT SUM(amount) FROM orders WHERE user_id IS NOT NULL),
        2
    ) AS percentage
FROM orders
WHERE user_id IS NOT NULL
GROUP BY user_id;
```

**Результат:**
```
+---------+------------+------------+
| user_id | user_total | percentage |
+---------+------------+------------+
| 101     | 125.00     | 33.33      |
| 102     | 250.00     | 66.67      |
+---------+------------+------------+
```

### Pivot таблицы (crosstab)

```sql
-- Пользователи по статусам (pivot)
SELECT
    user_id,
    SUM(CASE WHEN status = 'paid' THEN amount ELSE 0 END) AS paid,
    SUM(CASE WHEN status = 'pending' THEN amount ELSE 0 END) AS pending,
    SUM(CASE WHEN status = 'cancelled' THEN amount ELSE 0 END) AS cancelled
FROM orders
WHERE user_id IS NOT NULL
GROUP BY user_id;
```

**Результат:**
```
+---------+--------+---------+-----------+
| user_id | paid   | pending | cancelled |
+---------+--------+---------+-----------+
| 101     | 125.00 | 0.00    | 0.00      |
| 102     | 150.00 | 100.00  | 0.00      |
+---------+--------+---------+-----------+
```

### Moving aggregates (скользящая агрегация)

```sql
-- Нарастающий итог (cumulative sum)
SELECT
    id,
    amount,
    SUM(amount) OVER (ORDER BY id) AS cumulative_total
FROM orders
WHERE user_id IS NOT NULL
ORDER BY id;
```

**Результат:**
```
+----+--------+------------------+
| id | amount | cumulative_total |
+----+--------+------------------+
| 1  | 50.00  | 50.00            |
| 2  | 75.00  | 125.00           |
| 3  | 100.00 | 225.00           |
| 5  | 150.00 | 375.00           |
+----+--------+------------------+
```

## Производительность

### Индексы для GROUP BY

```sql
-- Индекс ускоряет GROUP BY и ORDER BY
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

SELECT user_id, status, SUM(amount)
FROM orders
GROUP BY user_id, status;
-- Может использовать Index Scan вместо Sort
```

### Partial aggregates

```sql
-- Материализованное представление для медленных агрегаций
CREATE MATERIALIZED VIEW user_stats AS
SELECT
    user_id,
    COUNT(*) AS order_count,
    SUM(amount) AS total_spent,
    AVG(amount) AS avg_order
FROM orders
GROUP BY user_id;

CREATE INDEX ON user_stats(user_id);

-- Быстрый доступ
SELECT * FROM user_stats WHERE user_id = 101;

-- Обновление (периодически)
REFRESH MATERIALIZED VIEW user_stats;
```

### EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, SUM(amount)
FROM orders
GROUP BY user_id;
```

**Смотрим на:**
- `HashAggregate` - для небольших групп
- `GroupAggregate` - для отсортированных данных
- `Sort` - если есть, возможно нужен индекс

## Распространённые ошибки

### Ошибка 1: Забыли GROUP BY

```sql
-- ❌ Ошибка
SELECT user_id, SUM(amount) FROM orders;
-- ERROR: column "user_id" must appear in GROUP BY

-- ✅ Правильно
SELECT user_id, SUM(amount) FROM orders GROUP BY user_id;
```

### Ошибка 2: Агрегация в WHERE

```sql
-- ❌ Ошибка (агрегатные функции не в WHERE!)
SELECT user_id, SUM(amount)
FROM orders
WHERE SUM(amount) > 100
GROUP BY user_id;
-- ERROR: aggregate functions are not allowed in WHERE

-- ✅ Правильно (HAVING после GROUP BY)
SELECT user_id, SUM(amount)
FROM orders
GROUP BY user_id
HAVING SUM(amount) > 100;
```

### Ошибка 3: COUNT(*) vs COUNT(column) с LEFT JOIN

```sql
-- ❌ Неправильно (считает NULL строки)
SELECT users.name, COUNT(*)
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.name;
-- Charlie будет иметь count=1 (неправильно!)

-- ✅ Правильно
SELECT users.name, COUNT(orders.id)
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.name;
-- Charlie имеет count=0 (правильно)
```

### Ошибка 4: NULL в агрегации

```sql
-- SUM игнорирует NULL, но может вернуть NULL если все NULL
SELECT SUM(amount) FROM orders WHERE user_id IS NULL;
-- NULL (если нет строк) или 25.00

-- Использовать COALESCE
SELECT COALESCE(SUM(amount), 0) FROM orders WHERE user_id IS NULL;
-- 0 или 25.00
```

## Best Practices

1. ✅ Все неагрегатные колонки в SELECT должны быть в GROUP BY
2. ✅ Используйте HAVING для фильтрации групп, WHERE для строк
3. ✅ COUNT(column) для LEFT JOIN (не COUNT(*))
4. ✅ COALESCE для замены NULL на дефолтные значения
5. ✅ Индексы на колонках GROUP BY для производительности
6. ✅ FILTER или CASE для условной агрегации
7. ✅ Материализованные представления для тяжелых агрегаций
8. ❌ Не используйте агрегатные функции в WHERE
9. ✅ EXPLAIN для проверки производительности
10. ✅ Осторожно с NULL - может неожиданно влиять на результат

## Полезные запросы

### Статистика по дням

```sql
SELECT
    DATE(created_at) AS date,
    COUNT(*) AS orders,
    SUM(amount) AS revenue,
    AVG(amount) AS avg_order
FROM orders
GROUP BY DATE(created_at)
ORDER BY date DESC;
```

### Топ-10 пользователей

```sql
SELECT
    user_id,
    COUNT(*) AS order_count,
    SUM(amount) AS total_spent
FROM orders
WHERE user_id IS NOT NULL
GROUP BY user_id
ORDER BY total_spent DESC
LIMIT 10;
```

### Распределение по диапазонам

```sql
SELECT
    CASE
        WHEN amount < 50 THEN '0-50'
        WHEN amount < 100 THEN '50-100'
        WHEN amount < 200 THEN '100-200'
        ELSE '200+'
    END AS amount_range,
    COUNT(*) AS count
FROM orders
GROUP BY amount_range
ORDER BY amount_range;
```

## Связанные темы

- [[PostgreSQL - Основы]]
- [[SQL - JOIN операции]]
- [[SQL - Оконные функции]]
- [[PostgreSQL - Оптимизация запросов]]
- [[PostgreSQL - Типы индексов]]
