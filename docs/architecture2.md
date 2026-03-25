# Architecture — OnePay ↔ Kaspi Integration Service

> Этот документ является краткой архитектурной выжимкой.
> Источником истины по контрактам, логике, хранению и ошибкам являются:
> `api-contracts.md`, `request-validation-and-signature.md`, `outbound-kaspi-contracts.md`,
> `inbound-onepay-commands.md`, `lombard-api-contracts.md`, `internal-system-contracts.md`,
> `service-logic.md`, `data-model-and-state.md`, `error-codes.md`.

## 1. Общая концепция

Приложение представляет собой **интеграционный адаптер** между:
- Kaspi (внешняя система)
- OnePay / Ломбард (внутренние системы)

Архитектурный стиль:
- модульный монолит
- четкое разделение слоев
- stateless application layer
- audit-first обработка интеграционных событий

---

## 2. Контур системы

### Входящие вызовы (Kaspi → OnePay)
- contracts
- repayment
- calculate-repayment
- payment
- callback

### Исходящие вызовы (OnePay → Kaspi)
- invoice/payment registration
- status

### Внутренние вызовы (OnePay Adapter → Core Systems)
- contracts service
- repayment calculation
- payment creation
- invoice preparation / bill issuing
- repayment posting

---

## 3. Компоненты

### 3.1 HTTP Layer
- обработка запросов
- middleware
- response envelope

### 3.2 Security Layer
- проверка подписи
- проверка merchant_id
- проверка request_time
- защита от replay

### 3.3 Application Layer
Сервисы:
- ContractsService
- RepaymentService
- CalculateRepaymentService
- PaymentService
- InvoiceRegistration flow
- CallbackService
- StatusCheckService

### 3.4 Domain Layer
Сущности:
- Payment
- Contract
- Repayment
- ExternalPaymentReference
- CallbackEvent
- Status

### 3.5 Persistence Layer
- PostgreSQL (источник истины)
- Redis (idempotency / cache)
- audit/integration logs

### 3.6 Kaspi Client
- HTTP клиент для invoice/payment registration
- HTTP клиент для status

### 3.7 Background Jobs
- status polling
- retry internal posting
- retry
- reconciliation

---

## 4. Потоки

### Основной поток оплаты
1. Kaspi → repayment/calculate либо OnePay → Kaspi invoice registration
2. Kaspi → payment или клиент переходит к оплате в Kaspi app
3. Kaspi → callback
4. OnePay сохраняет внешний платежный факт
5. OnePay → внутреннее проведение

### Fallback поток
1. callback не пришел
2. background job вызывает status
3. синхронизация состояния

### Ключевые правила
- callback является основным источником истины по оплате
- return_url не является подтверждением оплаты
- повторный callback не вызывает повторных side effects
- подтвержденный внешний платеж не должен теряться даже при ошибке внутреннего posting

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
- payment_links
- payment_logs
- merchant_fees

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
- audit trail of inbound/outbound events

---

## 9. Архитектурное решение

Отдельный сервис (adapter), который:
- не содержит бизнес-логики шлюза
- не хранит чувствительные данные
- масштабируется горизонтально