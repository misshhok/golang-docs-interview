# Git - Gitflow и trunk-based development

Gitflow и Trunk-based development — два популярных подхода к организации работы с ветками в Git. Выбор зависит от размера команды, частоты релизов и процессов разработки.

## Gitflow

Gitflow — это модель ветвления с несколькими долгоживущими ветками и строгими правилами их использования. Разработана Vincent Driessen в 2010 году.

### Структура веток в Gitflow

```
main (production)      ●───────────●───────────●
                                  /           /
develop                ●───●───●───────●───●
                        \     /         \
feature/*               ●───●            ●───●
                                        /
hotfix/*                              ●───●
                                     /
release/*                          ●───●
```

### Типы веток

#### 1. main (master)

- Основная ветка
- Содержит только production-ready код
- Каждый коммит — это новая версия в продакшене
- Запрещены прямые коммиты

```bash
# main всегда стабильная
git checkout main
git pull
```

#### 2. develop

- Ветка для интеграции фич
- Содержит код для следующего релиза
- Feature ветки создаются из develop и мержатся обратно

```bash
# Создать develop ветку
git checkout -b develop main
git push -u origin develop
```

#### 3. feature/* (короткоживущие)

- Для разработки новых фич
- Создаются из develop
- Мержатся обратно в develop

```bash
# Создать feature ветку
git checkout -b feature/user-authentication develop

# Работаем над фичей
git add .
git commit -m "Add JWT authentication"

# Мержим обратно в develop
git checkout develop
git merge --no-ff feature/user-authentication
git branch -d feature/user-authentication
git push origin develop
```

#### 4. release/* (короткоживущие)

- Подготовка к релизу
- Создаются из develop
- Мержатся в main И develop
- Только багфиксы, никаких новых фич

```bash
# Создать release ветку
git checkout -b release/1.2.0 develop

# Исправить баги, обновить версию
git commit -am "Bump version to 1.2.0"

# Мержим в main
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Version 1.2.0"

# Мержим обратно в develop
git checkout develop
git merge --no-ff release/1.2.0

# Удаляем release ветку
git branch -d release/1.2.0
```

#### 5. hotfix/* (короткоживущие)

- Срочные исправления в продакшене
- Создаются из main
- Мержатся в main И develop

```bash
# Создать hotfix ветку
git checkout -b hotfix/1.2.1 main

# Исправить критический баг
git commit -am "Fix critical security vulnerability"

# Мержим в main
git checkout main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix 1.2.1"

# Мержим в develop
git checkout develop
git merge --no-ff hotfix/1.2.1

# Удаляем hotfix ветку
git branch -d hotfix/1.2.1
```

### Практический workflow в Gitflow

```bash
# 1. Начать новую фичу
git checkout develop
git pull
git checkout -b feature/payment-gateway

# 2. Разработка
git add .
git commit -m "Implement Stripe integration"
git push -u origin feature/payment-gateway

# 3. Слияние в develop (через Pull Request на GitHub)
git checkout develop
git merge --no-ff feature/payment-gateway
git push origin develop
git branch -d feature/payment-gateway

# 4. Подготовка релиза
git checkout -b release/2.0.0 develop
# Исправления багов, обновление версии
git commit -am "Prepare release 2.0.0"

# 5. Релиз
git checkout main
git merge --no-ff release/2.0.0
git tag -a v2.0.0 -m "Release 2.0.0"
git push origin main --tags

# 6. Вернуть изменения в develop
git checkout develop
git merge --no-ff release/2.0.0
git push origin develop
git branch -d release/2.0.0
```

### Преимущества Gitflow

✅ **Чёткая структура** — понятно, где какой код
✅ **Поддержка нескольких версий** — можно поддерживать старые релизы
✅ **Изоляция продакшена** — main всегда стабильная
✅ **Хорошо для scheduled releases** — релизы раз в месяц/квартал

### Недостатки Gitflow

❌ **Сложность** — много веток, сложные merge
❌ **Медленная интеграция** — фичи долго не попадают в main
❌ **Merge conflicts** — много долгоживущих веток = больше конфликтов
❌ **Не подходит для CD** — Continuous Deployment требует простоты

## Trunk-based Development

Trunk-based development — упрощённый подход с одной основной веткой (trunk/main) и короткоживущими feature ветками.

### Структура

```
main (trunk)    ●───●───●───●───●───●───●───●───●
                 \   \ /   /   /   /
feature/*         ●───●   ●   ●   ●  (короткие ветки)
```

### Принципы

1. **Одна основная ветка** — main (trunk)
2. **Короткие feature ветки** — живут 1-2 дня максимум
3. **Частые коммиты в main** — минимум раз в день
4. **Feature flags** — новая функциональность скрыта за флагами
5. **Automated testing** — CI/CD проверяет каждый коммит

### Workflow в Trunk-based

```bash
# 1. Создать короткую feature ветку
git checkout -b add-logging
# Работа на несколько часов, максимум 1-2 дня

# 2. Частые коммиты
git add .
git commit -m "Add structured logging"
git push -u origin add-logging

# 3. Быстрый merge (в тот же день)
# Через Pull Request или напрямую
git checkout main
git pull
git merge --ff-only add-logging
git push origin main
git branch -d add-logging

# 4. Если фича не готова — используем feature flag
```

### Feature Flags

Позволяют деплоить незаконченную функциональность, скрывая её от пользователей.

```go
// Пример feature flag в коде
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    if featureFlags.IsEnabled("new_payment_flow") {
        // Новая логика (в разработке)
        handleNewPayment(w, r)
    } else {
        // Старая логика (работающая)
        handleOldPayment(w, r)
    }
}
```

```bash
# Feature flag в конфигурации
NEW_PAYMENT_FLOW=false  # для продакшена
NEW_PAYMENT_FLOW=true   # для тестирования
```

### Преимущества Trunk-based

✅ **Простота** — одна ветка, понятный процесс
✅ **Быстрая интеграция** — изменения сразу в main
✅ **Меньше конфликтов** — короткие ветки = меньше расхождений
✅ **Подходит для CI/CD** — код всегда готов к деплою
✅ **Быстрая обратная связь** — проблемы выявляются раньше

### Недостатки Trunk-based

❌ **Требует дисциплины** — нужны хорошие тесты и практики
❌ **Нужен CI/CD** — без автоматизации не работает
❌ **Feature flags** — добавляет сложность в код
❌ **Не подходит для долгих фич** — большие изменения сложно интегрировать

## Сравнение

| Критерий | Gitflow | Trunk-based |
|----------|---------|-------------|
| **Основные ветки** | main, develop | main |
| **Feature ветки** | Долгоживущие (недели) | Короткие (дни) |
| **Частота слияний** | Редко (спринт/релиз) | Часто (каждый день) |
| **Сложность** | Высокая | Низкая |
| **CI/CD** | Сложная настройка | Естественная поддержка |
| **Размер команды** | Любой | Лучше для малых/средних |
| **Частота релизов** | Редкие (месяц/квартал) | Частые (день/неделя) |
| **Конфликты** | Частые, сложные | Редкие, простые |

## Когда использовать Gitflow

✅ **Scheduled releases** — релизы по расписанию (раз в месяц/квартал)
✅ **Поддержка нескольких версий** — нужно поддерживать старые релизы
✅ **Строгий процесс** — regulated industries (банки, медицина)
✅ **Большая команда** — много разработчиков, нужна структура
✅ **Desktop/mobile приложения** — длительный процесс релиза

**Примеры:** Банковское ПО, Enterprise системы, desktop приложения

## Когда использовать Trunk-based

✅ **Continuous Deployment** — деплой несколько раз в день
✅ **SaaS приложения** — один production environment
✅ **Быстрые итерации** — нужна скорость разработки
✅ **Хороший CI/CD** — автоматизированное тестирование
✅ **Feature flags** — можно использовать флаги для скрытия фич

**Примеры:** Стартапы, SaaS продукты, веб-сервисы

## Гибридные подходы

Многие команды используют упрощённую версию Gitflow:

### GitHub Flow

Упрощённая альтернатива Gitflow:

```
main    ●───●───●───●───●
         \     /   \   /
feature   ●───●     ●───●
```

**Правила:**
1. main всегда deployable
2. Создать feature ветку из main
3. Регулярно push в feature ветку
4. Открыть Pull Request
5. После review смержить в main
6. Deploy сразу после merge

### GitLab Flow

Добавляет environment ветки к GitHub Flow:

```
main        ●───●───●───●
             \   \   \
pre-prod      ●───●───●
               \   \
production      ●───●
```

## Практические рекомендации

### Для Gitflow

```bash
# Установить git-flow extensions
# macOS:
brew install git-flow

# Инициализация
git flow init

# Начать feature
git flow feature start user-auth

# Закончить feature
git flow feature finish user-auth

# Начать release
git flow release start 1.2.0

# Закончить release
git flow release finish 1.2.0

# Hotfix
git flow hotfix start 1.2.1
git flow hotfix finish 1.2.1
```

### Для Trunk-based

```bash
# Короткие ветки
git checkout -b quick-fix
# Работа < 1 дня
git push -u origin quick-fix

# Создать PR и смержить в тот же день
# После merge сразу удалить ветку

# Использовать feature flags для больших фич
if config.FeatureEnabled("new-ui") {
    renderNewUI()
} else {
    renderOldUI()
}
```

## Вопросы с собеседований

**Вопрос:** В чём основная разница между Gitflow и Trunk-based development?

**Ответ:** Gitflow использует несколько долгоживущих веток (main, develop) и строгие правила их взаимодействия. Trunk-based использует одну основную ветку (main/trunk) и короткоживущие feature ветки (1-2 дня). Gitflow подходит для scheduled releases, Trunk-based — для Continuous Deployment.

**Вопрос:** Что такое feature flags и зачем они нужны в Trunk-based development?

**Ответ:** Feature flags (feature toggles) — это механизм включения/выключения функциональности через конфигурацию без изменения кода. Они позволяют деплоить незавершённую функциональность в продакшен, скрывая её от пользователей. Это критично для Trunk-based, где код мержится в main до завершения фичи.

**Вопрос:** Когда стоит использовать Gitflow вместо Trunk-based?

**Ответ:** Gitflow предпочтительнее когда:
- Релизы происходят по расписанию (раз в месяц/квартал)
- Нужна поддержка нескольких версий в продакшене
- Строгие требования к процессу (банки, медицина)
- Desktop/mobile приложения с длительным процессом релиза
Trunk-based лучше для веб-сервисов с частыми деплоями.

## Связанные темы

- [[Git - Ветвление и слияние]]
- [[Git - Rebase vs Merge]]
- [[CI-CD - Основные концепции]]
- [[Git - Основные команды]]
