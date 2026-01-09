# Go - Garbage Collector

Go использует автоматическую сборку мусора для управления памятью.

## Основы

### Что собирает GC

Garbage Collector освобождает память, на которую больше нет ссылок:

```go
func createData() {
    data := make([]byte, 1024*1024) // 1 MB
    // Используем data...
} // data выходит из области видимости - станет кандидатом на GC
```

### Алгоритм: Concurrent Mark & Sweep

Go GC работает конкурентно с программой:

1. **Mark Setup (STW)** - короткая пауза, подготовка к маркировке
2. **Marking (concurrent)** - маркировка достижимых объектов, программа работает
3. **Mark Termination (STW)** - короткая пауза, завершение маркировки
4. **Sweeping (concurrent)** - освобождение недостижимых объектов

**STW (Stop-The-World)** - программа приостанавливается на короткое время.

## Триггеры GC

### GOGC

Переменная окружения, контролирующая частоту GC:

```bash
# По умолчанию GOGC=100
# GC запускается когда heap вырастет на 100% относительно прошлого размера

# Более частый GC (меньше память, больше CPU)
GOGC=50 ./myapp

# Менее частый GC (больше память, меньше CPU)
GOGC=200 ./myapp

# Отключить автоматический GC (не рекомендуется!)
GOGC=off ./myapp
```

**Формула:** `heap_goal = live_heap + (live_heap + GC_roots) * GOGC/100`

### Ручной запуск

```go
import "runtime"

// Принудительно запустить GC
runtime.GC()

// Обычно не нужно! GC сам знает когда запускаться
```

## Мониторинг GC

### runtime.MemStats

```go
var m runtime.MemStats
runtime.ReadMemStats(&m)

fmt.Printf("Alloc = %v MB\n", m.Alloc/1024/1024)
fmt.Printf("TotalAlloc = %v MB\n", m.TotalAlloc/1024/1024)
fmt.Printf("Sys = %v MB\n", m.Sys/1024/1024)
fmt.Printf("NumGC = %v\n", m.NumGC)
fmt.Printf("PauseTotalNs = %v ms\n", m.PauseTotalNs/1e6)
```

### GODEBUG

```bash
# Логировать каждый GC цикл
GODEBUG=gctrace=1 ./myapp

# Вывод:
# gc 1 @0.001s 0%: 0.015+0.59+0.096 ms clock, 0.12+0.10/1.3/3.0+0.76 ms cpu, 4->4->3 MB, 5 MB goal, 8 P
```

### runtime/trace

```go
import (
    "os"
    "runtime/trace"
)

func main() {
    f, _ := os.Create("trace.out")
    defer f.Close()

    trace.Start(f)
    defer trace.Stop()

    // Ваш код
}

// Анализ: go tool trace trace.out
```

## Оптимизация для GC

### 1. Переиспользование объектов (sync.Pool)

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process() {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer bufferPool.Put(buf)
    buf.Reset()

    // Используем buf
}
```

### 2. Уменьшение количества аллокаций

```go
// ❌ Много аллокаций
func bad() {
    for i := 0; i < 1000; i++ {
        s := fmt.Sprintf("item %d", i) // Аллокация каждую итерацию
        process(s)
    }
}

// ✅ Одна аллокация
func good() {
    var buf strings.Builder
    for i := 0; i < 1000; i++ {
        buf.Reset()
        buf.WriteString("item ")
        buf.WriteString(strconv.Itoa(i))
        process(buf.String())
    }
}
```

### 3. Pre-allocation слайсов

```go
// ❌ Многократное перевыделение
result := []int{}
for i := 0; i < 10000; i++ {
    result = append(result, i)
}

// ✅ Одно выделение
result := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    result = append(result, i)
}
```

### 4. Избегайте утечек памяти

```go
// ❌ Утечка памяти - slice сохраняет ссылку на весь массив
func leak() []byte {
    data := make([]byte, 1024*1024) // 1 MB
    return data[:10] // Возвращает 10 байт, но держит 1 MB!
}

// ✅ Копируем нужную часть
func noLeak() []byte {
    data := make([]byte, 1024*1024)
    result := make([]byte, 10)
    copy(result, data[:10])
    return result // data может быть освобожден
}
```

## Поколенческий GC (отсутствует)

Go НЕ использует поколенческий GC (в отличие от Java):
- Все объекты равны
- Нет "молодого" и "старого" поколений
- Проще, но может быть менее эффективно для некоторых паттернов

## Finalizers (финализаторы)

```go
type File struct {
    fd int
}

func NewFile(name string) (*File, error) {
    f := &File{fd: open(name)}

    // Финализатор вызовется когда f станет недостижим
    runtime.SetFinalizer(f, func(f *File) {
        close(f.fd)
    })

    return f, nil
}

// ⚠️ Не полагайтесь на finalizers!
// - Время вызова не гарантировано
// - Может вообще не вызваться
// - Используйте defer для гарантированной очистки
```

## GC Tuning (настройка)

### Цель: < 1ms паузы

Go GC оптимизирован для малых пауз (< 1ms), а не для максимальной throughput.

### Настройка GOGC

```go
import "runtime/debug"

// Программно изменить GOGC
debug.SetGCPercent(200) // Эквивалент GOGC=200

// Отключить автоматический GC
debug.SetGCPercent(-1)
```

### Мягкий лимит памяти (Go 1.19+)

```go
// Установить мягкий лимит памяти
debug.SetMemoryLimit(1024 * 1024 * 1024) // 1 GB

// GC будет чаще запускаться при приближении к лимиту
```

## Ballast паттерн (устаревший)

```go
// ❌ Устаревший паттерн (до Go 1.19)
// Выделяли большой массив для уменьшения частоты GC
ballast := make([]byte, 10*1024*1024*1024) // 10 GB
_ = ballast

// ✅ В Go 1.19+ используйте SetMemoryLimit
debug.SetMemoryLimit(10 * 1024 * 1024 * 1024)
```

## Когда GC становится проблемой

### Симптомы

1. Частые GC паузы в логах
2. Высокий % времени в GC (> 10%)
3. Большой размер heap
4. Высокая частота аллокаций

### Диагностика

```go
// Профилирование памяти
import _ "net/http/pprof"
go func() {
    http.ListenAndServe("localhost:6060", nil)
}()

// http://localhost:6060/debug/pprof/heap
// go tool pprof http://localhost:6060/debug/pprof/heap
```

### Решения

1. Уменьшить количество аллокаций
2. Использовать sync.Pool для переиспользования
3. Увеличить GOGC (больше память, меньше CPU на GC)
4. Использовать SetMemoryLimit
5. Рефакторинг для уменьшения heap

## Best Practices

1. ✅ Доверяйте GC - он хорошо настроен по умолчанию
2. ✅ Используйте sync.Pool для горячих путей
3. ✅ Pre-allocate слайсы когда знаете размер
4. ✅ Профилируйте перед оптимизацией
5. ✅ Используйте SetMemoryLimit в контейнерах
6. ❌ Не вызывайте runtime.GC() без веской причины
7. ❌ Не используйте finalizers для критичной очистки
8. ❌ Не игнорируйте утечки памяти

## Сравнение с другими GC

| Язык | Алгоритм | Паузы | Throughput |
|------|----------|-------|------------|
| Go | Concurrent M&S | < 1ms | Средний |
| Java | Generational + G1/ZGC | 1-10ms | Высокий |
| C# | Generational | 1-10ms | Высокий |
| Python | Reference counting | Нет пауз | Низкий |

Go GC оптимизирован для:
- ✅ Низкая latency (малые паузы)
- ✅ Предсказуемость
- ❌ Не максимальная throughput

## Связанные темы

- [[Go - Управление памятью]]
- [[Go - Профилирование (pprof)]]
- [[Go - Пакет sync]]
