# Redis - Типы данных и команды

Подробный справочник по всем типам данных Redis и их командам с примерами.

## String (строки)

### Базовые операции

```bash
# SET - установить значение
SET key "value"
SET user:1:name "Alice"

# GET - получить значение
GET user:1:name
# "Alice"

# MSET / MGET - множественные операции
MSET key1 "value1" key2 "value2" key3 "value3"
MGET key1 key2 key3
# ["value1", "value2", "value3"]

# GETSET - установить новое и вернуть старое
GETSET counter 0
# (nil) или старое значение

# SETNX - SET if Not eXists
SETNX lock "locked"
# 1 (успех) или 0 (уже существует)

# SETEX - SET with EXpire
SETEX session:abc "data" 3600  # TTL 3600 секунд

# PSETEX - SET with milliseconds expire
PSETEX key 5000 "value"  # 5000 мс
```

### Работа с числами

```bash
# INCR / DECR - инкремент/декремент
SET counter 10
INCR counter      # 11
DECR counter      # 10

# INCRBY / DECRBY - на N
INCRBY counter 5  # 15
DECRBY counter 3  # 12

# INCRBYFLOAT - float инкремент
INCRBYFLOAT price 10.5
# "22.5"

# Атомарные операции (thread-safe)
INCR views:page1  # Счетчик просмотров
```

### Работа со строками

```bash
# APPEND - добавить в конец
SET message "Hello"
APPEND message " World"
# 11 (новая длина)
GET message
# "Hello World"

# STRLEN - длина строки
STRLEN message
# 11

# GETRANGE - подстрока
SETRANGE message 6 "Redis"
# "Hello Redis"

# GETRANGE - получить подстрку
GETRANGE message 0 4
# "Hello"

# SETBIT / GETBIT - битовые операции
SETBIT bitmap 0 1
GETBIT bitmap 0
# 1

# BITCOUNT - количество единичных бит
BITCOUNT bitmap
```

### Команды с TTL

```bash
# EXPIRE - установить TTL (секунды)
SET key "value"
EXPIRE key 60

# PEXPIRE - TTL в миллисекундах
PEXPIRE key 60000

# EXPIREAT - истекает в Unix timestamp
EXPIREAT key 1735689600

# TTL - оставшееся время (секунды)
TTL key
# 59 (секунд)
# -1 (без TTL)
# -2 (не существует)

# PTTL - оставшееся время (миллисекунды)
PTTL key

# PERSIST - убрать TTL
PERSIST key
```

## Hash (хеш-таблицы)

### Базовые операции

```bash
# HSET - установить поле
HSET user:1 name "Alice"
HSET user:1 age 30
HSET user:1 email "alice@example.com"

# HMSET - множественная установка (deprecated, используйте HSET)
HSET user:1 name "Alice" age 30 email "alice@example.com"

# HGET - получить поле
HGET user:1 name
# "Alice"

# HMGET - множественное получение
HMGET user:1 name email
# ["Alice", "alice@example.com"]

# HGETALL - все поля и значения
HGETALL user:1
# ["name", "Alice", "age", "30", "email", "alice@example.com"]

# HKEYS - все ключи
HKEYS user:1
# ["name", "age", "email"]

# HVALS - все значения
HVALS user:1
# ["Alice", "30", "alice@example.com"]

# HLEN - количество полей
HLEN user:1
# 3

# HEXISTS - проверить существование поля
HEXISTS user:1 name
# 1 (exists) или 0 (not exists)

# HDEL - удалить поля
HDEL user:1 email
# 1 (количество удаленных)

# HSETNX - SET if Not eXists
HSETNX user:1 city "Moscow"
# 1 (успех) или 0 (уже существует)
```

### Числовые операции

```bash
# HINCRBY - инкремент integer поля
HSET user:1 age 30
HINCRBY user:1 age 1
# 31

# HINCRBYFLOAT - инкремент float
HSET product:1 price 99.99
HINCRBYFLOAT product:1 price 10.00
# "109.99"
```

### Сканирование

```bash
# HSCAN - итеративное сканирование
HSCAN user:1 0 MATCH name* COUNT 10
# ["0", ["name", "Alice"]]
```

## List (списки)

### Добавление элементов

```bash
# LPUSH - добавить в начало (left)
LPUSH tasks "task1"
LPUSH tasks "task2"
# ["task2", "task1"]

# RPUSH - добавить в конец (right)
RPUSH tasks "task3"
# ["task2", "task1", "task3"]

# LPUSHX / RPUSHX - только если список существует
LPUSHX tasks "task0"

# LINSERT - вставить перед/после элемента
LINSERT tasks BEFORE "task1" "task0.5"
LINSERT tasks AFTER "task1" "task1.5"
```

### Получение элементов

```bash
# LINDEX - элемент по индексу
LINDEX tasks 0
# "task2"

# LRANGE - диапазон элементов
LRANGE tasks 0 -1  # Все элементы
LRANGE tasks 0 9   # Первые 10
LRANGE tasks -3 -1 # Последние 3

# LLEN - длина списка
LLEN tasks
# 3
```

### Удаление элементов

```bash
# LPOP - извлечь первый (удалить и вернуть)
LPOP tasks
# "task2"

# RPOP - извлечь последний
RPOP tasks
# "task3"

# LREM - удалить N вхождений элемента
LREM tasks 1 "task1"  # Удалить первое вхождение
LREM tasks -1 "task1" # Удалить последнее вхождение
LREM tasks 0 "task1"  # Удалить все вхождения

# LTRIM - оставить только диапазон
LTRIM tasks 0 99  # Оставить первые 100 элементов

# LSET - установить элемент по индексу
LSET tasks 0 "new_task"
```

### Блокирующие операции

```bash
# BLPOP - блокирующий LPOP (ждет если пусто)
BLPOP tasks 10
# Ждет 10 секунд, возвращает ["tasks", "element"] или nil

# BRPOP - блокирующий RPOP
BRPOP tasks 0
# Ждет бесконечно (timeout=0)

# BRPOPLPUSH - атомарный перенос между списками
BRPOPLPUSH source destination 5
```

### Продвинутые операции

```bash
# RPOPLPUSH - переместить элемент из одного списка в другой
RPOPLPUSH source destination
# Возвращает элемент

# Реализация надежной очереди
RPUSH processing_queue "task1"
# Worker берет задачу
RPOPLPUSH processing_queue in_progress
# После обработки
LREM in_progress 1 "task1"
```

## Set (множества)

### Базовые операции

```bash
# SADD - добавить элементы
SADD tags "redis" "database" "nosql"
# 3 (количество добавленных)

# SISMEMBER - проверить членство
SISMEMBER tags "redis"
# 1 (true) или 0 (false)

# SMEMBERS - все элементы
SMEMBERS tags
# ["redis", "database", "nosql"]

# SCARD - количество элементов
SCARD tags
# 3

# SREM - удалить элементы
SREM tags "nosql"
# 1 (количество удаленных)
```

### Случайные элементы

```bash
# SRANDMEMBER - случайный элемент (не удаляет)
SRANDMEMBER tags
# "redis"

# SRANDMEMBER с count
SRANDMEMBER tags 2
# ["redis", "database"]

# SPOP - случайный элемент (удаляет)
SPOP tags
# "database"

# SPOP с count
SPOP tags 2
```

### Операции с множествами

```bash
SADD set1 "a" "b" "c"
SADD set2 "b" "c" "d"

# SINTER - пересечение
SINTER set1 set2
# ["b", "c"]

# SINTERSTORE - сохранить результат
SINTERSTORE result set1 set2
# 2

# SUNION - объединение
SUNION set1 set2
# ["a", "b", "c", "d"]

# SUNIONSTORE - сохранить результат
SUNIONSTORE result set1 set2

# SDIFF - разность (в set1, но не в set2)
SDIFF set1 set2
# ["a"]

# SDIFFSTORE - сохранить результат
SDIFFSTORE result set1 set2

# SMOVE - переместить элемент
SMOVE set1 set2 "a"
```

### Сканирование

```bash
# SSCAN - итеративное сканирование
SSCAN tags 0 MATCH redis* COUNT 10
```

## Sorted Set (упорядоченные множества)

### Добавление и удаление

```bash
# ZADD - добавить элементы с баллами
ZADD leaderboard 100 "Alice"
ZADD leaderboard 200 "Bob" 150 "Charlie"
# 3 (количество добавленных)

# ZADD с опциями
ZADD leaderboard NX 300 "David"  # Только если не существует
ZADD leaderboard XX 250 "Alice"  # Только если существует
ZADD leaderboard GT 180 "Bob"    # Только если новый балл больше
ZADD leaderboard LT 90 "Alice"   # Только если новый балл меньше

# ZREM - удалить элементы
ZREM leaderboard "Alice"
# 1

# ZREMRANGEBYRANK - удалить по рангу
ZREMRANGEBYRANK leaderboard 0 0  # Удалить первый

# ZREMRANGEBYSCORE - удалить по баллу
ZREMRANGEBYSCORE leaderboard 0 100

# ZREMRANGEBYLEX - удалить лексикографически
ZREMRANGEBYLEX leaderboard [a [f
```

### Получение элементов

```bash
# ZRANGE - по рангу (возрастание)
ZRANGE leaderboard 0 -1
# ["Alice", "Charlie", "Bob"]

# С баллами
ZRANGE leaderboard 0 -1 WITHSCORES
# ["Alice", "100", "Charlie", "150", "Bob", "200"]

# ZREVRANGE - по рангу (убывание)
ZREVRANGE leaderboard 0 9
# Топ-10

# ZRANGEBYSCORE - по баллу (возрастание)
ZRANGEBYSCORE leaderboard 100 200
# ["Alice", "Charlie", "Bob"]

# С LIMIT
ZRANGEBYSCORE leaderboard -inf +inf LIMIT 0 10
# Первые 10

# ZREVRANGEBYSCORE - по баллу (убывание)
ZREVRANGEBYSCORE leaderboard 200 100

# ZRANGEBYLEX - лексикографически
ZRANGEBYLEX leaderboard [a [z
```

### Информация об элементах

```bash
# ZSCORE - получить балл
ZSCORE leaderboard "Bob"
# "200"

# ZRANK - ранг (позиция, возрастание)
ZRANK leaderboard "Bob"
# 2 (0-based)

# ZREVRANK - ранг (убывание)
ZREVRANK leaderboard "Bob"
# 0

# ZCARD - количество элементов
ZCARD leaderboard
# 3

# ZCOUNT - количество в диапазоне баллов
ZCOUNT leaderboard 100 200
# 3

# ZLEXCOUNT - лексикографический count
ZLEXCOUNT leaderboard [a [z
```

### Изменение баллов

```bash
# ZINCRBY - инкремент балла
ZINCRBY leaderboard 50 "Alice"
# "150" (новый балл)

# ZPOPMIN / ZPOPMAX - извлечь min/max
ZPOPMIN leaderboard
# ["Alice", "100"]

ZPOPMAX leaderboard 2
# ["Bob", "200", "Charlie", "150"]

# BZPOPMIN / BZPOPMAX - блокирующие версии
BZPOPMIN leaderboard 10
```

### Операции с множествами

```bash
# ZINTERSTORE - пересечение
ZADD set1 1 "a" 2 "b" 3 "c"
ZADD set2 4 "b" 5 "c" 6 "d"

ZINTERSTORE result 2 set1 set2
# result = ["b", "c"] с суммой баллов

# С весами
ZINTERSTORE result 2 set1 set2 WEIGHTS 2 3
# Баллы умножаются на веса

# С агрегацией
ZINTERSTORE result 2 set1 set2 AGGREGATE MAX
# MIN, MAX, SUM

# ZUNIONSTORE - объединение
ZUNIONSTORE result 2 set1 set2
```

### Сканирование

```bash
# ZSCAN - итеративное сканирование
ZSCAN leaderboard 0 MATCH A* COUNT 10
```

## Дополнительные типы данных

### Bitmaps

```bash
# SETBIT / GETBIT
SETBIT visits:2024-01-09 123 1  # User 123 посетил
GETBIT visits:2024-01-09 123
# 1

# BITCOUNT - количество единиц
BITCOUNT visits:2024-01-09
# 15 (посетителей)

# BITOP - битовые операции
BITOP AND result key1 key2
BITOP OR result key1 key2
BITOP XOR result key1 key2
BITOP NOT result key

# BITPOS - позиция первого бита
BITPOS visits:2024-01-09 1  # Первый визит
```

### HyperLogLog

Вероятностная структура для подсчета уникальных элементов.

```bash
# PFADD - добавить элементы
PFADD visitors "user1" "user2" "user3"
# 1 (приблизительное количество)

# PFCOUNT - подсчитать уникальные
PFCOUNT visitors
# ~3 (погрешность ~0.81%)

# PFMERGE - объединить
PFADD visitors1 "user1" "user2"
PFADD visitors2 "user2" "user3"
PFMERGE visitors_all visitors1 visitors2
PFCOUNT visitors_all
# ~3
```

**Применение:** Уникальные посетители, distinct count для больших данных.

### Geospatial (геопространственные)

```bash
# GEOADD - добавить координаты
GEOADD locations 37.617635 55.755814 "Moscow"
GEOADD locations 30.315868 59.939095 "Saint Petersburg"

# GEOPOS - получить координаты
GEOPOS locations "Moscow"
# [["37.617635", "55.755814"]]

# GEODIST - расстояние между точками
GEODIST locations "Moscow" "Saint Petersburg" km
# "634.3915"

# GEORADIUS - поиск в радиусе (deprecated)
GEORADIUS locations 37.6 55.7 100 km

# GEOSEARCH - поиск в радиусе (новый)
GEOSEARCH locations FROMLONLAT 37.6 55.7 BYRADIUS 100 km

# GEOSEARCHSTORE - сохранить результат
GEOSEARCHSTORE result locations FROMLONLAT 37.6 55.7 BYRADIUS 100 km
```

### Streams (потоки)

```bash
# XADD - добавить событие
XADD events * user "Alice" action "login"
# "1609459200000-0" (ID события)

# XREAD - читать события
XREAD STREAMS events 0
# [["events", [["1609459200000-0", ["user", "Alice", "action", "login"]]]]]

# XRANGE - диапазон событий
XRANGE events - +
# Все события

# XLEN - количество событий
XLEN events
# 1

# Consumer groups
XGROUP CREATE events mygroup 0
XREADGROUP GROUP mygroup consumer1 STREAMS events >

# XACK - подтвердить обработку
XACK events mygroup "1609459200000-0"
```

## Ключи и метаданные

### Работа с ключами

```bash
# EXISTS - проверить существование
EXISTS key1 key2 key3
# 2 (количество существующих)

# DEL - удалить ключи
DEL key1 key2
# 2 (количество удаленных)

# UNLINK - асинхронное удаление (для больших структур)
UNLINK huge_list

# KEYS - найти ключи (❌ НЕ использовать в production!)
KEYS user:*

# SCAN - итеративный поиск (✅ Используйте это)
SCAN 0 MATCH user:* COUNT 100
# ["17", ["user:1", "user:2"]]

# TYPE - тип значения
TYPE user:1
# "hash"

# RENAME - переименовать ключ
RENAME old_key new_key

# RENAMENX - переименовать если new_key не существует
RENAMENX old_key new_key
# 1 (успех) или 0 (new_key уже существует)

# MOVE - переместить в другую БД
MOVE key 1  # В БД номер 1

# RANDOMKEY - случайный ключ
RANDOMKEY

# DUMP / RESTORE - сериализация
DUMP key
# Бинарные данные
RESTORE new_key 0 "binary_data"
```

## Транзакции

```bash
# MULTI - начать транзакцию
MULTI
# OK

SET key1 "value1"
SET key2 "value2"
INCR counter

# EXEC - выполнить
EXEC
# [OK, OK, 1]

# DISCARD - отменить
DISCARD

# WATCH - оптимистичная блокировка
WATCH balance
balance = GET balance
MULTI
SET balance (balance - 100)
EXEC
# nil если balance изменился

# UNWATCH - снять наблюдение
UNWATCH
```

## Pub/Sub

```bash
# SUBSCRIBE - подписаться на каналы
SUBSCRIBE news sports

# PSUBSCRIBE - подписаться по паттерну
PSUBSCRIBE news:*

# PUBLISH - опубликовать сообщение
PUBLISH news "Breaking news!"
# 1 (количество получателей)

# UNSUBSCRIBE - отписаться
UNSUBSCRIBE news

# PUNSUBSCRIBE - отписаться от паттерна
PUNSUBSCRIBE news:*

# PUBSUB CHANNELS - список каналов
PUBSUB CHANNELS

# PUBSUB NUMSUB - количество подписчиков
PUBSUB NUMSUB news sports

# PUBSUB NUMPAT - количество паттерн-подписок
PUBSUB NUMPAT
```

## Скрипты (Lua)

```bash
# EVAL - выполнить Lua скрипт
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey "value"

# EVALSHA - выполнить кешированный скрипт
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# "sha1_hash"
EVALSHA "sha1_hash" 1 mykey

# SCRIPT EXISTS - проверить наличие скрипта
SCRIPT EXISTS sha1_hash1 sha1_hash2

# SCRIPT FLUSH - удалить все скрипты
SCRIPT FLUSH

# SCRIPT KILL - убить выполняющийся скрипт
SCRIPT KILL
```

**Пример атомарного INCR с ограничением:**

```bash
EVAL "
    local current = redis.call('GET', KEYS[1])
    if current == false then current = 0 else current = tonumber(current) end
    if current < tonumber(ARGV[1]) then
        return redis.call('INCR', KEYS[1])
    else
        return nil
    end
" 1 counter 100
```

## Административные команды

```bash
# INFO - информация о сервере
INFO
INFO stats
INFO memory

# CONFIG GET / SET - конфигурация
CONFIG GET maxmemory
CONFIG SET maxmemory 2gb

# DBSIZE - количество ключей
DBSIZE

# FLUSHDB - очистить текущую БД
FLUSHDB

# FLUSHALL - очистить все БД
FLUSHALL

# SAVE / BGSAVE - сохранить на диск
SAVE      # Синхронно
BGSAVE    # Асинхронно

# LASTSAVE - время последнего сохранения
LASTSAVE

# SHUTDOWN - остановить сервер
SHUTDOWN SAVE

# CLIENT LIST - список клиентов
CLIENT LIST

# CLIENT KILL - отключить клиента
CLIENT KILL 127.0.0.1:12345

# SLOWLOG - медленные запросы
SLOWLOG GET 10

# MONITOR - отладка (все команды в реальном времени)
MONITOR

# MEMORY USAGE - использование памяти ключом
MEMORY USAGE key

# MEMORY STATS - статистика памяти
MEMORY STATS
```

## Best Practices

1. ✅ Используйте SCAN вместо KEYS в production
2. ✅ Pipeline для множественных команд
3. ✅ Hash для объектов с полями
4. ✅ Sorted Set для рейтингов/лидербордов
5. ✅ List для очередей с BRPOP
6. ✅ Set для уникальных элементов/тегов
7. ✅ TTL для автоматической очистки
8. ❌ Избегайте очень больших ключей (> 1MB)
9. ✅ HyperLogLog для approximate distinct count
10. ✅ Lua скрипты для атомарных операций

## Связанные темы

- [[Redis - Key-Value хранилище]]
- [[Redis - Кэширование стратегии]]
- [[Типы СУБД - SQL vs NoSQL]]
- [[Go - Пакет sync]]
