# План для Sonnet агентов - База знаний Golang собеседований

**Цель:** Завершить создание полной базы знаний для подготовки к Senior Golang собеседованиям в российских компаниях (Яндекс, Авито, Т-Банк, Озон).

**Язык заметок:** Русский
**Формат:** Obsidian Markdown с wiki-links `[[Название заметки]]`

---

## Общий контекст

Это Obsidian vault с заметками для подготовки к техническому интервью. Уже создано 91 заметка (Golang Core, алгоритмы, базы данных, микросервисы, Docker). Нужно создать ещё ~63 заметки.

**Структура каждой заметки:**
1. Заголовок H1 с названием темы
2. Краткое введение (1-2 предложения)
3. Основное содержание с примерами кода на Go
4. Практические примеры/задачи где уместно
5. Раздел "Связанные темы" со ссылками на другие заметки

---

## МОДУЛЬ A: Задачи на Concurrency (может выполнять Инстанс 1)

**Контекст:** Concurrency - самая важная тема на Golang собеседованиях. Всегда спрашивают!

### A1. Go - Затенение переменных (Shadowing).md
**Что включить:**
- Определение shadowing - когда переменная во внутреннем scope "перекрывает" внешнюю
- Пример с `:=` в if/for блоках
- Классическая ловушка с `err` в цепочке вызовов
- Как go vet помогает найти shadowing
- Код с вопросом "что выведет?" и ответом

```go
// Пример для заметки
x := 1
if true {
    x := 2  // shadowing! новая переменная
    fmt.Println(x) // 2
}
fmt.Println(x) // 1 (оригинальная)
```

**Связи:** `[[Go - Переменные и константы]]`, `[[Go - Указатели - Подводные камни]]`

---

### A2. Go - Race Condition и Data Race.md
**Что включить:**
- Разница между race condition и data race
- Data race: одновременный доступ к памяти (один пишет)
- Race condition: логическая ошибка из-за порядка выполнения
- Пример data race с map (panic: concurrent map writes)
- Как запустить `go run -race`
- Способы исправления: mutex, channels, atomic

```go
// Data race пример
var counter int
for i := 0; i < 1000; i++ {
    go func() { counter++ }()
}
```

**Связи:** `[[Go - Mutex и RWMutex]]`, `[[Go - Атомарные операции]]`, `[[Go - Deadlock]]`

---

### A3. Go - Deadlock.md
**Что включить:**
- Что такое deadlock (взаимная блокировка)
- 4 условия Coffman (mutual exclusion, hold and wait, no preemption, circular wait)
- Примеры deadlock в Go:
  - Небуферизированный канал без получателя
  - Двойная блокировка mutex
  - Циклическая зависимость горутин
- Сообщение Go: "fatal error: all goroutines are asleep - deadlock!"
- Как избежать

```go
// Deadlock пример
ch := make(chan int)
ch <- 1  // deadlock! никто не читает
```

**Связи:** `[[Go - Каналы (channels)]]`, `[[Go - Mutex и RWMutex]]`, `[[Go - Race Condition и Data Race]]`

---

### A4. Задача - Worker Pool.md
**Что включить:**
- Зачем нужен (ограничение параллелизма, переиспользование горутин)
- Компоненты: jobs channel, results channel, workers
- Полная реализация с комментариями
- Варианты: с graceful shutdown, с rate limiting
- Вопросы с собеседований про worker pool

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        results <- j * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // Запускаем 3 воркера
    for w := 1; w <= 3; w++ {
        go worker(w, jobs, results)
    }

    // Отправляем задачи
    for j := 1; j <= 9; j++ {
        jobs <- j
    }
    close(jobs)

    // Собираем результаты
    for a := 1; a <= 9; a++ {
        <-results
    }
}
```

**Связи:** `[[Go - Горутины (goroutines)]]`, `[[Go - Каналы (channels)]]`, `[[Задача - Semaphore pattern]]`

---

### A5. Задача - Pipeline pattern.md
**Что включить:**
- Паттерн конвейера: каждая стадия - горутина с входным и выходным каналом
- Применение: обработка потоков данных, ETL
- Пример: генератор -> фильтр -> трансформер -> потребитель
- Как корректно закрывать каналы в pipeline

**Связи:** `[[Go - Каналы (channels)]]`, `[[Задача - Fan-in Fan-out]]`

---

### A6. Задача - Fan-in Fan-out.md
**Что включить:**
- Fan-out: один источник -> много обработчиков
- Fan-in: много источников -> один получатель
- Реализация fan-in через select или отдельную горутину
- Применение: параллельная обработка с объединением результатов

**Связи:** `[[Задача - Pipeline pattern]]`, `[[Задача - Worker Pool]]`

---

### A7. Задача - Graceful Shutdown.md
**Что включить:**
- Зачем нужен (не потерять данные, завершить транзакции)
- Использование context.WithCancel
- Обработка сигналов OS (signal.Notify)
- Паттерн с done channel
- Timeout на завершение

**Связи:** `[[Go - Context]]`, `[[Go - Горутины (goroutines)]]`, `[[Задача - Worker Pool]]`

---

### A8. Задача - Semaphore pattern.md
**Что включить:**
- Семафор через буферизированный канал
- Ограничение количества одновременных операций
- Пример: ограничение HTTP запросов

```go
sem := make(chan struct{}, 10) // макс 10 одновременно

for _, url := range urls {
    sem <- struct{}{} // захватываем слот
    go func(url string) {
        defer func() { <-sem }() // освобождаем слот
        fetch(url)
    }(url)
}
```

**Связи:** `[[Задача - Worker Pool]]`, `[[Задача - Rate Limiter]]`

---

### A9. Задача - Timeout pattern.md
**Что включить:**
- Таймаут через select + time.After
- Таймаут через context.WithTimeout
- Разница подходов
- Утечки горутин при неправильном таймауте

**Связи:** `[[Go - Select statement]]`, `[[Go - Context]]`, `[[Go - Пакет time]]`

---

## МОДУЛЬ B: Алгоритмические задачи (может выполнять Инстанс 2)

**Контекст:** Классические задачи с LeetCode, которые дают на собеседованиях.

### B1. Задача - Two Sum.md
**Что включить:**
- Условие: найти два числа, дающих target
- Brute force O(n²) vs HashMap O(n)
- Полное решение на Go с объяснением
- Вариации: отсортированный массив, three sum

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int)
    for i, num := range nums {
        if j, ok := seen[target-num]; ok {
            return []int{j, i}
        }
        seen[num] = i
    }
    return nil
}
```

**Связи:** `[[Go - Карты (maps)]]`, `[[HashMap - Реализация и особенности]]`, `[[Алгоритмическая сложность (Big O)]]`

---

### B2. Задача - Valid Parentheses.md
**Что включить:**
- Условие: проверить правильность скобок ()[]{}
- Решение через стек
- Пошаговое объяснение алгоритма

**Связи:** `[[Стек и очередь]]`, `[[Алгоритмическая сложность (Big O)]]`

---

### B3. Задача - Merge Intervals.md
**Что включить:**
- Условие: объединить пересекающиеся интервалы
- Сортировка + один проход
- Сложность O(n log n)

**Связи:** `[[Сортировки - Быстрая, слиянием, пузырьком]]`

---

### B4. Задача - Group Anagrams.md
**Что включить:**
- Группировка слов-анаграмм
- Ключ: отсортированная строка или счетчик букв
- Использование map

**Связи:** `[[Go - Карты (maps)]]`, `[[Алгоритмы на строках]]`

---

### B5. Задача - Find Duplicates.md
**Что включить:**
- Поиск дубликатов в массиве
- Через HashSet (map[int]struct{})
- Через сортировку
- Floyd's cycle detection для O(1) памяти

**Связи:** `[[HashSet]]`, `[[Go - Карты (maps)]]`

---

### B6. Задача - Maximum Subarray.md
**Что включить:**
- Kadane's algorithm
- Объяснение почему работает
- Сложность O(n)

**Связи:** `[[Динамическое программирование - Основы]]`, `[[Алгоритмическая сложность (Big O)]]`

---

### B7. Задача - Climbing Stairs.md
**Что включить:**
- Базовая задача на DP
- Связь с числами Фибоначчи
- Решения: рекурсия, мемоизация, итеративное

**Связи:** `[[Динамическое программирование - Основы]]`

---

### B8. Задача - Implement Queue using Stacks.md
**Что включить:**
- Два стека для реализации очереди
- Амортизированная O(1)

**Связи:** `[[Стек и очередь]]`, `[[Амортизированная сложность]]`

---

### B9. Задача - Palindrome Check.md
**Что включить:**
- Два указателя с концов
- Обработка регистра и не-буквенных символов

**Связи:** `[[Два указателя (Two Pointers)]]`, `[[Go - Строки и UTF-8]]`

---

### B10. Задача - Container With Most Water.md
**Что включить:**
- Два указателя
- Почему двигаем меньший

**Связи:** `[[Два указателя (Two Pointers)]]`

---

## МОДУЛЬ C: Тестирование (может выполнять Инстанс 3)

### C1. Go - Unit тестирование.md
**Что включить:**
- Пакет testing
- Naming convention: `TestXxx`, `_test.go`
- t.Run для подтестов
- t.Error vs t.Fatal
- go test флаги (-v, -run, -cover)

**Связи:** `[[Go - Table-driven тесты]]`, `[[Go - Моки и стабы]]`

---

### C2. Go - Table-driven тесты.md
**Что включить:**
- Паттерн table-driven
- Структура тестовых случаев
- Параллельные тесты с t.Parallel()

**Связи:** `[[Go - Unit тестирование]]`

---

### C3. Go - Моки и стабы.md
**Что включить:**
- Интерфейсы для тестируемости
- Ручные моки
- Библиотеки: gomock, testify/mock
- Dependency injection

**Связи:** `[[Go - Интерфейсы]]`, `[[Go - Unit тестирование]]`

---

### C4. E2E тестирование.md
**Что включить:**
- Что такое E2E
- Тестирование HTTP с httptest
- Тестирование с базой данных (testcontainers)
- Fixtures и seeds

**Связи:** `[[Go - Unit тестирование]]`, `[[Go - Пакет net-http]]`

---

### C5. Test Coverage.md
**Что включить:**
- go test -cover
- go tool cover -html
- Метрики покрытия
- Что должно покрываться

**Связи:** `[[Go - Unit тестирование]]`

---

## МОДУЛЬ D: Git (может выполнять Инстанс 3)

### D1. Git - Основные команды.md
**Что включить:**
- init, clone, add, commit, push, pull
- status, log, diff
- .gitignore

**Связи:** `[[Git - Ветвление и слияние]]`

---

### D2. Git - Ветвление и слияние.md
**Что включить:**
- branch, checkout, switch
- merge (fast-forward, 3-way)
- Разрешение конфликтов

**Связи:** `[[Git - Основные команды]]`, `[[Git - Rebase vs Merge]]`

---

### D3. Git - Gitflow и trunk-based development.md
**Что включить:**
- Gitflow: main, develop, feature, release, hotfix
- Trunk-based: короткие ветки, feature flags
- Когда что использовать

**Связи:** `[[Git - Ветвление и слияние]]`, `[[CI-CD - Основные концепции]]`

---

### D4. Git - Rebase vs Merge.md
**Что включить:**
- Как работает rebase
- Когда использовать rebase (линейная история)
- Когда использовать merge (сохранить историю)
- Golden rule: не rebase публичные ветки

**Связи:** `[[Git - Ветвление и слияние]]`

---

### D5. Git - Cherry-pick и работа с историей.md
**Что включить:**
- cherry-pick
- reset (soft, mixed, hard)
- revert
- reflog для восстановления

**Связи:** `[[Git - Основные команды]]`

---

## МОДУЛЬ E: Аутентификация (может выполнять Инстанс 4)

### E1. OAuth 2.0.md
**Что включить:**
- Роли: Resource Owner, Client, Authorization Server, Resource Server
- Grant types: Authorization Code, Client Credentials, etc.
- Access Token и Refresh Token
- Scopes

**Связи:** `[[JWT (JSON Web Tokens)]]`, `[[Безопасность API]]`

---

### E2. JWT (JSON Web Tokens).md
**Что включить:**
- Структура: header.payload.signature
- Алгоритмы: HS256, RS256
- Claims: iss, sub, exp, iat
- Валидация токена
- Хранение (cookie vs localStorage)

**Связи:** `[[OAuth 2.0]]`, `[[Безопасность API]]`

---

### E3. Basic Authentication.md
**Что включить:**
- Формат: Authorization: Basic base64(user:password)
- Когда использовать
- Проблемы безопасности
- Реализация в Go

**Связи:** `[[HTTP протокол]]`, `[[Безопасность API]]`

---

### E4. Session-based аутентификация.md
**Что включить:**
- Как работают сессии
- Session ID в cookie
- Серверное хранение (Redis, DB)
- Сравнение с JWT

**Связи:** `[[Redis - Key-Value хранилище]]`, `[[JWT (JSON Web Tokens)]]`

---

### E5. HTTPS и TLS.md
**Что включить:**
- Как работает TLS handshake
- Сертификаты
- mTLS (mutual TLS)
- Настройка в Go

**Связи:** `[[Go - TLS и HTTPS]]`, `[[HTTP протокол]]`

---

### E6. Безопасность API.md
**Что включить:**
- OWASP Top 10
- Rate limiting
- Input validation
- CORS
- SQL injection prevention

**Связи:** `[[REST API - Дизайн и best practices]]`, `[[JWT (JSON Web Tokens)]]`

---

## МОДУЛЬ F: DevOps - Kubernetes (может выполнять Инстанс 5)

### F1. Kubernetes - Основы.md
**Что включить:**
- Что такое K8s и зачем
- Архитектура: Master (API Server, etcd, Scheduler, Controller Manager) + Nodes (kubelet, kube-proxy)
- kubectl основные команды

**Связи:** `[[Docker - Основы]]`, `[[Kubernetes - Pods и Deployments]]`

---

### F2. Kubernetes - Pods и Deployments.md
**Что включить:**
- Pod: минимальная единица
- ReplicaSet
- Deployment: rolling updates, rollback
- YAML манифесты

**Связи:** `[[Kubernetes - Основы]]`, `[[Kubernetes - Services и Ingress]]`

---

### F3. Kubernetes - Services и Ingress.md
**Что включить:**
- Service types: ClusterIP, NodePort, LoadBalancer
- Service discovery
- Ingress для HTTP маршрутизации
- Ingress Controllers

**Связи:** `[[Kubernetes - Pods и Deployments]]`, `[[HTTP протокол]]`

---

### F4. Kubernetes - ConfigMaps и Secrets.md
**Что включить:**
- ConfigMap для конфигурации
- Secrets для чувствительных данных
- Монтирование как volume или env
- Шифрование secrets

**Связи:** `[[Kubernetes - Pods и Deployments]]`, `[[Переменные среды и .env]]`

---

## МОДУЛЬ G: DevOps - CI/CD и мониторинг (может выполнять Инстанс 5)

### G1. CI-CD - Основные концепции.md
**Что включить:**
- CI: автоматическая сборка, тесты, линтеры
- CD: Continuous Delivery vs Deployment
- Stages: build, test, deploy
- Артефакты

**Связи:** `[[GitLab CI-CD]]`, `[[GitHub Actions]]`

---

### G2. GitLab CI-CD.md
**Что включить:**
- .gitlab-ci.yml структура
- Jobs, stages, artifacts
- Runners
- Environments

**Связи:** `[[CI-CD - Основные концепции]]`

---

### G3. GitHub Actions.md
**Что включить:**
- Workflow файлы
- Events, jobs, steps
- Actions marketplace
- Secrets

**Связи:** `[[CI-CD - Основные концепции]]`

---

### G4. Метрики и наблюдаемость.md
**Что включить:**
- Three pillars: Metrics, Logs, Traces
- RED метрики (Rate, Errors, Duration)
- USE метрики (Utilization, Saturation, Errors)
- SLI, SLO, SLA

**Связи:** `[[Prometheus]]`, `[[Grafana]]`, `[[Логирование - Best practices]]`

---

### G5. Prometheus.md
**Что включить:**
- Pull model
- Типы метрик: Counter, Gauge, Histogram, Summary
- PromQL basics
- Alerting

**Связи:** `[[Метрики и наблюдаемость]]`, `[[Grafana]]`

---

### G6. Grafana.md
**Что включить:**
- Dashboards
- Data sources
- Panels и визуализации
- Alerting

**Связи:** `[[Prometheus]]`, `[[Метрики и наблюдаемость]]`

---

### G7. Логирование - Best practices.md
**Что включить:**
- Structured logging (JSON)
- Log levels
- Correlation IDs
- Популярные библиотеки: zap, zerolog

**Связи:** `[[Метрики и наблюдаемость]]`

---

## МОДУЛЬ H: Фреймворки Go (может выполнять Инстанс 6)

### H1. Gin - Web фреймворк.md
**Что включить:**
- Routing
- Middleware
- Binding и валидация
- Пример CRUD API

**Связи:** `[[Go - Пакет net-http]]`, `[[REST API - Дизайн и best practices]]`

---

### H2. Gorilla Mux.md
**Что включить:**
- Router
- Path variables
- Middleware
- Сравнение с net/http

**Связи:** `[[Go - Пакет net-http]]`

---

### H3. Echo.md
**Что включить:**
- Features
- Middleware
- Binding
- Сравнение с Gin

**Связи:** `[[Gin - Web фреймворк]]`

---

### H4. GORM - ORM для Go.md
**Что включить:**
- Models и migrations
- CRUD операции
- Relations
- Hooks
- Transactions

**Связи:** `[[PostgreSQL - Основы]]`, `[[Go - Структуры (struct)]]`

---

### H5. sqlx.md
**Что включить:**
- Расширение database/sql
- StructScan, NamedQuery
- Когда использовать вместо GORM

**Связи:** `[[GORM - ORM для Go]]`, `[[PostgreSQL - Основы]]`

---

### H6. Chi.md
**Что включить:**
- Lightweight router
- Middleware
- Совместимость с net/http

**Связи:** `[[Go - Пакет net-http]]`, `[[Gorilla Mux]]`

---

## МОДУЛЬ I: Паттерны (может выполнять Инстанс 6)

### I1. Паттерны проектирования в Go.md
**Что включить:**
- Особенности Go: нет классов, есть интерфейсы
- Композиция вместо наследования
- Категории: порождающие, структурные, поведенческие

**Связи:** `[[Go - Интерфейсы]]`, `[[Singleton, Factory, Builder в Go]]`

---

### I2. Singleton, Factory, Builder в Go.md
**Что включить:**
- Singleton через sync.Once
- Factory functions
- Builder для сложных объектов
- Functional options pattern

**Связи:** `[[Go - WaitGroup и Once]]`, `[[Паттерны проектирования в Go]]`

---

### I3. Repository паттерн.md
**Что включить:**
- Абстракция доступа к данным
- Интерфейс Repository
- Реализация с PostgreSQL
- Тестирование с моками

**Связи:** `[[Go - Интерфейсы]]`, `[[GORM - ORM для Go]]`

---

### I4. FSM (Конечные автоматы).md
**Что включить:**
- Состояния и переходы
- Применение: заказы, платежи, workflow
- Реализация в Go

**Связи:** `[[Паттерны проектирования в Go]]`

---

### I5. SOLID принципы.md
**Что включить:**
- S - Single Responsibility
- O - Open/Closed
- L - Liskov Substitution (через интерфейсы)
- I - Interface Segregation
- D - Dependency Inversion
- Примеры на Go

**Связи:** `[[Go - Интерфейсы]]`, `[[Паттерны проектирования в Go]]`

---

### I6. Clean Architecture.md
**Что включить:**
- Layers: Entities, Use Cases, Interfaces, Frameworks
- Dependency rule
- Структура проекта в Go
- Пример

**Связи:** `[[SOLID принципы]]`, `[[Микросервисная архитектура]]`

---

## МОДУЛЬ J: Системные инструменты (может выполнять любой инстанс)

### J1. Bash - Основы скриптинга.md
**Что включить:**
- Базовый синтаксис
- Переменные, условия, циклы
- Полезные команды для backend
- Примеры скриптов

**Связи:** `[[Makefile]]`, `[[CI-CD - Основные концепции]]`

---

### J2. Переменные среды и .env.md
**Что включить:**
- os.Getenv в Go
- .env файлы
- godotenv библиотека
- 12-factor app

**Связи:** `[[Go - Пакеты и модули]]`, `[[Docker - Docker Compose]]`

---

### J3. Makefile.md
**Что включить:**
- Targets, dependencies
- Полезные targets для Go: build, test, lint, run
- Переменные
- .PHONY

**Связи:** `[[Bash - Основы скриптинга]]`, `[[CI-CD - Основные концепции]]`

---

### J4. Taskfile.md
**Что включить:**
- Современная альтернатива Make
- YAML синтаксис
- Сравнение с Makefile

**Связи:** `[[Makefile]]`

---

## Распределение по инстансам

| Инстанс | Модули | Заметок |
|---------|--------|---------|
| 1 | A (Concurrency) | 9 |
| 2 | B (Алгоритмы) | 10 |
| 3 | C + D (Тестирование + Git) | 10 |
| 4 | E (Аутентификация) | 6 |
| 5 | F + G (DevOps) | 11 |
| 6 | H + I (Фреймворки + Паттерны) | 12 |
| Любой | J (Системные инструменты) | 4 |

**Итого: 62 заметки**

---

## Инструкции для инстансов

1. **Перед началом** прочитай INDEX.md и несколько существующих заметок для понимания стиля
2. **Создавай заметки** в корне vault (не в подпапках)
3. **Используй wiki-links** `[[Название]]` для связей
4. **Пиши на русском** языке
5. **Добавляй примеры кода** на Go с комментариями
6. **После создания** обнови PROGRESS.md

---

## После завершения всех модулей

1. Обновить INDEX.md со всеми новыми ссылками
2. Проверить все wiki-links на битые ссылки
3. Обновить PROGRESS.md с финальной статистикой
