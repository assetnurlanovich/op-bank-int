# Service Logic

## 1. Назначение

Документ фиксирует логику работы каждого сервиса приложения OnePay ↔ Kaspi и показывает, какие вызовы уходят:
- в Ломбард;
- в Kaspi;
- в локальное хранилище OnePay.

Документ дополняет:
- `docs/api-contracts.md`
- `docs/lombard-api-contracts.md`
- `docs/outbound-kaspi-contracts.md`
- `docs/data-model-and-state.md`

---

## 2. Соответствие сервисов OnePay и API Ломбарда

| Сервис OnePay | Назначение | Вызовы в Ломбард |
|---------------|------------|------------------|
| `Contracts Service` | вернуть список договоров клиента | `GET /internal/lombard/v1/contracts` |
| `Repayment Service` | вернуть параметры погашения и обеспечить `payment_id` OnePay | `GET /internal/lombard/v1/repayments/info`, `POST /internal/lombard/v1/payments/ensure` |
| `Calculate Repayment Service` | пересчитать сумму частичного/полного погашения | `POST /internal/lombard/v1/repayments/calculate` |
| `Payment Service` | вернуть параметры ранее созданного платежа | `GET /internal/lombard/v1/payments/{payment_id}` |
| `Invoice Registration / Payment Preparation flow` | подготовить invoice-based платеж и зарегистрировать его в Kaspi | `POST /internal/lombard/v1/kaspi-invoices`, `PUT /internal/lombard/v1/payments/{payment_id}/external-reference` |
| `Callback Processing Service` | сохранить callback, обновить статус, инициировать внутреннее проведение | `POST /internal/lombard/v1/payments/{payment_id}/posting` только при подтвержденном успешном платеже |
| `Payment Status Check Service` | выполнить fallback-сверку через Kaspi | прямых вызовов в Ломбард нет, кроме опционального ручного reconciliation вне MVP |

---

## 3. Логика сервисов

### 3.1. Contracts Service

Вход:
- `GET /api/kaspi/contracts`

Основная логика:
1. Принять запрос от Kaspi.
2. Провалидировать `merchant_id`, `iin`, `request_time`, `sign`.
3. Проверить replay window.
4. Вызвать Ломбард `GET /internal/lombard/v1/contracts`.
5. Смаппить ответ Ломбарда во внешний Kaspi-формат.
6. Сохранить audit-запись inbound/outbound.
7. Вернуть ответ Kaspi.

Маппинг ошибок:
- `client_not_found` -> `1002`
- `contracts_not_available` -> `1003`, если бизнес считает это недоступностью оплаты
- технические ошибки -> 5xx / controlled technical error

Ключевые правила:
- сервис не создает платеж;
- сервис не меняет состояние в локальной БД, кроме audit trail;
- список договоров всегда приходит из Ломбарда как из источника истины.

### 3.2. Repayment Service

Вход:
- `GET /api/kaspi/repayment`

Основная логика:
1. Принять и провалидировать запрос Kaspi.
2. Вызвать Ломбард `GET /internal/lombard/v1/repayments/info`.
3. Создать или обновить локальную запись в `payments` и `payment_links` по `(merchant_id, iin, contract_number)`.
4. Получить локальный `payment_id` OnePay.
5. Вызвать Ломбард `POST /internal/lombard/v1/payments/ensure` с `(payment_id, merchant_id, contract_number, iin, repayment_type, reference_id)`.
6. Сохранить `bank_id`, `source_id`, денежный snapshot и связь с Ломбардом.
6. Сформировать ответ Kaspi.
7. Записать audit и, при необходимости, статус history.

Ключевые правила:
- `payment_id` OnePay должен существовать уже на этапе `repayment`;
- повторный запрос по той же связке `(merchant_id, iin, contract_number)` не должен создавать новый локальный платеж;
- сервис не подтверждает оплату, а только подготавливает платеж к дальнейшему сценарию.

### 3.3. Calculate Repayment Service

Вход:
- `GET /api/kaspi/calculate-repayment`

Основная логика:
1. Принять и провалидировать запрос Kaspi.
2. Убедиться, что локально существует платеж с переданным `payment_id`.
3. Вызвать Ломбард `POST /internal/lombard/v1/repayments/calculate`.
4. Сохранить в локальном `payments` актуальные суммы расчета: `amount`, `fee`, `currency`.
5. Вернуть Kaspi рассчитанный ответ либо контролируемую бизнес-ошибку.
6. Записать audit.

Маппинг ошибок:
- `amount_below_minimum` -> `1005`
- `amount_above_full_due` -> `1006`
- `payment_not_found` -> `1007`

Ключевые правила:
- расчет использует decimal-арифметику;
- сервис не меняет подтвержденный статус платежа;
- одинаковый запрос должен давать детерминированный ответ в пределах одного бизнес-момента времени.

### 3.4. Payment Service

Вход:
- `GET /api/kaspi/payment`

Основная логика:
1. Принять и провалидировать запрос Kaspi.
2. Найти локальный платеж по `(merchant_id, payment_id)`.
3. Если платеж отсутствует, вернуть `1007`.
4. Если платеж уже финализирован и повторная оплата запрещена, вернуть `1008`.
5. Вызвать Ломбард `GET /internal/lombard/v1/payments/{payment_id}` для получения канонических параметров платежа по id OnePay.
6. Объединить данные Ломбарда с локальным `bank_id` и текущим интеграционным состоянием OnePay.
7. Вернуть Kaspi параметры платежа.
8. Записать audit.

Ключевые правила:
- `return_url` приходит из Ломбарда;
- `reference_id` во внешнем Kaspi API маппится во внутреннее поле `bank_id`;
- внешняя привязка к банку приходит из локального интеграционного состояния OnePay;
- redirect по `return_url` не считается подтверждением факта оплаты.

### 3.5. Invoice Registration / Payment Preparation Flow

Вход:
- внутренний вызов со стороны Ломбарда / OnePay UI, а не запрос от Kaspi

Основная логика:
1. Принять внутреннюю команду на оплату через Kaspi.
2. Вызвать Ломбард `POST /internal/lombard/v1/kaspi-invoices`.
3. Получить суммы, `return_url`, описание и, при наличии, `source_id`.
4. Создать или обновить локальный `payments` со статусом `2` (`PENDING`).
5. Вызвать Kaspi `RegisterInvoice`.
6. Если Kaspi вернул успех, сохранить raw request/response.
7. Сохранить `reference_id` Kaspi как `bank_id` локально и в Ломбарде через `PUT /internal/lombard/v1/payments/{payment_id}/external-reference`.
8. Убедиться, что локальный платеж находится в статусе `2` (`PENDING`).
9. Вернуть в вызывающий контур информацию для клиента.

Ветка ошибки:
1. Если Kaspi вернул бизнес-ошибку, локально сохранить failed attempt.
2. Не выполнять `AttachExternalReference`.
3. Не считать платеж оплаченным.
4. Вернуть безопасную бизнес-ошибку вызывающему контуру.

Ключевые правила:
- регистрация invoice не является оплатой;
- внешний `reference_id` сохраняется во внутреннем поле `bank_id` до возврата успеха наружу;
- при технической ошибке Kaspi допустим controlled retry.

### 3.6. Callback Processing Service

Вход:
- `POST /api/kaspi/callback`

Основная логика:
1. Принять callback.
2. Провалидировать обязательные поля, `merchant_id`, `request_time`, `sign`.
3. Найти платеж по `(merchant_id, payment_id)`.
4. В одной БД-транзакции:
5. Проверить business idempotency по `(merchant_id, payment_id, reference_id, status)`.
6. Если callback дубликат, вернуть идемпотентный успешный ответ без side effects.
7. Если callback новый, сохранить raw payload и event metadata в `payment_logs`.
8. Сохранить `bank_id`, `rnn`, внешний статус и audit trail.
9. Обновить агрегат `payments` и записать изменение состояния в `payment_logs`.
10. Если внешний статус успешный, зафиксировать внешний факт оплаты.
11. После фиксации внешнего успеха вызвать Ломбард `POST /internal/lombard/v1/payments/{payment_id}/posting`, где `payment_id` — это id OnePay.
12. Если posting успешен, сохранить успешный результат в `payment_logs`, а статус платежа оставить `1` (`SUCCESS`).
13. Если posting упал технически, сохранить внешний успех, статус платежа оставить `1` (`SUCCESS`) и поставить запись об ошибке в `payment_logs`.
14. Вернуть техническое подтверждение Kaspi.

Ключевые правила:
- callback является главным источником истины по оплате;
- внешний успех должен быть сохранен до внутреннего posting;
- повторный callback не должен повторно запускать posting;
- неуспешный callback не вызывает внутреннее проведение.

### 3.7. Payment Status Check Service

Вход:
- background job или manual trigger

Основная логика:
1. Выбрать локальные платежи, требующие проверки.
2. Создать запись в `payment_logs` с типом background-job.
3. Вызвать Kaspi `GetPaymentStatus`.
4. Сохранить raw request/response.
5. Сравнить внешний статус с локальным состоянием.
6. Если статус подтверждает pending, оставить платеж в ожидании.
7. Если статус подтверждает failure, обновить локальный платеж в статус `6`.
8. Если статус подтверждает success и callback не был получен, зафиксировать внешний успех и запустить ту же ветку внутреннего posting, что и для callback.
9. Если есть противоречие с уже подтвержденным callback success, не затирать локальный факт, а отправить кейс в retry / reconciliation.

Ключевые правила:
- `status` является fallback, а не первичным источником истины;
- при конфликте между `status` и ранее принятым callback приоритет у callback;
- автоматическая сверка не должна откатывать уже подтвержденный успех.

---

## 4. Какие сервисы не ходят в Ломбард постоянно

- `Signature & Security Layer` не ходит в Ломбард, он работает локально на стороне OnePay.
- `Persistence Layer` не ходит в Ломбард, а обслуживает локальное интеграционное состояние.
- `Payment Status Check Service` в MVP ходит только в Kaspi; в Ломбард он нужен только если позже появится отдельный reconciliation API.

---

## 5. Границы ответственности OnePay и Ломбарда

### 5.1. Что делает OnePay

- принимает запросы Kaspi;
- проверяет подпись и replay window;
- ведет локальную БД интеграционного состояния;
- хранит callback и audit trail;
- вызывает Kaspi outbound API;
- вызывает внутренние API Ломбарда;
- управляет retry и reconciliation.

### 5.2. Что делает Ломбард

- хранит договоры и задолженность;
- определяет бизнес-правила расчета;
- создает внутренний платеж;
- выдает `return_url` и бизнес-параметры платежа;
- выполняет внутреннее проведение подтвержденного платежа.

---

## 6. Минимальный MVP-поток вызовов OnePay -> Ломбард

1. `contracts` -> `GET /contracts`
2. `repayment` -> `GET /repayments/info` + `POST /payments/ensure`
3. `calculate-repayment` -> `POST /repayments/calculate`
4. `payment` -> `GET /payments/{payment_id}`
5. `invoice flow` -> `POST /kaspi-invoices` + `PUT /payments/{payment_id}/external-reference`
6. `callback success` -> `POST /payments/{payment_id}/posting`
