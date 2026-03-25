# Internal System Contracts

## 1. Назначение

Этот документ определяет минимально достаточные внутренние контракты OnePay / Ломбарда, которые нужны агенту для полной реализации приложения.
Без этих контрактов невозможно детерминированно написать application layer, integration adapters и интеграционные тесты.

Важно:
- этот документ описывает application ports;
- минимальный transport-level HTTP API Ломбарда зафиксирован отдельно в `docs/lombard-api-contracts.md`;
- application layer OnePay должен зависеть от портов ниже, а не от конкретного HTTP-клиента.
- если этот документ расходится с `docs/lombard-api-contracts.md`, источником истины считается `docs/lombard-api-contracts.md` для transport и `docs/data-model-and-state.md` для локальной модели хранения OnePay.

Важное правило:
на старте transport внутренних интеграций может быть HTTP или gRPC, но application layer приложения должен зависеть не от transport, а от явных портов.

---

## 2. Общие требования к внутренним портам

- каждый порт должен быть оформлен как интерфейс application layer;
- каждый вызов должен принимать `context`;
- каждый вызов должен логироваться с `request_id` / `trace_id`;
- все state-changing операции должны быть идемпотентны;
- ошибки должны разделяться на:
  `not_found`, `validation`, `business`, `temporary`, `internal`;
- все денежные поля передаются как decimal-значения;
- все даты и время передаются в UTC или вместе с явно оговоренным timezone policy.

---

## 3. Порт A - ContractsProvider

## 3.1. Назначение

Получение списка договоров клиента для endpoint `contracts`.

## 3.2. Метод

`ListContracts(ctx, request) -> response`

## 3.3. Request

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| merchant_id | integer | да | Мерчант / ломбард |
| iin | string(12) | да | ИИН клиента |

## 3.4. Response

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| contracts[] | array | да | Список договоров |
| contracts[].contract_number | string(1..50) | да | Номер договора |
| contracts[].contract_status | string(1..30) | да | Статус договора |

## 3.5. Ошибки

- `not_found` -> клиент не найден;
- `business` -> клиент найден, но договоры недоступны;
- `temporary` / `internal` -> техническая ошибка upstream.

---

## 4. Порт B - RepaymentProvider

## 4.1. Назначение

Возвращает данные для `repayment` и `calculate-repayment`.

## 4.2. Методы

`GetRepaymentInfo(ctx, request) -> response`

`CalculateRepayment(ctx, request) -> response`

## 4.3. GetRepaymentInfo Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| repayment_type | integer | да |

## 4.4. GetRepaymentInfo Response

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| repayment_type | integer | да | `1` или `2` |
| monthly_due_amount | number(18,2) | нет | Только для `repayment_type = 1` |
| interest_amount | number(18,2) | да | Проценты |
| penalty_amount | number(18,2) | да | Штрафы |
| full_due_amount | number(18,2) | да | Полная задолженность |
| min_payment_amount | number(18,2) | нет | Только для `repayment_type = 2` |
| max_payment_amount | number(18,2) | нет | Только для `repayment_type = 2` |
| fee_amount | number(18,2) | нет | Комиссия, если рассчитывается сразу |
| total_amount | number(18,2) | нет | Итог, если рассчитывается сразу |
| currency | string(3) | да | Валюта |

## 4.5. CalculateRepayment Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| payment_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| entered_amount | number(18,2) | да |

## 4.6. CalculateRepayment Response

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

## 4.7. Бизнес-правила

- если `entered_amount < min_payment_amount`, провайдер должен вернуть отдельную business error для маппинга на `1005`;
- если `entered_amount > full_due_amount`, провайдер должен вернуть отдельную business error для маппинга на `1006`;
- ответ должен быть детерминированным для одинакового входа в пределах одного бизнес-момента времени.

---

## 5. Порт C - PaymentRegistry

## 5.1. Назначение

Создание, чтение и актуализация внутреннего платежа.

Примечание:
- этот порт описывает logical/application-level модель OnePay;
- transport-level вызов в Ломбард `POST /internal/lombard/v1/payments/ensure` использует уже созданный `payment_id` OnePay как correlation id;
- поэтому application-level идемпотентность и transport-level идемпотентность могут быть выражены разными ключами, но не должны противоречить друг другу.

## 5.2. Методы

`CreateOrGetPayment(ctx, request) -> payment`

`GetPayment(ctx, merchantID, paymentID) -> payment`

`AttachExternalReference(ctx, request) -> payment`

`MarkExternalStatus(ctx, request) -> payment`

## 5.3. CreateOrGetPayment Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| repayment_type | integer | да |

## 5.4. Payment Model

| Поле | Тип | Обяз. | Описание |
|------|-----|-------|----------|
| payment_id | integer | да | Идентификатор платежа OnePay |
| merchant_id | integer | да | Мерчант |
| source_id | string(1..64) | нет | Идентификатор платежа в Ломбарде, если он существует |
| status | integer | да | Интеграционный статус платежа в OnePay по кодам Kaspi |
| type | integer | да | Тип платежа OnePay |
| bank_id | string(1..64) | нет | Банковский идентификатор. Во внешних Kaspi-контрактах соответствует `reference_id` |
| rnn | string(1..64) | нет | Внешний номер операции |
| amount | number(18,2) | нет | Полная сумма платежа, то есть `total_amount` |
| fee | number(18,2) | нет | Комиссия |
| currency | string(3) | нет | Валюта |
| return_url | string(1..255) | нет | URL возврата, если известен |

## 5.5. Идемпотентность

- `CreateOrGetPayment` обязан быть идемпотентным по бизнес-ключу `(merchant_id, iin, contract_number)`;
- повторный запрос не должен создавать второй платеж;
- один `payment_id` OnePay должен соответствовать ровно одному бизнес-контексту `(merchant_id, iin, contract_number)`;
- `AttachExternalReference` не должен терять уже сохраненный `bank_id`, если он совпадает;
- смена `bank_id` для уже привязанного платежа должна считаться конфликтом и уходить в reconciliation;

---

## 6. Порт D - InternalPostingService

## 6.1. Назначение

Выполняет внутреннее проведение подтвержденного внешнего платежа.

## 6.2. Метод

`PostRepayment(ctx, request) -> response`

## 6.3. Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| payment_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | нет |
| repayment_type | integer | да |
| amount | number(18,2) | да |
| fee | number(18,2) | да |
| bank_id | string(1..64) | да |
| rnn | string(1..64) | нет |
| external_status | integer | да |

## 6.4. Правила

- вызов разрешен только после подтвержденного внешнего платежного факта;
- операция обязана быть идемпотентной по `payment_id`;
- при повторном вызове тот же `payment_id` не должен проводиться второй раз;
- ошибка posting не должна откатывать уже сохраненный внешний факт оплаты.

---

## 7. Порт E - InvoicePreparationService

## 7.1. Назначение

Готовит invoice-based платеж на стороне OnePay перед вызовом `RegisterInvoice`.

## 7.2. Метод

`PrepareKaspiInvoice(ctx, request) -> response`

## 7.3. Request

| Поле | Тип | Обяз. |
|------|-----|-------|
| merchant_id | integer | да |
| contract_number | string(1..50) | да |
| iin | string(12) | да |
| requested_amount | number(18,2) | да |
| repayment_type | integer | да |

## 7.4. Response

| Поле | Тип | Обяз. |
|------|-----|-------|
| payment_id | integer | да |
| payment_amount | number(18,2) | да |
| fee_amount | number(18,2) | да |
| total_amount | number(18,2) | да |
| currency | string(3) | да |
| return_url | string(1..255) | да |
| description | string(1..255) | нет |

---

## 8. Порт F - NotificationService

## 8.1. Назначение

Необязательный на старте порт для отправки клиентских уведомлений после регистрации invoice или обработки результата.

## 8.2. Правило

- если адаптер отсутствует, его можно реализовать как no-op;
- отсутствие уведомления не должно ломать платежный сценарий.

---

## 9. Маппинг ошибок внутренних портов во внешний API

| Внутренняя категория | Внешний код / поведение |
|----------------------|--------------------------|
| `not_found` по клиенту | `1002` |
| `business` по недоступному договору | `1003` |
| `business` amount below minimum | `1005` |
| `business` amount above full due | `1006` |
| `not_found` по платежу | `1007` |
| `conflict` / already processed | `1008` |
| `validation` | контролируемая ошибка валидации |
| `temporary` / `internal` | техническая ошибка приложения |

---

## 10. Требования к тестам

- для каждого порта должны существовать mock/fake реализации;
- application services тестируются против интерфейсов, а не против HTTP-клиентов;
- для `InternalPostingService` обязателен тест на идемпотентный повтор;
- для `PaymentRegistry` обязателен тест на повторное создание по одной связке `(merchant_id, iin, contract_number)`.
