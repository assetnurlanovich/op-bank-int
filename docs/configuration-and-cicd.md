# Configuration and CI/CD

## 1. Назначение

Этот документ закрывает пробел по эксплуатационной части.
По нему агент может написать:
- конфигурационный слой;
- `Dockerfile` и локальный `docker-compose`;
- CI pipeline;
- CD pipeline;
- smoke checks, миграции и rollback-процедуры.

---

## 2. Среды

| Среда | Назначение |
|-------|------------|
| `local` | Локальная разработка |
| `dev` | Интеграционная среда для общей проверки |
| `test` | Автотесты и ephemeral environments |
| `prod` | Боевая среда |

## 2.1. Общие правила

- каждая среда использует собственные БД, Redis, Kaspi URL и merchant secrets;
- `prod` никогда не использует тестовые секреты;
- локальная среда допускает mock Kaspi вместо реального банка;
- application config читается из env, а не из зашитых констант.

---

## 3. Обязательные конфигурационные параметры

## 3.1. Базовые

| Переменная | Обяз. | Пример | Назначение |
|------------|-------|--------|------------|
| `APP_NAME` | да | `op-bank-int` | Имя приложения |
| `APP_ENV` | да | `dev` | Среда исполнения |
| `HTTP_PORT` | да | `8080` | Порт HTTP API |
| `LOG_LEVEL` | да | `info` | Уровень логирования |
| `SHUTDOWN_TIMEOUT` | да | `10s` | Graceful shutdown timeout |

## 3.2. PostgreSQL

| Переменная | Обяз. | Назначение |
|------------|-------|------------|
| `POSTGRES_DSN` | да | Подключение к PostgreSQL |
| `POSTGRES_MAX_OPEN_CONNS` | да | Максимум открытых коннектов |
| `POSTGRES_MAX_IDLE_CONNS` | да | Максимум idle-коннектов |
| `POSTGRES_CONN_MAX_LIFETIME` | да | Время жизни коннекта |

## 3.3. Redis

| Переменная | Обяз. | Назначение |
|------------|-------|------------|
| `REDIS_ADDR` | да | Адрес Redis |
| `REDIS_PASSWORD` | нет | Пароль Redis |
| `REDIS_DB` | нет | Номер БД |

## 3.4. HTTP server и security

| Переменная | Обяз. | Назначение |
|------------|-------|------------|
| `HTTP_READ_TIMEOUT` | да | Таймаут чтения |
| `HTTP_WRITE_TIMEOUT` | да | Таймаут записи |
| `HTTP_IDLE_TIMEOUT` | да | Idle timeout |
| `REPLAY_WINDOW_SECONDS` | да | Допустимое окно для `request_time` |
| `MASK_SENSITIVE_LOG_FIELDS` | да | Включение маскирования |

## 3.5. Kaspi outbound

| Переменная | Обяз. | Назначение |
|------------|-------|------------|
| `KASPI_BASE_URL` | да | Базовый URL Kaspi |
| `KASPI_INVOICE_PATH` | да | Path invoice registration |
| `KASPI_STATUS_PATH` | да | Path status check |
| `KASPI_TIMEOUT` | да | Таймаут вызовов в Kaspi |
| `KASPI_STATUS_RETRY_COUNT` | да | Retry count для status |
| `KASPI_STATUS_RETRY_BACKOFF` | да | Backoff между retry |

## 3.6. Merchant config

## Канонический способ для production:

`MERCHANTS_JSON`

Пример структуры:

```json
[
  {
    "id": 1,
    "code": "lombard-main",
    "short_name": "Lombard Main",
    "secret": "secret-value",
    "email": "ops@example.kz",
    "active": true,
    "replay_sec": 300,
    "bank_url": "https://sandbox.bank.example"
  }
]
```

## Упрощенный способ для local/dev:

- допускается обратная совместимость с:
  `ONEPAY_MERCHANT_ID`
  `KASPI_SECRET`

- но runtime config приложения должен нормализовать эти значения в общую merchant-модель.

## 3.7. Background jobs

| Переменная | Обяз. | Назначение |
|------------|-------|------------|
| `STATUS_POLL_INTERVAL` | да | Частота запуска job |
| `STATUS_POLL_BATCH_SIZE` | да | Размер батча |
| `STATUS_CHECK_INITIAL_DELAY` | да | Задержка до первой fallback-проверки |
| `POSTING_RETRY_INTERVAL` | да | Интервал retry внутреннего posting |
| `POSTING_RETRY_MAX_ATTEMPTS` | да | Максимум retry |

## 3.8. Health and metrics

| Переменная | Обяз. | Назначение |
|------------|-------|------------|
| `HEALTHCHECK_PATH` | да | Например `/healthz` |
| `READINESS_PATH` | да | Например `/readyz` |
| `METRICS_PATH` | да | Например `/metrics` |

---

## 4. Репозиторные deliverables

- агент должен в итоге создать:
- `Dockerfile`;
- `docker-compose.yml` для local/dev;
- `Makefile`;
- конфиг CI в `.github/workflows/` или эквивалент целевой CI-системы;
- `deploy/` manifests или chart values;
- миграции в `db/migrations/`;
- OpenAPI spec в `api/openapi.yaml`.

---

## 5. CI pipeline

## 5.1. Trigger rules

- push в feature branch;
- pull request / merge request;
- push в main;
- ручной запуск release pipeline.

## 5.2. Обязательные стадии

1. `format-check`
2. `lint`
3. `unit-tests`
4. `integration-tests`
5. `build-binary`
6. `build-image`
7. `migration-validate`
8. `openapi-validate`

## 5.3. Содержимое стадий

### `format-check`
- проверка форматирования Go-кода;
- проверка, что generated artifacts актуальны.

### `lint`
- `go vet`;
- статический анализ;
- проверка отсутствия secrets в репозитории.

### `unit-tests`
- запуск unit tests с race detector там, где это приемлемо по времени.

### `integration-tests`
- подъем PostgreSQL и Redis в CI;
- запуск handler и repository tests;
- прогон mock Kaspi server tests.

### `build-binary`
- сборка Linux binary;
- проверка, что бинарник стартует с `--help` или health bootstrap.

### `build-image`
- сборка OCI image;
- tag по commit SHA;
- опционально tag по semver release.

### `migration-validate`
- проверка, что миграции применяются на чистую БД;
- проверка, что приложение стартует после миграций.

### `openapi-validate`
- проверка синтаксиса OpenAPI;
- проверка соответствия route inventory.

---

## 6. CD pipeline

## 6.1. Общий порядок

1. Build image.
2. Push image в registry.
3. Применить DB migrations отдельным job.
4. Развернуть приложение.
5. Выполнить smoke tests.
6. Перевести rollout в success либо запустить rollback.

## 6.2. Smoke tests после deploy

- `GET /healthz` -> `200`;
- `GET /readyz` -> `200`;
- миграции применены;
- приложение подключается к PostgreSQL и Redis;
- тестовый подписанный запрос к одному из read-only endpoint проходит в non-prod.

## 6.3. Rollback policy

- rollback приложения выполняется отдельным deploy image предыдущей стабильной версии;
- rollback БД выполняется только если миграция явно поддерживает безопасный downgrade;
- если downgrade миграции небезопасен, используется strategy forward-fix.

---

## 7. Контейнеризация

## 7.1. Dockerfile requirements

- multi-stage build;
- runtime image без toolchain;
- запуск от non-root user;
- экспорт HTTP-порта;
- команда запуска - один бинарник API сервиса.

## 7.2. docker-compose для локальной среды

- сервисы:
  `app`
  `postgres`
  `redis`
  `mock-kaspi`

- должны быть volumes для Postgres;
- должны быть healthchecks;
- должна быть возможность применить миграции одной командой.

---

## 8. Kubernetes / deploy contract

- deployment должен иметь readiness и liveness probes;
- config и secrets должны приходить через `ConfigMap` / `Secret` или эквивалент;
- миграции должны выполняться как отдельный `Job`, а не внутри основного startup path;
- реплики приложения должны масштабироваться горизонтально;
- pod disruption не должен приводить к потере подтвержденного платежного факта.

---

## 9. Observability в delivery-процессе

- обязательны structured logs в stdout;
- обязательны Prometheus-совместимые metrics;
- trace context должен прокидываться в outbound integrations;
- alert rules минимум:
  spike signature failures
  callback processing failures
  status polling failures
  posting retries growth
  readiness failures

---

## 10. Требования к quality gate

- merge в main запрещен при падении любой обязательной стадии CI;
- coverage не является единственным gate, но критичные сценарии должны быть покрыты;
- обязательны тесты:
  duplicate callback
  invalid signature
  stale request
  fallback status
  posting retry after external success

---

## 11. Что агент должен считать definition of done

- приложение собирается в контейнер;
- миграции создают всю необходимую схему;
- все внешние endpoint покрыты handler/integration tests;
- есть mock/stub для outbound Kaspi;
- есть CI pipeline;
- есть deploy manifests;
- есть OpenAPI spec;
- есть README с командами запуска и тестирования.
