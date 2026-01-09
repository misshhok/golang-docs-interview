# go.mod и go.sum

## go.mod

Файл описания модуля и его зависимостей.

### Структура

```go
module github.com/username/myproject  // Имя модуля

go 1.21  // Минимальная версия Go

require (
    github.com/gin-gonic/gin v1.9.1      // Прямая зависимость
    gorm.io/gorm v1.25.5
)

require (
    github.com/golang/protobuf v1.5.3 // indirect - транзитивная зависимость
)

replace (
    github.com/old/pkg => github.com/new/pkg v1.2.3  // Замена пакета
    github.com/local/pkg => ../local/pkg             // Локальная замена
)

exclude github.com/broken/pkg v1.0.0  // Исключение версии

retract v1.5.0  // Отзыв собственной версии
```

### Директивы

**module** - имя модуля (import path):
```go
module github.com/username/myproject
```

**go** - минимальная версия Go:
```go
go 1.21
```

**require** - зависимости:
```go
require (
    github.com/gin-gonic/gin v1.9.1
    gorm.io/gorm v1.25.5
)
```

**replace** - замена зависимости:
```go
// Для тестирования локальных изменений
replace github.com/some/pkg => ../local/pkg

// Замена на fork
replace github.com/old/pkg => github.com/myusername/pkg v1.2.3
```

**exclude** - исключение версии:
```go
exclude github.com/broken/pkg v1.0.0
```

**retract** - отзыв версии (в go.mod самого пакета):
```go
retract [v1.5.0, v1.5.3] // Диапазон
retract v1.6.0           // Конкретная версия
```

### Команды

```bash
# Создать go.mod
go mod init github.com/username/project

# Добавить/обновить зависимость
go get github.com/gin-gonic/gin@v1.9.1
go get github.com/gin-gonic/gin@latest

# Удалить неиспользуемые, добавить недостающие
go mod tidy

# Обновить все зависимости
go get -u ./...

# Обновить только patch версии
go get -u=patch ./...

# Загрузить зависимости в кэш
go mod download

# Проверить целостность
go mod verify

# Создать vendor/
go mod vendor

# Посмотреть зависимости
go list -m all

# Граф зависимостей
go mod graph

# Причина зависимости
go mod why github.com/some/package
```

## go.sum

Контрольные суммы для проверки целостности модулей.

### Структура

```
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:hPrL7YrpYKXt5YId3A/Tnip5kqbEAP+KLuI3SUcPTeU=
```

**Формат:**
```
<module> <version> <hash>
<module> <version>/go.mod <hash>
```

- Первая строка - хэш содержимого модуля
- Вторая строка - хэш go.mod файла

**Важно:** go.sum должен быть в git!

### Проверка

```bash
# Проверить что зависимости не изменились
go mod verify

# ✅ Успех
all modules verified

# ❌ Ошибка
github.com/some/pkg v1.2.3: dir has been modified
```

## Минимальная версия (MVS)

Go использует Minimal Version Selection:

```
Проект требует:
  A v1.2 (который требует C v1.3)
  B v1.4 (который требует C v1.5)

Go выберет: C v1.5 (максимальная минимальная версия)
```

## Версионирование

### Semantic Versioning

```
v1.2.3
│ │ └── PATCH - баг-фиксы, обратная совместимость
│ └──── MINOR - новые фичи, обратная совместимость
└────── MAJOR - breaking changes
```

### Pseudo-versions

Для коммитов без тегов:

```
v0.0.0-20230101120000-abc123456789
│ │ │  │              └── commit hash
│ │ │  └── timestamp
│ │ └── patch
│ └── minor
└── major
```

### Major версии >= v2

```go
// go.mod проекта v2
module github.com/username/myproject/v2

// Импорт в коде
import "github.com/username/myproject/v2"
```

## Работа с приватными репозиториями

```bash
# Переменная окружения
export GOPRIVATE=github.com/mycompany/*

# В go.env
go env -w GOPRIVATE=github.com/mycompany/*

# Git credentials
git config --global url."https://username:token@github.com/".insteadOf "https://github.com/"

# SSH
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

## Vendor

Копия зависимостей в проекте:

```bash
# Создать vendor/
go mod vendor

# Использовать vendor/ при сборке
go build -mod=vendor

# По умолчанию vendor используется если есть
```

**Когда использовать:**
- ✅ Гарантия доступности зависимостей
- ✅ Контроль точных версий
- ✅ Ускорение CI/CD
- ❌ Увеличивает размер репозитория

## Прокси модулей

```bash
# По умолчанию
GOPROXY=https://proxy.golang.org,direct

# Несколько прокси
GOPROXY=https://proxy1.com,https://proxy2.com,direct

# Отключить прокси
GOPROXY=direct

# Приватные модули без прокси
GOPRIVATE=github.com/mycompany/*
```

## Workspace (Go 1.18+)

Для работы с несколькими модулями:

```bash
# go.work
go 1.21

use (
    ./moduleA
    ./moduleB
)

replace github.com/some/pkg => ../local/pkg
```

```bash
# Создать workspace
go work init ./moduleA ./moduleB

# Добавить модуль
go work use ./moduleC

# Синхронизировать
go work sync
```

## Частые проблемы

### 1. Несовместимость версий

```bash
# Обновить go.mod и go.sum
go mod tidy

# Проверить конфликты
go list -m all
```

### 2. Контрольная сумма не совпадает

```bash
# Очистить кэш
go clean -modcache

# Заново загрузить
go mod download
```

### 3. Приватный репозиторий недоступен

```bash
# Настроить GOPRIVATE
go env -w GOPRIVATE=github.com/mycompany/*

# Проверить git credentials
git config --global --list
```

## Best Practices

1. ✅ Всегда коммитьте go.mod и go.sum
2. ✅ Используйте go mod tidy регулярно
3. ✅ Указывайте конкретные версии в CI/CD
4. ✅ Используйте replace для локальной разработки
5. ✅ Настройте GOPRIVATE для приватных репозиториев
6. ✅ Регулярно обновляйте зависимости (go get -u)
7. ❌ Не редактируйте go.sum вручную
8. ❌ Не игнорируйте go.sum в .gitignore

## Пример типичного workflow

```bash
# 1. Создать новый проект
go mod init github.com/username/myproject

# 2. Добавить зависимость
go get github.com/gin-gonic/gin

# 3. Написать код, импортировать пакеты
# import "github.com/gin-gonic/gin"

# 4. Очистить неиспользуемые зависимости
go mod tidy

# 5. Проверить
go mod verify

# 6. Закоммитить
git add go.mod go.sum
git commit -m "Add dependencies"
```

## Связанные темы

- [[Go - Пакеты и модули]]
- [[Go - Основы синтаксиса]]
