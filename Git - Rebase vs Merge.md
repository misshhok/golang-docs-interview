# Git - Rebase vs Merge

Rebase и merge — два способа интеграции изменений из одной ветки в другую. Выбор между ними влияет на читаемость истории и безопасность работы в команде.

## Merge

Merge объединяет две ветки, создавая merge commit.

### Как работает merge

```
До merge:
main       ●───●───●───M
            \         /
feature      ●───●───●

После merge:
main       ●───●───●───M (merge commit)
            \         /
feature      ●───●───●
```

### Команды merge

```bash
# Слить feature ветку в текущую
git checkout main
git merge feature/auth

# Merge с созданием merge commit (даже если возможен fast-forward)
git merge --no-ff feature/auth

# Merge только если возможен fast-forward
git merge --ff-only feature/auth
```

### Характеристики merge

✅ **Сохраняет всю историю** — видно, когда и откуда были изменения
✅ **Безопасно** — не изменяет существующие коммиты
✅ **Просто откатить** — можно сделать `git revert` merge commit
✅ **Честная история** — показывает реальный порядок событий

❌ **Загромождает историю** — много merge commits
❌ **Нелинейная история** — сложнее читать `git log`
❌ **Merge commits** — дополнительные коммиты без смысловой нагрузки

## Rebase

Rebase переносит коммиты из одной ветки на вершину другой, создавая линейную историю.

### Как работает rebase

```
До rebase:
main       ●───●───●───●
            \
feature      ●───●───●

После rebase:
main       ●───●───●───●
                        \
feature                  ●'──●'──●'
```

Коммиты из feature "переписываются" (●') поверх main.

### Команды rebase

```bash
# Rebase текущей ветки на main
git checkout feature/auth
git rebase main

# Интерактивный rebase (см. ниже)
git rebase -i main

# Продолжить после разрешения конфликтов
git rebase --continue

# Пропустить коммит
git rebase --skip

# Отменить rebase
git rebase --abort
```

### Характеристики rebase

✅ **Линейная история** — легче читать и понимать
✅ **Чистая история** — нет лишних merge commits
✅ **Проще для bisect** — легче искать баги
✅ **Красивый git log** — последовательная история

❌ **Переписывает историю** — изменяет существующие коммиты
❌ **Опасно для публичных веток** — может сломать работу команды
❌ **Сложнее разрешать конфликты** — конфликты для каждого коммита
❌ **Теряется контекст слияния** — непонятно, когда ветки объединились

## Сравнение

| Критерий | Merge | Rebase |
|----------|-------|--------|
| **История** | Нелинейная, все коммиты | Линейная, переписанные коммиты |
| **Merge commits** | Да | Нет |
| **Изменяет коммиты** | Нет | Да (создаёт новые SHA) |
| **Конфликты** | Один раз | Для каждого коммита |
| **Безопасность** | Безопасно | Опасно для публичных веток |
| **Читаемость** | Сложнее | Легче |
| **Откат** | Просто (revert) | Сложно |

## Интерактивный rebase

Интерактивный rebase позволяет редактировать историю коммитов.

```bash
# Rebase последних 3 коммитов
git rebase -i HEAD~3

# Rebase от конкретного коммита
git rebase -i abc123
```

### Открывается редактор

```
pick abc123 Add user model
pick def456 Add user repository
pick ghi789 Fix typo in user model

# Commands:
# p, pick = use commit
# r, reword = use commit, but edit message
# e, edit = use commit, but stop for amending
# s, squash = use commit, but meld into previous commit
# f, fixup = like squash, but discard commit message
# d, drop = remove commit
```

### Действия в интерактивном rebase

#### 1. Squash — объединить коммиты

```
pick abc123 Add user model
squash def456 Add user tests
squash ghi789 Update documentation

# Результат: один коммит со всеми изменениями
```

#### 2. Reword — изменить сообщение

```
reword abc123 Add user model  # откроется редактор для изменения
pick def456 Add tests
```

#### 3. Edit — изменить коммит

```
edit abc123 Add user model
pick def456 Add tests

# После edit:
git add forgotten_file.go
git commit --amend
git rebase --continue
```

#### 4. Drop — удалить коммит

```
pick abc123 Add user model
drop def456 Add debug logs
pick ghi789 Add tests
```

#### 5. Reorder — изменить порядок

```
pick ghi789 Add tests
pick abc123 Add user model  # поменяли местами
pick def456 Add repository
```

## Golden Rule of Rebase

**⚠️ НИКОГДА НЕ ДЕЛАЙТЕ REBASE ПУБЛИЧНЫХ ВЕТОК!**

```bash
# ❌ ОПАСНО! Не делайте это:
git checkout main
git rebase feature/auth  # main — публичная ветка!

# ✅ ПРАВИЛЬНО:
git checkout feature/auth
git rebase main  # feature — ваша локальная ветка
```

**Почему опасно:**
- Rebase изменяет SHA коммитов
- Другие разработчики работают со старыми SHA
- Возникают дублирующиеся коммиты и конфликты

### Что считается публичной веткой

- `main` / `master`
- `develop`
- Любая ветка, из которой работают другие разработчики
- Ветка, отправленная в удалённый репозиторий и используемая в PR

### Когда можно делать rebase

✅ Ваша локальная ветка, не отправленная в remote
✅ Ваша feature ветка, где вы работаете один
✅ После уведомления команды (force push в свою ветку)

## Практические сценарии

### Сценарий 1: Обновление feature ветки

**С merge:**
```bash
git checkout feature/auth
git merge main

# История:
# feature: ●───●───●───M
#           \         /
# main:      ●───●───●
```

**С rebase:**
```bash
git checkout feature/auth
git rebase main

# История (линейная):
# feature:          ●'──●'──●'
# main:    ●───●───●
```

### Сценарий 2: Очистка истории перед merge

```bash
# У вас много мелких коммитов
git log --oneline
# ghi789 Fix typo
# def456 Fix another typo
# abc123 Add feature

# Объединяем в один коммит
git rebase -i HEAD~3

# В редакторе:
pick abc123 Add feature
fixup def456 Fix another typo
fixup ghi789 Fix typo

# Результат: один чистый коммит
```

### Сценарий 3: Pull with rebase

```bash
# Вместо pull (fetch + merge)
git pull origin main

# Используем pull с rebase
git pull --rebase origin main

# Настроить по умолчанию для ветки
git config branch.feature/auth.rebase true

# Или глобально
git config --global pull.rebase true
```

## Разрешение конфликтов

### При merge

Конфликты решаются один раз для всего merge:

```bash
git merge feature/auth
# CONFLICT (content): Merge conflict in user.go

# Решаем конфликт в user.go
git add user.go
git commit  # завершаем merge
```

### При rebase

Конфликты решаются для каждого коммита:

```bash
git rebase main
# CONFLICT (content): Merge conflict in user.go

# Решаем конфликт
git add user.go
git rebase --continue

# Может быть ещё конфликт в следующем коммите!
# CONFLICT (content): Merge conflict in auth.go

# Решаем и продолжаем
git add auth.go
git rebase --continue
```

## Force Push после rebase

После rebase нужен force push, так как история изменилась.

```bash
# После rebase
git rebase main

# Обычный push не сработает
git push origin feature/auth
# Ошибка: Updates were rejected

# Force push (опасно!)
git push --force origin feature/auth

# Безопаснее: force-with-lease
git push --force-with-lease origin feature/auth
```

**--force-with-lease** проверяет, что удалённая ветка не изменилась с момента последнего fetch.

## Когда использовать merge

✅ Слияние feature ветки в main (через Pull Request)
✅ Слияние публичных веток
✅ Хотите сохранить контекст, когда произошло слияние
✅ Работаете в команде и не хотите рисковать
✅ Важна точная история изменений

## Когда использовать rebase

✅ Обновление своей feature ветки из main
✅ Очистка истории перед созданием PR
✅ Локальные коммиты, не отправленные в remote
✅ Хотите линейную историю
✅ Исправление мелких коммитов (squash)

## Рекомендованный workflow

### 1. Во время разработки feature

```bash
# Периодически обновляем feature ветку
git checkout feature/auth
git fetch origin
git rebase origin/main  # линейная история
```

### 2. Перед созданием Pull Request

```bash
# Очищаем историю
git rebase -i origin/main

# Squash мелких коммитов
# Исправляем сообщения коммитов
# Удаляем ненужные коммиты

# Force push в свою ветку
git push --force-with-lease origin feature/auth
```

### 3. При слиянии в main

```bash
# Через Pull Request с merge (не rebase!)
# Это создаст merge commit и сохранит историю
```

### Итого

```
Локальная работа → rebase (чистая история)
Публичное слияние → merge (безопасность)
```

## Настройки Git

```bash
# Pull всегда с rebase
git config --global pull.rebase true

# Автоматический stash перед rebase
git config --global rebase.autoStash true

# Показывать оригинальные ветки в reflog
git config --global rebase.abbreviateCommands false
```

## Вопросы с собеседований

**Вопрос:** В чём основная разница между merge и rebase?

**Ответ:** Merge объединяет ветки, создавая merge commit и сохраняя всю историю. Rebase переносит коммиты из одной ветки на вершину другой, создавая линейную историю. Merge безопасен и сохраняет контекст, rebase даёт чистую историю, но переписывает коммиты.

**Вопрос:** Что такое "Golden Rule of Rebase"?

**Ответ:** Никогда не делайте rebase публичных веток (main, develop, или веток, используемых другими разработчиками). Rebase изменяет SHA коммитов, что может сломать работу других разработчиков. Rebase безопасен только для локальных веток или ваших feature веток, где вы работаете один.

**Вопрос:** Когда использовать merge, а когда rebase?

**Ответ:**
- **Merge** — для слияния feature веток в main (через PR), для публичных веток, когда важна точная история
- **Rebase** — для обновления feature ветки из main, для очистки локальной истории перед PR, для создания линейной истории

**Вопрос:** Что делать если после rebase нужно отправить изменения в remote?

**Ответ:** Использовать `git push --force-with-lease`, а не `--force`. `--force-with-lease` проверяет, что удалённая ветка не изменилась с момента последнего fetch, и отклоняет push если кто-то успел внести изменения. Это предотвращает случайную перезапись чужих коммитов.

## Связанные темы

- [[Git - Ветвление и слияние]]
- [[Git - Основные команды]]
- [[Git - Cherry-pick и работа с историей]]
- [[Git - Gitflow и trunk-based development]]
