# Git - Основные команды

Git — это распределённая система контроля версий, которая позволяет отслеживать изменения в коде, работать в команде и управлять историей проекта. Знание основных команд Git — обязательное требование на собеседованиях.

## Основные концепции

### Три состояния файлов

```
Working Directory  →  Staging Area  →  Repository
(рабочая копия)      (индекс)         (коммиты)

  git add →            git commit →
```

- **Working Directory** — текущее состояние файлов на диске
- **Staging Area (Index)** — подготовленные изменения для коммита
- **Repository** — история коммитов

## Инициализация и клонирование

### git init

Создаёт новый Git репозиторий.

```bash
# Создать новый репозиторий в текущей папке
git init

# Создать репозиторий в новой папке
git init my-project
```

Создаётся скрытая папка `.git` со всей историей и конфигурацией.

### git clone

Клонирует существующий репозиторий.

```bash
# Клонировать репозиторий
git clone https://github.com/user/repo.git

# Клонировать в папку с другим именем
git clone https://github.com/user/repo.git my-folder

# Клонировать конкретную ветку
git clone -b develop https://github.com/user/repo.git
```

## Работа с изменениями

### git status

Показывает состояние рабочей копии и staging area.

```bash
git status

# Краткий вывод
git status -s
```

Пример вывода:
```
On branch main
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   README.md

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
        modified:   main.go

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        config.yaml
```

### git add

Добавляет изменения в staging area.

```bash
# Добавить конкретный файл
git add main.go

# Добавить все изменённые файлы
git add .

# Добавить все файлы с расширением .go
git add *.go

# Добавить все файлы в папке
git add src/

# Интерактивное добавление (выбрать части файла)
git add -p
```

### git commit

Создаёт коммит с изменениями из staging area.

```bash
# Коммит с сообщением
git commit -m "Add user authentication"

# Коммит с подробным описанием
git commit -m "Add user authentication" -m "Implemented JWT-based auth"

# Добавить все изменения и закоммитить (без git add)
git commit -am "Fix bug in login handler"

# Изменить последний коммит (добавить файлы или исправить сообщение)
git commit --amend

# Изменить сообщение последнего коммита
git commit --amend -m "New message"
```

**Хорошее сообщение коммита:**
```
Краткое описание (до 50 символов)

Подробное объяснение изменений (если нужно):
- Что изменено
- Почему изменено
- Какие проблемы решает
```

## Просмотр истории

### git log

Показывает историю коммитов.

```bash
# Полная история
git log

# Компактный вывод (одна строка на коммит)
git log --oneline

# Последние N коммитов
git log -5

# С графом веток
git log --graph --oneline --all

# История конкретного файла
git log -- main.go

# Коммиты за период
git log --since="2 weeks ago"
git log --after="2024-01-01" --before="2024-01-31"

# Коммиты конкретного автора
git log --author="Ivan"

# Поиск по сообщению коммита
git log --grep="bug fix"
```

Красивый формат:
```bash
git log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
```

### git diff

Показывает изменения в файлах.

```bash
# Изменения в working directory (не добавленные в staging)
git diff

# Изменения в staging area (что будет закоммичено)
git diff --staged
# или
git diff --cached

# Изменения между коммитами
git diff abc123..def456

# Изменения между ветками
git diff main..feature

# Изменения в конкретном файле
git diff main.go

# Статистика изменений
git diff --stat
```

## Синхронизация с удалённым репозиторием

### git remote

Управление удалёнными репозиториями.

```bash
# Показать список remote
git remote -v

# Добавить remote
git remote add origin https://github.com/user/repo.git

# Удалить remote
git remote remove origin

# Переименовать remote
git remote rename origin upstream

# Изменить URL remote
git remote set-url origin https://github.com/user/new-repo.git
```

### git fetch

Загружает изменения с удалённого репозитория, но не применяет их.

```bash
# Загрузить все ветки
git fetch origin

# Загрузить конкретную ветку
git fetch origin main

# Загрузить и удалить удалённые ветки, которых больше нет
git fetch --prune
```

### git pull

Загружает и применяет изменения (fetch + merge).

```bash
# Загрузить и смержить текущую ветку
git pull

# Загрузить конкретную ветку
git pull origin main

# Pull с rebase вместо merge
git pull --rebase

# Pull всех веток
git pull --all
```

### git push

Отправляет коммиты на удалённый репозиторий.

```bash
# Отправить текущую ветку
git push

# Отправить конкретную ветку
git push origin main

# Отправить и установить upstream для ветки
git push -u origin feature-auth

# Отправить все ветки
git push --all

# Отправить теги
git push --tags

# Force push (⚠️ опасно!)
git push --force
# Безопаснее:
git push --force-with-lease
```

## Отмена изменений

### git restore

Восстанавливает файлы из коммитов.

```bash
# Отменить изменения в файле (восстановить из staging)
git restore main.go

# Убрать файл из staging (unstage)
git restore --staged main.go

# Восстановить файл из конкретного коммита
git restore --source=abc123 main.go
```

### git reset

Перемещает HEAD и изменяет историю.

```bash
# Убрать файлы из staging (soft reset)
git reset HEAD main.go

# Отменить последний коммит, но сохранить изменения
git reset --soft HEAD~1

# Отменить последний коммит, убрать из staging, но сохранить в working directory
git reset --mixed HEAD~1
# или просто:
git reset HEAD~1

# Отменить последний коммит и все изменения (⚠️ опасно!)
git reset --hard HEAD~1

# Вернуться к конкретному коммиту
git reset --hard abc123
```

**Типы reset:**
- `--soft` — только перемещает HEAD, изменения остаются в staging
- `--mixed` — перемещает HEAD, убирает из staging, файлы не трогает (по умолчанию)
- `--hard` — удаляет все изменения (опасно!)

### git revert

Создаёт новый коммит, отменяющий изменения.

```bash
# Отменить конкретный коммит
git revert abc123

# Отменить последний коммит
git revert HEAD

# Отменить без создания коммита сразу
git revert --no-commit abc123
```

**Разница между reset и revert:**
- `reset` — изменяет историю (удаляет коммиты)
- `revert` — создаёт новый коммит с отменой (история сохраняется)

## .gitignore

Файл `.gitignore` указывает, какие файлы не должны попадать в Git.

```gitignore
# Скомпилированные файлы
*.exe
*.o
*.so
*.dll

# Зависимости
vendor/
node_modules/

# Конфигурация
.env
config.local.yaml

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db

# Логи
*.log
logs/

# Билды
bin/
build/
dist/

# Go специфичное
*.test
coverage.out
```

Применить .gitignore к уже отслеживаемым файлам:
```bash
# Удалить из Git, но оставить на диске
git rm --cached file.txt

# Удалить всю папку из Git
git rm -r --cached vendor/
```

## Полезные команды

### git stash

Временно сохраняет изменения.

```bash
# Сохранить изменения
git stash

# Сохранить с описанием
git stash save "Work in progress on feature"

# Показать список stash
git stash list

# Применить последний stash
git stash apply

# Применить и удалить последний stash
git stash pop

# Применить конкретный stash
git stash apply stash@{2}

# Удалить stash
git stash drop stash@{0}

# Очистить все stash
git stash clear
```

### git clean

Удаляет неотслеживаемые файлы.

```bash
# Показать, что будет удалено (dry run)
git clean -n

# Удалить неотслеживаемые файлы
git clean -f

# Удалить неотслеживаемые файлы и папки
git clean -fd

# Удалить и файлы из .gitignore
git clean -fX
```

### git show

Показывает информацию о коммите.

```bash
# Показать последний коммит
git show

# Показать конкретный коммит
git show abc123

# Показать файл из коммита
git show abc123:main.go

# Показать изменения в файле
git show abc123 -- main.go
```

## Практический workflow

```bash
# 1. Клонировать репозиторий
git clone https://github.com/company/project.git
cd project

# 2. Создать ветку для новой фичи
git checkout -b feature/user-auth

# 3. Внести изменения
# ... редактируем файлы ...

# 4. Посмотреть статус
git status

# 5. Добавить изменения
git add .

# 6. Проверить что добавили
git diff --staged

# 7. Создать коммит
git commit -m "Add JWT authentication"

# 8. Отправить на GitHub
git push -u origin feature/user-auth

# 9. Обновить ветку с изменениями из main
git checkout main
git pull
git checkout feature/user-auth
git merge main

# 10. Финальный push
git push
```

## Вопросы с собеседований

**Вопрос:** В чём разница между `git pull` и `git fetch`?

**Ответ:** `git fetch` загружает изменения с удалённого репозитория, но не применяет их — позволяет просмотреть изменения перед слиянием. `git pull` делает `fetch` + `merge` за один раз — загружает и сразу сливает изменения в текущую ветку.

**Вопрос:** Как отменить последний коммит?

**Ответ:** Зависит от того, нужно ли сохранить изменения:
- `git reset --soft HEAD~1` — отменить коммит, изменения остаются в staging
- `git reset HEAD~1` — отменить коммит, изменения в working directory
- `git reset --hard HEAD~1` — полностью удалить коммит и изменения
- `git revert HEAD` — создать новый коммит, отменяющий изменения (не изменяет историю)

**Вопрос:** Что такое staging area и зачем она нужна?

**Ответ:** Staging area (индекс) — это промежуточная область между рабочей копией и репозиторием. Она позволяет выборочно добавлять изменения в коммит (`git add`), а не коммитить всё сразу. Это даёт контроль над тем, что войдёт в коммит, и позволяет группировать логически связанные изменения.

## Связанные темы

- [[Git - Ветвление и слияние]]
- [[Git - Rebase vs Merge]]
- [[Git - Gitflow и trunk-based development]]
- [[CI-CD - Основные концепции]]