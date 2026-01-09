# Go - Профилирование (pprof)

Анализ производительности и потребления ресурсов с помощью pprof.

## Типы профилей

- **CPU** - где программа тратит процессорное время
- **Heap** - аллокации памяти
- **Goroutine** - текущие горутины
- **Block** - где горутины блокируются
- **Mutex** - конкуренция за мьютексы
- **Threadcreate** - создание OS threads

## net/http/pprof - профилирование веб-сервера

### Включение pprof

```go
import (
    _ "net/http/pprof"
    "net/http"
)

func main() {
    // Запуск pprof сервера
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()

    // Ваш основной код...
}
```

### Доступные endpoints

```bash
# CPU profile
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.prof

# Heap profile
curl http://localhost:6060/debug/pprof/heap > heap.prof

# Goroutines
curl http://localhost:6060/debug/pprof/goroutine > goroutine.prof

# Все доступные профили
open http://localhost:6060/debug/pprof/
```

## runtime/pprof - профилирование standalone программ

### CPU Profiling

```go
import (
    "os"
    "runtime/pprof"
)

func main() {
    // Создать файл для CPU profile
    f, err := os.Create("cpu.prof")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()

    // Начать профилирование
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    // Ваш код для профилирования
    doWork()
}
```

### Heap Profiling

```go
func main() {
    doWork()

    // Создать heap profile
    f, err := os.Create("heap.prof")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()

    runtime.GC() // Запустить GC перед профилированием
    pprof.WriteHeapProfile(f)
}
```

### Goroutine Profile

```go
f, _ := os.Create("goroutine.prof")
defer f.Close()

pprof.Lookup("goroutine").WriteTo(f, 0)
```

## Анализ профилей

### Интерактивный режим

```bash
# Открыть CPU profile
go tool pprof cpu.prof

# Открыть heap profile
go tool pprof heap.prof

# Или напрямую с URL
go tool pprof http://localhost:6060/debug/pprof/heap
```

### Основные команды в pprof

```
(pprof) top          # Top функции по потреблению
(pprof) top10        # Top 10
(pprof) list funcName # Исходный код функции
(pprof) web          # Визуализация в браузере (требует graphviz)
(pprof) pdf          # Сохранить как PDF
(pprof) png          # Сохранить как PNG
(pprof) help         # Справка
```

### Web UI (рекомендуется)

```bash
# Запустить web интерфейс
go tool pprof -http=:8080 cpu.prof

# Или напрямую с сервера
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/heap

# Откроется браузер с интерактивным UI
```

## Чтение результатов

### CPU Profile

```
(pprof) top
Showing nodes accounting for 5.50s, 78.57% of 7.00s total
      flat  flat%   sum%        cum   cum%
     2.00s 28.57% 28.57%      3.50s 50.00%  main.processData
     1.50s 21.43% 50.00%      1.50s 21.43%  runtime.scanobject
     1.00s 14.29% 64.29%      1.00s 14.29%  main.compute
     0.50s  7.14% 71.43%      5.00s 71.43%  main.main
```

**Колонки:**
- `flat` - время в самой функции (без вызовов)
- `flat%` - процент от общего времени
- `sum%` - кумулятивный процент
- `cum` - время в функции + все вызовы
- `cum%` - процент кумулятивного времени

### Heap Profile

```
(pprof) top
Showing nodes accounting for 512MB, 89.47% of 573MB total
      flat  flat%   sum%        cum   cum%
   256MB 44.68% 44.68%    256MB 44.68%  main.allocateData
   128MB 22.37% 67.05%    384MB 67.05%  main.processItems
    64MB 11.18% 78.23%     64MB 11.18%  bytes.makeSlice
```

## Flame Graph

```bash
# Установить
go install github.com/google/pprof@latest

# Создать flame graph
pprof -http=:8080 cpu.prof

# В браузере выбрать VIEW -> Flame Graph
```

## Профилирование в тестах

### CPU

```bash
go test -cpuprofile=cpu.prof -bench=.
go tool pprof cpu.prof
```

### Memory

```bash
go test -memprofile=mem.prof -bench=.
go tool pprof mem.prof
```

### Блокировки

```bash
go test -blockprofile=block.prof -bench=.
go tool pprof block.prof
```

## Примеры оптимизации

### До оптимизации

```go
func processData(data []string) []string {
    result := []string{} // ❌ Без pre-allocation
    for _, item := range data {
        result = append(result, strings.ToUpper(item))
    }
    return result
}
```

Профиль показывает много времени в `growslice` (перевыделение слайса).

### После оптимизации

```go
func processData(data []string) []string {
    result := make([]string, 0, len(data)) // ✅ Pre-allocate
    for _, item := range data {
        result = append(result, strings.ToUpper(item))
    }
    return result
}
```

## Continuous Profiling

### Production профилирование

```go
import (
    _ "net/http/pprof"
)

func main() {
    // Запуск pprof на отдельном порту (не публичном!)
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()

    // Основной сервер
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

⚠️ **Безопасность:** Не открывайте pprof порт наружу!

### Периодический сбор профилей

```go
func collectProfiles() {
    ticker := time.NewTicker(10 * time.Minute)
    defer ticker.Stop()

    for range ticker.C {
        // CPU profile
        f, _ := os.Create(fmt.Sprintf("cpu-%d.prof", time.Now().Unix()))
        pprof.StartCPUProfile(f)
        time.Sleep(30 * time.Second)
        pprof.StopCPUProfile()
        f.Close()

        // Heap profile
        f, _ = os.Create(fmt.Sprintf("heap-%d.prof", time.Now().Unix()))
        runtime.GC()
        pprof.WriteHeapProfile(f)
        f.Close()
    }
}
```

## Сравнение профилей

```bash
# Собрать baseline
go tool pprof -proto cpu-before.prof > before.pb.gz

# После оптимизации
go tool pprof -proto cpu-after.prof > after.pb.gz

# Сравнить
go tool pprof -base=before.pb.gz after.pb.gz
```

## Типичные проблемы

### 1. Много времени в GC

```
(pprof) top
  30.00% runtime.gcBgMarkWorker
  15.00% runtime.scanobject
```

**Решение:** Уменьшить аллокации, использовать sync.Pool

### 2. Горячие функции

```
(pprof) top
  50.00% strings.Replace
```

**Решение:** Оптимизировать или кэшировать результаты

### 3. Утечка горутин

```
(pprof) top goroutine
  10000 goroutines stuck in channel receive
```

**Решение:** Проверить закрытие каналов, использовать context

### 4. Контention на мьютексах

```bash
go test -mutexprofile=mutex.prof
go tool pprof mutex.prof
```

**Решение:** Уменьшить критические секции, использовать RWMutex

## trace - детальная трассировка

```go
import "runtime/trace"

func main() {
    f, _ := os.Create("trace.out")
    defer f.Close()

    trace.Start(f)
    defer trace.Stop()

    // Ваш код
    doWork()
}
```

```bash
# Анализ trace
go tool trace trace.out
```

Trace показывает:
- Timeline горутин
- GC events
- Network/Syscall blocking
- Processor utilization

## Best Practices

1. ✅ Профилируйте перед оптимизацией (не гадайте!)
2. ✅ Профилируйте на реальных данных
3. ✅ Используйте web UI для удобства
4. ✅ Сравнивайте до/после оптимизации
5. ✅ Профилируйте в production (с осторожностью)
6. ✅ Собирайте CPU profile на 30+ секунд
7. ✅ Запускайте GC перед heap профилем
8. ❌ Не оптимизируйте без измерений
9. ❌ Не открывайте pprof порт наружу
10. ❌ Не профилируйте слишком короткое время

## Инструменты

```bash
# pprof
go tool pprof

# trace
go tool trace

# benchstat (сравнение)
go install golang.org/x/perf/cmd/benchstat@latest

# graphviz (для визуализации)
brew install graphviz  # macOS
apt install graphviz   # Linux
```

## Checklist для production

- [ ] pprof endpoint доступен только внутренне
- [ ] Логируются периодические CPU/heap profiles
- [ ] Мониторинг goroutine count
- [ ] Алерты на рост памяти
- [ ] Retention policy для старых профилей
- [ ] Документация как собирать и анализировать

## Связанные темы

- [[Go - Бенчмаркинг]]
- [[Go - Garbage Collector]]
- [[Go - Управление памятью]]
- [[Go - Горутины (goroutines)]]
