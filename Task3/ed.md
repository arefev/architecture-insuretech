# Переход на Event-Driven архитектуру

## 1. Анализ текущей архитектуры

В текущей архитектуре сервисы взаимодействуют преимущественно через **синхронные REST-запросы**.

Основные потоки данных:

1. **core-app → ins-product-aggregator (каждые 15 минут)**
2. **ins-comp-settlement → ins-product-aggregator (раз в сутки)**
3. **ins-comp-settlement → core-app (раз в сутки)**

ins-product-aggregator при каждом запросе:

* обращается к **5 страховым компаниям**
* агрегирует предложения
* возвращает результат

## 2. Проблемы текущей архитектуры

### 2.1 Синхронные зависимости между сервисами

Сервисы напрямую зависят друг от друга через REST API.

Проблемы:

* задержки ответа
* ошибки сети
* cascading failures

Например:

```bash
core-app → ins-product-aggregator → insurance companies
```

Если страховая компания отвечает медленно, **вся цепочка деградирует**.

### 2.2 Высокая latency агрегатора

При каждом запросе:

```bash
ins-product-aggregator
   → company1
   → company2
   → company3
   → company4
   → company5
```

После подключения **10 страховых компаний** latency увеличится в 2 раза.

Риски:

* таймауты
* перегрузка агрегатора
* ухудшение UX

### 2.3 Дублирование данных

Сервисы:

* core-app
* ins-comp-settlement

хранят **локальные копии тарифов и продуктов**.

Проблемы:

* данные могут быть **устаревшими**
* несогласованность данных
* сложная синхронизация

### 2.4 Polling архитектура

Сейчас используется **pull модель**:

```bash
core-app → каждые 15 минут
ins-comp-settlement → раз в сутки
```

Недостатки:

* задержка обновления данных
* лишняя нагрузка
* неэффективное использование ресурсов

### 2.5 Batch интеграция между сервисами

```bash
ins-comp-settlement → core-app
```

раз в сутки получает список страховок.

Риски:

* большой payload
* долгий запрос
* вероятность таймаута

### 2.6 Масштабирование при росте страховых компаний

При увеличении числа партнёров:

```bash
5 → 10 → 15 компаний
```

нагрузка на:

* ins-product-aggregator
* сеть
* API партнёров

будет расти линейно.

## 3. Предлагаемая архитектура

Рекомендуется перейти к [**Event-Driven архитектуре**](./InsureTechContainer.png).

Основная идея:

* сервисы **публикуют события**
* другие сервисы **подписываются**

Для передачи событий используется **event streaming платформа**.

Например:

* Apache Kafka
* Redpanda
* Pulsar

## 4. Основные события системы

### 1. Обновление продуктов

Событие:

```bash
ProductUpdated
```

Публикуется:

```bash
ins-product-aggregator
```

Подписчики:

```bash
core-app
ins-comp-settlement
```

### 2. Оформление страховки

Событие:

```bash
InsuranceCreated
```

Публикуется:

```bash
core-app
```

Подписчик:

```bash
ins-comp-settlement
```

## 5. Новая схема потоков данных

### Обновление продуктов

```bash
insurance companies
        ↓
ins-product-aggregator
        ↓
   Event Bus
  (ProductUpdated)
        ↓
  core-app
  ins-comp-settlement
```

Теперь:

* данные обновляются **сразу**
* polling больше не нужен

---

### Передача оформленных страховок

Сейчас:

```bash
ins-comp-settlement → REST → core-app
```

Будет:

```bash
core-app
   ↓
InsuranceCreated event
   ↓
Event Bus
   ↓
ins-comp-settlement
```

## 6. Новая контейнерная архитектура

Обновлённая схема контейнеров:

```bash
                   +-------------------+
                   |  Insurance        |
                   |  Companies        |
                   +---------+---------+
                             |
                             |
                    +--------v--------+
                    | ins-product-    |
                    | aggregator      |
                    +--------+--------+
                             |
                    ProductUpdated event
                             |
                     +-------v--------+
                     | Event Streaming |
                     |  Platform       |
                     | (Kafka)         |
                     +---+---------+---+
                         |         |
                         |         |
                +--------v-+   +--v---------------+
                | core-app |   | ins-comp-        |
                |          |   | settlement       |
                +----+-----+   +--------+---------+
                     |
                     |
           InsuranceCreated event
                     |
                     v
              Event Streaming
```

## 7. Преимущества новой архитектуры

### Снижение latency

Теперь сервисы получают данные **асинхронно**, без ожидания REST ответа.

---

### Уменьшение нагрузки

Нет:

```bash
periodic polling
```

---

### Масштабируемость

Добавление новых страховых компаний:

```bash
+1 integration
```

не влияет на другие сервисы.

---

### Слабая связанность сервисов

Сервисы больше не зависят друг от друга напрямую.

---

## 8. Использование Transactional Outbox

Да, рекомендуется использовать **паттерн Transactional Outbox**.

### Причина

Нужно избежать ситуации:

```bash
данные записались в БД
но событие не отправилось
```

или наоборот.

---

### Как работает Outbox

При создании страховки:

```bash
1 запись в insurance table
2 запись в outbox table
```

В одной транзакции.

---

### Далее

Специальный процесс:

```bash
Outbox Processor
```

читает таблицу и отправляет события в Kafka.

---

### Поток

```bash
core-app
   |
DB Transaction
   |
insurance table
outbox table
   |
Outbox relay
   |
Kafka
   |
ins-comp-settlement
```

## 9. Где применяем Transactional Outbox

Используем:

### core-app

для событий

```bash
InsuranceCreated
InsuranceUpdated
```

---

### ins-product-aggregator

для событий

```bash
ProductUpdated
TariffUpdated
```

## 10. Итог

Переход на Event-Driven архитектуру позволит:

* снизить связанность сервисов
* уменьшить latency
* повысить устойчивость системы
* упростить масштабирование
* устранить проблемы polling интеграций

Использование **event streaming платформы (Kafka)** и **паттерна Transactional Outbox** обеспечит надёжную доставку событий и согласованность данных между сервисами.
