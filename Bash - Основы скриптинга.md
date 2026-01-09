# Bash - Основы скриптинга

Bash (Bourne Again Shell) — это командный интерпретатор и язык скриптов, широко используемый в Unix-подобных системах. Для backend-разработчика умение писать bash-скрипты критически важно для автоматизации рутинных задач, деплоя и DevOps операций.

## Базовый синтаксис

### Shebang и запуск скрипта

```bash
#!/bin/bash
# Первая строка указывает интерпретатор

echo "Hello, World!"
```

Чтобы запустить скрипт:
```bash
chmod +x script.sh  # Делаем файл исполняемым
./script.sh         # Запускаем
```

### Комментарии

```bash
# Это однострочный комментарий

: '
Это многострочный
комментарий
'
```

## Переменные

### Объявление и использование

```bash
#!/bin/bash

# Объявление переменной (БЕЗ пробелов вокруг =)
name="Golang Developer"
age=25

# Использование переменных
echo "Name: $name"
echo "Age: ${age}"  # Фигурные скобки для явного указания границ

# Команды в переменных
current_date=$(date +%Y-%m-%d)
echo "Today: $current_date"
```

### Переменные окружения

```bash
#!/bin/bash

# Чтение переменных окружения
echo "User: $USER"
echo "Home: $HOME"
echo "Path: $PATH"

# Установка переменной для текущего скрипта
export DB_HOST="localhost"
export DB_PORT=5432
```

### Специальные переменные

```bash
#!/bin/bash

# $0 - имя скрипта
# $1, $2, ... - аргументы командной строки
# $# - количество аргументов
# $@ - все аргументы как отдельные слова
# $? - код возврата последней команды
# $$ - PID текущего процесса

echo "Script name: $0"
echo "First argument: $1"
echo "All arguments: $@"
echo "Number of arguments: $#"
```

## Условия

### if-then-else

```bash
#!/bin/bash

age=18

if [ $age -ge 18 ]; then
    echo "Adult"
elif [ $age -ge 13 ]; then
    echo "Teenager"
else
    echo "Child"
fi
```

### Операторы сравнения

**Для чисел:**
```bash
-eq  # равно (equal)
-ne  # не равно (not equal)
-gt  # больше (greater than)
-ge  # больше или равно (greater or equal)
-lt  # меньше (less than)
-le  # меньше или равно (less or equal)
```

**Для строк:**
```bash
=    # равно
!=   # не равно
-z   # строка пустая (zero length)
-n   # строка не пустая (non-zero length)
```

**Для файлов:**
```bash
-f file  # файл существует и это обычный файл
-d dir   # директория существует
-e path  # путь существует
-r file  # файл доступен для чтения
-w file  # файл доступен для записи
-x file  # файл исполняемый
```

### Пример проверки файла

```bash
#!/bin/bash

config_file="config.yaml"

if [ -f "$config_file" ]; then
    echo "Config file exists"
    if [ -r "$config_file" ]; then
        echo "Config is readable"
    else
        echo "❌ Cannot read config"
        exit 1
    fi
else
    echo "❌ Config file not found"
    exit 1
fi
```

### Логические операторы

```bash
#!/bin/bash

# И (AND) - &&
if [ $age -ge 18 ] && [ $age -le 65 ]; then
    echo "Working age"
fi

# ИЛИ (OR) - ||
if [ "$status" = "active" ] || [ "$status" = "pending" ]; then
    echo "Processing..."
fi

# НЕ (NOT) - !
if [ ! -f "backup.tar.gz" ]; then
    echo "Backup not found"
fi
```

## Циклы

### For loop

```bash
#!/bin/bash

# Итерация по списку
for name in Alice Bob Charlie; do
    echo "Hello, $name"
done

# Итерация по файлам
for file in *.go; do
    echo "Processing $file"
    go fmt "$file"
done

# C-style for loop
for ((i=1; i<=5; i++)); do
    echo "Iteration $i"
done
```

### While loop

```bash
#!/bin/bash

counter=1

while [ $counter -le 5 ]; do
    echo "Counter: $counter"
    ((counter++))
done

# Чтение файла построчно
while IFS= read -r line; do
    echo "Line: $line"
done < input.txt
```

### Until loop

```bash
#!/bin/bash

# Выполняется пока условие ложно
counter=1

until [ $counter -gt 5 ]; do
    echo "Counter: $counter"
    ((counter++))
done
```

## Функции

```bash
#!/bin/bash

# Объявление функции
greet() {
    local name=$1  # local делает переменную локальной
    echo "Hello, $name!"
}

# Функция с возвратом значения
add() {
    local result=$(($1 + $2))
    echo $result  # "возвращаем" через echo
}

# Вызов функций
greet "Gopher"
sum=$(add 10 20)
echo "Sum: $sum"
```

## Полезные команды для backend-разработчика

### Проверка статуса сервиса

```bash
#!/bin/bash

check_service() {
    local service_name=$1

    if systemctl is-active --quiet "$service_name"; then
        echo "✅ $service_name is running"
        return 0
    else
        echo "❌ $service_name is not running"
        return 1
    fi
}

check_service "postgresql"
check_service "redis"
```

### Проверка портов

```bash
#!/bin/bash

check_port() {
    local host=$1
    local port=$2

    if nc -z "$host" "$port" 2>/dev/null; then
        echo "✅ Port $port on $host is open"
    else
        echo "❌ Port $port on $host is closed"
    fi
}

check_port "localhost" 5432  # PostgreSQL
check_port "localhost" 6379  # Redis
check_port "localhost" 8080  # API
```

### Backup скрипт

```bash
#!/bin/bash

# Скрипт для бэкапа базы данных
backup_database() {
    local db_name=$1
    local backup_dir="/backups"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="${backup_dir}/${db_name}_${timestamp}.sql"

    echo "Creating backup of $db_name..."

    # Создаем директорию если не существует
    mkdir -p "$backup_dir"

    # Делаем бэкап
    pg_dump "$db_name" > "$backup_file"

    if [ $? -eq 0 ]; then
        echo "✅ Backup created: $backup_file"

        # Сжимаем
        gzip "$backup_file"
        echo "✅ Compressed: ${backup_file}.gz"

        # Удаляем старые бэкапы (старше 7 дней)
        find "$backup_dir" -name "${db_name}_*.sql.gz" -mtime +7 -delete
        echo "✅ Old backups cleaned"
    else
        echo "❌ Backup failed"
        exit 1
    fi
}

backup_database "myapp_production"
```

### Деплой скрипт

```bash
#!/bin/bash

set -e  # Прервать выполнение при ошибке

APP_NAME="myapp"
REPO_DIR="/var/www/${APP_NAME}"
BUILD_DIR="${REPO_DIR}/build"

echo "🚀 Starting deployment..."

# 1. Обновляем код
echo "📥 Pulling latest code..."
cd "$REPO_DIR"
git pull origin main

# 2. Устанавливаем зависимости
echo "📦 Installing dependencies..."
go mod download

# 3. Запускаем тесты
echo "🧪 Running tests..."
go test ./...

# 4. Собираем приложение
echo "🔨 Building application..."
go build -o "$BUILD_DIR/$APP_NAME" ./cmd/main.go

# 5. Перезапускаем сервис
echo "🔄 Restarting service..."
sudo systemctl restart "$APP_NAME"

# 6. Проверяем статус
sleep 2
if systemctl is-active --quiet "$APP_NAME"; then
    echo "✅ Deployment successful!"
else
    echo "❌ Deployment failed - service is not running"
    exit 1
fi
```

## Обработка ошибок

```bash
#!/bin/bash

# set -e: прервать при ошибке
# set -u: ошибка при использовании неопределенной переменной
# set -o pipefail: ошибка если любая команда в pipe провалилась
set -euo pipefail

# Trap для обработки ошибок
trap 'echo "❌ Error on line $LINENO"' ERR

# Проверка кода возврата
go build ./...
if [ $? -ne 0 ]; then
    echo "❌ Build failed"
    exit 1
fi

echo "✅ Build successful"
```

## Практический пример: CI скрипт

```bash
#!/bin/bash

set -euo pipefail

echo "🏗️  Starting CI pipeline..."

# Функция для вывода секций
section() {
    echo ""
    echo "======================================"
    echo "$1"
    echo "======================================"
}

# 1. Проверка кода
section "🔍 Linting"
golangci-lint run ./...

# 2. Форматирование
section "📝 Formatting check"
if [ -n "$(gofmt -l .)" ]; then
    echo "❌ Code is not formatted. Run: go fmt ./..."
    exit 1
fi

# 3. Тесты
section "🧪 Running tests"
go test -v -race -coverprofile=coverage.out ./...

# 4. Coverage
section "📊 Test coverage"
coverage=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | sed 's/%//')
echo "Coverage: ${coverage}%"

if (( $(echo "$coverage < 80" | bc -l) )); then
    echo "❌ Coverage is below 80%"
    exit 1
fi

# 5. Сборка
section "🔨 Building"
go build -o ./bin/app ./cmd/main.go

echo ""
echo "✅ CI pipeline completed successfully!"
```

## Best Practices

1. ✅ **Всегда используйте `set -euo pipefail`** в начале скрипта
2. ✅ **Цитируйте переменные**: `"$var"` вместо `$var`
3. ✅ **Проверяйте входные параметры**
4. ✅ **Используйте функции** для переиспользования кода
5. ✅ **Добавляйте комментарии** для сложной логики
6. ✅ **Логируйте важные действия** с emoji для читаемости
7. ❌ **Не игнорируйте коды возврата** команд
8. ❌ **Не используйте `eval`** без крайней необходимости

## Вопросы с собеседований

**Вопрос:** Чем отличается `$@` от `$*`?

**Ответ:**
- `$@` — все аргументы как отдельные слова: `"$1" "$2" "$3"`
- `$*` — все аргументы как одна строка: `"$1 $2 $3"`
- При использовании в кавычках разница существенна для обработки аргументов с пробелами

**Вопрос:** Что делает `set -e`?

**Ответ:** Прерывает выполнение скрипта при первой ошибке (ненулевой код возврата команды). Это важно для CI/CD скриптов, чтобы не продолжать выполнение после ошибки.

**Вопрос:** Как перенаправить stderr в stdout?

**Ответ:**
```bash
command 2>&1          # stderr в stdout
command > file 2>&1   # оба потока в файл
command &> file       # короткая форма (bash 4+)
```

## Связанные темы

- [[Makefile]]
- [[Taskfile]]
- [[CI-CD - Основные концепции]]
- [[Переменные среды и .env]]
- [[GitLab CI-CD]]
