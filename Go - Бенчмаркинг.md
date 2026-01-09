# Go - Бенчмаркинг

Измерение производительности кода с помощью встроенного фреймворка бенчмарков.

## Базовый бенчмарк

```go
// fibonacci_test.go
package main

import "testing"

func BenchmarkFibonacci(b *testing.B) {
    for i := 0; i < b.N; i++ {
        fibonacci(20)
    }
}

// Запуск: go test -bench=.
// Вывод: BenchmarkFibonacci-8   30000   50234 ns/op
```

**b.N** - количество итераций, подбирается автоматически для стабильных результатов.

## ReportAllocs - измерение аллокаций

```go
func BenchmarkStringConcat(b *testing.B) {
    b.ReportAllocs() // Показать аллокации

    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "text"
        }
    }
}

// Вывод:
// BenchmarkStringConcat-8  10000  112584 ns/op  503824 B/op  99 allocs/op
//                                                ^^^^^^^^      ^^^^^^^^^^^^
//                                                байт/op       аллокаций/op
```

## Сброс таймера

```go
func BenchmarkWithSetup(b *testing.B) {
    // Дорогая подготовка
    data := generateLargeDataset()

    b.ResetTimer() // Сбросить таймер после setup

    for i := 0; i < b.N; i++ {
        process(data)
    }
}
```

## StopTimer / StartTimer

```go
func BenchmarkWithCleanup(b *testing.B) {
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        data := setup() // Не измеряется
        b.StartTimer()

        process(data)
    }
}
```

## Sub-benchmarks

```go
func BenchmarkSort(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            data := generateData(size)
            b.ResetTimer()

            for i := 0; i < b.N; i++ {
                sort.Ints(data)
            }
        })
    }
}

// Запуск конкретного: go test -bench=Sort/size=1000
```

## Сравнение реализаций

```go
func BenchmarkStringConcatPlus(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        s := ""
        for j := 0; j < 100; j++ {
            s += "text"
        }
    }
}

func BenchmarkStringConcatBuilder(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var builder strings.Builder
        for j := 0; j < 100; j++ {
            builder.WriteString("text")
        }
        _ = builder.String()
    }
}

// Сравнение:
// BenchmarkStringConcatPlus-8      10000  112584 ns/op  503824 B/op  99 allocs/op
// BenchmarkStringConcatBuilder-8  200000    6543 ns/op     856 B/op   6 allocs/op
```

## Запуск бенчмарков

```bash
# Все бенчмарки
go test -bench=.

# Конкретный бенчмарк
go test -bench=BenchmarkFibonacci

# С паттерном
go test -bench=String

# Запустить N раз для стабильности
go test -bench=. -benchtime=10s

# Указать количество итераций
go test -bench=. -benchtime=1000x

# С профилированием CPU
go test -bench=. -cpuprofile=cpu.prof

# С профилированием памяти
go test -bench=. -memprofile=mem.prof

# Несколько раз (benchstat)
go test -bench=. -count=10 > bench.txt
```

## benchstat - сравнение результатов

```bash
# Установка
go install golang.org/x/perf/cmd/benchstat@latest

# Запуск old version
go test -bench=. -count=10 > old.txt

# Изменения в коде...

# Запуск new version
go test -bench=. -count=10 > new.txt

# Сравнение
benchstat old.txt new.txt

# Вывод:
# name        old time/op  new time/op  delta
# Fibonacci   50.2µs ± 2%  30.1µs ± 1%  -40.04%  (p=0.000 n=10+10)
```

## Параллельные бенчмарки

```go
func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            // Код для бенчмарка
            doWork()
        }
    })
}

// Тестирует с разным количеством горутин: -cpu=1,2,4,8
```

## Примеры оптимизаций

### String Builder vs concatenation

```go
func BenchmarkConcatPlus(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var s string
        for j := 0; j < 100; j++ {
            s += "text"
        }
    }
}
// 112584 ns/op  503824 B/op  99 allocs/op

func BenchmarkConcatBuilder(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var builder strings.Builder
        builder.Grow(400) // Pre-allocate
        for j := 0; j < 100; j++ {
            builder.WriteString("text")
        }
        _ = builder.String()
    }
}
// 1234 ns/op  512 B/op  1 allocs/op  ✅ Гораздо быстрее!
```

### Pre-allocation

```go
func BenchmarkAppendNoPrealloc(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        s := []int{}
        for j := 0; j < 1000; j++ {
            s = append(s, j)
        }
    }
}
// 45678 ns/op  57344 B/op  11 allocs/op

func BenchmarkAppendPrealloc(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        s := make([]int, 0, 1000)
        for j := 0; j < 1000; j++ {
            s = append(s, j)
        }
    }
}
// 8765 ns/op  8192 B/op  1 allocs/op  ✅ Быстрее и меньше аллокаций
```

### sync.Pool

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func BenchmarkWithoutPool(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        buf := new(bytes.Buffer)
        buf.WriteString("data")
        _ = buf.String()
    }
}
// 234 ns/op  128 B/op  2 allocs/op

func BenchmarkWithPool(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        buf := bufferPool.Get().(*bytes.Buffer)
        buf.Reset()
        buf.WriteString("data")
        _ = buf.String()
        bufferPool.Put(buf)
    }
}
// 123 ns/op  64 B/op  1 allocs/op  ✅ Меньше аллокаций
```

## Table-driven бенчмарки

```go
func BenchmarkFibonacci(b *testing.B) {
    testCases := []struct {
        name string
        n    int
    }{
        {"n=10", 10},
        {"n=20", 20},
        {"n=30", 30},
    }

    for _, tc := range testCases {
        b.Run(tc.name, func(b *testing.B) {
            for i := 0; i < b.N; i++ {
                fibonacci(tc.n)
            }
        })
    }
}
```

## Профилирование в бенчмарках

```bash
# CPU profile
go test -bench=. -cpuprofile=cpu.prof
go tool pprof cpu.prof

# Memory profile
go test -bench=. -memprofile=mem.prof
go tool pprof mem.prof

# Trace
go test -bench=. -trace=trace.out
go tool trace trace.out
```

## Интерпретация результатов

```
BenchmarkFibonacci-8    30000    50234 ns/op    2048 B/op    15 allocs/op
│                  │    │        │               │            │
│                  │    │        │               │            └─ аллокаций за операцию
│                  │    │        │               └─ байт выделено за операцию
│                  │    │        └─ наносекунд за операцию
│                  │    └─ количество итераций
│                  └─ количество CPU (GOMAXPROCS)
└─ имя бенчмарка
```

## Best Practices

1. ✅ Используйте `b.ReportAllocs()` для отслеживания аллокаций
2. ✅ `b.ResetTimer()` после дорогой подготовки
3. ✅ Sub-benchmarks для сравнения вариантов
4. ✅ Запускайте несколько раз с benchstat для точности
5. ✅ Бенчмаркте на реальных данных, близких к production
6. ✅ Профилируйте если результаты неожиданные
7. ✅ Измеряйте ДО оптимизации (baseline)
8. ❌ Не оптимизируйте без измерений
9. ❌ Не делайте микрооптимизации без веской причины

## Типичные ошибки

### 1. Компилятор оптимизирует код

```go
// ❌ Компилятор может удалить этот код
func BenchmarkBad(b *testing.B) {
    for i := 0; i < b.N; i++ {
        result := compute()
        // result не используется - может быть удалено!
    }
}

// ✅ Сохраните результат
var Result int

func BenchmarkGood(b *testing.B) {
    var r int
    for i := 0; i < b.N; i++ {
        r = compute()
    }
    Result = r // Предотвращает оптимизацию
}
```

### 2. Забыли b.ResetTimer

```go
// ❌ Setup включен в измерение
func BenchmarkBad(b *testing.B) {
    data := expensiveSetup() // Измеряется!
    for i := 0; i < b.N; i++ {
        process(data)
    }
}

// ✅ Setup исключен
func BenchmarkGood(b *testing.B) {
    data := expensiveSetup()
    b.ResetTimer() // Сброс после setup
    for i := 0; i < b.N; i++ {
        process(data)
    }
}
```

### 3. Общие данные в параллельных бенчмарках

```go
// ❌ Race condition!
func BenchmarkParallelBad(b *testing.B) {
    counter := 0
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            counter++ // Не потокобезопасно!
        }
    })
}

// ✅ Локальные данные
func BenchmarkParallelGood(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        counter := 0
        for pb.Next() {
            counter++ // Локальная переменная
        }
    })
}
```

## Связанные темы

- [[Go - Профилирование (pprof)]]
- [[Go - Управление памятью]]
- [[Go - Unit тестирование]]
