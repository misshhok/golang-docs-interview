# Задача - Concurrent Map

Классическая задача на собеседованиях. Проверяет понимание sync.RWMutex, sync.Map, шардирования и выбора правильного подхода для конкретного сценария.

## Условие задачи

```
Реализовать потокобезопасную map.
Требования:
1. Get(key) — получить значение
2. Set(key, value) — установить значение
3. Delete(key) — удалить значение
4. Потокобезопасность при конкурентном доступе
5. Высокая производительность при read-heavy нагрузке
```

## Решение 1: sync.RWMutex

```go
type SafeMap[K comparable, V any] struct {
    data map[K]V
    mu   sync.RWMutex
}

func NewSafeMap[K comparable, V any]() *SafeMap[K, V] {
    return &SafeMap[K, V]{
        data: make(map[K]V),
    }
}

func (m *SafeMap[K, V]) Get(key K) (V, bool) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    val, ok := m.data[key]
    return val, ok
}

func (m *SafeMap[K, V]) Set(key K, value V) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.data[key] = value
}

func (m *SafeMap[K, V]) Delete(key K) {
    m.mu.Lock()
    defer m.mu.Unlock()
    delete(m.data, key)
}

func (m *SafeMap[K, V]) Len() int {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return len(m.data)
}

func (m *SafeMap[K, V]) Keys() []K {
    m.mu.RLock()
    defer m.mu.RUnlock()

    keys := make([]K, 0, len(m.data))
    for k := range m.data {
        keys = append(keys, k)
    }
    return keys
}
```

## Решение 2: sync.Map (стандартная библиотека)

```go
// sync.Map оптимизирована для двух сценариев:
// 1. Ключи записываются один раз, но читаются много раз
// 2. Много горутин читают/записывают непересекающиеся наборы ключей

type CacheWithSyncMap struct {
    data sync.Map
}

func NewCacheWithSyncMap() *CacheWithSyncMap {
    return &CacheWithSyncMap{}
}

func (c *CacheWithSyncMap) Get(key string) (interface{}, bool) {
    return c.data.Load(key)
}

func (c *CacheWithSyncMap) Set(key string, value interface{}) {
    c.data.Store(key, value)
}

func (c *CacheWithSyncMap) Delete(key string) {
    c.data.Delete(key)
}

func (c *CacheWithSyncMap) GetOrSet(key string, value interface{}) (interface{}, bool) {
    // Возвращает существующее значение если есть, иначе сохраняет новое
    return c.data.LoadOrStore(key, value)
}

func (c *CacheWithSyncMap) Range(fn func(key, value interface{}) bool) {
    c.data.Range(fn)
}
```

## Решение 3: Шардированная Map (высокая производительность)

```go
const numShards = 32

type ShardedMap[V any] struct {
    shards [numShards]*shard[V]
}

type shard[V any] struct {
    data map[string]V
    mu   sync.RWMutex
}

func NewShardedMap[V any]() *ShardedMap[V] {
    m := &ShardedMap[V]{}
    for i := 0; i < numShards; i++ {
        m.shards[i] = &shard[V]{
            data: make(map[string]V),
        }
    }
    return m
}

func (m *ShardedMap[V]) getShard(key string) *shard[V] {
    // FNV-1a хеш для распределения по шардам
    hash := fnv32(key)
    return m.shards[hash%numShards]
}

func fnv32(key string) uint32 {
    hash := uint32(2166136261)
    for i := 0; i < len(key); i++ {
        hash ^= uint32(key[i])
        hash *= 16777619
    }
    return hash
}

func (m *ShardedMap[V]) Get(key string) (V, bool) {
    shard := m.getShard(key)
    shard.mu.RLock()
    defer shard.mu.RUnlock()
    val, ok := shard.data[key]
    return val, ok
}

func (m *ShardedMap[V]) Set(key string, value V) {
    shard := m.getShard(key)
    shard.mu.Lock()
    defer shard.mu.Unlock()
    shard.data[key] = value
}

func (m *ShardedMap[V]) Delete(key string) {
    shard := m.getShard(key)
    shard.mu.Lock()
    defer shard.mu.Unlock()
    delete(shard.data, key)
}

func (m *ShardedMap[V]) Len() int {
    total := 0
    for i := 0; i < numShards; i++ {
        m.shards[i].mu.RLock()
        total += len(m.shards[i].data)
        m.shards[i].mu.RUnlock()
    }
    return total
}
```

## Решение 4: С атомарными операциями (Copy-on-Write)

```go
type COWMap[K comparable, V any] struct {
    data atomic.Value // хранит map[K]V
}

func NewCOWMap[K comparable, V any]() *COWMap[K, V] {
    m := &COWMap[K, V]{}
    m.data.Store(make(map[K]V))
    return m
}

func (m *COWMap[K, V]) Get(key K) (V, bool) {
    data := m.data.Load().(map[K]V)
    val, ok := data[key]
    return val, ok
}

func (m *COWMap[K, V]) Set(key K, value V) {
    for {
        oldData := m.data.Load().(map[K]V)

        // Создаём копию
        newData := make(map[K]V, len(oldData)+1)
        for k, v := range oldData {
            newData[k] = v
        }
        newData[key] = value

        // Атомарно заменяем (CAS паттерн)
        if m.data.CompareAndSwap(oldData, newData) {
            return
        }
        // Если не удалось — кто-то изменил, пробуем снова
    }
}

func (m *COWMap[K, V]) Delete(key K) {
    for {
        oldData := m.data.Load().(map[K]V)

        newData := make(map[K]V, len(oldData))
        for k, v := range oldData {
            if k != key {
                newData[k] = v
            }
        }

        if m.data.CompareAndSwap(oldData, newData) {
            return
        }
    }
}
```

## Решение 5: С callback и расширенным API

```go
type ConcurrentMap[K comparable, V any] struct {
    data map[K]V
    mu   sync.RWMutex
}

func NewConcurrentMap[K comparable, V any]() *ConcurrentMap[K, V] {
    return &ConcurrentMap[K, V]{
        data: make(map[K]V),
    }
}

// GetOrCompute — атомарно получает или вычисляет значение
func (m *ConcurrentMap[K, V]) GetOrCompute(key K, compute func() V) V {
    // Сначала пробуем читать
    m.mu.RLock()
    val, ok := m.data[key]
    m.mu.RUnlock()

    if ok {
        return val
    }

    // Не нашли — захватываем write lock
    m.mu.Lock()
    defer m.mu.Unlock()

    // Double-check: возможно кто-то уже добавил
    if val, ok := m.data[key]; ok {
        return val
    }

    // Вычисляем и сохраняем
    val = compute()
    m.data[key] = val
    return val
}

// Update — атомарно обновляет значение
func (m *ConcurrentMap[K, V]) Update(key K, fn func(V, bool) V) V {
    m.mu.Lock()
    defer m.mu.Unlock()

    oldVal, exists := m.data[key]
    newVal := fn(oldVal, exists)
    m.data[key] = newVal
    return newVal
}

// ForEach — итерация со снимком
func (m *ConcurrentMap[K, V]) ForEach(fn func(K, V) bool) {
    m.mu.RLock()
    // Создаём снимок ключей
    keys := make([]K, 0, len(m.data))
    for k := range m.data {
        keys = append(keys, k)
    }
    m.mu.RUnlock()

    for _, key := range keys {
        m.mu.RLock()
        val, ok := m.data[key]
        m.mu.RUnlock()

        if ok && !fn(key, val) {
            break
        }
    }
}
```

## Пример использования

```go
func main() {
    // RWMutex версия
    cache := NewSafeMap[string, int]()

    var wg sync.WaitGroup

    // Писатели
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            for j := 0; j < 100; j++ {
                cache.Set(fmt.Sprintf("key%d", j), n*100+j)
            }
        }(i)
    }

    // Читатели
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for j := 0; j < 100; j++ {
                cache.Get(fmt.Sprintf("key%d", j))
            }
        }()
    }

    wg.Wait()
    fmt.Printf("Final size: %d\n", cache.Len())
}
```

## Бенчмарк сравнения

```go
func BenchmarkRWMutexMap(b *testing.B) {
    m := NewSafeMap[string, int]()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            m.Set("key", 1)
            m.Get("key")
        }
    })
}

func BenchmarkSyncMap(b *testing.B) {
    var m sync.Map
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            m.Store("key", 1)
            m.Load("key")
        }
    })
}

func BenchmarkShardedMap(b *testing.B) {
    m := NewShardedMap[int]()
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            m.Set("key", 1)
            m.Get("key")
        }
    })
}

/*
Типичные результаты (read-heavy, 90% reads):
BenchmarkRWMutexMap-8    5000000    300 ns/op
BenchmarkSyncMap-8      10000000    150 ns/op
BenchmarkShardedMap-8   15000000    100 ns/op

Write-heavy (50% writes):
BenchmarkRWMutexMap-8    2000000    600 ns/op
BenchmarkSyncMap-8       3000000    500 ns/op
BenchmarkShardedMap-8   10000000    120 ns/op
*/
```

## Сравнение подходов

| Подход | Read | Write | Память | Когда использовать |
|--------|------|-------|--------|-------------------|
| sync.RWMutex | Средне | Средне | Минимум | Универсальный случай |
| sync.Map | Быстро | Медленно | Много | Read-heavy, stable keys |
| Sharded | Быстро | Быстро | Средне | Высокая нагрузка |
| COW | Очень быстро | Очень медленно | Много | 99% reads |

## Типичные ошибки

### Ошибка 1: Не использовать defer

```go
// ❌ НЕПРАВИЛЬНО: при панике lock не освободится
func (m *SafeMap) Get(key string) int {
    m.mu.RLock()
    val := m.data[key]
    m.mu.RUnlock() // Не выполнится при панике
    return val
}

// ✅ ПРАВИЛЬНО: defer гарантирует unlock
func (m *SafeMap) Get(key string) int {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return m.data[key]
}
```

### Ошибка 2: Lock внутри другого Lock

```go
// ❌ НЕПРАВИЛЬНО: Deadlock!
func (m *SafeMap) Transfer(from, to string, amount int) {
    m.mu.Lock()
    defer m.mu.Unlock()

    fromVal := m.Get(from) // Get тоже берёт RLock — deadlock!
    // ...
}

// ✅ ПРАВИЛЬНО: одна критическая секция
func (m *SafeMap) Transfer(from, to string, amount int) {
    m.mu.Lock()
    defer m.mu.Unlock()

    fromVal := m.data[from] // Прямой доступ без дополнительных локов
    m.data[from] = fromVal - amount
    m.data[to] = m.data[to] + amount
}
```

### Ошибка 3: Возврат внутренних данных

```go
// ❌ НЕПРАВИЛЬНО: возвращаем ссылку на внутренний slice
func (m *SafeMap) GetUsers() []User {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return m.data["users"].([]User) // Внешний код может изменить!
}

// ✅ ПРАВИЛЬНО: возвращаем копию
func (m *SafeMap) GetUsers() []User {
    m.mu.RLock()
    defer m.mu.RUnlock()

    users := m.data["users"].([]User)
    result := make([]User, len(users))
    copy(result, users)
    return result
}
```

### Ошибка 4: Race при check-then-act

```go
// ❌ НЕПРАВИЛЬНО: гонка между проверкой и записью
func (m *SafeMap) SetIfNotExists(key string, val int) bool {
    if _, ok := m.Get(key); ok { // RLock
        return false
    }
    m.Set(key, val) // Lock — между ними другая горутина может записать!
    return true
}

// ✅ ПРАВИЛЬНО: атомарная операция
func (m *SafeMap) SetIfNotExists(key string, val int) bool {
    m.mu.Lock()
    defer m.mu.Unlock()

    if _, ok := m.data[key]; ok {
        return false
    }
    m.data[key] = val
    return true
}
```

## Когда что использовать

```
┌─────────────────────────────────────────────────────────┐
│                    Выбор реализации                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Мало данных, простой случай?                           │
│      ↓ Да                                               │
│  → sync.RWMutex                                         │
│                                                         │
│  Ключи записываются редко, читаются часто?              │
│      ↓ Да                                               │
│  → sync.Map                                             │
│                                                         │
│  Высокая конкурентность, много горутин?                 │
│      ↓ Да                                               │
│  → Sharded Map                                          │
│                                                         │
│  99%+ операций — чтение?                                │
│      ↓ Да                                               │
│  → Copy-on-Write (atomic.Value)                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## Вопросы с собеседований

### Вопрос 1: Почему sync.Map не всегда лучше RWMutex?

<details>
<summary>Ответ</summary>

sync.Map оптимизирована для специфических случаев:
1. Ключи записываются один раз, читаются много раз
2. Горутины работают с непересекающимися наборами ключей

В других случаях sync.Map может быть медленнее из-за:
- Двух внутренних map (read/dirty)
- Атомарных операций вместо mutex
- Больший расход памяти

</details>

### Вопрос 2: Как выбрать количество шардов?

<details>
<summary>Ответ</summary>

Эмпирические правила:
- **GOMAXPROCS * 4-8** — хороший стартовый вариант
- **Степень двойки** (16, 32, 64) — быстрее операция mod
- **Бенчмарк** — измерить на реальной нагрузке

Слишком мало шардов — много конкуренции
Слишком много — overhead на память и хеширование

</details>

### Вопрос 3: Можно ли итерировать по concurrent map?

<details>
<summary>Ответ</summary>

Да, но есть нюансы:

1. **sync.Map.Range** — итерация "best effort", не гарантирует консистентность
2. **RWMutex** — держать RLock во время итерации блокирует записи
3. **Snapshot** — создать копию и итерировать по ней

```go
// Snapshot подход
func (m *SafeMap) Snapshot() map[string]int {
    m.mu.RLock()
    defer m.mu.RUnlock()

    snapshot := make(map[string]int, len(m.data))
    for k, v := range m.data {
        snapshot[k] = v
    }
    return snapshot
}
```

</details>

## Связанные темы

- [[Go - Mutex и RWMutex]]
- [[Go - Атомарные операции]]
- [[Go - sync.Map]]
- [[Go - Внутреннее устройство map]]
- [[Go - Race Condition и Data Race]]
- [[Задача - LRU Cache]]
- [[Задача - TTL Cache]]
