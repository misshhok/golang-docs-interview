# Git - Cherry-pick и работа с историей

Git предоставляет мощные инструменты для манипуляции историей коммитов: cherry-pick, reset, revert, reflog. Эти команды позволяют исправлять ошибки, переносить изменения между ветками и восстанавливать потерянные коммиты.

## git cherry-pick

Cherry-pick позволяет применить конкретный коммит из одной ветки в другую, не делая полноценный merge.

### Базовое использование

```bash
# Применить коммит abc123 к текущей ветке
git cherry-pick abc123

# Применить несколько коммитов
git cherry-pick abc123 def456 ghi789

# Применить диапазон коммитов (не включая start)
git cherry-pick start..end

# Применить диапазон коммитов (включая start)
git cherry-pick start^..end
```

### Пример сценария

```
main       ●───●───●
            \
hotfix      ●───●───F (важный фикс)
            \
feature          ●───●───●
```

Нужно применить фикс F к feature ветке:

```bash
git checkout feature
git cherry-pick F

# Результат:
main       ●───●───●
            \
hotfix      ●───●───F
            \
feature          ●───●───●───F'
```

F' — это копия коммита F с новым SHA.

### Опции cherry-pick

```bash
# Применить, но не коммитить (оставить в staging)
git cherry-pick --no-commit abc123

# Добавить информацию об оригинальном коммите
git cherry-pick -x abc123
# В сообщении появится: (cherry picked from commit abc123)

# Изменить сообщение коммита
git cherry-pick --edit abc123

# Продолжить после разрешения конфликтов
git cherry-pick --continue

# Пропустить коммит
git cherry-pick --skip

# Отменить cherry-pick
git cherry-pick --abort
```

### Разрешение конфликтов

```bash
git cherry-pick abc123
# CONFLICT (content): Merge conflict in user.go

# Решаем конфликт
git add user.go
git cherry-pick --continue
```

## git reset

Reset перемещает HEAD и опционально изменяет staging area и working directory.

### Три режима reset

#### 1. --soft (только HEAD)

Перемещает HEAD, изменения остаются в staging area.

```bash
git reset --soft HEAD~1

# Отменяет последний коммит
# Изменения из коммита остаются в staging (git add)
```

**Использование:**
- Переделать последний коммит
- Объединить несколько коммитов в один

```bash
# Переделать последний коммит
git reset --soft HEAD~1
# Внести изменения
git add fixed_file.go
git commit -m "Better commit message"
```

#### 2. --mixed (HEAD + staging, по умолчанию)

Перемещает HEAD и убирает изменения из staging, но файлы не трогает.

```bash
git reset HEAD~1
# или
git reset --mixed HEAD~1

# Отменяет коммит
# Убирает из staging
# Файлы сохраняются с изменениями
```

**Использование:**
- Убрать файлы из staging
- Отменить коммит, но сохранить изменения

```bash
# Убрать файл из staging
git reset HEAD file.go
# Теперь эквивалентно:
git restore --staged file.go
```

#### 3. --hard (HEAD + staging + working directory)

⚠️ **ОПАСНО!** Удаляет все изменения!

```bash
git reset --hard HEAD~1

# Отменяет коммит
# Убирает из staging
# УДАЛЯЕТ изменения из файлов
```

**Использование:**
- Полностью откатить изменения
- Вернуться к конкретному коммиту

```bash
# Откатить все изменения
git reset --hard HEAD

# Вернуться к коммиту abc123
git reset --hard abc123
```

### Примеры reset

```bash
# Отменить последний коммит, оставить изменения
git reset HEAD~1

# Отменить последние 3 коммита
git reset HEAD~3

# Вернуться к конкретному коммиту
git reset abc123

# Удалить все локальные изменения
git reset --hard HEAD

# Синхронизировать с remote (удалить локальные коммиты)
git reset --hard origin/main
```

## git revert

Revert создаёт новый коммит, отменяющий изменения. В отличие от reset, не изменяет историю.

### Базовое использование

```bash
# Отменить последний коммит
git revert HEAD

# Отменить конкретный коммит
git revert abc123

# Отменить несколько коммитов
git revert abc123 def456

# Отменить диапазон коммитов
git revert abc123..def456
```

### Отличия от reset

```
reset:
●───●───●───X───Y  (до)
●───●               (после) - история изменена!

revert:
●───●───●───X───Y       (до)
●───●───●───X───Y───R   (после) - новый коммит R отменяет Y
```

### Опции revert

```bash
# Отменить без создания коммита (оставить в staging)
git revert --no-commit abc123

# Отменить merge commit
git revert -m 1 abc123  # 1 = родитель (main ветка)

# Продолжить после разрешения конфликтов
git revert --continue

# Отменить revert
git revert --abort
```

### Когда использовать revert

✅ Отменить коммит в публичной ветке (main, develop)
✅ Сохранить историю изменений
✅ Откатить изменения, которые уже в продакшене

```bash
# Откатить изменения в main
git checkout main
git revert abc123  # безопасно, история сохранена
```

## git reflog

Reflog — это лог всех изменений HEAD. Позволяет восстановить "потерянные" коммиты.

### Просмотр reflog

```bash
# Показать историю HEAD
git reflog

# Вывод:
# abc123 HEAD@{0}: commit: Add feature
# def456 HEAD@{1}: checkout: moving from main to feature
# ghi789 HEAD@{2}: reset: moving to HEAD~1
# jkl012 HEAD@{3}: commit: Fix bug

# Показать последние 10 записей
git reflog -10

# Показать reflog конкретной ветки
git reflog show main
```

### Восстановление потерянных коммитов

```bash
# Случайно сделали hard reset
git reset --hard HEAD~3
# Коммиты "потеряны"!

# Находим их в reflog
git reflog
# abc123 HEAD@{1}: commit: Important feature
# def456 HEAD@{2}: commit: Another change

# Восстанавливаем
git reset --hard abc123
# или
git reset --hard HEAD@{1}
```

### Восстановление удалённой ветки

```bash
# Случайно удалили ветку
git branch -D feature/important

# Находим последний коммит в reflog
git reflog | grep feature/important
# abc123 HEAD@{5}: commit: Last commit in feature

# Восстанавливаем ветку
git branch feature/important abc123
```

## git reset vs git revert

| Ситуация | Использовать |
|----------|--------------|
| Локальные коммиты (не pushed) | `git reset` |
| Публичные коммиты (в main/develop) | `git revert` |
| Нужно сохранить историю | `git revert` |
| Нужно переписать историю | `git reset` |
| Работа в команде | `git revert` |

## Практические сценарии

### Сценарий 1: Перенести коммит из одной ветки в другую

```bash
# Коммит случайно попал в feature1, нужен в feature2
git checkout feature2
git cherry-pick abc123

# Удалить из feature1 (если нужно)
git checkout feature1
git reset --hard HEAD~1
```

### Сценарий 2: Объединить несколько коммитов в один

```bash
# Последние 3 коммита — мелкие фиксы, объединяем
git reset --soft HEAD~3

# Все изменения в staging
git commit -m "Complete feature implementation"
```

### Сценарий 3: Откатить ошибочный merge

```bash
# Merge был ошибочным
git log
# M abc123 Merge branch 'feature'

# Откатить merge (в публичной ветке)
git revert -m 1 abc123

# Или удалить merge (локально)
git reset --hard HEAD~1
```

### Сценарий 4: Восстановить случайно удалённые коммиты

```bash
# Сделали git reset --hard и потеряли работу
git reflog
# Находим нужный коммит
# def456 HEAD@{3}: commit: My important work

git reset --hard def456
# Работа восстановлена!
```

### Сценарий 5: Применить фикс из hotfix ветки

```bash
# Срочный фикс в hotfix, нужен в develop
git checkout develop
git cherry-pick hotfix-commit-sha

# Также применить к feature ветке
git checkout feature/new-api
git cherry-pick hotfix-commit-sha
```

## Изменение истории (осторожно!)

### git commit --amend

Изменить последний коммит.

```bash
# Добавить файл к последнему коммиту
git add forgotten_file.go
git commit --amend --no-edit

# Изменить сообщение последнего коммита
git commit --amend -m "Better message"

# Изменить автора
git commit --amend --author="Name <email@example.com>"
```

⚠️ Создаёт новый коммит с новым SHA!

### git rebase -i (интерактивный)

См. [[Git - Rebase vs Merge]] для подробностей.

```bash
# Редактировать последние 3 коммита
git rebase -i HEAD~3

# Squash, reword, drop, reorder коммиты
```

### git filter-branch (продвинутое)

Переписать всю историю (например, удалить файл со всех коммитов).

```bash
# Удалить файл из всей истории
git filter-branch --tree-filter 'rm -f passwords.txt' HEAD

# Изменить email во всех коммитах
git filter-branch --env-filter '
if [ "$GIT_AUTHOR_EMAIL" = "old@email.com" ]; then
    export GIT_AUTHOR_EMAIL="new@email.com"
fi
' HEAD
```

⚠️ **ОЧЕНЬ опасно!** Изменяет всю историю.

Современная альтернатива: `git filter-repo`

## Best Practices

1. ✅ **Используйте revert для публичных веток** — безопасно для команды
2. ✅ **Reset только для локальных изменений** — до push в remote
3. ✅ **Cherry-pick для точечных изменений** — не злоупотребляйте
4. ✅ **Проверяйте reflog при проблемах** — можно восстановить почти всё
5. ❌ **Не меняйте публичную историю** — используйте только для своих веток
6. ❌ **Не делайте reset --hard без проверки** — можно потерять работу
7. ⚠️ **Осторожно с --force push** — используйте --force-with-lease

## Восстановление в экстренных случаях

### Потерял коммиты после reset --hard

```bash
git reflog
git reset --hard HEAD@{1}
```

### Случайно удалил ветку

```bash
git reflog | grep branch-name
git branch branch-name <commit-sha>
```

### Испортил историю rebase

```bash
git reflog
# Найти состояние до rebase
git reset --hard HEAD@{5}
```

### Удалил файл и закоммитил

```bash
# Восстановить файл из предыдущего коммита
git checkout HEAD~1 -- file.go
git commit -m "Restore deleted file"
```

## Вопросы с собеседований

**Вопрос:** Что делает git cherry-pick и когда его использовать?

**Ответ:** `git cherry-pick` применяет конкретный коммит из одной ветки в другую, создавая копию с новым SHA. Используется для переноса отдельных изменений (например, hotfix в разные ветки) без полного merge. Важно: создаёт дубликат коммита, а не перемещает его.

**Вопрос:** В чём разница между git reset и git revert?

**Ответ:**
- `reset` — перемещает HEAD и изменяет историю, подходит для локальных изменений
- `revert` — создаёт новый коммит, отменяющий изменения, не трогает историю, подходит для публичных веток

Для публичных веток (main) всегда используйте `revert`, для локальных можно `reset`.

**Вопрос:** Что такое git reflog и зачем он нужен?

**Ответ:** `git reflog` — это лог всех изменений HEAD (checkout, commit, reset, rebase и т.д.). Позволяет восстановить "потерянные" коммиты после `reset --hard`, случайного удаления ветки или неудачного rebase. По сути, это страховка от потери данных в Git.

**Вопрос:** Как восстановить случайно удалённые коммиты?

**Ответ:**
1. Использовать `git reflog` для поиска SHA потерянного коммита
2. Выполнить `git reset --hard <commit-sha>` или `git checkout -b recovery <commit-sha>`

Reflog хранит историю ~90 дней по умолчанию, так что почти всегда можно восстановить данные.

## Связанные темы

- [[Git - Основные команды]]
- [[Git - Ветвление и слияние]]
- [[Git - Rebase vs Merge]]
- [[Git - Gitflow и trunk-based development]]
