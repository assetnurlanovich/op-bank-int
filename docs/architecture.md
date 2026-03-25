# Архитектура приложения интеграции OnePay ↔ Kaspi

## 1. Назначение

Приложение интеграции OnePay ↔ Kaspi представляет собой отдельное серверное приложение, выделенное из общего контура Платежного Шлюза.

Назначение приложения:
- предоставлять внешние API OnePay для вызовов со стороны Kaspi;
- инициировать исходящие вызовы в Kaspi для сценариев, стартующих на стороне OnePay / Ломбарда;
- выполнять исходящие вызовы в API Kaspi, необходимые для проверки статуса операций;
- обеспечивать безопасную и идемпотентную обработку платежных событий;
- изолировать банковскую интеграцию Kaspi от остальных компонентов Платежного Шлюза;
- обеспечить возможность дальнейшего масштабирования на других банковских партнеров.

Данное приложение не является полным Платежным Шлюзом и не реализует весь набор функций gateway-платформы. Оно является специализированным интеграционным сервисом для банковского взаимодействия.

---

## 2. Архитектурная роль в ландшафте систем

Приложение находится между:
- системой Kaspi;
- внутренними системами OnePay / Ломбарда.

С точки зрения архитектурной роли приложение выполняет функции:
- внешнего API-адаптера;
- оркестратора интеграционного сценария;
- процессора callback-уведомлений;
- сервиса резервной сверки статусов;
- журнала интеграционных событий.

Приложение должно быть реализовано как самостоятельный deployable-unit с независимым циклом поставки, тестирования и эксплуатации.

---

## 3. Границы ответственности

### 3.1. Что входит в ответственность приложения

Приложение должно:
- принимать запросы Kaspi к сервисам OnePay;
- валидировать параметры запроса и подпись;
- маршрутизировать запрос в соответствующую внутреннюю бизнес-логику;
- формировать ответы в согласованном JSON-формате;
- создавать и сопровождать платежную операцию в части интеграционного состояния;
- инициировать регистрацию счета / платежа в Kaspi для сценария оплаты, начатого на стороне OnePay / Ломбарда;
- принимать callback от Kaspi;
- обеспечивать идемпотентную обработку callback;
- инициировать резервную проверку статуса через сервис Kaspi `status`;
- вести аудит запросов, ответов и переходов статусов;
- предоставлять технические точки контроля: healthcheck, readiness, metrics, logs, tracing.

### 3.2. Что не входит в ответственность приложения

Приложение не должно:
- реализовывать весь контур универсального Платежного Шлюза;
- обслуживать карточные данные, PCI Vault или tokenization;
- реализовывать back-office, merchant portal, ledger или billing общего шлюза;
- хранить или обрабатывать функционал, не относящийся к интеграции с Kaspi;
- подменять собой учетную систему Ломбарда.

---

## 4. Архитектурный стиль

Для сервиса принимается следующий архитектурный стиль:
- модульный монолит на старте;
- stateless application layer;
- четкое разделение слоев transport / application / domain / infrastructure;
- внешнее взаимодействие по REST/JSON поверх HTTPS;
- хранение состояния в PostgreSQL;
- использование Redis для технического кэша и краткоживущих ключей;
- синхронная обработка запросов;
- фоновые задачи для status polling / retry / reconciliation.

Такой подход выбран как достаточный для первого этапа интеграции при сохранении возможности дальнейшего выделения отдельных компонентов.

---

## 5. Логическая схема компонентов

## 5.1. Компоненты приложения

Приложение включает следующие логические компоненты:

### A. HTTP API Layer
Входной слой для внешних запросов:
- `GET /api/kaspi/contracts`
- `GET /api/kaspi/repayment`
- `GET /api/kaspi/calculate-repayment`
- `GET /api/kaspi/payment`
- `POST /api/kaspi/callback`

Функции слоя:
- прием HTTP-запросов;
- парсинг параметров;
- базовая валидация;
- корреляция request id / trace id;
- возврат унифицированного JSON-ответа.

### B. Signature & Security Layer
Компонент, отвечающий за:
- проверку `merchant_id`;
- проверку `request_time`;
- валидацию подписи `sign`;
- защиту от replay-атак в рамках допустимого временного окна;
- маскирование чувствительных данных в логах.

### C. Application Services Layer
Слой сценарной логики:
- Contracts Service;
- Repayment Service;
- Calculate Repayment Service;
- Payment Service;
- Invoice Registration / Payment Preparation flow;
- Callback Processing Service;
- Payment Status Check Service.

### D. Domain Layer
Содержит предметные сущности и правила:
- Merchant;
- Contract;
- Repayment Info;
- Payment;
- Payment Attempt;
- External Payment Reference;
- Callback Event;
- Payment Status;
- Reconciliation Result;
- Idempotency Key.

### E. Persistence Layer
Отвечает за хранение:
- платежей;
- callback-событий;
- журналов интеграции;
- idempotency-ключей;
- результатов сверки статусов;
- технических задач на повторную проверку.

### F. Kaspi Client
Исходящий HTTP-клиент для вызовов в Kaspi:
- регистрация счета / платежа для invoice-based сценария;
- `status`;
- при необходимости дальнейших исходящих сервисов по мере развития протокола.

### G. Background Jobs
Фоновый контур:
- повторная проверка статуса платежей;
- retry внутреннего posting после подтвержденного внешнего платежа;
- reconciliation спорных операций;
- обработка зависших операций;
- техническая очистка и housekeeping.

### H. Observability Layer
Компонент наблюдаемости:
- structured logging;
- metrics;
- tracing;
- alerting hooks.

---

## 6. Схема взаимодействия

### 6.1. Входящие потоки от Kaspi

Kaspi вызывает API приложения:
1. contracts;
2. repayment;
3. calculate-repayment;
4. payment;
5. callback.

### 6.2. Исходящие потоки в Kaspi

Приложение вызывает API Kaspi:
1. сервис регистрации счета / платежа — для сценария оплаты, начатого на стороне OnePay / Ломбарда;
2. status — резервная сверка при отсутствии callback, ошибке callback или спорной операции.

### 6.3. Внутренние потоки

Приложение взаимодействует с внутренними сервисами OnePay / Ломбарда:
- сервисом договоров;
- сервисом расчета задолженности;
- сервисом создания/поиска платежей;
- сервисом подготовки платежа и выставления счета клиенту;
- сервисом проведения погашения;
- сервисом уведомлений;
- внутренним audit/log storage.

На старте эти взаимодействия могут быть реализованы через HTTP/gRPC или через адаптеры к существующим внутренним API.

### 6.4. Ключевые архитектурные правила сценариев

- callback является основным источником истины по факту оплаты;
- `return_url` не является подтверждением успешной оплаты;
- подтвержденный внешний платеж должен быть сохранен до запуска необратимых внутренних side effects;
- сбой внутреннего posting после успешного callback не должен терять внешний факт оплаты и переводит операцию в retry / reconciliation;
- все входящие и исходящие интеграционные сообщения должны быть доступны для аудита.

---

## 7. Слои приложения

## 7.1. Transport Layer

Назначение:
- работа с HTTP;
- middleware;
- serialization / deserialization;
- mapping HTTP ↔ application DTO.

Папки:
- `internal/transport/http`
- `internal/transport/http/handlers`
- `internal/transport/http/middleware`

## 7.2. Application Layer

Назначение:
- реализация пользовательских и интеграционных сценариев;
- координация domain logic и infrastructure adapters.

Папки:
- `internal/app/contracts`
- `internal/app/repayment`
- `internal/app/calculate`
- `internal/app/payment`
- `internal/app/callback`
- `internal/app/statuscheck`

## 7.3. Domain Layer

Назначение:
- бизнес-сущности;
- статусы;
- инварианты;
- правила перехода состояний.

Папки:
- `internal/domain/payment`
- `internal/domain/contract`
- `internal/domain/merchant`
- `internal/domain/common`

## 7.4. Infrastructure Layer

Назначение:
- postgres repositories;
- redis adapters;
- kaspi http client;
- internal system clients;
- scheduler/jobs;
- telemetry.

Папки:
- `internal/store`
- `internal/cache`
- `internal/integration/kaspi`
- `internal/integration/onepay`
- `internal/jobs`
- `internal/telemetry`

---

## 8. Предлагаемая структура репозитория

```text
.
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── app/
│   │   ├── contracts/
│   │   ├── repayment/
│   │   ├── calculate/
│   │   ├── payment/
│   │   ├── callback/
│   │   └── statuscheck/
│   ├── domain/
│   │   ├── common/
│   │   ├── merchant/
│   │   ├── contract/
│   │   └── payment/
│   ├── transport/
│   │   └── http/
│   │       ├── handlers/
│   │       ├── middleware/
│   │       └── dto/
│   ├── integration/
│   │   ├── kaspi/
│   │   └── onepay/
│   ├── store/
│   ├── cache/
│   ├── jobs/
│   ├── config/
│   ├── telemetry/
│   └── platform/
├── db/
│   ├── migrations/
│   └── queries/
├── config/
├── docs/
├── deploy/
└── scripts/
```