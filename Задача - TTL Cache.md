# Задача - TTL Cache

Частая задача на собеседованиях. Проверяет понимание конкурентного доступа, горутин для фоновой очистки и управления временем жизни данных.

## Условие задачи

```
Реализовать кэш с автоматическим удалением элементов по истечении TTL.
Требования:
1. Set(key, value, ttl) — добавить элемент с временем жизни
2. Get(key) — получить элемент (если не истёк)
3. Delete(key) — удалить элемент
4. Потокобезопасность
5. Автоматическая очистка истёкших элементов
```

## Решение 1: С фоновой очисткой (goroutine)

```go
type item struct {
    value      interface{}
    expiration time.Time
}

type TTLCache struct {
    items map[string]item
    mu    sync.RWMutex
    stop  chan struct{}
}

func NewTTLCache(cleanupInterval time.Duration) *TTLCache {
    cache := &TTLCache{
        items: make(map[string]item),
        stop:  make(chan struct{}),
    }

    // Фоновая горутина для очистки
    go cache.cleanupLoop(cleanupInterval)

    return cache
}

func (c *TTLCache) Set(key string, value interface{}, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.items[key] = item{
        value:      value,
        expiration: time.Now().Add(ttl),
    }
}

func (c *TTLCache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    it, exists := c.items[key]
    if !exists {
        return nil, false
    }

    // Проверяем истёк ли TTL
    if time.Now().After(it.expiration) {
        return nil, false
    }

    return it.value, true
}

func (c *TTLCache) Delete(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    delete(c.items, key)
}

func (c *TTLCache) cleanupLoop(interval time.Duration) {
    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            c.cleanup()
        case <-c.stop:
            return
        }
    }
}

func (c *TTLCache) cleanup() {
    c.mu.Lock()
    defer c.mu.Unlock()

    now := time.Now()
    for key, it := range c.items {
        if now.After(it.expiration) {
            delete(c.items, key)
        }
    }
}

func (c *TTLCache) Stop() {
    close(c.stop)
}
```

## Решение 2: Ленивая очистка (без горутины)

```go
type TTLCacheLazy struct {
    items map[string]item
    mu    sync.RWMutex
}

func NewTTLCacheLazy() *TTLCacheLazy {
    return &TTLCacheLazy{
        items: make(map[string]item),
    }
}

func (c *TTLCacheLazy) Set(key string, value interface{}, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.items[key] = item{
        value:      value,
        expiration: time.Now().Add(ttl),
    }
}

func (c *TTLCacheLazy) Get(key string) (interface{}, bool) {
    c.mu.Lock() // Write lock для возможного удаления
    defer c.mu.Unlock()

    it, exists := c.items[key]
    if !exists {
        return nil, false
    }

    if time.Now().After(it.expiration) {
        // Ленивое удаление
        delete(c.items, key)
        return nil, false
    }

    return it.value, true
}
```

## Решение 3: С использованием heap для эффективной очистки

```go
import "container/heap"

type expirationHeap []struct {
    key        string
    expiration time.Time
}

func (h expirationHeap) Len() int            { return len(h) }
func (h expirationHeap) Less(i, j int) bool  { return h[i].expiration.Before(h[j].expiration) }
func (h expirationHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *expirationHeap) Push(x interface{}) { *h = append(*h, x.(struct{ key string; expiration time.Time })) }
func (h *expirationHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[0 : n-1]
    return x
}

type TTLCacheHeap struct {
    items   map[string]item
    expHeap expirationHeap
    mu      sync.Mutex
    stop    chan struct{}
}

func NewTTLCacheHeap() *TTLCacheHeap {
    cache := &TTLCacheHeap{
        items:   make(map[string]item),
        expHeap: make(expirationHeap, 0),
        stop:    make(chan struct{}),
    }

    go cache.cleanupLoop()
    return cache
}

func (c *TTLCacheHeap) Set(key string, value interface{}, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()

    exp := time.Now().Add(ttl)
    c.items[key] = item{value: value, expiration: exp}

    heap.Push(&c.expHeap, struct {
        key        string
        expiration time.Time
    }{key, exp})
}

func (c *TTLCacheHeap) cleanupLoop() {
    for {
        c.mu.Lock()

        if len(c.expHeap) == 0 {
            c.mu.Unlock()
            time.Sleep(time.Second)
            continue
        }

        // Смотрим на ближайший к истечению элемент
        next := c.expHeap[0]
        now := time.Now()

        if now.Before(next.expiration) {
            // Ещё рано — ждём
            waitTime := next.expiration.Sub(now)
            c.mu.Unlock()

            select {
            case <-time.After(waitTime):
            case <-c.stop:
                return
            }
            continue
        }

        // Удаляем истёкший
        heap.Pop(&c.expHeap)
        if it, exists := c.items[next.key]; exists {
            if it.expiration.Equal(next.expiration) {
                delete(c.items, next.key)
            }
        }
        c.mu.Unlock()
    }
}
```

## Решение 4: С использованием sync.Map

```go
type TTLCacheSyncMap struct {
    items sync.Map
    stop  chan struct{}
}

type itemWithExp struct {
    value      interface{}
    expiration time.Time
}

func NewTTLCacheSyncMap(cleanupInterval time.Duration) *TTLCacheSyncMap {
    cache := &TTLCacheSyncMap{
        stop: make(chan struct{}),
    }

    go func() {
        ticker := time.NewTicker(cleanupInterval)
        defer ticker.Stop()

        for {
            select {
            case <-ticker.C:
                now := time.Now()
                cache.items.Range(func(key, value interface{}) bool {
                    it := value.(itemWithExp)
                    if now.After(it.expiration) {
                        cache.items.Delete(key)
                    }
                    return true
                })
            case <-c.stop:
                return
            }
        }
    }()

    return cache
}

func (c *TTLCacheSyncMap) Set(key string, value interface{}, ttl time.Duration) {
    c.items.Store(key, itemWithExp{
        value:      value,
        expiration: time.Now().Add(ttl),
    })
}

func (c *TTLCacheSyncMap) Get(key string) (interface{}, bool) {
    val, exists := c.items.Load(key)
    if !exists {
        return nil, false
    }

    it := val.(itemWithExp)
    if time.Now().After(it.expiration) {
        c.items.Delete(key)
        return nil, false
    }

    return it.value, true
}
```

## Пример использования

```go
func main() {
    cache := NewTTLCache(time.Second) // Очистка каждую секунду
    defer cache.Stop()

    // Добавляем элементы
    cache.Set("user:1", "John", 5*time.Second)
    cache.Set("user:2", "Jane", 2*time.Second)

    // Читаем
    if val, ok := cache.Get("user:1"); ok {
        fmt.Println("user:1 =", val) // John
    }

    // Ждём истечения TTL
    time.Sleep(3 * time.Second)

    // user:2 истёк
    if _, ok := cache.Get("user:2"); !ok {
        fmt.Println("user:2 expired")
    }

    // user:1 ещё живой
    if val, ok := cache.Get("user:1"); ok {
        fmt.Println("user:1 still alive:", val)
    }
}
```

## Сравнение подходов

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| Фоновая горутина | Предсказуемое использование памяти | Нужно останавливать горутину |
| Ленивая очистка | Просто, нет горутин | Память может расти |
| Heap | Эффективная очистка O(log n) | Сложность реализации |
| sync.Map | Хорошо для read-heavy | Нет гарантий порядка очистки |

## Типичные ошибки

### Ошибка 1: Утечка горутины

```go
// ❌ НЕПРАВИЛЬНО: горутина никогда не остановится
func NewCache() *Cache {
    c := &Cache{}
    go func() {
        for { // Бесконечный цикл!
            time.Sleep(time.Second)
            c.cleanup()
        }
    }()
    return c
}

// ✅ ПРАВИЛЬНО: добавить канал stop
func (c *Cache) Stop() {
    close(c.stop)
}
```

### Ошибка 2: Гонка при проверке и удалении

```go
// ❌ НЕПРАВИЛЬНО: между RUnlock и Lock другая горутина может изменить items
func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    it, exists := c.items[key]
    c.mu.RUnlock()

    if time.Now().After(it.expiration) {
        c.mu.Lock()
        delete(c.items, key) // Может удалить новый элемент!
        c.mu.Unlock()
        return nil, false
    }
    // ...
}
```

## Вопросы с собеседований

### Вопрос 1: Когда использовать фоновую очистку, а когда ленивую?

**Ответ:**

**Фоновая очистка:**
- Важно ограничить потребление памяти
- Предсказуемое поведение
- Есть возможность корректно остановить

**Ленивая очистка:**
- Простота реализации
- Нет накладных расходов на горутину
- Допустимо временное превышение памяти

---

### Вопрос 2: Почему нельзя использовать RLock в Get при ленивой очистке?

**Ответ:**

При ленивой очистке Get может удалить элемент. Операция delete требует write lock. Использование RLock + последующий Lock создаёт гонку: между освобождением RLock и захватом Lock другая горутина может изменить состояние.

---

### Вопрос 3: Как реализовать callback при удалении?

**Ответ:**

```go
type TTLCache struct {
    // ...
    onEvict func(key string, value interface{})
}

func (c *TTLCache) cleanup() {
    c.mu.Lock()
    defer c.mu.Unlock()

    now := time.Now()
    for key, it := range c.items {
        if now.After(it.expiration) {
            if c.onEvict != nil {
                c.onEvict(key, it.value)
            }
            delete(c.items, key)
        }
    }
}
```

---

## Связанные темы

- [[Задача - LRU Cache]]
- [[Go - Mutex и RWMutex]]
- [[Go - Горутины (goroutines)]]
- [[Redis - Кэширование стратегии]]
- [[Go - Пакет time]]
