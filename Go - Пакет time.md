# Go - Пакет time

Работа с датой и временем в Go.

## Time

### Текущее время

```go
now := time.Now()
fmt.Println(now) // 2024-01-15 14:30:45.123456 +0300 MSK
```

### Создание времени

```go
// Конкретная дата
t := time.Date(2024, time.January, 15, 14, 30, 0, 0, time.UTC)

// UTC
utc := time.Now().UTC()

// Unix timestamp
unix := time.Unix(1705323045, 0)

// Парсинг строки
t, err := time.Parse("2006-01-02", "2024-01-15")
t, err = time.Parse(time.RFC3339, "2024-01-15T14:30:00Z")
```

### Форматирование

```go
now := time.Now()

// Стандартные форматы
now.Format(time.RFC3339)     // "2024-01-15T14:30:45Z"
now.Format(time.Kitchen)     // "2:30PM"

// Кастомный формат (reference time: Mon Jan 2 15:04:05 MST 2006)
now.Format("2006-01-02")           // "2024-01-15"
now.Format("02/01/2006")           // "15/01/2024"
now.Format("15:04:05")             // "14:30:45"
now.Format("January 2, 2006")      // "January 15, 2024"
now.Format("Mon, 02 Jan 2006")     // "Mon, 15 Jan 2024"
```

**⚠️ Используйте reference time: `Mon Jan 2 15:04:05 MST 2006`**

## Duration

### Создание

```go
d := 5 * time.Second           // 5s
d = 100 * time.Millisecond     // 100ms
d = 2 * time.Hour + 30 * time.Minute // 2h30m
d = time.ParseDuration("1h30m") // 1h30m
```

### Операции

```go
// Добавление
future := time.Now().Add(24 * time.Hour)

// Вычитание
past := time.Now().Add(-1 * time.Hour)

// Разница
duration := time2.Sub(time1)

// Компоненты
d := 1*time.Hour + 30*time.Minute + 45*time.Second
d.Hours()      // 1.5125
d.Minutes()    // 90.75
d.Seconds()    // 5445
d.Milliseconds() // 5445000
```

## Сравнение

```go
t1 := time.Now()
t2 := time.Now().Add(1 * time.Hour)

t1.Before(t2) // true
t1.After(t2)  // false
t1.Equal(t2)  // false

// Для сравнения используйте Before/After, не == !
```

## Sleep и Ticker

### Sleep

```go
time.Sleep(2 * time.Second) // Спит 2 секунды
```

### Timer

```go
timer := time.NewTimer(2 * time.Second)
<-timer.C // Блокируется на 2 секунды
fmt.Println("Timer fired")

// Или с select
select {
case <-timer.C:
    fmt.Println("Timer fired")
}
```

### Ticker

```go
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()

for i := 0; i < 5; i++ {
    <-ticker.C
    fmt.Println("Tick", i)
}
```

### After / AfterFunc

```go
// After - channel который получит значение через duration
select {
case <-time.After(5 * time.Second):
    fmt.Println("Timeout")
case result := <-ch:
    fmt.Println("Got result:", result)
}

// AfterFunc - вызовет функцию через duration
time.AfterFunc(2*time.Second, func() {
    fmt.Println("2 seconds passed")
})
```

## Timezone

```go
// Загрузить timezone
loc, err := time.LoadLocation("America/New_York")
if err != nil {
    log.Fatal(err)
}

// Конвертировать
t := time.Now()
ny := t.In(loc)

// UTC
utc := t.UTC()

// Local
local := t.Local()
```

## Монотонное время

```go
// Для измерения интервалов используйте монотонные часы
start := time.Now()
// операция...
elapsed := time.Since(start)
fmt.Printf("Took %v\n", elapsed)

// Или
duration := time.Since(start)
```

## Примеры паттернов

### Таймаут

```go
select {
case result := <-ch:
    fmt.Println(result)
case <-time.After(5 * time.Second):
    fmt.Println("Timeout")
}
```

### Периодическая задача

```go
ticker := time.NewTicker(10 * time.Second)
defer ticker.Stop()

for {
    select {
    case <-ticker.C:
        doWork()
    case <-ctx.Done():
        return
    }
}
```

### Retry с backoff

```go
func retryWithBackoff(fn func() error, maxRetries int) error {
    backoff := time.Second

    for i := 0; i < maxRetries; i++ {
        if err := fn(); err == nil {
            return nil
        }

        if i < maxRetries-1 {
            time.Sleep(backoff)
            backoff *= 2 // Exponential backoff
        }
    }

    return errors.New("max retries exceeded")
}
```

### Debounce

```go
func debounce(d time.Duration, fn func()) func() {
    var timer *time.Timer
    var mu sync.Mutex

    return func() {
        mu.Lock()
        defer mu.Unlock()

        if timer != nil {
            timer.Stop()
        }

        timer = time.AfterFunc(d, fn)
    }
}
```

## Best Practices

1. ✅ Используйте time.Since() для измерения интервалов
2. ✅ Всегда Stop() таймеры и тикеры
3. ✅ Используйте context для таймаутов вместо time.After в продакшене
4. ✅ Храните время в UTC
5. ✅ Используйте Before/After для сравнения, не ==
6. ❌ Не используйте time.Sleep в продакшен коде для ожидания (используйте context)

## Связанные темы

- [[Go - Context]]
- [[Go - Каналы (channels)]]
- [[Go - Select statement]]
