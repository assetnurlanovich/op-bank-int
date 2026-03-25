# Inbound OnePay Commands

## 1. Назначение

Этот документ фиксирует внутренний inbound API, которым Ломбард / OnePay запускает сценарий invoice-based оплаты через Kaspi.

Этот API не предназначен для Kaspi.
Он используется только во внутреннем trusted-контуре между Ломбардом и приложением OnePay ↔ Kaspi.

---

## 2. Source of truth

Для данного сценария документы используются в следующем порядке:
1. этот документ — как источник истины по inbound-команде запуска invoice-based flow;
2. `docs/outbound-kaspi-contracts.md` — как источник истины по вызову OnePay -> Kaspi;
3. `docs/lombard-api-contracts.md` — как источник истины по вызовам OnePay -> Ломбард;
4. `docs/service-logic.md` — как источник истины по orchestration-логике.

---

## 3. Аутентификация и transport

- protocol: HTTP/JSON;
- base path: `/internal/onepay/v1`;
- аутентификация: только `Authorization: Bearer <token>`;
- обязательны заголовки `X-Request-ID` и `X-Trace-ID`;
- этот API не использует `sign`.

---

## 4. Операция - Start Kaspi Invoice Payment

### 4.1. Назначение

Команда запускает сценарий:
1. OnePay принимает внутреннюю команду;
2. получает/подготавливает бизнес-данные в Ломбарде;
3. регистрирует invoice / payment в Kaspi;
4. возвращает вызывающей стороне идентификаторы и параметры для дальнейшего UX-сценария.

### 4.2. HTTP contract

`POST /internal/onepay/v1/kaspi-payments`

### 4.3. Request

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| command_id | string(1..64) | да | Идемпотентный идентификатор команды запуска |
| merchant_id | integer | да | Идентификатор мерчанта / ломбарда |
| contract_number | string(1..50) | да | Номер залогового билета / договора |
| iin | string(12) | да | ИИН клиента |
| repayment_type | integer | да | `1` или `2` |
| requested_amount | number(18,2) | да | Запрошенная сумма, если сценарий стартует с суммы |

### 4.4. Response

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| payment_id | integer | да | Идентификатор платежа OnePay |
| source_id | string(1..64) | нет | Собственный id платежа Ломбарда, если он существует |
| reference_id | string(1..64) | да | Идентификатор платежа / invoice в Kaspi |
| status | integer | да | Статус платежа в OnePay, для созданного invoice ожидается `2` |
| payment_url | string(1..255) | нет | Ссылка на оплату / открытие сценария, если Kaspi ее возвращает |
| return_url | string(1..255) | да | URL возврата клиента |
| expires_at | string(date-time) | нет | Срок действия invoice |

### 4.5. Идемпотентность

- команда обязана быть идемпотентной по `(merchant_id, command_id)`;
- повтор команды с тем же `(merchant_id, command_id)` должен возвращать тот же результат;
- повтор команды с тем же `command_id`, но другим бизнес-контекстом должен возвращать `409 Conflict`.

### 4.6. Правила обработки

- OnePay не должен считать платеж успешным только по факту успешного создания invoice;
- после успешной регистрации invoice OnePay обязан сохранить `reference_id` до возврата ответа;
- итоговое подтверждение оплаты приходит только через callback или fallback status;
- HTTP-level API Ломбарда и Kaspi для дальнейших шагов определены в `docs/lombard-api-contracts.md` и `docs/outbound-kaspi-contracts.md`.

---

## 5. HTTP status policy

| Ситуация | HTTP status |
|----------|-------------|
| Успешное выполнение команды | `200 OK` |
| Ошибка валидации входной команды | `400 Bad Request` |
| Команда с тем же `command_id`, но другим контекстом | `409 Conflict` |
| Бизнес-ошибка подготовки платежа | `422 Unprocessable Entity` |
| Временная ошибка upstream | `503 Service Unavailable` |
| Неожиданная внутренняя ошибка | `500 Internal Server Error` |

Ошибочный ответ:

```json
{
  "error": {
    "category": "business",
    "code": "invoice_prepare_failed",
    "message": "Unable to prepare Kaspi payment",
    "retryable": false
  }
}
```
