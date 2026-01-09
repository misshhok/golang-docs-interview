# FSM (Конечные автоматы)

FSM (Finite State Machine, конечный автомат) — это модель вычислений, в которой объект может находиться в одном из конечного числа состояний и переходить между ними в ответ на события. В бизнес-приложениях FSM используется для моделирования жизненного цикла объектов.

## Основные понятия

### Компоненты FSM

**Состояния (States)** — набор всех возможных состояний системы

**События (Events)** — триггеры, вызывающие переходы

**Переходы (Transitions)** — правила перехода из состояния в состояние при событии

**Действия (Actions)** — операции, выполняемые при переходе

## Применение в реальных задачах

### Примеры использования

✅ **Статусы заказа:** Создан → Оплачен → Отправлен → Доставлен
✅ **Workflow документов:** Черновик → На согласовании → Утвержден → Опубликован
✅ **Платежные транзакции:** Инициирован → Обработка → Успех/Отклонен
✅ **Состояния пользователя:** Гость → Зарегистрирован → Активен → Заблокирован

## Простая реализация FSM в Go

### Базовый пример - Заказ

```go
package main

import (
    "fmt"
)

// State - состояние заказа
type OrderState string

const (
    OrderPending    OrderState = "pending"
    OrderPaid       OrderState = "paid"
    OrderShipped    OrderState = "shipped"
    OrderDelivered  OrderState = "delivered"
    OrderCancelled  OrderState = "cancelled"
)

// Event - событие
type OrderEvent string

const (
    EventPay      OrderEvent = "pay"
    EventShip     OrderEvent = "ship"
    EventDeliver  OrderEvent = "deliver"
    EventCancel   OrderEvent = "cancel"
)

// Order - заказ с FSM
type Order struct {
    ID    int
    State OrderState
}

// Transition - попытка перехода
func (o *Order) Transition(event OrderEvent) error {
    // Определяем разрешенные переходы
    transitions := map[OrderState]map[OrderEvent]OrderState{
        OrderPending: {
            EventPay:    OrderPaid,
            EventCancel: OrderCancelled,
        },
        OrderPaid: {
            EventShip:   OrderShipped,
            EventCancel: OrderCancelled,
        },
        OrderShipped: {
            EventDeliver: OrderDelivered,
        },
        // OrderDelivered и OrderCancelled — финальные состояния
    }

    // Проверяем возможен ли переход
    possibleTransitions, ok := transitions[o.State]
    if !ok {
        return fmt.Errorf("no transitions from state %s", o.State)
    }

    nextState, ok := possibleTransitions[event]
    if !ok {
        return fmt.Errorf("cannot transition from %s on event %s", o.State, event)
    }

    // Выполняем переход
    oldState := o.State
    o.State = nextState

    fmt.Printf("Order %d: %s -> %s\n", o.ID, oldState, nextState)

    return nil
}

func main() {
    order := &Order{ID: 1, State: OrderPending}

    // Успешные переходы
    order.Transition(EventPay)     // pending -> paid
    order.Transition(EventShip)    // paid -> shipped
    order.Transition(EventDeliver) // shipped -> delivered

    // Ошибка: нельзя отменить доставленный заказ
    err := order.Transition(EventCancel)
    if err != nil {
        fmt.Println("Error:", err)
    }
}
```

## FSM с действиями (Actions)

```go
package fsm

import "fmt"

type State string
type Event string

// Action - действие при переходе
type Action func() error

// Transition - переход с действием
type Transition struct {
    From   State
    Event  Event
    To     State
    Action Action // Опционально
}

// FSM - конечный автомат
type FSM struct {
    current     State
    transitions map[State]map[Event]Transition
}

func NewFSM(initial State) *FSM {
    return &FSM{
        current:     initial,
        transitions: make(map[State]map[Event]Transition),
    }
}

// AddTransition - добавить переход
func (f *FSM) AddTransition(from State, event Event, to State, action Action) {
    if f.transitions[from] == nil {
        f.transitions[from] = make(map[Event]Transition)
    }

    f.transitions[from][event] = Transition{
        From:   from,
        Event:  event,
        To:     to,
        Action: action,
    }
}

// CurrentState - текущее состояние
func (f *FSM) CurrentState() State {
    return f.current
}

// Can - проверка возможности перехода
func (f *FSM) Can(event Event) bool {
    _, ok := f.transitions[f.current][event]
    return ok
}

// Trigger - вызвать событие
func (f *FSM) Trigger(event Event) error {
    transitions, ok := f.transitions[f.current]
    if !ok {
        return fmt.Errorf("no transitions from state %s", f.current)
    }

    transition, ok := transitions[event]
    if !ok {
        return fmt.Errorf("cannot transition from %s on event %s", f.current, event)
    }

    // Выполняем действие (если есть)
    if transition.Action != nil {
        if err := transition.Action(); err != nil {
            return fmt.Errorf("action failed: %w", err)
        }
    }

    // Переходим в новое состояние
    f.current = transition.To

    return nil
}
```

### Использование FSM с действиями

```go
package main

import (
    "fmt"
    "myapp/fsm"
)

const (
    StateDraft     fsm.State = "draft"
    StateReview    fsm.State = "review"
    StateApproved  fsm.State = "approved"
    StatePublished fsm.State = "published"
    StateRejected  fsm.State = "rejected"
)

const (
    EventSubmit  fsm.Event = "submit"
    EventApprove fsm.Event = "approve"
    EventReject  fsm.Event = "reject"
    EventPublish fsm.Event = "publish"
    EventRevise  fsm.Event = "revise"
)

type Document struct {
    ID      int
    Title   string
    Content string
    FSM     *fsm.FSM
}

func NewDocument(id int, title string) *Document {
    doc := &Document{
        ID:    id,
        Title: title,
        FSM:   fsm.NewFSM(StateDraft),
    }

    // Настраиваем переходы с действиями
    doc.FSM.AddTransition(StateDraft, EventSubmit, StateReview, func() error {
        fmt.Printf("Document %d submitted for review\n", doc.ID)
        // Отправить уведомление ревьюерам
        return nil
    })

    doc.FSM.AddTransition(StateReview, EventApprove, StateApproved, func() error {
        fmt.Printf("Document %d approved\n", doc.ID)
        // Логирование
        return nil
    })

    doc.FSM.AddTransition(StateReview, EventReject, StateRejected, func() error {
        fmt.Printf("Document %d rejected\n", doc.ID)
        // Уведомить автора
        return nil
    })

    doc.FSM.AddTransition(StateApproved, EventPublish, StatePublished, func() error {
        fmt.Printf("Document %d published\n", doc.ID)
        // Публикация в системе
        return nil
    })

    doc.FSM.AddTransition(StateRejected, EventRevise, StateDraft, func() error {
        fmt.Printf("Document %d revised\n", doc.ID)
        return nil
    })

    return doc
}

func (d *Document) Submit() error {
    return d.FSM.Trigger(EventSubmit)
}

func (d *Document) Approve() error {
    return d.FSM.Trigger(EventApprove)
}

func (d *Document) Reject() error {
    return d.FSM.Trigger(EventReject)
}

func (d *Document) Publish() error {
    return d.FSM.Trigger(EventPublish)
}

func main() {
    doc := NewDocument(1, "Важный документ")

    // Workflow документа
    doc.Submit()  // draft -> review
    doc.Approve() // review -> approved
    doc.Publish() // approved -> published

    // Проверка возможности перехода
    if doc.FSM.Can(EventReject) {
        doc.Reject()
    } else {
        fmt.Println("Cannot reject published document")
    }
}
```

## FSM для платежных транзакций

```go
package payment

import (
    "fmt"
    "time"
)

type PaymentState string

const (
    PaymentInitiated  PaymentState = "initiated"
    PaymentProcessing PaymentState = "processing"
    PaymentSucceeded  PaymentState = "succeeded"
    PaymentFailed     PaymentState = "failed"
    PaymentRefunded   PaymentState = "refunded"
)

type PaymentEvent string

const (
    EventProcess PaymentEvent = "process"
    EventSucceed PaymentEvent = "succeed"
    EventFail    PaymentEvent = "fail"
    EventRefund  PaymentEvent = "refund"
)

type Payment struct {
    ID            string
    Amount        float64
    State         PaymentState
    FailureReason string
    CreatedAt     time.Time
    UpdatedAt     time.Time
}

func (p *Payment) Transition(event PaymentEvent) error {
    transitions := map[PaymentState]map[PaymentEvent]PaymentState{
        PaymentInitiated: {
            EventProcess: PaymentProcessing,
        },
        PaymentProcessing: {
            EventSucceed: PaymentSucceeded,
            EventFail:    PaymentFailed,
        },
        PaymentSucceeded: {
            EventRefund: PaymentRefunded,
        },
    }

    possibleTransitions, ok := transitions[p.State]
    if !ok {
        return fmt.Errorf("no transitions from state %s", p.State)
    }

    nextState, ok := possibleTransitions[event]
    if !ok {
        return fmt.Errorf("cannot %s from state %s", event, p.State)
    }

    // Выполняем действия в зависимости от события
    switch event {
    case EventProcess:
        fmt.Printf("Processing payment %s...\n", p.ID)
        // Вызов платежного шлюза
    case EventSucceed:
        fmt.Printf("Payment %s succeeded\n", p.ID)
        // Отправка чека
    case EventFail:
        fmt.Printf("Payment %s failed\n", p.ID)
        // Логирование причины
    case EventRefund:
        fmt.Printf("Payment %s refunded\n", p.ID)
        // Возврат средств
    }

    p.State = nextState
    p.UpdatedAt = time.Now()

    return nil
}
```

## FSM с сохранением в БД

```go
package order

import (
    "context"
    "database/sql"
    "fmt"
    "time"
)

type Order struct {
    ID        int
    State     OrderState
    CreatedAt time.Time
    UpdatedAt time.Time
}

type OrderService struct {
    db *sql.DB
}

// Transition с сохранением в БД
func (s *OrderService) Transition(ctx context.Context, orderID int, event OrderEvent) error {
    // Начинаем транзакцию
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // Получаем текущее состояние
    var order Order
    err = tx.QueryRowContext(ctx,
        "SELECT id, state, created_at, updated_at FROM orders WHERE id = $1 FOR UPDATE",
        orderID,
    ).Scan(&order.ID, &order.State, &order.CreatedAt, &order.UpdatedAt)

    if err != nil {
        return fmt.Errorf("failed to get order: %w", err)
    }

    // Проверяем возможность перехода
    transitions := getTransitions()
    possibleTransitions, ok := transitions[order.State]
    if !ok {
        return fmt.Errorf("no transitions from state %s", order.State)
    }

    nextState, ok := possibleTransitions[event]
    if !ok {
        return fmt.Errorf("cannot transition from %s on event %s", order.State, event)
    }

    // Обновляем состояние
    _, err = tx.ExecContext(ctx,
        "UPDATE orders SET state = $1, updated_at = $2 WHERE id = $3",
        nextState, time.Now(), orderID,
    )

    if err != nil {
        return fmt.Errorf("failed to update order state: %w", err)
    }

    // Логируем переход
    _, err = tx.ExecContext(ctx,
        "INSERT INTO order_state_history (order_id, from_state, to_state, event, created_at) VALUES ($1, $2, $3, $4, $5)",
        orderID, order.State, nextState, event, time.Now(),
    )

    if err != nil {
        return fmt.Errorf("failed to log transition: %w", err)
    }

    // Коммитим транзакцию
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("failed to commit: %w", err)
    }

    return nil
}
```

## Визуализация FSM

```
Состояния заказа:

    ┌─────────┐
    │ Pending │
    └────┬────┘
         │
    ┌────▼────┐     ┌───────────┐
    │   Pay   │────▶│ Cancelled │
    └────┬────┘     └───────────┘
         │
    ┌────▼────┐
    │  Paid   │
    └────┬────┘
         │
    ┌────▼────┐
    │ Shipped │
    └────┬────┘
         │
    ┌────▼─────┐
    │Delivered │
    └──────────┘
```

## Best Practices

### ✅ Правильно

```go
// Использовать типизированные константы
type State string
const (
    StateNew State = "new"
    StateDone State = "done"
)

// Явно определять таблицу переходов
transitions := map[State]map[Event]State{
    StateNew: {
        EventStart: StateInProgress,
    },
}

// Логировать все переходы
fmt.Printf("Transition: %s -[%s]-> %s\n", from, event, to)

// Использовать транзакции для атомарности
tx.Begin()
// Обновить состояние + логировать
tx.Commit()
```

### ❌ Неправильно

```go
// Использовать строки напрямую
order.State = "paid" // Опечатки не отловятся

// Проверять переходы через if-else
if state == "new" && event == "pay" {
    state = "paid"
} else if state == "paid" && event == "ship" {
    state = "shipped"
}
// Сложно поддерживать

// Не логировать переходы
// Невозможно отследить историю изменений
```

## Библиотеки для FSM в Go

**looplab/fsm** — популярная библиотека для FSM
```bash
go get github.com/looplab/fsm
```

```go
import "github.com/looplab/fsm"

orderFSM := fsm.NewFSM(
    "pending",
    fsm.Events{
        {Name: "pay", Src: []string{"pending"}, Dst: "paid"},
        {Name: "ship", Src: []string{"paid"}, Dst: "shipped"},
        {Name: "deliver", Src: []string{"shipped"}, Dst: "delivered"},
    },
    fsm.Callbacks{
        "enter_paid": func(e *fsm.Event) {
            fmt.Println("Order is paid!")
        },
    },
)

orderFSM.Event("pay")
orderFSM.Event("ship")
```

## Вопросы с собеседований

**Вопрос:** Когда использовать FSM вместо простых if-else?

**Ответ:** FSM подходит когда:
- Много состояний (>3-4)
- Сложная логика переходов
- Нужна история переходов
- Требуется валидация переходов
- Состояние имеет бизнес-значение (статусы заказа, workflow)

**Вопрос:** Как обеспечить thread-safety FSM?

**Ответ:** Использовать mutex при изменении состояния или сохранять состояние в БД с использованием `FOR UPDATE` lock в транзакции.

**Вопрос:** Как отличить FSM от State паттерна?

**Ответ:** State паттерн — это ООП реализация FSM, где каждое состояние — это отдельный класс/объект. FSM — это общая концепция, которая может быть реализована разными способами, включая State паттерн.

## Связанные темы

- [[Паттерны проектирования в Go]]
- [[PostgreSQL - Блокировки и изоляция транзакций]]
- [[GORM - ORM для Go]]
- [[Repository паттерн]]