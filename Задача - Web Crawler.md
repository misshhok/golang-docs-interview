# Задача - Web Crawler

Популярная задача на собеседованиях в Яндекс. Проверяет понимание BFS/DFS, горутин, дедупликации и ограничения параллелизма.

## Условие задачи

```
Написать веб-краулер, который обходит страницы начиная с заданного URL.
Требования:
1. Найти все ссылки на странице
2. Рекурсивно обойти найденные ссылки
3. Не посещать одну страницу дважды
4. Ограничить глубину обхода
5. Ограничить количество параллельных запросов
6. Обрабатывать только ссылки в пределах того же домена
```

## Решение 1: BFS с горутинами

```go
type Crawler struct {
    visited   map[string]bool
    mu        sync.Mutex
    maxDepth  int
    maxWorkers int
    domain    string
}

type Task struct {
    URL   string
    Depth int
}

func NewCrawler(maxDepth, maxWorkers int, domain string) *Crawler {
    return &Crawler{
        visited:    make(map[string]bool),
        maxDepth:   maxDepth,
        maxWorkers: maxWorkers,
        domain:     domain,
    }
}

func (c *Crawler) Crawl(startURL string) []string {
    var results []string
    var resultsMu sync.Mutex

    tasks := make(chan Task, 1000)
    var wg sync.WaitGroup

    // Запускаем воркеров
    for i := 0; i < c.maxWorkers; i++ {
        go func() {
            for task := range tasks {
                links := c.processURL(task.URL, task.Depth, tasks, &wg)

                resultsMu.Lock()
                results = append(results, links...)
                resultsMu.Unlock()
            }
        }()
    }

    // Начинаем с первого URL
    wg.Add(1)
    tasks <- Task{URL: startURL, Depth: 0}

    // Ждём завершения всех задач
    wg.Wait()
    close(tasks)

    return results
}

func (c *Crawler) processURL(url string, depth int, tasks chan<- Task, wg *sync.WaitGroup) []string {
    defer wg.Done()

    // Проверяем, не посещали ли уже
    c.mu.Lock()
    if c.visited[url] {
        c.mu.Unlock()
        return nil
    }
    c.visited[url] = true
    c.mu.Unlock()

    // Проверяем глубину
    if depth > c.maxDepth {
        return nil
    }

    // Получаем ссылки со страницы
    links, err := c.fetchLinks(url)
    if err != nil {
        return nil
    }

    var result []string
    result = append(result, url)

    // Добавляем новые задачи
    for _, link := range links {
        if c.isSameDomain(link) && !c.isVisited(link) {
            wg.Add(1)
            select {
            case tasks <- Task{URL: link, Depth: depth + 1}:
            default:
                wg.Done() // Очередь полна, пропускаем
            }
        }
    }

    return result
}

func (c *Crawler) fetchLinks(url string) ([]string, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var links []string

    tokenizer := html.NewTokenizer(resp.Body)
    for {
        tt := tokenizer.Next()
        switch tt {
        case html.ErrorToken:
            return links, nil
        case html.StartTagToken:
            token := tokenizer.Token()
            if token.Data == "a" {
                for _, attr := range token.Attr {
                    if attr.Key == "href" {
                        absoluteURL := c.toAbsoluteURL(url, attr.Val)
                        if absoluteURL != "" {
                            links = append(links, absoluteURL)
                        }
                    }
                }
            }
        }
    }
}

func (c *Crawler) isSameDomain(url string) bool {
    parsed, err := neturl.Parse(url)
    if err != nil {
        return false
    }
    return parsed.Host == c.domain
}

func (c *Crawler) isVisited(url string) bool {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.visited[url]
}

func (c *Crawler) toAbsoluteURL(base, href string) string {
    baseURL, err := neturl.Parse(base)
    if err != nil {
        return ""
    }
    hrefURL, err := neturl.Parse(href)
    if err != nil {
        return ""
    }
    return baseURL.ResolveReference(hrefURL).String()
}
```

## Решение 2: Рекурсивное с WaitGroup

```go
func CrawlRecursive(url string, depth int, fetcher Fetcher) []string {
    var visited sync.Map
    var results []string
    var resultsMu sync.Mutex
    var wg sync.WaitGroup

    var crawl func(url string, depth int)
    crawl = func(url string, depth int) {
        defer wg.Done()

        if depth <= 0 {
            return
        }

        // Атомарная проверка и установка
        if _, loaded := visited.LoadOrStore(url, true); loaded {
            return
        }

        resultsMu.Lock()
        results = append(results, url)
        resultsMu.Unlock()

        links, err := fetcher.Fetch(url)
        if err != nil {
            return
        }

        for _, link := range links {
            wg.Add(1)
            go crawl(link, depth-1)
        }
    }

    wg.Add(1)
    go crawl(url, depth)
    wg.Wait()

    return results
}

type Fetcher interface {
    Fetch(url string) ([]string, error)
}
```

## Решение 3: С context и отменой

```go
func CrawlWithContext(ctx context.Context, startURL string, maxDepth, maxWorkers int) ([]string, error) {
    visited := make(map[string]bool)
    var mu sync.Mutex
    var results []string

    sem := make(chan struct{}, maxWorkers)
    var wg sync.WaitGroup

    var crawl func(url string, depth int)
    crawl = func(url string, depth int) {
        defer wg.Done()

        // Проверяем отмену
        select {
        case <-ctx.Done():
            return
        default:
        }

        if depth > maxDepth {
            return
        }

        // Проверка visited
        mu.Lock()
        if visited[url] {
            mu.Unlock()
            return
        }
        visited[url] = true
        mu.Unlock()

        // Захватываем слот семафора
        select {
        case sem <- struct{}{}:
            defer func() { <-sem }()
        case <-ctx.Done():
            return
        }

        // Делаем запрос
        links, err := fetchLinksWithContext(ctx, url)
        if err != nil {
            return
        }

        mu.Lock()
        results = append(results, url)
        mu.Unlock()

        for _, link := range links {
            wg.Add(1)
            go crawl(link, depth+1)
        }
    }

    wg.Add(1)
    go crawl(startURL, 0)
    wg.Wait()

    return results, ctx.Err()
}

func fetchLinksWithContext(ctx context.Context, url string) ([]string, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    // Парсим HTML и извлекаем ссылки...
    return parseLinks(resp.Body, url)
}
```

## Пример использования

```go
func main() {
    crawler := NewCrawler(3, 10, "example.com")

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    results, err := CrawlWithContext(ctx, "https://example.com", 3, 10)
    if err != nil {
        log.Printf("Crawl interrupted: %v", err)
    }

    fmt.Printf("Found %d pages:\n", len(results))
    for _, url := range results {
        fmt.Println(url)
    }
}
```

## Типичные ошибки

### Ошибка 1: Гонка при проверке visited

```go
// ❌ НЕПРАВИЛЬНО: гонка между проверкой и записью
if !visited[url] {      // Чтение
    visited[url] = true // Запись — другая горутина может успеть между ними
    // ...
}

// ✅ ПРАВИЛЬНО: атомарная операция
if _, loaded := visited.LoadOrStore(url, true); loaded {
    return // Уже посещали
}
```

### Ошибка 2: Забыть ограничить глубину

```go
// ❌ НЕПРАВИЛЬНО: бесконечная рекурсия
func crawl(url string) {
    links := fetch(url)
    for _, link := range links {
        go crawl(link) // Никогда не остановится!
    }
}

// ✅ ПРАВИЛЬНО: с ограничением глубины
func crawl(url string, depth int) {
    if depth <= 0 {
        return
    }
    // ...
    go crawl(link, depth-1)
}
```

### Ошибка 3: Неконтролируемое создание горутин

```go
// ❌ НЕПРАВИЛЬНО: миллионы горутин
for _, link := range links {
    go crawl(link) // Без ограничений!
}

// ✅ ПРАВИЛЬНО: семафор
sem := make(chan struct{}, maxWorkers)
for _, link := range links {
    sem <- struct{}{} // Блокируемся если много горутин
    go func(l string) {
        defer func() { <-sem }()
        crawl(l)
    }(link)
}
```

## Сложность

| Аспект | Значение |
|--------|----------|
| Время | O(N) где N — количество страниц |
| Память | O(N) для visited + O(maxWorkers) горутин |
| Сетевые запросы | Ограничены maxWorkers |

## Вопросы с собеседований

### Вопрос 1: Как избежать повторного посещения страниц?

**Ответ:**

Использовать `sync.Map` или `map` с мьютексом для хранения посещённых URL.

```go
// sync.Map — атомарная проверка и добавление
if _, loaded := visited.LoadOrStore(url, true); loaded {
    return // Уже посещали
}
```

Важно: проверка и добавление должны быть атомарными!

---

### Вопрос 2: Как обрабатывать редиректы?

**Ответ:**

HTTP клиент Go по умолчанию следует редиректам. Можно настроить:

```go
client := &http.Client{
    CheckRedirect: func(req *http.Request, via []*http.Request) error {
        if len(via) >= 10 {
            return errors.New("too many redirects")
        }
        // Добавить финальный URL в visited
        return nil
    },
}
```

---

### Вопрос 3: Как обрабатывать robots.txt?

**Ответ:**

1. Перед началом краулинга скачать robots.txt
2. Распарсить правила (можно использовать библиотеку)
3. Перед каждым запросом проверять, разрешён ли URL

```go
import "github.com/temoto/robotstxt"

func (c *Crawler) isAllowed(url string) bool {
    return c.robots.TestAgent(url, "MyBot")
}
```

---

## Связанные темы

- [[BFS и DFS]]
- [[Go - Горутины (goroutines)]]
- [[Задача - Worker Pool]]
- [[Задача - Semaphore pattern]]
- [[Go - Context]]
- [[Go - Пакет net-http]]
