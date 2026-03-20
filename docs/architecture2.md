# Architecture — OnePay ↔ Kaspi Integration Service

## 1. Общая концепция

Приложение представляет собой **интеграционный адаптер** между:
- Kaspi (внешняя система)
- OnePay / Ломбард (внутренние системы)

Архитектурный стиль:
- модульный монолит
- четкое разделение слоев
- stateless application layer

---

## 2. Контур системы

### Входящие вызовы (Kaspi → OnePay)
- contracts
- repayment
- calculate-repayment
- payment
- callback

### Исходящие вызовы (OnePay → Kaspi)
- status

### Внутренние вызовы (OnePay Adapter → Core Systems)
- contracts service
- repayment calculation
- payment creation
- repayment posting

---

## 3. Компоненты

### 3.1 HTTP Layer
- обработка запросов
- middleware
- response envelope

### 3.2 Security Layer
- проверка подписи
- проверка request_time
- защита от replay

### 3.3 Application Layer
Сервисы:
- ContractsService
- RepaymentService
- CalculateRepaymentService
- PaymentService
- CallbackService
- StatusCheckService

### 3.4 Domain Layer
Сущности:
- Payment
- Contract
- Repayment
- CallbackEvent
- Status

### 3.5 Persistence Layer
- PostgreSQL (источник истины)
- Redis (idempotency / cache)

### 3.6 Kaspi Client
- HTTP клиент для status

### 3.7 Background Jobs
- status polling
- retry
- reconciliation

---

## 4. Потоки

### Основной поток оплаты
1. Kaspi → repayment/calculate
2. Kaspi → payment
3. Kaspi → callback
4. OnePay → внутреннее проведение

### Fallback поток
1. callback не пришел
2. background job вызывает status
3. синхронизация состояния

---

## 5. Идемпотентность

Ключевые правила:
- callback обрабатывается один раз
- повторный callback не вызывает side effects
- уникальность по (payment_id + reference_id)

---

## 6. Хранение данных

Основные таблицы:
- payments
- payment_status_history
- callbacks
- idempotency_keys
- status_checks

---

## 7. Статусы

Внешние:
- 1 SUCCESS
- 2 PENDING
- 5 CANCELLED
- 6 FAILED

Внутренние:
- NEW
- WAITING_PAYMENT
- CALLBACK_RECEIVED
- SUCCESS
- FAILED
- RECONCILIATION_REQUIRED

---

## 8. Наблюдаемость

- structured logging
- metrics
- tracing
- health endpoints

---

## 9. Архитектурное решение

Отдельный сервис (adapter), который:
- не содержит бизнес-логики шлюза
- не хранит чувствительные данные
- масштабируется горизонтально