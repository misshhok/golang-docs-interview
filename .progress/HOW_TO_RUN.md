# Инструкция по запуску параллельных инстансов Sonnet

## Способ 1: Несколько терминалов с Claude Code

Открой несколько терминалов и в каждом запусти `claude` с разными модулями.

### Терминал 1 - Модуль A (Concurrency)
```bash
cd /Users/michaelsmirnov/Desktop/obs-docs
claude
```

Промпт:
```
Ты работаешь над МОДУЛЕМ A из плана в .progress/SONNET_PLAN.md

Твоя задача: создать 9 заметок по теме Concurrency для подготовки к Golang собеседованиям.

Перед началом:
1. Прочитай .progress/SONNET_PLAN.md - там детальное описание каждой заметки
2. Прочитай .progress/TEMPLATE.md - шаблон структуры заметки
3. Посмотри пример хорошей заметки: Go - Указатели - Подводные камни.md

Список заметок для создания:
1. Go - Затенение переменных (Shadowing).md
2. Go - Race Condition и Data Race.md
3. Go - Deadlock.md
4. Задача - Worker Pool.md
5. Задача - Pipeline pattern.md
6. Задача - Fan-in Fan-out.md
7. Задача - Graceful Shutdown.md
8. Задача - Semaphore pattern.md
9. Задача - Timeout pattern.md

После создания каждой заметки отмечай её как [x] в .progress/PROGRESS.md

Язык заметок: русский. Код на Go с комментариями. Используй wiki-links [[название]].
```

---

### Терминал 2 - Модуль B (Алгоритмы)
```bash
cd /Users/michaelsmirnov/Desktop/obs-docs
claude
```

Промпт:
```
Ты работаешь над МОДУЛЕМ B из плана в .progress/SONNET_PLAN.md

Твоя задача: создать 10 заметок с алгоритмическими задачами для подготовки к Golang собеседованиям.

Перед началом:
1. Прочитай .progress/SONNET_PLAN.md - там детальное описание каждой заметки
2. Прочитай .progress/TEMPLATE.md - шаблон структуры заметки
3. Посмотри существующие задачи: Задача - LRU Cache.md, Задача - Rate Limiter.md

Список заметок для создания:
1. Задача - Two Sum.md
2. Задача - Valid Parentheses.md
3. Задача - Merge Intervals.md
4. Задача - Group Anagrams.md
5. Задача - Find Duplicates.md
6. Задача - Maximum Subarray.md
7. Задача - Climbing Stairs.md
8. Задача - Implement Queue using Stacks.md
9. Задача - Palindrome Check.md
10. Задача - Container With Most Water.md

Каждая задача должна включать: условие, brute force решение, оптимальное решение, объяснение сложности.

После создания каждой заметки отмечай её как [x] в .progress/PROGRESS.md
```

---

### Терминал 3 - Модуль C+D (Тестирование + Git)
```bash
cd /Users/michaelsmirnov/Desktop/obs-docs
claude
```

Промпт:
```
Ты работаешь над МОДУЛЯМИ C и D из плана в .progress/SONNET_PLAN.md

Твоя задача: создать 10 заметок по тестированию в Go и работе с Git.

Перед началом:
1. Прочитай .progress/SONNET_PLAN.md - там детальное описание каждой заметки
2. Прочитай .progress/TEMPLATE.md - шаблон структуры заметки

МОДУЛЬ C - Тестирование (5 заметок):
1. Go - Unit тестирование.md
2. Go - Table-driven тесты.md
3. Go - Моки и стабы.md
4. E2E тестирование.md
5. Test Coverage.md

МОДУЛЬ D - Git (5 заметок):
1. Git - Основные команды.md
2. Git - Ветвление и слияние.md
3. Git - Gitflow и trunk-based development.md
4. Git - Rebase vs Merge.md
5. Git - Cherry-pick и работа с историей.md

После создания каждой заметки отмечай её как [x] в .progress/PROGRESS.md
```

---

### Терминал 4 - Модуль E (Аутентификация)
```bash
cd /Users/michaelsmirnov/Desktop/obs-docs
claude
```

Промпт:
```
Ты работаешь над МОДУЛЕМ E из плана в .progress/SONNET_PLAN.md

Твоя задача: создать 6 заметок по аутентификации и безопасности API.

Перед началом:
1. Прочитай .progress/SONNET_PLAN.md - там детальное описание каждой заметки
2. Прочитай .progress/TEMPLATE.md - шаблон структуры заметки

Список заметок для создания:
1. OAuth 2.0.md
2. JWT (JSON Web Tokens).md
3. Basic Authentication.md
4. Session-based аутентификация.md
5. HTTPS и TLS.md
6. Безопасность API.md

После создания каждой заметки отмечай её как [x] в .progress/PROGRESS.md
```

---

### Терминал 5 - Модуль G (CI/CD + Мониторинг)
```bash
cd /Users/michaelsmirnov/Desktop/obs-docs
claude
```

Промпт:
```
Ты работаешь над МОДУЛЕМ G из плана в .progress/SONNET_PLAN.md

Твоя задача: создать 5 заметок по CI/CD и мониторингу (часть уже создана).

Перед началом:
1. Прочитай .progress/SONNET_PLAN.md - там детальное описание каждой заметки
2. Прочитай .progress/TEMPLATE.md - шаблон структуры заметки

Список заметок для создания:
1. GitHub Actions.md
2. Метрики и наблюдаемость.md
3. Prometheus.md
4. Grafana.md
5. Логирование - Best practices.md

После создания каждой заметки отмечай её как [x] в .progress/PROGRESS.md
```

---

### Терминал 6 - Модули H+I (Фреймворки + Паттерны)
```bash
cd /Users/michaelsmirnov/Desktop/obs-docs
claude
```

Промпт:
```
Ты работаешь над МОДУЛЯМИ H и I из плана в .progress/SONNET_PLAN.md

Твоя задача: создать 12 заметок по Go фреймворкам и паттернам проектирования.

Перед началом:
1. Прочитай .progress/SONNET_PLAN.md - там детальное описание каждой заметки
2. Прочитай .progress/TEMPLATE.md - шаблон структуры заметки

МОДУЛЬ H - Фреймворки (6 заметок):
1. Gin - Web фреймворк.md
2. Gorilla Mux.md
3. Echo.md
4. GORM - ORM для Go.md
5. sqlx.md
6. Chi.md

МОДУЛЬ I - Паттерны (6 заметок):
1. Паттерны проектирования в Go.md
2. Singleton, Factory, Builder в Go.md
3. Repository паттерн.md
4. FSM (Конечные автоматы).md
5. SOLID принципы.md
6. Clean Architecture.md

После создания каждой заметки отмечай её как [x] в .progress/PROGRESS.md
```

---

### Терминал 7 - Модуль J (Системные инструменты)
```bash
cd /Users/michaelsmirnov/Desktop/obs-docs
claude
```

Промпт:
```
Ты работаешь над МОДУЛЕМ J из плана в .progress/SONNET_PLAN.md

Твоя задача: создать 4 заметки по системным инструментам.

Перед началом:
1. Прочитай .progress/SONNET_PLAN.md - там детальное описание каждой заметки
2. Прочитай .progress/TEMPLATE.md - шаблон структуры заметки

Список заметок для создания:
1. Bash - Основы скриптинга.md
2. Переменные среды и .env.md
3. Makefile.md
4. Taskfile.md

После создания каждой заметки отмечай её как [x] в .progress/PROGRESS.md
```

---

## Способ 2: Использование модели через /model

Если у тебя ограничение на параллельные сессии, можно последовательно:

```bash
cd /Users/michaelsmirnov/Desktop/obs-docs
claude --model sonnet
```

И давать промпты по одному модулю за раз.

---

## Важные моменты

### Конфликты при записи в PROGRESS.md
Если несколько инстансов пытаются одновременно обновить PROGRESS.md, могут быть конфликты.

**Решение:** Каждый инстанс может просто создавать заметки, а ты потом вручную обновишь PROGRESS.md, или инстансы могут обновлять только свою секцию.

### Проверка wiki-links
После завершения всех модулей нужно проверить, что все wiki-links работают. Это можно сделать в Obsidian через плагин или вручную.

### Рекомендуемый порядок
1. **Сначала модули A и B** (критически важные задачи)
2. **Затем C, D, E** (тестирование, git, безопасность)
3. **Потом G, H, I, J** (devops, фреймворки, паттерны)

---

## Минимальный промпт (если один терминал)

```
Создай заметки по плану в .progress/SONNET_PLAN.md

Начни с МОДУЛЯ A (Concurrency) - это самое важное для собеседований.

Перед созданием заметок прочитай:
- .progress/SONNET_PLAN.md (детали каждой заметки)
- .progress/TEMPLATE.md (шаблон)
- Go - Указатели - Подводные камни.md (пример хорошей заметки)

Язык: русский. Код: Go с комментариями. Связи: [[wiki-links]].
```
