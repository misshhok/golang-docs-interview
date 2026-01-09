# PostgreSQL - Типы индексов

Индексы ускоряют поиск данных, но замедляют запись. Выбор правильного типа индекса критичен для производительности.

## Зачем нужны индексы?

**Без индекса:** Sequential Scan - O(n)
```sql
SELECT * FROM users WHERE email = 'user@example.com';
-- Сканирует ВСЕ строки таблицы
```

**С индексом:** Index Scan - O(log n)
```sql
CREATE INDEX idx_users_email ON users(email);
-- Теперь поиск за O(log n)
```

## Типы индексов в PostgreSQL

### 1. B-Tree (по умолчанию)

**Когда использовать:** 90% случаев. Подходит для большинства запросов.

**Поддерживает:**
- Равенство (`=`)
- Сравнения (`<`, `>`, `<=`, `>=`)
- Диапазоны (`BETWEEN`)
- Сортировку (`ORDER BY`)
- Поиск префиксов (`LIKE 'prefix%'`)

```sql
-- Создание B-Tree индекса
CREATE INDEX idx_users_email ON users(email);

-- Явное указание B-Tree (необязательно)
CREATE INDEX idx_users_email ON users USING btree(email);

-- Составной индекс
CREATE INDEX idx_users_last_first ON users(last_name, first_name);
```

**Структура:** Сбалансированное дерево

```
                [M]
               /   \
           [D,G]   [T,X]
          /  |  \   |  \
        [A] [E] [J] [P] [W]
```

**Примеры запросов:**

```sql
-- ✅ Использует индекс
SELECT * FROM users WHERE email = 'user@example.com';
SELECT * FROM users WHERE age > 18;
SELECT * FROM users WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
SELECT * FROM users ORDER BY email;
SELECT * FROM users WHERE email LIKE 'admin%';

-- ❌ НЕ использует индекс
SELECT * FROM users WHERE email LIKE '%@gmail.com';  -- % в начале
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';  -- функция
```

**Составной индекс (многоколоночный):**

```sql
CREATE INDEX idx_users_location ON users(country, city, district);

-- ✅ Использует индекс (префиксы)
WHERE country = 'RU'
WHERE country = 'RU' AND city = 'Moscow'
WHERE country = 'RU' AND city = 'Moscow' AND district = 'Center'

-- ❌ НЕ использует индекс (пропущен префикс)
WHERE city = 'Moscow'
WHERE district = 'Center'
```

### 2. Hash

**Когда использовать:** Только для точного совпадения (`=`). Быстрее B-Tree для равенства, но только для этого.

**НЕ поддерживает:** `<`, `>`, `BETWEEN`, `ORDER BY`, `LIKE`

```sql
CREATE INDEX idx_users_email_hash ON users USING hash(email);

-- ✅ Использует Hash индекс
SELECT * FROM users WHERE email = 'user@example.com';

-- ❌ НЕ использует Hash индекс
SELECT * FROM users WHERE email > 'a';
SELECT * FROM users ORDER BY email;
```

**Когда выбрать Hash:**
- Только запросы с `=`
- Высокая кардинальность (много уникальных значений)
- Экономия места (Hash индекс меньше B-Tree)

**Недостатки:**
- Нельзя восстановить после краша (нужен REINDEX)
- Не реплицируется

### 3. GiST (Generalized Search Tree)

**Когда использовать:** Геометрические типы, полнотекстовый поиск, диапазоны, массивы.

```sql
-- Геометрия (PostGIS)
CREATE INDEX idx_locations_point ON locations USING gist(geom);

SELECT * FROM locations WHERE ST_DWithin(geom, ST_MakePoint(55.75, 37.61), 1000);

-- Полнотекстовый поиск
CREATE INDEX idx_articles_search ON articles USING gist(to_tsvector('english', content));

SELECT * FROM articles WHERE to_tsvector('english', content) @@ to_tsquery('postgresql');

-- Диапазоны
CREATE INDEX idx_bookings_period ON bookings USING gist(period);

SELECT * FROM bookings WHERE period && '[2024-01-01, 2024-01-31]'::daterange;
```

**Поддерживает:**
- Геометрические операторы (PostGIS)
- Полнотекстовый поиск (`@@`)
- Диапазоны (`&&`, `@>`, `<@`)
- Массивы (`&&`, `@>`, `<@`)

### 4. GIN (Generalized Inverted Index)

**Когда использовать:** Массивы, JSONB, полнотекстовый поиск.

```sql
-- JSONB
CREATE INDEX idx_users_metadata ON users USING gin(metadata);

SELECT * FROM users WHERE metadata @> '{"role": "admin"}';
SELECT * FROM users WHERE metadata ? 'phone';

-- Массивы
CREATE INDEX idx_posts_tags ON posts USING gin(tags);

SELECT * FROM posts WHERE tags @> ARRAY['postgresql', 'database'];
SELECT * FROM posts WHERE tags && ARRAY['tutorial'];

-- Полнотекстовый поиск (быстрее GiST для FTS)
CREATE INDEX idx_articles_fts ON articles USING gin(to_tsvector('russian', content));

SELECT * FROM articles WHERE to_tsvector('russian', content) @@ to_tsquery('russian', 'база & данных');
```

**GIN vs GiST:**

| Критерий | GIN | GiST |
|----------|-----|------|
| Скорость поиска | ⚡ Быстрее | Медленнее |
| Скорость вставки | 🐌 Медленнее | ⚡ Быстрее |
| Размер индекса | 📦 Больше | 📦 Меньше |
| Для полнотекстового поиска | ✅ Рекомендуется | Допустимо |

**Правило:** GIN для read-heavy, GiST для write-heavy.

### 5. BRIN (Block Range Index)

**Когда использовать:** Огромные таблицы с естественной сортировкой (временные ряды, логи).

**Особенность:** Индексирует диапазоны блоков, а не отдельные строки.

```sql
CREATE INDEX idx_logs_created_brin ON logs USING brin(created_at);

-- Очень маленький размер индекса!
-- 1GB таблица → 100KB индекс
```

**Когда эффективен:**
```sql
-- ✅ Хорошо (данные физически отсортированы по created_at)
SELECT * FROM logs WHERE created_at > '2024-01-01';

-- ❌ Плохо (данные не отсортированы)
SELECT * FROM users WHERE created_at > '2024-01-01';
```

**Пример:** Таблица логов

```sql
-- Данные вставляются последовательно по времени
CREATE TABLE logs (
    id bigserial,
    created_at timestamp DEFAULT now(),
    message text
);

CREATE INDEX idx_logs_time ON logs USING brin(created_at);
```

**Размер:**
```sql
SELECT pg_size_pretty(pg_relation_size('idx_logs_time'));
-- 48 KB (вместо 50 MB для B-Tree!)
```

### 6. SP-GiST (Space-Partitioned GiST)

**Когда использовать:** Данные с неравномерным распределением (IP адреса, телефоны, географические точки).

```sql
-- IP адреса
CREATE INDEX idx_connections_ip ON connections USING spgist(ip_address inet_ops);

SELECT * FROM connections WHERE ip_address << '192.168.0.0/16';

-- Точки (quad-tree)
CREATE INDEX idx_locations_point ON locations USING spgist(point);
```

**Отличие от GiST:** Разделяет пространство по-разному (лучше для неравномерных данных).

## Специальные типы индексов

### Partial Index (Частичный индекс)

Индексируем только часть строк.

```sql
-- Индекс только для активных пользователей
CREATE INDEX idx_users_active_email ON users(email) WHERE is_active = true;

-- Индекс только для недавних записей
CREATE INDEX idx_logs_recent ON logs(created_at) WHERE created_at > '2024-01-01';

-- ✅ Использует индекс
SELECT * FROM users WHERE email = 'user@example.com' AND is_active = true;

-- ❌ НЕ использует индекс (условие не совпадает)
SELECT * FROM users WHERE email = 'user@example.com' AND is_active = false;
```

**Преимущества:**
- Меньший размер индекса
- Быстрее обновление
- Фокус на "горячих" данных

### Expression Index (Индекс по выражению)

Индекс на результат функции или выражения.

```sql
-- Поиск без учета регистра
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- Поиск по части даты
CREATE INDEX idx_orders_year ON orders(EXTRACT(YEAR FROM created_at));

SELECT * FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2024;

-- JSONB поле
CREATE INDEX idx_users_settings_theme ON users((metadata->>'theme'));

SELECT * FROM users WHERE metadata->>'theme' = 'dark';
```

### Covering Index (Индекс с INCLUDE)

Добавляем дополнительные колонки в индекс (только для чтения, не для поиска).

```sql
CREATE INDEX idx_users_email_include ON users(email) INCLUDE (first_name, last_name);

-- Index-Only Scan (не нужно обращаться к таблице!)
SELECT first_name, last_name FROM users WHERE email = 'user@example.com';
```

**Преимущество:** Избегаем обращения к heap (таблице), читаем только индекс.

### Unique Index

```sql
-- Уникальность + индекс
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Уникальность с условием
CREATE UNIQUE INDEX idx_users_active_email ON users(email) WHERE is_active = true;
```

## Выбор индекса: блок-схема

```
Что индексируем?
│
├─ Точное равенство (=)?
│  ├─ Да, только = → Hash
│  └─ Нет → B-Tree
│
├─ JSONB / массивы?
│  ├─ Read-heavy → GIN
│  └─ Write-heavy → GiST
│
├─ Геометрия (PostGIS)?
│  └─ GiST или SP-GiST
│
├─ Огромная таблица с естественной сортировкой?
│  └─ BRIN
│
└─ Обычный случай
   └─ B-Tree (по умолчанию)
```

## Проверка использования индекса

### EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'user@example.com';
```

**Смотрим на:**
- `Seq Scan` ❌ - не использует индекс
- `Index Scan` ✅ - использует индекс
- `Index Only Scan` ✅✅ - идеально
- `Bitmap Index Scan` ✅ - для нескольких индексов

### Пример вывода

```
Index Scan using idx_users_email on users  (cost=0.42..8.44 rows=1 width=100)
  Index Cond: (email = 'user@example.com'::text)
  Buffers: shared hit=4
```

## Когда индекс НЕ используется?

### 1. Функция на колонке

```sql
-- ❌ Не использует индекс
WHERE LOWER(email) = 'user@example.com'

-- ✅ Создать expression index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

### 2. Неявное приведение типов

```sql
-- phone_number - varchar, но ищем по integer
-- ❌ Не использует индекс
WHERE phone_number = 123456789

-- ✅ Правильно
WHERE phone_number = '123456789'
```

### 3. OR условия

```sql
-- ❌ Может не использовать индексы
WHERE email = 'a@b.com' OR phone = '123'

-- ✅ Использует индексы (если есть на обеих колонках)
WHERE email = 'a@b.com'
UNION
WHERE phone = '123'
```

### 4. LIKE с % в начале

```sql
-- ❌ Не использует индекс
WHERE email LIKE '%gmail.com'

-- ✅ Использует индекс
WHERE email LIKE 'admin%'

-- ✅ Для поиска подстроки - полнотекстовый поиск или pg_trgm
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_email_trgm ON users USING gin(email gin_trgm_ops);
```

### 5. Маленькая таблица

```sql
-- PostgreSQL может решить, что Seq Scan быстрее
-- Если таблица < 1000 строк, индекс может игнорироваться
```

## Мониторинг индексов

### Неиспользуемые индексы

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Размер индексов

```sql
SELECT
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Index Bloat (раздувание)

```sql
-- После многих UPDATE/DELETE индекс может раздуться
REINDEX INDEX idx_users_email;

-- Или всю таблицу
REINDEX TABLE users;

-- Или всю базу
REINDEX DATABASE mydb;
```

## Best Practices

1. ✅ B-Tree для большинства случаев
2. ✅ GIN для JSONB и массивов (read-heavy)
3. ✅ BRIN для временных рядов
4. ✅ Partial index для "горячих" данных
5. ✅ Expression index для функций
6. ✅ INCLUDE для covering index
7. ❌ Не создавайте индексы на маленькие таблицы (< 1000 строк)
8. ❌ Не индексируйте часто изменяемые колонки
9. ✅ Регулярно анализируйте использование индексов
10. ✅ REINDEX при bloat

## Стоимость индексов

**Плюсы:**
- ⚡ Ускорение SELECT (O(log n) вместо O(n))
- ⚡ Ускорение JOIN, ORDER BY, GROUP BY

**Минусы:**
- 🐌 Замедление INSERT, UPDATE, DELETE
- 💾 Дополнительное место на диске
- 🔄 Требуют обслуживания (VACUUM, REINDEX)

**Правило:** Индексировать колонки в WHERE, JOIN, ORDER BY, которые часто используются.

## Пример: выбор индекса для разных запросов

```sql
CREATE TABLE orders (
    id bigserial PRIMARY KEY,  -- автоматически B-Tree
    user_id bigint,
    status varchar(20),
    created_at timestamp DEFAULT now(),
    metadata jsonb
);

-- Поиск по user_id (часто)
CREATE INDEX idx_orders_user ON orders(user_id);

-- Поиск недавних активных заказов
CREATE INDEX idx_orders_status_time ON orders(status, created_at)
WHERE status IN ('pending', 'processing');

-- Поиск по JSONB
CREATE INDEX idx_orders_metadata ON orders USING gin(metadata);

-- Временной ряд (если таблица огромная)
CREATE INDEX idx_orders_created_brin ON orders USING brin(created_at);
```

## Связанные темы

- [[PostgreSQL - Основы]]
- [[PostgreSQL - Оптимизация запросов]]
- [[Алгоритмическая сложность (Big O)]]
- [[Деревья - Основы]]
- [[HashMap - Реализация и особенности]]
