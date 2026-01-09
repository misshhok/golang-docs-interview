# Git - Ветвление и слияние

Ветвление (branching) — одна из ключевых возможностей Git, позволяющая работать над разными задачами изолированно. Слияние (merging) объединяет изменения из разных веток.

## Зачем нужны ветки

Ветки позволяют:
- Работать над новой функциональностью, не влияя на основной код
- Изолировать эксперименты
- Параллельно разрабатывать несколько фич
- Легко переключаться между задачами
- Откатываться к стабильной версии

```
main    ●───●───●───●───●───●
             \       \     /
feature       ●───●───●───●
```

## Основы работы с ветками

### git branch

Управление ветками.

```bash
# Показать список веток
git branch

# Показать все ветки (включая удалённые)
git branch -a

# Создать новую ветку
git branch feature/authentication

# Удалить ветку
git branch -d feature/old-feature

# Принудительно удалить ветку (даже если не смержена)
git branch -D feature/experimental

# Переименовать ветку
git branch -m old-name new-name

# Переименовать текущую ветку
git branch -m new-name

# Показать последний коммит каждой ветки
git branch -v

# Показать смерженные ветки
git branch --merged

# Показать несмерженные ветки
git branch --no-merged
```

### git checkout

Переключение между ветками (старый способ).

```bash
# Переключиться на ветку
git checkout main

# Создать и переключиться на новую ветку
git checkout -b feature/new-api

# Переключиться на ветку и обновить её с remote
git checkout -b feature/auth origin/feature/auth

# Вернуться на предыдущую ветку
git checkout -
```

### git switch

Переключение между ветками (новый способ, с Git 2.23).

```bash
# Переключиться на ветку
git switch main

# Создать и переключиться на новую ветку
git switch -c feature/new-api

# Вернуться на предыдущую ветку
git switch -
```

**Разница между checkout и switch:**
- `git switch` — только для переключения веток (проще и безопаснее)
- `git checkout` — универсальная команда (ветки, файлы, коммиты)

## Типы слияния (merge)

### Fast-Forward Merge

Происходит, когда в основной ветке нет новых коммитов.

```
До:
main    ●───●
             \
feature       ●───●───●

После:
main    ●───●───●───●───●
```

```bash
git checkout main
git merge feature/auth

# Вывод:
# Fast-forward
# main abc123..def456
```

### 3-Way Merge (рекурсивное слияние)

Происходит, когда в обеих ветках есть новые коммиты.

```
До:
main    ●───●───●───●
             \
feature       ●───●───●

После:
main    ●───●───●───●───●  (merge commit)
             \         /
feature       ●───●───●
```

```bash
git checkout main
git merge feature/api

# Создаётся merge commit
```

Git находит общего предка (base) и делает трёхстороннее слияние.

### No-Fast-Forward Merge

Принудительно создаёт merge commit даже когда возможен fast-forward.

```bash
# Всегда создавать merge commit
git merge --no-ff feature/auth
```

Это полезно для сохранения истории — видно, что была отдельная ветка.

## Основные команды merge

```bash
# Слияние ветки в текущую
git merge feature/auth

# Слияние с созданием merge commit
git merge --no-ff feature/auth

# Слияние только если возможен fast-forward
git merge --ff-only feature/auth

# Слияние с описанием
git merge feature/auth -m "Merge authentication feature"

# Отменить слияние (до коммита)
git merge --abort

# Продолжить слияние после разрешения конфликтов
git merge --continue
```

## Разрешение конфликтов

Конфликт возникает, когда одни и те же строки изменены в разных ветках.

### Пример конфликта

```go
// main ветка:
func GetUser(id int) string {
    return "User from database"
}

// feature ветка:
func GetUser(id int) string {
    return "User from cache"
}
```

При слиянии Git не может автоматически решить, какую версию использовать:

```bash
git merge feature/cache

# Вывод:
# Auto-merging user.go
# CONFLICT (content): Merge conflict in user.go
# Automatic merge failed; fix conflicts and then commit the result.
```

### Файл с конфликтом

```go
func GetUser(id int) string {
<<<<<<< HEAD
    return "User from database"
=======
    return "User from cache"
>>>>>>> feature/cache
}
```

**Маркеры конфликта:**
- `<<<<<<< HEAD` — начало версии из текущей ветки
- `=======` — разделитель
- `>>>>>>> feature/cache` — конец версии из сливаемой ветки

### Разрешение конфликта

**Вариант 1: Вручную редактировать**

```go
// Решаем оставить обе проверки
func GetUser(id int) string {
    // Сначала проверяем кеш
    if cached := getCached(id); cached != "" {
        return cached
    }
    // Если нет в кеше, берём из БД
    return getUserFromDB(id)
}
```

Удаляем маркеры конфликта и сохраняем:

```bash
git add user.go
git commit -m "Merge feature/cache: combine DB and cache logic"
```

**Вариант 2: Выбрать одну из версий**

```bash
# Принять версию из текущей ветки (ours)
git checkout --ours user.go

# Принять версию из сливаемой ветки (theirs)
git checkout --theirs user.go

git add user.go
git commit
```

**Вариант 3: Использовать merge tool**

```bash
# Запустить графический инструмент
git mergetool

# Настроить mergetool (например, VSCode)
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

### Просмотр конфликтов

```bash
# Показать файлы с конфликтами
git status

# Показать конфликты
git diff

# Список конфликтующих файлов
git diff --name-only --diff-filter=U

# Отменить слияние
git merge --abort
```

## Стратегии слияния

### Recursive (по умолчанию)

Стандартная стратегия для большинства случаев.

```bash
git merge -s recursive feature/auth

# С опциями:
# Предпочитать нашу версию при конфликтах
git merge -X ours feature/auth

# Предпочитать их версию при конфликтах
git merge -X theirs feature/auth
```

### Ours

Игнорирует все изменения из другой ветки.

```bash
git merge -s ours feature/experimental
```

Создаётся merge commit, но фактически используется только текущая ветка.

### Octopus

Для слияния более двух веток одновременно.

```bash
git merge feature-1 feature-2 feature-3
```

## Удалённые ветки

### Работа с remote ветками

```bash
# Показать удалённые ветки
git branch -r

# Показать все ветки
git branch -a

# Создать локальную ветку из удалённой
git checkout -b feature/auth origin/feature/auth
# Или:
git switch -c feature/auth origin/feature/auth

# Отслеживать удалённую ветку
git branch --set-upstream-to=origin/feature/auth

# Удалить удалённую ветку
git push origin --delete feature/old-feature

# Обновить список удалённых веток
git fetch --prune
# или короче:
git fetch -p
```

### Настройка отслеживания

```bash
# Отправить ветку и настроить отслеживание
git push -u origin feature/auth

# Теперь можно просто:
git push
git pull
```

## Практические сценарии

### Сценарий 1: Создание feature ветки

```bash
# Обновляем main
git checkout main
git pull

# Создаём ветку для новой фичи
git checkout -b feature/user-profile

# Работаем, делаем коммиты
git add .
git commit -m "Add user profile page"

# Отправляем на remote
git push -u origin feature/user-profile

# Создаём Pull Request на GitHub
```

### Сценарий 2: Обновление feature ветки из main

```bash
# Находясь в feature ветке
git checkout feature/user-profile

# Обновляем main
git fetch origin main

# Вариант 1: Merge
git merge origin/main

# Вариант 2: Rebase (см. [[Git - Rebase vs Merge]])
git rebase origin/main
```

### Сценарий 3: Удаление смерженной ветки

```bash
# После слияния через Pull Request
git checkout main
git pull

# Удалить локальную ветку
git branch -d feature/user-profile

# Удалить удалённую ветку (если не удалилась автоматически)
git push origin --delete feature/user-profile

# Очистить устаревшие ссылки
git fetch --prune
```

## Best Practices

1. ✅ **Создавайте ветки для каждой задачи** — изолируйте изменения
2. ✅ **Используйте описательные имена веток**:
   - `feature/user-authentication`
   - `bugfix/fix-login-error`
   - `hotfix/security-patch`
3. ✅ **Регулярно обновляйте feature ветку из main** — избегайте больших конфликтов
4. ✅ **Удаляйте смерженные ветки** — держите репозиторий чистым
5. ✅ **Используйте --no-ff для важных фич** — сохраняйте историю
6. ❌ **Не коммитьте напрямую в main/master** — используйте Pull Requests
7. ❌ **Не оставляйте долгоживущие feature ветки** — чаще мержите

## Naming Convention для веток

```
<type>/<short-description>

Типы:
- feature/   - новая функциональность
- bugfix/    - исправление бага
- hotfix/    - срочное исправление в продакшене
- refactor/  - рефакторинг
- docs/      - документация
- test/      - добавление тестов
- chore/     - рутинные задачи

Примеры:
feature/add-payment-gateway
bugfix/fix-null-pointer-error
hotfix/patch-security-vulnerability
refactor/improve-error-handling
```

## Вопросы с собеседований

**Вопрос:** В чём разница между fast-forward и 3-way merge?

**Ответ:** Fast-forward происходит, когда в основной ветке нет новых коммитов после создания feature ветки — Git просто перемещает указатель. 3-way merge происходит, когда в обеих ветках есть новые коммиты — Git создаёт специальный merge commit, объединяющий изменения из обеих веток.

**Вопрос:** Как разрешить конфликт слияния?

**Ответ:**
1. Открыть файл с конфликтом и найти маркеры `<<<<<<<`, `=======`, `>>>>>>>`
2. Решить, какую версию оставить или объединить изменения вручную
3. Удалить маркеры конфликта
4. Сделать `git add файл` и `git commit`
Альтернативно можно использовать `git checkout --ours/--theirs` для выбора одной из версий.

**Вопрос:** Когда использовать --no-ff флаг при слиянии?

**Ответ:** `--no-ff` создаёт merge commit даже когда возможен fast-forward. Используйте его для важных feature веток, чтобы сохранить в истории информацию о том, что была отдельная ветка разработки. Это помогает понять логическую структуру изменений и при необходимости откатить всю фичу одним revert.

## Связанные темы

- [[Git - Основные команды]]
- [[Git - Rebase vs Merge]]
- [[Git - Gitflow и trunk-based development]]
- [[Git - Cherry-pick и работа с историей]]