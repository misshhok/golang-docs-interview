# SQL - Оконные функции

Оконные функции (Window Functions) выполняют вычисления на наборе строк, связанных с текущей строкой, **не группируя результат**.

## Отличие от GROUP BY

**GROUP BY** - схлопывает строки в группы:
```sql
SELECT user_id, SUM(amount)
FROM orders
GROUP BY user_id;
-- 2 строки (по одной на user_id)
```

**Window Functions** - сохраняет все строки:
```sql
SELECT
    user_id,
    amount,
    SUM(amount) OVER (PARTITION BY user_id) AS user_total
FROM orders;
-- 5 строк (все заказы + итог по пользователю)
```

## Синтаксис

```sql
function_name(...) OVER (
    [PARTITION BY column]
    [ORDER BY column]
    [frame_clause]
)
```

**Компоненты:**
- `PARTITION BY` - разбивает данные на группы (как GROUP BY, но не схлопывает)
- `ORDER BY` - сортировка внутри окна
- `frame_clause` - определяет границы окна (ROWS, RANGE)

## Тестовые данные

```sql
CREATE TABLE sales (
    id int,
    employee varchar(50),
    department varchar(50),
    amount decimal(10,2),
    sale_date date
);

INSERT INTO sales VALUES
    (1, 'Alice', 'Sales', 1000, '2024-01-01'),
    (2, 'Bob', 'Sales', 1500, '2024-01-02'),
    (3, 'Charlie', 'Sales', 800, '2024-01-03'),
    (4, 'David', 'IT', 2000, '2024-01-01'),
    (5, 'Eve', 'IT', 1800, '2024-01-02'),
    (6, 'Frank', 'IT', 2200, '2024-01-03');
```

## Агрегатные функции как оконные

### SUM OVER

```sql
-- Общая сумма по каждому отделу (для каждой строки)
SELECT
    employee,
    department,
    amount,
    SUM(amount) OVER (PARTITION BY department) AS dept_total
FROM sales;
```

**Результат:**
```
+----------+------------+--------+------------+
| employee | department | amount | dept_total |
+----------+------------+--------+------------+
| Alice    | Sales      | 1000   | 3300       |
| Bob      | Sales      | 1500   | 3300       |
| Charlie  | Sales      | 800    | 3300       |
| David    | IT         | 2000   | 6000       |
| Eve      | IT         | 1800   | 6000       |
| Frank    | IT         | 2200   | 6000       |
+----------+------------+--------+------------+
```

### AVG, MIN, MAX, COUNT OVER

```sql
SELECT
    employee,
    department,
    amount,
    AVG(amount) OVER (PARTITION BY department) AS dept_avg,
    MIN(amount) OVER (PARTITION BY department) AS dept_min,
    MAX(amount) OVER (PARTITION BY department) AS dept_max,
    COUNT(*) OVER (PARTITION BY department) AS dept_count
FROM sales;
```

**Результат:**
```
+----------+------------+--------+----------+----------+----------+------------+
| employee | department | amount | dept_avg | dept_min | dept_max | dept_count |
+----------+------------+--------+----------+----------+----------+------------+
| Alice    | Sales      | 1000   | 1100.00  | 800      | 1500     | 3          |
| Bob      | Sales      | 1500   | 1100.00  | 800      | 1500     | 3          |
| Charlie  | Sales      | 800    | 1100.00  | 800      | 1500     | 3          |
| David    | IT         | 2000   | 2000.00  | 1800     | 2200     | 3          |
| Eve      | IT         | 1800   | 2000.00  | 1800     | 2200     | 3          |
| Frank    | IT         | 2200   | 2000.00  | 1800     | 2200     | 3          |
+----------+------------+--------+----------+----------+----------+------------+
```

## Ranking Functions (функции ранжирования)

### ROW_NUMBER - уникальный номер строки

```sql
-- Номер каждого сотрудника в отделе по сумме продаж
SELECT
    employee,
    department,
    amount,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY amount DESC) AS rank
FROM sales;
```

**Результат:**
```
+----------+------------+--------+------+
| employee | department | amount | rank |
+----------+------------+--------+------+
| Bob      | Sales      | 1500   | 1    |
| Alice    | Sales      | 1000   | 2    |
| Charlie  | Sales      | 800    | 3    |
| Frank    | IT         | 2200   | 1    |
| David    | IT         | 2000   | 2    |
| Eve      | IT         | 1800   | 3    |
+----------+------------+--------+------+
```

**ROW_NUMBER** всегда уникален, даже для одинаковых значений.

### RANK - ранг с пропусками

```sql
INSERT INTO sales VALUES (7, 'George', 'Sales', 1500, '2024-01-04');

SELECT
    employee,
    amount,
    RANK() OVER (ORDER BY amount DESC) AS rank
FROM sales
WHERE department = 'Sales';
```

**Результат:**
```
+----------+--------+------+
| employee | amount | rank |
+----------+--------+------+
| Bob      | 1500   | 1    |
| George   | 1500   | 1    | -- Одинаковые значения = одинаковый ранг
| Alice    | 1000   | 3    | -- Пропуск! (не 2)
| Charlie  | 800    | 4    |
+----------+--------+------+
```

**RANK** пропускает номера при одинаковых значениях.

### DENSE_RANK - ранг без пропусков

```sql
SELECT
    employee,
    amount,
    DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rank
FROM sales
WHERE department = 'Sales';
```

**Результат:**
```
+----------+--------+------------+
| employee | amount | dense_rank |
+----------+--------+------------+
| Bob      | 1500   | 1          |
| George   | 1500   | 1          |
| Alice    | 1000   | 2          | -- Нет пропуска!
| Charlie  | 800    | 3          |
+----------+--------+------------+
```

**DENSE_RANK** не пропускает номера.

### Сравнение функций ранжирования

```sql
SELECT
    employee,
    amount,
    ROW_NUMBER() OVER (ORDER BY amount DESC) AS row_num,
    RANK() OVER (ORDER BY amount DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rank
FROM sales
WHERE department = 'Sales';
```

**Результат:**
```
+----------+--------+---------+------+------------+
| employee | amount | row_num | rank | dense_rank |
+----------+--------+---------+------+------------+
| Bob      | 1500   | 1       | 1    | 1          |
| George   | 1500   | 2       | 1    | 1          |
| Alice    | 1000   | 3       | 3    | 2          |
| Charlie  | 800    | 4       | 4    | 3          |
+----------+--------+---------+------+------------+
```

### NTILE - разбиение на N групп

```sql
-- Разбить на 3 группы (терциль)
SELECT
    employee,
    amount,
    NTILE(3) OVER (ORDER BY amount DESC) AS tile
FROM sales;
```

**Результат:**
```
+----------+--------+------+
| employee | amount | tile |
+----------+--------+------+
| Frank    | 2200   | 1    | -- Топ 33%
| David    | 2000   | 1    |
| Eve      | 1800   | 2    | -- Средние 33%
| Bob      | 1500   | 2    |
| George   | 1500   | 2    |
| Alice    | 1000   | 3    | -- Нижние 33%
| Charlie  | 800    | 3    |
+----------+--------+------+
```

**Применение:** Квартили, децили для анализа.

## Value Functions (функции доступа к значениям)

### LAG - предыдущее значение

```sql
SELECT
    employee,
    sale_date,
    amount,
    LAG(amount) OVER (PARTITION BY department ORDER BY sale_date) AS prev_amount,
    amount - LAG(amount) OVER (PARTITION BY department ORDER BY sale_date) AS diff
FROM sales;
```

**Результат:**
```
+----------+------------+--------+-------------+--------+
| employee | sale_date  | amount | prev_amount | diff   |
+----------+------------+--------+-------------+--------+
| Alice    | 2024-01-01 | 1000   | NULL        | NULL   |
| Bob      | 2024-01-02 | 1500   | 1000        | 500    |
| Charlie  | 2024-01-03 | 800    | 1500        | -700   |
| David    | 2024-01-01 | 2000   | NULL        | NULL   |
| Eve      | 2024-01-02 | 1800   | 2000        | -200   |
| Frank    | 2024-01-03 | 2200   | 1800        | 400    |
+----------+------------+--------+-------------+--------+
```

**LAG** смотрит на предыдущую строку в окне.

### LEAD - следующее значение

```sql
SELECT
    employee,
    sale_date,
    amount,
    LEAD(amount) OVER (PARTITION BY department ORDER BY sale_date) AS next_amount
FROM sales;
```

**Результат:**
```
+----------+------------+--------+-------------+
| employee | sale_date  | amount | next_amount |
+----------+------------+--------+-------------+
| Alice    | 2024-01-01 | 1000   | 1500        |
| Bob      | 2024-01-02 | 1500   | 800         |
| Charlie  | 2024-01-03 | 800    | NULL        |
| David    | 2024-01-01 | 2000   | 1800        |
| Eve      | 2024-01-02 | 1800   | 2200        |
| Frank    | 2024-01-03 | 2200   | NULL        |
+----------+------------+--------+-------------+
```

**LEAD** смотрит на следующую строку.

### FIRST_VALUE, LAST_VALUE - первое/последнее значение

```sql
SELECT
    employee,
    department,
    amount,
    FIRST_VALUE(employee) OVER (PARTITION BY department ORDER BY amount DESC) AS top_seller,
    LAST_VALUE(employee) OVER (
        PARTITION BY department
        ORDER BY amount DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS worst_seller
FROM sales;
```

**Результат:**
```
+----------+------------+--------+------------+--------------+
| employee | department | amount | top_seller | worst_seller |
+----------+------------+--------+------------+--------------+
| Bob      | Sales      | 1500   | Bob        | Charlie      |
| Alice    | Sales      | 1000   | Bob        | Charlie      |
| Charlie  | Sales      | 800    | Bob        | Charlie      |
| Frank    | IT         | 2200   | Frank      | Eve          |
| David    | IT         | 2000   | Frank      | Eve          |
| Eve      | IT         | 1800   | Frank      | Eve          |
+----------+------------+--------+------------+--------------+
```

**ВАЖНО:** `LAST_VALUE` требует явного указания frame (`ROWS BETWEEN ...`), иначе вернет текущую строку!

### NTH_VALUE - N-ое значение

```sql
-- Второй лучший продавец в отделе
SELECT
    employee,
    department,
    amount,
    NTH_VALUE(employee, 2) OVER (
        PARTITION BY department
        ORDER BY amount DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_best
FROM sales;
```

## Frame Clause (границы окна)

Определяет, какие строки включать в окно для вычислений.

### ROWS vs RANGE

**ROWS** - физические строки:
```sql
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
-- Предыдущая строка + текущая + следующая
```

**RANGE** - логические значения:
```sql
RANGE BETWEEN 100 PRECEDING AND 100 FOLLOWING
-- Все строки, где значение в пределах ±100 от текущей
```

### Примеры frame

```sql
-- Скользящее среднее (3 строки)
SELECT
    sale_date,
    amount,
    AVG(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS moving_avg_3
FROM sales
ORDER BY sale_date;
```

**Результат:**
```
+------------+--------+-------------+
| sale_date  | amount | moving_avg_3|
+------------+--------+-------------+
| 2024-01-01 | 1000   | 1500.00     | -- (1000+2000)/2
| 2024-01-01 | 2000   | 1600.00     | -- (1000+2000+1500)/3
| 2024-01-02 | 1500   | 1766.67     | -- (2000+1500+1800)/3
| 2024-01-02 | 1800   | 1700.00     | -- (1500+1800+800)/3
| 2024-01-03 | 800    | 1600.00     | -- (1800+800+2200)/3
| 2024-01-03 | 2200   | 1500.00     | -- (800+2200)/2
+------------+--------+-------------+
```

### Frame синтаксис

```sql
{ROWS | RANGE} BETWEEN frame_start AND frame_end

frame_start/frame_end:
  - UNBOUNDED PRECEDING    -- До начала окна
  - N PRECEDING            -- N строк/значений до текущей
  - CURRENT ROW            -- Текущая строка
  - N FOLLOWING            -- N строк/значений после текущей
  - UNBOUNDED FOLLOWING    -- До конца окна
```

**Примеры:**

```sql
-- Все строки от начала до текущей (cumulative sum)
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- Все строки в окне
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- Скользящее окно (5 строк)
ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING

-- Только предыдущая строка
ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING
```

## Кумулятивные вычисления

### Cumulative Sum (нарастающий итог)

```sql
SELECT
    sale_date,
    employee,
    amount,
    SUM(amount) OVER (
        PARTITION BY department
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_total
FROM sales
ORDER BY department, sale_date;
```

**Результат:**
```
+------------+----------+--------+------------------+
| sale_date  | employee | amount | cumulative_total |
+------------+----------+--------+------------------+
| 2024-01-01 | David    | 2000   | 2000             |
| 2024-01-02 | Eve      | 1800   | 3800             |
| 2024-01-03 | Frank    | 2200   | 6000             |
| 2024-01-01 | Alice    | 1000   | 1000             |
| 2024-01-02 | Bob      | 1500   | 2500             |
| 2024-01-03 | Charlie  | 800    | 3300             |
+------------+----------+--------+------------------+
```

### Running Average

```sql
SELECT
    sale_date,
    amount,
    AVG(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_avg
FROM sales
ORDER BY sale_date;
```

## Практические примеры

### Top N по группам

```sql
-- Топ-2 продавца в каждом отделе
SELECT *
FROM (
    SELECT
        employee,
        department,
        amount,
        RANK() OVER (PARTITION BY department ORDER BY amount DESC) AS rank
    FROM sales
) ranked
WHERE rank <= 2;
```

**Результат:**
```
+----------+------------+--------+------+
| employee | department | amount | rank |
+----------+------------+--------+------+
| Frank    | IT         | 2200   | 1    |
| David    | IT         | 2000   | 2    |
| Bob      | Sales      | 1500   | 1    |
| George   | Sales      | 1500   | 1    |
| Alice    | Sales      | 1000   | 3    |
+----------+------------+--------+------+
```

Если нужно ровно 2 (без дубликатов):
```sql
WHERE rank <= 2 -- RANK
-- или
WHERE row_num <= 2 -- ROW_NUMBER
```

### Процент от общего по группе

```sql
SELECT
    employee,
    department,
    amount,
    ROUND(
        100.0 * amount / SUM(amount) OVER (PARTITION BY department),
        2
    ) AS pct_of_dept
FROM sales;
```

**Результат:**
```
+----------+------------+--------+-------------+
| employee | department | amount | pct_of_dept |
+----------+------------+--------+-------------+
| Alice    | Sales      | 1000   | 30.30       |
| Bob      | Sales      | 1500   | 45.45       |
| Charlie  | Sales      | 800    | 24.24       |
| David    | IT         | 2000   | 33.33       |
| Eve      | IT         | 1800   | 30.00       |
| Frank    | IT         | 2200   | 36.67       |
+----------+------------+--------+-------------+
```

### Gap Analysis (пропуски в последовательности)

```sql
CREATE TABLE logins (user_id int, login_date date);
INSERT INTO logins VALUES
    (1, '2024-01-01'),
    (1, '2024-01-02'),
    (1, '2024-01-04'),  -- Пропуск!
    (1, '2024-01-05');

SELECT
    user_id,
    login_date,
    LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS prev_login,
    login_date - LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS days_gap
FROM logins;
```

**Результат:**
```
+---------+------------+------------+----------+
| user_id | login_date | prev_login | days_gap |
+---------+------------+------------+----------+
| 1       | 2024-01-01 | NULL       | NULL     |
| 1       | 2024-01-02 | 2024-01-01 | 1        |
| 1       | 2024-01-04 | 2024-01-02 | 2        | -- Пропуск!
| 1       | 2024-01-05 | 2024-01-04 | 1        |
+---------+------------+------------+----------+
```

### Удаление дубликатов

```sql
CREATE TABLE duplicates (id int, name varchar(50), value int);
INSERT INTO duplicates VALUES
    (1, 'Alice', 100),
    (2, 'Alice', 100),  -- Дубликат
    (3, 'Bob', 200);

-- Удалить дубликаты, оставив первый
DELETE FROM duplicates
WHERE id IN (
    SELECT id
    FROM (
        SELECT
            id,
            ROW_NUMBER() OVER (PARTITION BY name, value ORDER BY id) AS rn
        FROM duplicates
    ) ranked
    WHERE rn > 1
);
```

### Year-over-Year сравнение

```sql
CREATE TABLE revenue (year int, month int, amount decimal(10,2));
INSERT INTO revenue VALUES
    (2023, 1, 1000),
    (2023, 2, 1200),
    (2024, 1, 1500),
    (2024, 2, 1400);

SELECT
    year,
    month,
    amount,
    LAG(amount) OVER (PARTITION BY month ORDER BY year) AS prev_year,
    amount - LAG(amount) OVER (PARTITION BY month ORDER BY year) AS yoy_change
FROM revenue;
```

**Результат:**
```
+------+-------+--------+-----------+------------+
| year | month | amount | prev_year | yoy_change |
+------+-------+--------+-----------+------------+
| 2023 | 1     | 1000   | NULL      | NULL       |
| 2024 | 1     | 1500   | 1000      | 500        |
| 2023 | 2     | 1200   | NULL      | NULL       |
| 2024 | 2     | 1400   | 1200      | 200        |
+------+-------+--------+-----------+------------+
```

## Производительность

### Индексы

```sql
-- Индекс на колонки в PARTITION BY и ORDER BY
CREATE INDEX idx_sales_dept_date ON sales(department, sale_date);

SELECT
    employee,
    SUM(amount) OVER (PARTITION BY department ORDER BY sale_date)
FROM sales;
-- Может использовать индекс для сортировки
```

### Избегайте вложенных оконных функций

```sql
-- ❌ Медленно (вложенные OVER)
SELECT
    employee,
    amount,
    RANK() OVER (
        ORDER BY (SUM(amount) OVER (PARTITION BY department))
    )
FROM sales;

-- ✅ Быстрее (CTE или подзапрос)
WITH dept_totals AS (
    SELECT
        department,
        SUM(amount) AS total
    FROM sales
    GROUP BY department
)
SELECT
    s.employee,
    s.amount,
    RANK() OVER (ORDER BY dt.total)
FROM sales s
JOIN dept_totals dt ON s.department = dt.department;
```

### EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    employee,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY amount DESC)
FROM sales;
```

Смотрим на `WindowAgg` node.

## Распространённые ошибки

### Ошибка 1: LAST_VALUE без frame

```sql
-- ❌ Вернет текущую строку!
SELECT
    employee,
    LAST_VALUE(employee) OVER (PARTITION BY department ORDER BY amount DESC)
FROM sales;

-- ✅ Правильно
SELECT
    employee,
    LAST_VALUE(employee) OVER (
        PARTITION BY department
        ORDER BY amount DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    )
FROM sales;
```

### Ошибка 2: Забыли ORDER BY

```sql
-- ❌ ROW_NUMBER без ORDER BY - случайный порядок!
SELECT
    employee,
    ROW_NUMBER() OVER (PARTITION BY department)
FROM sales;

-- ✅ Правильно
SELECT
    employee,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY amount DESC)
FROM sales;
```

### Ошибка 3: Оконная функция в WHERE

```sql
-- ❌ Нельзя использовать в WHERE!
SELECT employee, amount
FROM sales
WHERE RANK() OVER (ORDER BY amount DESC) <= 3;
-- ERROR: window functions are not allowed in WHERE

-- ✅ Правильно (подзапрос или CTE)
SELECT *
FROM (
    SELECT
        employee,
        amount,
        RANK() OVER (ORDER BY amount DESC) AS rank
    FROM sales
) ranked
WHERE rank <= 3;
```

## Best Practices

1. ✅ Используйте оконные функции вместо SELF JOIN
2. ✅ Индексы на PARTITION BY и ORDER BY колонках
3. ✅ Явно указывайте frame для LAST_VALUE
4. ✅ CTE для читаемости сложных запросов
5. ✅ ROW_NUMBER для уникальных рангов
6. ✅ RANK/DENSE_RANK когда нужны одинаковые ранги
7. ❌ Не используйте оконные функции в WHERE (только в подзапросе)
8. ✅ EXPLAIN для проверки производительности
9. ✅ LAG/LEAD для сравнения с предыдущими/следующими значениями
10. ✅ Именованные окна (WINDOW) для переиспользования

### Именованные окна

```sql
SELECT
    employee,
    amount,
    ROW_NUMBER() OVER w AS rn,
    RANK() OVER w AS rank,
    SUM(amount) OVER w AS cumulative
FROM sales
WINDOW w AS (PARTITION BY department ORDER BY amount DESC);
```

## Связанные темы

- [[SQL - Агрегация и группировка]]
- [[SQL - JOIN операции]]
- [[PostgreSQL - Основы]]
- [[PostgreSQL - Оптимизация запросов]]
- [[PostgreSQL - Типы индексов]]
