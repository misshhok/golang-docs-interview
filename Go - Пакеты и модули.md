# Go - Пакеты и модули

## Пакеты

Пакет - это коллекция исходных файлов в одной директории, которые компилируются вместе.

### Объявление пакета

```go
// main.go
package main // Исполняемый пакет

func main() {
    fmt.Println("Hello")
}
```

```go
// utils.go
package utils // Библиотечный пакет

func Add(a, b int) int {
    return a + b
}
```

**Правила:**
- Все файлы в директории должны иметь одно имя пакета
- `package main` - специальный пакет для исполняемых программ
- Имя пакета обычно совпадает с именем директории

### Импорт пакетов

```go
import "fmt"
import "strings"

// Или группировкой
import (
    "fmt"
    "strings"
    "os"
)

// С алиасом
import (
    f "fmt"
    str "strings"
)

// Импорт только для side effects
import _ "net/http/pprof"

// Импорт всех экспортируемых идентификаторов (не рекомендуется!)
import . "fmt"
```

### Видимость (экспорт)

```go
package mypackage

// Экспортируемые (публичные) - заглавная буква
var PublicVar = "visible"
func PublicFunc() {}
type PublicType struct {
    PublicField  int    // Экспортируемое поле
    privateField string // Приватное поле
}

// Не экспортируемые (приватные) - строчная буква
var privateVar = "hidden"
func privateFunc() {}
type privateType struct{}
```

### Структура проекта

```
myproject/
├── go.mod
├── go.sum
├── main.go
├── config/
│   └── config.go         (package config)
├── handlers/
│   ├── user.go           (package handlers)
│   └── auth.go           (package handlers)
└── internal/
    └── database/
        └── db.go         (package database)
```

**internal/** - специальная директория, пакеты из неё доступны только родительскому проекту.

## Модули

Модуль - это коллекция связанных пакетов с версионированием.

### go.mod

Файл описания модуля:

```go
module github.com/username/myproject

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    gorm.io/gorm v1.25.5
)

require (
    github.com/golang/protobuf v1.5.3 // indirect
)

replace github.com/old/package => github.com/new/package v1.2.3
```

Подробнее: [[go.mod и go.sum]]

### Команды модулей

```bash
# Инициализация модуля
go mod init github.com/username/project

# Добавление зависимостей
go get github.com/gin-gonic/gin

# Обновление зависимостей
go get -u ./...

# Удаление неиспользуемых зависимостей
go mod tidy

# Загрузка зависимостей
go mod download

# Проверка модуля
go mod verify

# Создание vendor/
go mod vendor
```

### Версионирование

Go modules использует Semantic Versioning:

```
v1.2.3
│ │ └── PATCH (баг-фиксы)
│ └──── MINOR (новые фичи, обратная совместимость)
└────── MAJOR (breaking changes)
```

```bash
# Конкретная версия
go get github.com/pkg/errors@v0.9.1

# Последняя версия
go get github.com/pkg/errors@latest

# Конкретный коммит
go get github.com/pkg/errors@abc123

# Ветка
go get github.com/pkg/errors@master
```

### Major versions >= v2

```go
// go.mod
module github.com/username/myproject/v2

go 1.21

require (
    github.com/some/package/v3 v3.2.1
)
```

```go
// Импорт
import "github.com/some/package/v3"
```

## Init функция

Выполняется автоматически при инициализации пакета:

```go
package config

var Config AppConfig

func init() {
    // Инициализация при импорте пакета
    Config = loadConfig()
}

func init() {
    // Можно несколько init()
    setupLogging()
}
```

**Порядок выполнения:**
1. Инициализация импортированных пакетов
2. Инициализация переменных пакета
3. Функции init() в порядке объявления
4. main() (если есть)

## Циклические зависимости

```
package A imports package B
package B imports package A
❌ Ошибка: import cycle not allowed
```

**Решение:** выделить общий код в третий пакет

## Best Practices

1. ✅ Одно имя пакета для всех файлов в директории
2. ✅ Короткие, описательные имена пакетов (lowercase, без underscores)
3. ✅ Используйте internal/ для приватных пакетов
4. ✅ go mod tidy после изменения зависимостей
5. ✅ Vendor для контроля зависимостей
6. ❌ Не используйте `import .`
7. ❌ Избегайте циклических зависимостей

## Связанные темы

- [[go.mod и go.sum]]
- [[Go - Основы синтаксиса]]
