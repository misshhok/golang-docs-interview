# SQL - JOIN операции

JOIN объединяет данные из нескольких таблиц на основе связи между ними.

## Типы JOIN

PostgreSQL поддерживает 5 основных типов JOIN:
- INNER JOIN
- LEFT (OUTER) JOIN
- RIGHT (OUTER) JOIN
- FULL (OUTER) JOIN
- CROSS JOIN

## Тестовые данные

```sql
CREATE TABLE users (
    id int PRIMARY KEY,
    name varchar(50)
);

CREATE TABLE orders (
    id int PRIMARY KEY,
    user_id int,
    amount decimal(10,2)
);

INSERT INTO users VALUES
    (1, 'Alice'),
    (2, 'Bob'),
    (3, 'Charlie');

INSERT INTO orders VALUES
    (101, 1, 100.00),
    (102, 1, 200.00),
    (103, 2, 150.00),
    (104, NULL, 50.00);  -- Заказ без пользователя
```

**Данные:**
```
users:                orders:
+----+---------+      +-----+---------+--------+
| id | name    |      | id  | user_id | amount |
+----+---------+      +-----+---------+--------+
| 1  | Alice   |      | 101 | 1       | 100.00 |
| 2  | Bob     |      | 102 | 1       | 200.00 |
| 3  | Charlie |      | 103 | 2       | 150.00 |
+----+---------+      | 104 | NULL    | 50.00  |
                      +-----+---------+--------+
```

## 1. INNER JOIN

Возвращает только строки, где **есть совпадение** в обеих таблицах.

```sql
SELECT
    users.name,
    orders.id AS order_id,
    orders.amount
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

**Результат:**
```
+---------+----------+--------+
| name    | order_id | amount |
+---------+----------+--------+
| Alice   | 101      | 100.00 |
| Alice   | 102      | 200.00 |
| Bob     | 103      | 150.00 |
+---------+----------+--------+
```

**Заметьте:**
- Charlie отсутствует (нет заказов)
- Заказ 104 отсутствует (user_id = NULL)

**Визуализация:**
```
users ∩ orders
    [Alice]
    [Bob]
```

### Сокращенный синтаксис

```sql
-- INNER можно опустить
SELECT * FROM users JOIN orders ON users.id = orders.user_id;

-- Если имена колонок совпадают - USING
SELECT * FROM users JOIN orders USING (id);  -- Если id одинаковое имя

-- Еще короче - NATURAL JOIN (автоматически по одинаковым именам)
SELECT * FROM users NATURAL JOIN orders;  -- Осторожно!
```

## 2. LEFT (OUTER) JOIN

Возвращает **все строки из левой таблицы** + совпадения из правой (NULL если нет).

```sql
SELECT
    users.name,
    orders.id AS order_id,
    orders.amount
FROM users
LEFT JOIN orders ON users.id = orders.user_id;
```

**Результат:**
```
+---------+----------+--------+
| name    | order_id | amount |
+---------+----------+--------+
| Alice   | 101      | 100.00 |
| Alice   | 102      | 200.00 |
| Bob     | 103      | 150.00 |
| Charlie | NULL     | NULL   |
+---------+----------+--------+
```

**Заметьте:**
- Charlie присутствует (даже без заказов)
- Для Charlie order_id и amount = NULL

**Визуализация:**
```
users + (users ∩ orders)
    [Alice] ✅
    [Bob] ✅
    [Charlie] ✅ (NULL для orders)
```

### Найти пользователей БЕЗ заказов

```sql
SELECT users.name
FROM users
LEFT JOIN orders ON users.id = orders.user_id
WHERE orders.id IS NULL;
```

**Результат:**
```
+---------+
| name    |
+---------+
| Charlie |
+---------+
```

## 3. RIGHT (OUTER) JOIN

Возвращает **все строки из правой таблицы** + совпадения из левой (NULL если нет).

```sql
SELECT
    users.name,
    orders.id AS order_id,
    orders.amount
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;
```

**Результат:**
```
+---------+----------+--------+
| name    | order_id | amount |
+---------+----------+--------+
| Alice   | 101      | 100.00 |
| Alice   | 102      | 200.00 |
| Bob     | 103      | 150.00 |
| NULL    | 104      | 50.00  |
+---------+----------+--------+
```

**Заметьте:**
- Заказ 104 присутствует (даже без пользователя)
- Для заказа 104 name = NULL

**Практика:** RIGHT JOIN используется редко. Обычно переписывают как LEFT JOIN.

```sql
-- Эквивалентно
SELECT * FROM orders LEFT JOIN users ON orders.user_id = users.id;
```

## 4. FULL (OUTER) JOIN

Возвращает **все строки из обеих таблиц**. NULL где нет совпадений.

```sql
SELECT
    users.name,
    orders.id AS order_id,
    orders.amount
FROM users
FULL JOIN orders ON users.id = orders.user_id;
```

**Результат:**
```
+---------+----------+--------+
| name    | order_id | amount |
+---------+----------+--------+
| Alice   | 101      | 100.00 |
| Alice   | 102      | 200.00 |
| Bob     | 103      | 150.00 |
| Charlie | NULL     | NULL   |
| NULL    | 104      | 50.00  |
+---------+----------+--------+
```

**Визуализация:**
```
users ∪ orders
    [Alice] ✅
    [Bob] ✅
    [Charlie] ✅ (NULL для orders)
    [Order 104] ✅ (NULL для users)
```

### Найти несовпадения

```sql
-- Пользователи без заказов + заказы без пользователей
SELECT *
FROM users
FULL JOIN orders ON users.id = orders.user_id
WHERE users.id IS NULL OR orders.id IS NULL;
```

**Результат:**
```
+---------+----------+--------+
| name    | order_id | amount |
+---------+----------+--------+
| Charlie | NULL     | NULL   |
| NULL    | 104      | 50.00  |
+---------+----------+--------+
```

## 5. CROSS JOIN

Декартово произведение: каждая строка из первой таблицы × каждая строка из второй.

```sql
SELECT
    users.name,
    orders.id AS order_id
FROM users
CROSS JOIN orders;
```

**Результат:**
```
+---------+----------+
| name    | order_id |
+---------+----------+
| Alice   | 101      |
| Alice   | 102      |
| Alice   | 103      |
| Alice   | 104      |
| Bob     | 101      |
| Bob     | 102      |
| Bob     | 103      |
| Bob     | 104      |
| Charlie | 101      |
| Charlie | 102      |
| Charlie | 103      |
| Charlie | 104      |
+---------+----------+
```

**Количество строк:** `users.count × orders.count = 3 × 4 = 12`

**Когда использовать:**
- Генерация комбинаций
- Заполнение таблиц (календари, матрицы)

```sql
-- Генерация дат на год
SELECT date::date
FROM generate_series('2024-01-01', '2024-12-31', '1 day') AS date;

-- Комбинации продуктов и размеров
SELECT products.name, sizes.size
FROM products
CROSS JOIN (VALUES ('S'), ('M'), ('L'), ('XL')) AS sizes(size);
```

## Сравнение JOIN типов

| JOIN Type | Левая таблица | Правая таблица | Условие |
|-----------|---------------|----------------|---------|
| INNER | Только совпадения | Только совпадения | ON |
| LEFT | Все строки | Только совпадения + NULL | ON |
| RIGHT | Только совпадения + NULL | Все строки | ON |
| FULL | Все строки | Все строки | ON |
| CROSS | Все строки | Все строки | Нет (декартово) |

## Множественные JOIN

### 3 таблицы

```sql
CREATE TABLE products (
    id int PRIMARY KEY,
    name varchar(50)
);

CREATE TABLE order_items (
    order_id int,
    product_id int,
    quantity int
);

INSERT INTO products VALUES (1, 'Laptop'), (2, 'Mouse');
INSERT INTO order_items VALUES (101, 1, 2), (101, 2, 1);

-- JOIN трех таблиц
SELECT
    users.name AS customer,
    orders.id AS order_id,
    products.name AS product,
    order_items.quantity
FROM users
JOIN orders ON users.id = orders.user_id
JOIN order_items ON orders.id = order_items.order_id
JOIN products ON order_items.product_id = products.id;
```

**Результат:**
```
+----------+----------+---------+----------+
| customer | order_id | product | quantity |
+----------+----------+---------+----------+
| Alice    | 101      | Laptop  | 2        |
| Alice    | 101      | Mouse   | 1        |
+----------+----------+---------+----------+
```

### Порядок JOIN имеет значение!

```sql
-- Оптимизатор может переставлять, но логически:

-- 1. users JOIN orders → промежуточный результат
-- 2. result JOIN order_items → промежуточный результат
-- 3. result JOIN products → финальный результат
```

## SELF JOIN

JOIN таблицы с самой собой.

```sql
CREATE TABLE employees (
    id int PRIMARY KEY,
    name varchar(50),
    manager_id int
);

INSERT INTO employees VALUES
    (1, 'Alice', NULL),    -- CEO
    (2, 'Bob', 1),         -- Менеджер Alice
    (3, 'Charlie', 1),     -- Менеджер Alice
    (4, 'David', 2);       -- Менеджер Bob

-- Найти сотрудников и их менеджеров
SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

**Результат:**
```
+----------+---------+
| employee | manager |
+----------+---------+
| Alice    | NULL    |
| Bob      | Alice   |
| Charlie  | Alice   |
| David    | Bob     |
+----------+---------+
```

### Найти коллег (с одним менеджером)

```sql
SELECT
    e1.name AS employee1,
    e2.name AS employee2,
    m.name AS manager
FROM employees e1
JOIN employees e2 ON e1.manager_id = e2.manager_id AND e1.id < e2.id
JOIN employees m ON e1.manager_id = m.id;
```

**Результат:**
```
+-----------+-----------+---------+
| employee1 | employee2 | manager |
+-----------+-----------+---------+
| Bob       | Charlie   | Alice   |
+-----------+-----------+---------+
```

## JOIN с условиями (WHERE vs ON)

### ON - условие JOIN

```sql
SELECT *
FROM users
LEFT JOIN orders ON users.id = orders.user_id AND orders.amount > 100;
```

**Результат:**
```
+---------+----------+--------+
| name    | order_id | amount |
+---------+----------+--------+
| Alice   | 102      | 200.00 |
| Bob     | 103      | 150.00 |
| Charlie | NULL     | NULL   |
+---------+----------+--------+
```

Заказ 101 (amount=100) не попал, но Alice есть с NULL.

### WHERE - фильтр результата

```sql
SELECT *
FROM users
LEFT JOIN orders ON users.id = orders.user_id
WHERE orders.amount > 100;
```

**Результат:**
```
+---------+----------+--------+
| name    | order_id | amount |
+---------+----------+--------+
| Alice   | 102      | 200.00 |
| Bob     | 103      | 150.00 |
+---------+----------+--------+
```

Charlie отфильтровался (orders.amount IS NULL не > 100).

**Правило:**
- `ON` - когда JOIN
- `WHERE` - после JOIN

## Производительность JOIN

### 1. Используйте индексы на JOIN колонках

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Теперь JOIN быстрый (Index Scan вместо Seq Scan)
SELECT * FROM users JOIN orders ON users.id = orders.user_id;
```

### 2. JOIN меньших таблиц первыми

```sql
-- ❌ Плохо (большая таблица первая)
SELECT *
FROM large_table
JOIN small_table ON large_table.id = small_table.foreign_id;

-- ✅ Хорошо (маленькая таблица первая)
SELECT *
FROM small_table
JOIN large_table ON small_table.foreign_id = large_table.id;
```

PostgreSQL оптимизатор обычно делает это сам, но явный порядок помогает.

### 3. Избегайте CROSS JOIN больших таблиц

```sql
-- ❌ ОПАСНО!
SELECT * FROM table1 CROSS JOIN table2;
-- Если table1 = 10000 строк, table2 = 10000 строк
-- Результат = 100,000,000 строк!
```

### 4. EXPLAIN для проверки

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM users
JOIN orders ON users.id = orders.user_id;
```

**Смотрим на:**
- `Nested Loop` - для маленьких таблиц
- `Hash Join` - для средних таблиц
- `Merge Join` - для больших отсортированных таблиц

## Сложные примеры

### Агрегация с JOIN

```sql
-- Сумма заказов по пользователям
SELECT
    users.name,
    COALESCE(SUM(orders.amount), 0) AS total_spent
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.id, users.name
ORDER BY total_spent DESC;
```

**Результат:**
```
+---------+-------------+
| name    | total_spent |
+---------+-------------+
| Alice   | 300.00      |
| Bob     | 150.00      |
| Charlie | 0.00        |
+---------+-------------+
```

### Подзапросы vs JOIN

```sql
-- Вариант 1: подзапрос (для каждого пользователя отдельный запрос)
SELECT
    name,
    (SELECT COUNT(*) FROM orders WHERE user_id = users.id) AS order_count
FROM users;

-- Вариант 2: JOIN (один запрос)
SELECT
    users.name,
    COUNT(orders.id) AS order_count
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.id, users.name;
```

**JOIN обычно быстрее!**

### Множественные условия JOIN

```sql
-- JOIN по нескольким колонкам
SELECT *
FROM table1
JOIN table2 ON table1.id = table2.id
           AND table1.type = table2.type
           AND table1.date = table2.date;
```

### JOIN с DISTINCT

```sql
-- Пользователи, у которых есть заказы (без дубликатов)
SELECT DISTINCT users.*
FROM users
JOIN orders ON users.id = orders.user_id;

-- Эквивалентно (но EXISTS может быть быстрее)
SELECT *
FROM users
WHERE EXISTS (SELECT 1 FROM orders WHERE orders.user_id = users.id);
```

## Распространённые ошибки

### Ошибка 1: Забыли GROUP BY при агрегации

```sql
-- ❌ Ошибка
SELECT
    users.name,
    SUM(orders.amount)
FROM users
JOIN orders ON users.id = orders.user_id;
-- ERROR: column "users.name" must appear in GROUP BY

-- ✅ Правильно
SELECT
    users.name,
    SUM(orders.amount)
FROM users
JOIN orders ON users.id = orders.user_id
GROUP BY users.name;
```

### Ошибка 2: Перепутали LEFT и INNER JOIN

```sql
-- ❌ Хотели всех пользователей, но получили только с заказами
SELECT users.name, COUNT(orders.id)
FROM users
JOIN orders ON users.id = orders.user_id
GROUP BY users.name;
-- Charlie отсутствует!

-- ✅ Правильно
SELECT users.name, COUNT(orders.id)
FROM users
LEFT JOIN orders ON users.id = orders.user_id
GROUP BY users.name;
```

### Ошибка 3: WHERE убивает LEFT JOIN

```sql
-- ❌ LEFT JOIN превратился в INNER
SELECT *
FROM users
LEFT JOIN orders ON users.id = orders.user_id
WHERE orders.amount > 100;
-- Charlie исчез (orders.amount IS NULL)

-- ✅ Условие в ON
SELECT *
FROM users
LEFT JOIN orders ON users.id = orders.user_id AND orders.amount > 100;
-- Charlie остался с NULL
```

## Best Practices

1. ✅ Используйте явный тип JOIN (INNER, LEFT) для читаемости
2. ✅ Создавайте индексы на JOIN колонках
3. ✅ Используйте EXPLAIN для проверки производительности
4. ✅ Фильтры для LEFT JOIN пишите в ON, не в WHERE
5. ✅ Для "есть связь" используйте EXISTS вместо JOIN + DISTINCT
6. ❌ Избегайте CROSS JOIN без необходимости
7. ❌ Не забывайте GROUP BY при агрегации
8. ✅ Используйте алиасы таблиц для краткости
9. ✅ COALESCE для замены NULL на значения по умолчанию
10. ✅ Тестируйте с реальными данными (edge cases)

## Связанные темы

- [[PostgreSQL - Основы]]
- [[PostgreSQL - Типы индексов]]
- [[PostgreSQL - Оптимизация запросов]]
- [[SQL - Агрегация и группировка]]
- [[SQL - Оконные функции]]
