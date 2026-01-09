# Задача - LRU Cache

Реализовать LRU (Least Recently Used) кэш с O(1) операциями.

## Условие

Реализуйте структуру данных LRU cache с ограниченной вместимостью:

```go
type LRUCache struct {
    // Ваши поля
}

func Constructor(capacity int) LRUCache

func (c *LRUCache) Get(key int) int

func (c *LRUCache) Put(key int, value int)
```

**Требования:**
- `Get(key)` - вернуть значение ключа или -1 если не найден. O(1)
- `Put(key, value)` - добавить/обновить. Если превышен capacity, удалить LRU элемент. O(1)

## Примеры

```go
cache := Constructor(2)

cache.Put(1, 1)
cache.Put(2, 2)
cache.Get(1)        // 1
cache.Put(3, 3)     // Удаляется key=2 (LRU)
cache.Get(2)        // -1 (не найден)
cache.Put(4, 4)     // Удаляется key=1 (LRU)
cache.Get(1)        // -1 (не найден)
cache.Get(3)        // 3
cache.Get(4)        // 4
```

## Подход

**Структуры данных:**
1. **HashMap** - для O(1) доступа по ключу
2. **Doubly Linked List** - для O(1) перемещения/удаления

```
HashMap: key → *Node

Doubly Linked List (MRU → LRU):
head ↔ [key=3, val=30] ↔ [key=1, val=10] ↔ tail
       (Most Recent)      (Least Recent)
```

**Операции:**
- `Get(key)` → найти в map → переместить в начало
- `Put(key, val)` → если есть, обновить и в начало; если нет, добавить в начало
- Если размер > capacity → удалить tail (LRU)

## Решение

```go
type Node struct {
    key, value int
    prev, next *Node
}

type LRUCache struct {
    capacity int
    cache    map[int]*Node
    head     *Node  // Dummy head (MRU end)
    tail     *Node  // Dummy tail (LRU end)
}

func Constructor(capacity int) LRUCache {
    head := &Node{}
    tail := &Node{}
    head.next = tail
    tail.prev = head

    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*Node),
        head:     head,
        tail:     tail,
    }
}

func (c *LRUCache) Get(key int) int {
    if node, ok := c.cache[key]; ok {
        c.moveToHead(node)
        return node.value
    }
    return -1
}

func (c *LRUCache) Put(key int, value int) {
    if node, ok := c.cache[key]; ok {
        // Обновить существующий
        node.value = value
        c.moveToHead(node)
    } else {
        // Добавить новый
        newNode := &Node{key: key, value: value}
        c.cache[key] = newNode
        c.addToHead(newNode)

        if len(c.cache) > c.capacity {
            // Удалить LRU
            removed := c.removeTail()
            delete(c.cache, removed.key)
        }
    }
}

// Вспомогательные методы

func (c *LRUCache) addToHead(node *Node) {
    node.prev = c.head
    node.next = c.head.next

    c.head.next.prev = node
    c.head.next = node
}

func (c *LRUCache) removeNode(node *Node) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (c *LRUCache) moveToHead(node *Node) {
    c.removeNode(node)
    c.addToHead(node)
}

func (c *LRUCache) removeTail() *Node {
    node := c.tail.prev
    c.removeNode(node)
    return node
}
```

## Пример использования

```go
func main() {
    cache := Constructor(2)

    cache.Put(1, 10)
    cache.Put(2, 20)

    fmt.Println(cache.Get(1))  // 10, [1,2] → [1,2] (1 most recent)

    cache.Put(3, 30)           // [1,2,3] → [3,1] (удалили 2)

    fmt.Println(cache.Get(2))  // -1 (не найден)
    fmt.Println(cache.Get(3))  // 30

    cache.Put(4, 40)           // [3,1,4] → [4,3] (удалили 1)

    fmt.Println(cache.Get(1))  // -1
    fmt.Println(cache.Get(3))  // 30
    fmt.Println(cache.Get(4))  // 40
}
```

## Визуализация

### Начальное состояние

```
capacity = 2
cache = {}
head ↔ tail
```

### Put(1, 10)

```
cache = {1: *node1}
head ↔ [1:10] ↔ tail
```

### Put(2, 20)

```
cache = {1: *node1, 2: *node2}
head ↔ [2:20] ↔ [1:10] ↔ tail
       (MRU)    (LRU)
```

### Get(1)

```
cache = {1: *node1, 2: *node2}
head ↔ [1:10] ↔ [2:20] ↔ tail
       (MRU)    (LRU)
```

### Put(3, 30) - вытесняем 2

```
cache = {1: *node1, 3: *node3}
head ↔ [3:30] ↔ [1:10] ↔ tail
       (MRU)    (LRU)
```

## Сложность

- **Время:**
  - Get: O(1)
  - Put: O(1)

- **Память:** O(capacity)

## Альтернативная реализация (container/list)

```go
import "container/list"

type LRUCache struct {
    capacity int
    cache    map[int]*list.Element
    list     *list.List
}

type entry struct {
    key   int
    value int
}

func Constructor(capacity int) LRUCache {
    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*list.Element),
        list:     list.New(),
    }
}

func (c *LRUCache) Get(key int) int {
    if elem, ok := c.cache[key]; ok {
        c.list.MoveToFront(elem)
        return elem.Value.(*entry).value
    }
    return -1
}

func (c *LRUCache) Put(key int, value int) {
    if elem, ok := c.cache[key]; ok {
        c.list.MoveToFront(elem)
        elem.Value.(*entry).value = value
    } else {
        elem := c.list.PushFront(&entry{key, value})
        c.cache[key] = elem

        if c.list.Len() > c.capacity {
            back := c.list.Back()
            if back != nil {
                c.list.Remove(back)
                delete(c.cache, back.Value.(*entry).key)
            }
        }
    }
}
```

## Вариации задачи

### LFU Cache (Least Frequently Used)

Вытесняем элемент с наименьшей частотой использования.

```go
type LFUCache struct {
    capacity  int
    minFreq   int
    cache     map[int]*Node
    freqMap   map[int]*list.List  // freq → list of nodes
}

// Сложнее реализация, но тоже O(1)
```

### TTL Cache (Time To Live)

Элементы истекают через определённое время.

```go
type CacheEntry struct {
    value      int
    expireTime time.Time
}

type TTLCache struct {
    cache map[int]CacheEntry
    ttl   time.Duration
}

func (c *TTLCache) Get(key int) int {
    if entry, ok := c.cache[key]; ok {
        if time.Now().Before(entry.expireTime) {
            return entry.value
        }
        delete(c.cache, key)  // Истёк
    }
    return -1
}
```

## Применение в реальности

### HTTP Cache

```go
type HTTPCache struct {
    cache *LRUCache
}

func (h *HTTPCache) Get(url string) (*http.Response, error) {
    hash := hashURL(url)
    if cached := h.cache.Get(hash); cached != -1 {
        return deserializeResponse(cached), nil
    }

    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }

    h.cache.Put(hash, serializeResponse(resp))
    return resp, nil
}
```

### Database Query Cache

```go
type QueryCache struct {
    cache *LRUCache
}

func (qc *QueryCache) Query(sql string) ([]Row, error) {
    hash := hashQuery(sql)
    if result := qc.cache.Get(hash); result != -1 {
        return deserializeRows(result), nil
    }

    rows, err := db.Query(sql)
    if err != nil {
        return nil, err
    }

    qc.cache.Put(hash, serializeRows(rows))
    return rows, nil
}
```

## Best Practices

1. ✅ Dummy head/tail упрощают граничные случаи
2. ✅ HashMap для O(1) lookup
3. ✅ Doubly linked list для O(1) удаление/перемещение
4. ✅ container/list для production кода
5. ❌ Не забывайте обновлять оба: map и list
6. ❌ Thread-safety: добавьте sync.RWMutex для concurrent доступа

## Concurrent LRU Cache

```go
type ConcurrentLRUCache struct {
    mu    sync.RWMutex
    cache *LRUCache
}

func (c *ConcurrentLRUCache) Get(key int) int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.cache.Get(key)
}

func (c *ConcurrentLRUCache) Put(key int, value int) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.cache.Put(key, value)
}
```

## Связанные темы

- [[Связные списки]]
- [[HashMap - Реализация и особенности]]
- [[Go - Указатели и ссылки]]
- [[Go - Пакет sync]]
