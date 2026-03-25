# API Contracts — OnePay ↔ Kaspi

## Общие правила типов и обязательности

Если для параметра не указано иное, параметр считается обязательным.

### Базовые типы
- `integer` — целое число без дробной части
- `number(18,2)` — числовое значение с двумя знаками после запятой
- `string` — строка
- `timestamp14` — дата и время в формате `YYYYMMDDHHMMSS`

### Общие обязательные технические параметры
Для всех запросов обязательны:
- `merchant_id: integer`
- `request_time: timestamp14`
- `sign: string`

### Общие правила валидации
- строковые параметры не должны быть пустыми после trim;
- числовые параметры должны быть больше либо равны нулю, если явно не указано иное;
- `merchant_id` должен быть допустимым идентификатором мерчанта;
- `request_time` должен соответствовать формату `YYYYMMDDHHMMSS`;
- запрос с отсутствующим обязательным параметром отклоняется;
- запрос с параметром неверного типа отклоняется;
- запрос с неверной подписью отклоняется.

> Правила типов, обязательности, валидации параметров и формирования подписи для всех endpoint описаны в документе `request-validation-and-signature.md`.  
> Для каждого endpoint в настоящем документе указаны конкретные параметры запроса и ответа.

## Общий формат ответа

```json
{
  "success": true,
  "data": {},
  "error_code": 0,
  "message": "Успешно"
}
```

## 1. contracts

`GET /api/kaspi/contracts`

### Параметры запроса

| Параметр      | Тип         | Обязательность | Описание |
|---------------|-------------|----------------|----------|
| merchant_id   | integer     | да             | Идентификатор мерчанта в OnePay |
| iin           | string(12)  | да             | ИИН клиента, 12 цифр |
| request_time  | timestamp14 | да             | Время формирования запроса в формате YYYYMMDDHHMMSS |
| sign          | string      | да             | Подпись запроса |

### Правила валидации
- `merchant_id` > 0
- `iin` должен содержать ровно 12 цифр
- `request_time` должен быть корректной датой/временем
- `sign` не должен быть пустым

### Правила формирования подписи
Подробные правила формирования и проверки подписи определены в документе `request-validation-and-signature.md`.
Для данного сервиса базовое поле подписи:

```text
iin
```

Формула подписи:
```text
sha512(iin + request_time + secret)
```

### Параметры ответа data

| Параметр      | Тип               | Обязательность | Описание |
|---------------|-------------------|----------------|----------|
| merchant_id   | integer           | да             | Идентификатор мерчанта в OnePay |
| iin           | string(12)        | да             | ИИН клиента, 12 цифр |
| contracts     | array of contract | да             | Список договоров клиента, может быть пустым |


### Структура элемента `contracts[]`
| Параметр        | Тип           | Обязательность | Описание |
|-----------------|---------------|----------------|----------|
| contract_number | string(1..50) | да             | Номер договора |
| contract_status | string(1..30) | да             | Статус договора |

### Пример запроса
```text
GET /api/kaspi/contracts?merchant_id=1&iin=900101300123&request_time=20260316093015&sign=8a5f...
```

### Примеры ответов

```json
{
  "success": true,
  "data": {
    "merchant_id": 1,
    "iin": "900101300123",
    "contracts": [
      {
        "contract_number": "DZ-458721",
        "contract_status": "ACTIVE"
      },
      {
        "contract_number": "DZ-458722",
        "contract_status": "ACTIVE"
      }
    ]
  },
  "error_code": 0,
  "message": "Список договоров получен"
}

{
  "success": true,
  "data": {
    "merchant_id": 1,
    "iin": "900101300123",
    "contracts": []
  },
  "error_code": 0,
  "message": "Договоры не найдены"
}

{
  "success": false,
  "data": {
    "merchant_id": 1,
    "iin": "900101300123"
  },
  "error_code": 1002,
  "message": "Клиент не найден"
}
```


## 2. repayment

`GET /api/kaspi/repayment`

### Параметры запроса

| Параметр        | Тип           | Обяз. | Описание |
|-----------------|---------------|-------|----------|
| merchant_id     | integer       | да    | Идентификатор мерчанта |
| contract_number | string(1..50) | да    | Номер договора |
| iin             | string(12)    | да    | ИИН клиента |
| repayment_type  | integer       | да    | Тип погашения: 1 — ежемесячное, 2 — частичное/полное |
| reference_id    | string(1..64) | да    | Идентификатор платежа на стороне Kaspi |
| request_time    | timestamp14   | да    | Время формирования запроса |
| sign            | string        | да    | Подпись запроса |

### Правила валидации
- `merchant_id` > 0
- `contract_number` не пустой
- `iin` должен содержать ровно 12 цифр
- `repayment_type` ∈ `{1, 2}`
- `reference_id` не должен быть пустым
- `request_time` должен быть корректной датой и временем
- `sign` не должен быть пустым

### Правила формирования подписи
Подробные правила формирования и проверки подписи определены в документе `request-validation-and-signature.md`.
Для данного сервиса базовое поле подписи:

```text
contract_number
```

Формула подписи:
```text
sha512(contract_number + request_time + secret)
```

### Параметры ответа data для repayment_type = 1
| Параметр           | Тип            | Обяз. | Описание |
|--------------------|----------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	да    |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| iin                |	string(12)    |	нет   |	ИИН клиента |
| contract_number    |	string(1..50) |	да    |	Номер договора |
| repayment_type     |	integer       |	да    |	Тип погашения, значение 1 |
| monthly_due_amount |	number(18,2)  |	да    |	Сумма ежемесячного платежа |
| interest_amount    |	number(18,2)  |	да    |	Сумма процентов |
| penalty_amount     |	number(18,2)  |	да    |	Сумма штрафов |
| full_due_amount    |	number(18,2)  |	да    |	Полная сумма задолженности |
| fee_amount         |	number(18,2)  |	да    |	Комиссия |
| total_amount       |	number(18,2)  |	да    |	Итоговая сумма к оплате |
| currency           |	string(3)     |	да    |	Валюта, например KZT |

### Параметры ответа data для repayment_type = 2
| Параметр           | Тип            | Обяз. | Описание |
|--------------------|----------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	да    |	Идентификатор платежа на стороне Kaspi |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| iin                |	string(12)    |	да    |	ИИН клиента |
| contract_number    |	string(1..50) |	да    |	Номер договора |
| repayment_type     |	integer       |	да    |	Тип погашения, значение 2 |
| interest_amount    |	number(18,2)  |	да    |	Сумма процентов |
| penalty_amount     |	number(18,2)  |	да    |	Сумма штрафов |
| full_due_amount    |	number(18,2)  |	да    |	Полная сумма задолженности |
| min_payment_amount |	number(18,2)  |	нет   |	Минимально допустимая сумма погашения |
| max_payment_amount |	number(18,2)  | нет   |	Максимально допустимая сумма погашения |
| currency           |	string(3)     |	да    |	Валюта, например KZT |

### Пример запроса (repayment_type=1)
```text
GET /api/kaspi/repayment?merchant_id=1&contract_number=DZ-458721&iin=900101300123&repayment_type=1&reference_id=KSP-SESSION-00001&request_time=20260316093210&sign=6b31...
```

### Примеры ответа (repayment_type=1)

```json
{
  "success": true,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-SESSION-00001",
    "payment_id": 202603160001,
    "iin": "900101300123",
    "contract_number": "DZ-458721",
    "repayment_type": 1,
    "monthly_due_amount": 15000.00,
    "interest_amount": 3000.00,
    "penalty_amount": 1200.00,
    "full_due_amount": 120000.00,
    "fee_amount": 327.27,
    "total_amount": 16527.27,
    "currency": "KZT"
  },
  "error_code": 0,
  "message": "Параметры ежемесячного погашения получены"
}
```

### Пример запроса (repayment_type=2)
```text
GET /api/kaspi/repayment?merchant_id=1&iin=900101300123&contract_number=DZ-458721&repayment_type=2&reference_id=KSP-SESSION-00002&request_time=20260316093420&sign=9f42...
```

### Примеры ответа (repayment_type=2)

```json
{
  "success": true,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-SESSION-00002",
    "payment_id": 202603160002,
    "iin": "900101300123",
    "contract_number": "DZ-458721",
    "repayment_type": 2,
    "interest_amount": 3000.00,
    "penalty_amount": 1200.00,
    "full_due_amount": 120000.00,
    "min_payment_amount": 1200.00,
    "max_payment_amount": 120000.00,
    "currency": "KZT"
  },
  "error_code": 0,
  "message": "Параметры частичного или полного погашения получены"
}
```


### Ответ с ошибкой

```json
{
  "success": false,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-SESSION-00002",
    "iin": "900101300123",
    "contract_number": "DZ-458721",
    "repayment_type": 2
  },
  "error_code": 1003,
  "message": "Договор недоступен для оплаты"
}
```

## 3. calculate-repayment

`GET /api/kaspi/calculate-repayment`

### Параметры запроса

| Параметр        | Тип           | Обяз. | Описание |
|-----------------|---------------|-------|----------|
| merchant_id     | integer       | да    | Идентификатор мерчанта |
| contract_number | string(1..50) | да    | Номер договора |
| iin             | string(12)    | да    | ИИН клиента |
| payment_id      | integer       | да    | Идентификатор платежа в OnePay |
| entered_amount  | number(18,2)  | да    | Введенная клиентом сумма |
| reference_id    | string(1..64) | да    | Идентификатор запроса/сессии |
| request_time    | timestamp14   | да    | Время формирования запроса |
| sign            | string        | да    | Подпись запроса |

### Правила валидации
- `merchant_id` > 0
- `payment_id` > 0
- `entered_amount` > 0
- `entered_amount` должен передаваться с точностью не более 2 знаков после запятой
- `contract_number` не пустой
- `reference_id` не пустой
- `iin содержит` 12 цифр

### Параметры ответа data при успешном расчете
| Параметр        | Тип           | Обяз. | Описание |
|-----------------|---------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	да    |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| iin                |	string(12)    |	нет   |	ИИН клиента |
| contract_number    |	string(1..50) |	да    |	Номер договора |
| repayment_type     |	integer       |	да    |	Тип погашения |
| entered_amount     |  number(18,2)  | да    | Введенная клиентом сумма |
| interest_amount    |	number(18,2)  |	да    |	Сумма процентов |
| penalty_amount     |	number(18,2)  |	да    |	Сумма штрафов |
| repayment_amount   |	number(18,2)  |	да    |	Сумма, направляемая на погашение |
| fee_amount         |	number(18,2)  |	да    |	Комиссия |
| total_amount       |	number(18,2)  |	да    |	Итоговая сумма к оплате |
| currency           |	string(3)     |	да    |	Валюта, например KZT |

### Параметры ответа data при ошибке 1005
| Параметр           | Тип            | Обяз. | Описание |
|--------------------|----------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	да    |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| iin                |	string(12)    |	нет   |	ИИН клиента |
| contract_number    |	string(1..50) |	да    |	Номер договора |
| repayment_type     |	integer       |	да    |	Тип погашения |
| entered_amount     |  number(18,2)  | да    | Введенная клиентом сумма |
| min_payment_amount |	number(18,2)  |	нет   |	Минимально допустимая сумма погашения |

### Параметры ответа data при ошибке 1006
| Параметр           | Тип            | Обяз. | Описание |
|--------------------|----------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	да    |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| contract_number    |	string(1..50) |	да    |	Номер договора |
| repayment_type     |	integer       |	да    |	Тип погашения |
| entered_amount     |  number(18,2)  | да    | Введенная клиентом сумма |
| full_due_amount    |	number(18,2)  |	да    |	Полная сумма задолженности |

### Пример запроса
```text
GET /api/kaspi/calculate-repayment?merchant_id=1&iin=900101300123&contract_number=DZ-458721&payment_id=20260316000002&entered_amount=30000.00&reference_id=KSP-SESSION-00002&request_time=20260316093655&sign=0c91...
```


### Успешный ответ

```json
{
  "success": true,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-SESSION-00002",
    "payment_id": 202603160002,
    "iin": "900101300123",
    "contract_number": "DZ-458721",
    "repayment_type": 2,
    "entered_amount": 30000.00,
    "interest_amount": 3000.00,
    "penalty_amount": 1200.00,
    "repayment_amount": 25800.00,
    "fee_amount": 612.24,
    "total_amount": 30612.24,
    "currency": "KZT"
  },
  "error_code": 0,
  "message": "Расчет платежа выполнен"
}
```

### Ответы с ошибкой: "сумма меньше минимально допустимой"

```json
{
  "success": false,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-SESSION-00002",
    "payment_id": 202603160002,
    "contract_number": "DZ-458721",
    "repayment_type": 2,
    "entered_amount": 1000.00,
    "min_payment_amount": 1200.00
  },
  "error_code": 1005,
  "message": "Введенная сумма меньше минимально допустимой"
}
```

### Ответы с ошибкой: "сумма больше полной задолженности"

```json
{
  "success": false,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-SESSION-00002",
    "payment_id": 202603160002,
    "contract_number": "DZ-458721",
    "repayment_type": 2,
    "entered_amount": 130000.00,
    "full_due_amount": 120000.00
  },
  "error_code": 1006,
  "message": "Сумма превышает полную задолженность"
}
```


## 4. payment

`GET /api/kaspi/payment`

### Параметры запроса

| Параметр      | Тип           | Обяз. | Описание |
|---------------|---------------|-------|----------|
| merchant_id   | integer       | да    | Идентификатор мерчанта |
| payment_id    | integer       | да    | Идентификатор платежа |
| reference_id  | string(1..64) | да    | Идентификатор платежа на стороне Каспи |
| request_time  | timestamp14   | да    | Время формирования запроса |
| sign          | string        | да    | Подпись запроса |

### Правила валидации
- `merchant_id` > 0
- `payment_id` > 0
- `reference_id` не должен быть пустым
- `request_time` должен быть корректной датой и временем
- `sign` не должен быть пустым

### Параметры ответа data при успешном ответе
| Параметр        | Тип           | Обяз. | Описание |
|-----------------|---------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	да    |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| iin                |	string(12)    |	да    |	ИИН клиента |
| contract_number    |	string(1..50) |	да    |	Номер договора |
| repayment_type     |	integer       |	да    |	Тип погашения |
| payment_amount     |  number(18,2)  | да    | Сумма погашения без комиссии |
| fee_amount         |	number(18,2)  |	да    |	Комиссия |
| total_amount       |	number(18,2)  |	да    |	Итоговая сумма к оплате |
| currency           |	string(3)     |	да    |	Валюта, например KZT |
| return_url         |	string(1..255)|	да    |	URL возврата клиента после завершения пользовательского сценария |

### Параметры ответа data при ошибке 1007
| Параметр           | Тип            | Обяз. | Описание |
|--------------------|----------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	да    |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |

### Параметры ответа data при ошибке 1008
| Параметр           | Тип            | Обяз. | Описание |
|--------------------|----------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	да    |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| status             |	integer       |	да    |	Текущий статус платежа |


### Пример запроса
```text
GET /api/kaspi/payment?merchant_id=1&payment_id=20260316000010&reference_id=KSP-SESSION-00003&request_time=20260316094020&sign=2d74...
```


### Успешный ответ

```json
{
  "success": true,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-SESSION-00003",
    "payment_id": 202603160010,
    "repayment_type": 1,
    "iin": "900101300123",
    "contract_number": "DZ-458721",
    "payment_amount": 15000.00,
    "fee_amount": 327.27,
    "total_amount": 15327.27,
    "currency": "KZT",
    "return_url": "https://lombard.example.kz/payment/result?payment_id=202603160010"
  },
  "error_code": 0,
  "message": "Параметры платежа получены"
}
```


### Ошибка: платеж не найден

```json
{
  "success": false,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-SESSION-00003",
    "payment_id": 202603160010
  },
  "error_code": 1007,
  "message": "Платеж не найден"
}
```


### Ошибка: платеж уже обработан

```json
{
  "success": false,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-SESSION-00003",
    "payment_id": 202603160010,
    "status": 1
  },
  "error_code": 1008,
  "message": "Платеж уже обработан"
}
```


## 5. callback

`POST /api/kaspi/callback`

### Параметры запроса

| Параметр        | Тип           | Обяз. | Описание |
|-----------------|---------------|-------|----------|
| merchant_id     | integer       | да    | Идентификатор мерчанта |
| payment_id      | integer       | да    | Идентификатор платежа |
| contract_number | string(1..50) | нет   | Номер договора |
| iin             | string(12)    | нет   | ИИН клиента |
| repayment_type  | integer       | нет   | 1 или 2 |
| status          | integer       | да    | Статус платежа: 1, 2, 5, 6 |
| payment_amount  | number(18,2)  | да    | Сумма погашения без комиссии |
| fee_amount      | number(18,2)  | да    | Сумма комиссии |
| total_amount    | number(18,2)  | да    | Общая сумма платежа |
| error_code      | integer       | нет   | Код ошибки на стороне Kaspi |
| message         | string(1..255)| нет   | Текст ошибки/комментарий |
| rnn             | string(1..64) | нет   | RNN операции |
| reference_id    | string(1..64) | да    | Идентификатор операции на стороне Kaspi |
| request_time    | timestamp14   | да    | Время формирования callback |
| sign            | string        | да    | Подпись callback |

### Правила валидации
- `merchant_id` > 0
- `payment_id` > 0
- `status` ∈ {1, 2, 5, 6}
- `payment_amount` >= 0
- `fee_amount` >= 0
- `total_amount` >= 0
- желательно проверять: `total_amount = payment_amount + fee_amount`
- если передан `repayment_type`, то `repayment_type` ∈ {1, 2}
- `reference_id` не должен быть пустым
- `request_time` должен быть корректной датой и временем
- `sign` не должен быть пустым

### Параметры ответа data при успешной обработке
| Параметр        | Тип           | Обяз. | Описание |
|-----------------|---------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	да    |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| rnn                | string(1..64)  | нет   | RNN операции |
| status             |	integer       |	да    |	Текущий статус платежа |

### Параметры ответа data при ошибке 1010
| Параметр           | Тип            | Обяз. | Описание |
|--------------------|----------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	нет   |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| rnn                | string(1..64)  | нет   | RNN операции |

### Параметры ответа data при ошибке 1009
| Параметр           | Тип            | Обяз. | Описание |
|--------------------|----------------|-------|----------|
| merchant_id        |	integer       |	да    |	Идентификатор мерчанта |
| reference_id       |	string(1..64) |	нет   |	Идентификатор платежа на стороне Каспи |
| payment_id         |	integer       |	да    |	Идентификатор платежа в OnePay |
| rnn                | string(1..64)  | нет   | RNN операции |


### Пример запроса
```http
POST /api/kaspi/callback
Content-Type: application/json
```

```json
{
  "merchant_id": 1,
  "payment_id": 20260316000010,
  "status": 1,
  "total_amount": 15327.27,
  "payment_amount": 15000.00,
  "fee_amount": 327.27,
  "rnn": "RNN-20260316-00003",
  "reference_id": "KSP-PAY-00003",
  "request_time": "20260316094230",
  "sign": "5af8..."
}
```


### Успешный ответ на успешный callback

```json
{
  "success": true,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-PAY-00003",
    "rnn": 9876543210123456,
    "payment_id": 202603160010,
    "status": 1
  },
  "error_code": 0,
  "message": "Платеж принят в обработку"
}
```


### Успешный ответ на повторный callback

```json
{
  "success": true,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-PAY-00003",
    "rnn": 9876543210123456,
    "payment_id": 202603160010,
    "status": 1
  },
  "error_code": 0,
  "message": "Callback ранее уже был обработан"
}
```


### Успешный ответ на неуспешный платеж

```json
{
  "success": true,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-PAY-00003",
    "rnn": 9876543210123456,
    "payment_id": 202603160010,
    "status": 2
  },
  "error_code": 0,
  "message": "Неуспешный статус платежа принят и сохранен"
}
```


### Ошибка: некорректный callback

```json
{
  "success": false,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-PAY-00003",
    "rnn": 9876543210123456
  },
  "error_code": 1010,
  "message": "Некорректные параметры callback"
}
```


### Ошибка: неверная подпись

```json
{
  "success": false,
  "data": {
    "merchant_id": 1,
    "reference_id": "KSP-PAY-00003",
    "rnn": 9876543210123456,
    "payment_id": 202603160010
  },
  "error_code": 1009,
  "message": "Неверная подпись запроса"
}
```

## Примечания для реализации
### Единообразие типов идентификаторов
Для целей реализации в этом документе приняты следующие правила:
- merchant_id — integer
- payment_id — integer
- reference_id — string
- rnn — string

Агент не должен смешивать строковые и числовые типы для reference_id и rnn в разных endpoint или примерах.

### Формирование подписи
Подробные правила формирования и проверки подписи определены в документе request-validation-and-signature.md.

### Проверка денежных полей
Для callback и расчетных endpoint рекомендуется использовать decimal-арифметику, а не float, чтобы избежать ошибок округления.

### Источник истины по статусу оплаты
Источник истины по подтверждению оплаты:
1. callback
2. fallback-проверка через status
return_url не является подтверждением факта оплаты.