# OnePay Bank Integration Service — Overview

## Назначение

Данное приложение реализует интеграцию OnePay с банками-партнерами.

На текущем этапе реализуется интеграция с Kaspi.

Приложение является отдельным серверным компонентом и не является Платежным Шлюзом. Оно выполняет роль адаптера между внешним API банка и внутренними системами OnePay / Ломбарда.

---

## Основные функции

Приложение обеспечивает:

- прием запросов Kaspi:
  - contracts
  - repayment
  - calculate-repayment
  - payment
  - callback

- вызовы в Kaspi:
  - invoice / payment registration
  - status (резервная сверка)

- обработку платежных событий:
  - создание платежа
  - обновление статусов
  - идемпотентная обработка callback

- интеграцию с внутренними системами:
  - получение договоров
  - расчет задолженности
  - подготовка invoice-based платежа
  - проведение погашения

---

## Архитектурная роль

Приложение:
- изолирует банковскую интеграцию от остальной системы;
- обеспечивает независимый lifecycle;
- позволяет масштабировать интеграции на другие банки.

## Приоритет документов

При конфликте документов использовать следующий приоритет:
1. `api-contracts.md`, `request-validation-and-signature.md`, `outbound-kaspi-contracts.md`, `inbound-onepay-commands.md`, `error-codes.md`
2. `lombard-api-contracts.md`, `internal-system-contracts.md`, `service-logic.md`, `data-model-and-state.md`
3. `configuration-and-cicd.md`, `nonfunctional-requirements.md`
4. `architecture.md`, `architecture2.md`, `overview.md`, `kaspi-protocol.md`

Если summary-документ расходится с документом более высокого приоритета, источником истины считается документ с более высоким приоритетом.

---

## Ключевые принципы

- stateless сервис
- идемпотентная обработка событий
- audit trail всех операций
- разделение внешнего и внутреннего контрактов
- fallback через status API

---

## Scope текущей версии

- интеграция только с Kaspi
- REST API (JSON)
- PostgreSQL + Redis
- синхронная обработка + фоновые задачи
- delivery через контейнеризацию и CI/CD pipeline по `docs/configuration-and-cicd.md`

---

## Out of scope

- функционал полного платежного шлюза
- работа с картами и PCI
- back-office