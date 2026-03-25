# Lombard API Contracts

## 1. Назначение

Этот документ фиксирует минимальный и легкий HTTP API Ломбарда, который должен вызывать сервис интеграции OnePay ↔ Kaspi.

Цель API:
- отдать OnePay бизнес-данные по договорам и задолженности;
- создать или найти внутренний платеж Ломбарда;
- подготовить invoice-based платеж;
- вернуть параметры платежа для Kaspi;
- выполнить внутреннее проведение подтвержденного платежа.

OnePay остается владельцем интеграционного состояния с Kaspi:
- проверка подписи и replay;
- audit inbound/outbound сообщений;
- хранение callback;
- fallback status check;
- внешние идентификаторы `reference_id` и `rnn` в контексте банковской интеграции.

Ломбард остается владельцем бизнес-смысла:
- договоры;
- задолженность;
- правила погашения;
- внутренний платеж;
- внутреннее проведение.

## Source of truth

При конфликте документов по внутренней интеграции использовать следующий приоритет:
1. `docs/lombard-api-contracts.md`
2. `docs/internal-system-contracts.md`
3. `docs/service-logic.md`
4. `docs/architecture.md`, `docs/architecture2.md`, `docs/kaspi-protocol.md`

---

## 2. Общие правила API Ломбарда

### 2.1. Transport

- protocol: HTTP/JSON;
- base path: `/internal/lombard/v1`;
- API работает только во внутреннем trusted-контуре;
- обязательны `X-Request-ID` и `X-Trace-ID`;
- аутентификация: только `Authorization: Bearer <token>`;
- подпись `sign` для этого API не нужна.

### 2.2. Формат успешного ответа

```json
{
  "data": {}
}
```

### 2.3. Формат ошибочного ответа

```json
{
  "error": {
    "category": "business",
    "code": "contract_not_available",
    "message": "Contract is not available for repayment",
    "retryable": false
  }
}
```

### 2.4. Категории ошибок

- `validation`
- `not_found`
- `business`
- `conflict`
- `temporary`
- `internal`

### 2.4.1. HTTP status policy

| Категория результата | HTTP status |
|----------------------|-------------|
| success | `200 OK` |
| `validation` | `400 Bad Request` |
| `not_found` | `404 Not Found` |
| `conflict` | `409 Conflict` |
| `business` | `422 Unprocessable Entity` |
| `temporary` | `503 Service Unavailable` |
| `internal` | `500 Internal Server Error` |

OnePay client обязан читать не только HTTP status, но и тело ошибки из раздела 2.3.

### 2.5. Общие правила данных

- все денежные поля передаются как decimal JSON number с точностью не более 2 знаков;
- все идентификаторы времени передаются в ISO-8601 UTC;
- `payment_id` в transport рекомендуется хранить как `bigint`;
- `payment_id` во внутренних HTTP-контрактах ниже означает идентификатор платежа OnePay и используется как correlation id между OnePay и Ломбардом;
- собственный идентификатор платежа Ломбарда, если он существует, передается отдельно как `source_id`;
- `reference_id` и `rnn` всегда строковые;
- все state-changing операции должны быть идемпотентными.

---

## 3. Операция A - List Client Contracts

### 3.1. Назначение

Используется сервисом OnePay `Contracts Service`.

### 3.2. HTTP contract

`GET /internal/lombard/v1/contracts?merchant_id=1&iin=900101300123`

### 3.3. Request

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| merchant_id | integer | да | Идентификатор мерчанта / ломбарда |
| iin | string(12) | да | ИИН клиента |

### 3.4. Response

| Поле | Тип | Обяз. |
|------|-----|-------|
| contracts[] | array | да |
| contracts[].contract_number | string(1..50) | да |
| contracts[].contract_status | string(1..30) | да |

### 3.5. Ошибки

- `not_found/client_not_found`
- `business/contracts_not_available`
- `temporary/upstream_timeout`

---

## 4. Операция B - Get Repayment Info

### 4.1. Назначение

Используется сервисом OnePay `Repayment Service`.

### 4.2. HTTP contract

`GET /internal/lombard/v1/repayments/info?merchant_id=1&contract_number=DZ-458721&iin=900101300123&repayment_type=2`

### 4.3. Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| repayment_type | integer | да |

### 4.4. Response

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| repayment_type | integer | да | `1` или `2` |
| monthly_due_amount | number(18,2) | нет | Только для `repayment_type = 1` |
| interest_amount | number(18,2) | да | Проценты |
| penalty_amount | number(18,2) | да | Штрафы |
| full_due_amount | number(18,2) | да | Полная задолженность |
| min_payment_amount | number(18,2) | нет | Только для `repayment_type = 2` |
| max_payment_amount | number(18,2) | нет | Только для `repayment_type = 2` |
| fee_amount | number(18,2) | нет | Комиссия, если известна сразу |
| total_amount | number(18,2) | нет | Итог, если известен сразу |
| currency | string(3) | да | Валюта |

### 4.5. Ошибки

- `not_found/client_not_found`
- `business/contract_not_available`
- `temporary/upstream_timeout`

---

## 5. Операция C - Calculate Repayment

### 5.1. Назначение

Используется сервисом OnePay `Calculate Repayment Service`.

### 5.2. HTTP contract

`POST /internal/lombard/v1/repayments/calculate`

### 5.3. Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| payment_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| entered_amount | number(18,2) | да |

### 5.4. Response

| Поле | Тип | Обяз. |
|------|-----|-------|
| repayment_type | integer | да |
| interest_amount | number(18,2) | да |
| penalty_amount | number(18,2) | да |
| repayment_amount | number(18,2) | да |
| fee_amount | number(18,2) | да |
| total_amount | number(18,2) | да |
| min_payment_amount | number(18,2) | нет |
| full_due_amount | number(18,2) | нет |
| currency | string(3) | да |

### 5.5. Ошибки

- `business/amount_below_minimum`
- `business/amount_above_full_due`
- `not_found/payment_not_found`
- `temporary/upstream_timeout`

---

## 6. Операция D - Ensure Payment

### 6.1. Назначение

Создает или возвращает существующий внутренний платеж Ломбарда.
Используется сервисом OnePay `Repayment Service`.

### 6.2. HTTP contract

`POST /internal/lombard/v1/payments/ensure`

### 6.3. Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| payment_id | integer | да |
| merchant_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| repayment_type | integer | да |
| reference_id | string(1..64) | да |

### 6.4. Response

| Поле | Тип | Обяз. |
|------|-----|-------|
| payment_id | integer | да |
| merchant_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| repayment_type | integer | да |
| source_id | string(1..64) | нет |
| status | string(1..40) | да |

### 6.5. Идемпотентность

- операция обязана быть идемпотентной по `(merchant_id, payment_id)`;
- повторный запрос должен вернуть тот же `payment_id`;
- повторный запрос с тем же `(merchant_id, payment_id)`, но другим `(iin, contract_number, repayment_type)` должен вернуть `conflict/payment_context_mismatch`.

Примечание:
- `payment_id` в этом endpoint — это id платежа OnePay;
- `source_id` — опциональный собственный id Ломбарда.

---

## 7. Операция E - Get Payment

### 7.1. Назначение

Возвращает канонические параметры внутреннего платежа Ломбарда.
Используется сервисом OnePay `Payment Service`.

### 7.2. HTTP contract

`GET /internal/lombard/v1/payments/{payment_id}?merchant_id=1`

### 7.3. Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| payment_id | integer | да |

Примечание:
- `payment_id` в path — это id платежа OnePay, по которому Ломбард находит связанный бизнес-контекст;
- если у Ломбарда есть собственный id, он возвращается отдельно как `source_id`.

### 7.4. Response

| Поле | Тип | Обяз. |
|------|-----|-------|
| payment_id | integer | да |
| merchant_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| repayment_type | integer | да |
| source_id | string(1..64) | нет |
| status | string(1..40) | да |
| payment_amount | number(18,2) | да |
| fee_amount | number(18,2) | да |
| total_amount | number(18,2) | да |
| currency | string(3) | да |
| return_url | string(1..255) | да |

### 7.5. Ошибки

- `not_found/payment_not_found`
- `conflict/payment_already_processed`
- `temporary/upstream_timeout`

---

## 8. Операция F - Prepare Kaspi Invoice

### 8.1. Назначение

Используется сценарием, который начинается на стороне Ломбарда.
OnePay вызывает этот endpoint перед `RegisterInvoice` в Kaspi.

### 8.2. HTTP contract

`POST /internal/lombard/v1/kaspi-invoices`

### 8.3. Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| requested_amount | number(18,2) | да |
| repayment_type | integer | да |

### 8.4. Response

| Поле | Тип | Обяз. |
|------|-----|-------|
| payment_id | integer | да |
| source_id | string(1..64) | нет |
| payment_amount | number(18,2) | да |
| fee_amount | number(18,2) | да |
| total_amount | number(18,2) | да |
| currency | string(3) | да |
| return_url | string(1..255) | да |
| description | string(1..255) | нет |

### 8.5. Правила

- операция создает внутренний платеж либо возвращает ранее подготовленный платеж по бизнес-ключу;
- в этом ответе еще нет `reference_id` Kaspi;
- подготовка платежа не означает подтверждение оплаты.

---

## 9. Операция G - Attach External Reference

### 9.1. Назначение

Сохраняет во внутренней системе Ломбарда внешний идентификатор Kaspi после успешной регистрации invoice.
Используется сервисом OnePay `Invoice Registration / Payment Preparation flow`.

### 9.2. HTTP contract

`PUT /internal/lombard/v1/payments/{payment_id}/external-reference`

### 9.3. Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| reference_id | string(1..64) | да |
| external_status | string(1..30) | нет |
| payment_url | string(1..255) | нет |
| expires_at | string(date-time) | нет |

Примечание:
- в transport-level API Ломбарда поле по-прежнему называется `reference_id`, потому что это идентификатор банка / Kaspi;
- во внутренней модели хранения OnePay это же значение рекомендуется сохранять в поле `bank_id`.

### 9.4. Response

| Поле | Тип | Обяз. |
|------|-----|-------|
| payment_id | integer | да |
| merchant_id | integer | да |
| reference_id | string(1..64) | да |
| status | string(1..40) | да |

### 9.5. Правила

- повторный вызов с тем же `reference_id` должен быть идемпотентным;
- попытка заменить уже закрепленный другой `reference_id` должна вернуть `conflict/reference_id_conflict`.
- `payment_id` в path — это id OnePay;
- собственный id Ломбарда при наличии не заменяет `payment_id`, а хранится как `source_id`.

---

## 10. Операция H - Post Confirmed Repayment

### 10.1. Назначение

Запускает внутреннее проведение после подтвержденного внешнего платежа.
Используется сервисом OnePay `Callback Processing Service`.

### 10.2. HTTP contract

`POST /internal/lombard/v1/payments/{payment_id}/posting`

### 10.3. Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | нет |
| repayment_type | integer | да |
| amount | number(18,2) | да |
| fee | number(18,2) | да |
| reference_id | string(1..64) | да |
| rnn | string(1..64) | нет |
| external_status | integer | да |

Примечание:
- в этом запросе `amount` означает полную сумму платежа, то есть `total_amount`;
- сумма без комиссии может быть вычислена в Ломбарде как `amount - fee`, если она требуется его бизнес-логике.

### 10.4. Response

| Поле | Тип | Обяз. |
|------|-----|-------|
| payment_id | integer | да |
| posting_status | string(1..30) | да |
| posted_at | string(date-time) | нет |
| posting_reference | string(1..64) | нет |

### 10.5. Правила

- вызов разрешен только для подтвержденного внешнего успеха;
- операция обязана быть идемпотентной по `payment_id`;
- повтор не должен приводить к двойному проведению;
- техническая ошибка должна позволять retry со стороны OnePay.
- `payment_id` в path — это id OnePay как correlation id.

---

## 11. Какие запросы OnePay делает в Ломбард

| Сценарий OnePay | Запрос в Ломбард |
|-----------------|------------------|
| Получить список договоров для Kaspi | `GET /internal/lombard/v1/contracts` |
| Получить параметры погашения | `GET /internal/lombard/v1/repayments/info` |
| Посчитать сумму частичного/полного погашения | `POST /internal/lombard/v1/repayments/calculate` |
| Создать или найти платеж для Kaspi-initiated потока | `POST /internal/lombard/v1/payments/ensure` |
| Получить параметры созданного платежа | `GET /internal/lombard/v1/payments/{payment_id}` |
| Подготовить invoice-based платеж | `POST /internal/lombard/v1/kaspi-invoices` |
| Сохранить внешний `reference_id` Kaspi | `PUT /internal/lombard/v1/payments/{payment_id}/external-reference` |
| Провести подтвержденный платеж внутри Ломбарда | `POST /internal/lombard/v1/payments/{payment_id}/posting` |

---

## 12. Маппинг на application ports

| Lombard HTTP API | Application port |
|------------------|------------------|
| `GET /contracts` | `ContractsProvider.ListContracts` |
| `GET /repayments/info` | `RepaymentProvider.GetRepaymentInfo` |
| `POST /repayments/calculate` | `RepaymentProvider.CalculateRepayment` |
| `POST /payments/ensure` | `PaymentRegistry.CreateOrGetPayment` |
| `GET /payments/{payment_id}` | `PaymentRegistry.GetPayment` |
| `PUT /payments/{payment_id}/external-reference` | `PaymentRegistry.AttachExternalReference` |
| `POST /payments/{payment_id}/posting` | `InternalPostingService.PostRepayment` |
| `POST /kaspi-invoices` | `InvoicePreparationService.PrepareKaspiInvoice` |
