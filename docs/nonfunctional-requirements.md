# Non-Functional Requirements

## 1. Purpose

This document defines non-functional requirements for the OnePay ↔ Kaspi integration application.

These requirements are mandatory implementation constraints for architecture, development, testing and operations.

---

## 2. Performance

## 2.1. External request latency

- Maximum processing time for external Kaspi request: **no more than 15 seconds**
- Normal target latency should be significantly lower than the hard limit
- Signature validation and idempotency checks must be lightweight

## 2.2. Concurrency

At the initial load profile the service must support:
- at least **10–15 concurrent external connections**
- at least **10+ payment-related operations per minute** without degradation

## 2.3. Background processing

Fallback status checks, retries and reconciliation jobs must not block synchronous request handling.

---

## 3. Availability and resiliency

## 3.1. Availability principles

The application must:
- remain available when individual payment operations fail;
- degrade gracefully when Kaspi-side API is slow or unavailable;
- preserve payment state during transient failures.

## 3.2. Resiliency mechanisms

The application must implement:
- timeouts for all outbound calls;
- retry policy only where business-safe;
- circuit breaker / failure isolation for outbound Kaspi calls;
- graceful shutdown;
- recovery jobs for stuck operations.

## 3.3. Fallback mechanism

If callback is missing, broken or disputed, the application must use Kaspi `status` API as fallback verification source.

---

## 4. Consistency and idempotency

## 4.1. Callback idempotency

Repeated callback processing must not:
- duplicate repayment posting;
- duplicate status transitions;
- create duplicate business records.

## 4.2. State consistency

State-changing operations must be:
- atomic;
- auditable;
- replay-safe.

## 4.3. Data durability

All confirmed external payment facts must be persisted before any irreversible internal action is considered successful.

---

## 5. Security

## 5.1. Transport security

- all external interaction must use HTTPS;
- plain HTTP is not allowed outside local development.

## 5.2. Request authenticity

Every incoming request must be validated by:
- `merchant_id`
- `request_time`
- `sign`

## 5.3. Replay protection

The application must enforce an allowed time window for `request_time` and reject suspicious stale requests.

## 5.4. Secret management

- secrets must not be stored in repository;
- secrets must be injected through environment variables or secret manager;
- different environments must use different secrets.

## 5.5. Secure logging

The service must avoid logging raw secrets and must mask sensitive fields when necessary.

---

## 6. Scalability

## 6.1. Application layer

The application layer must be stateless so that multiple instances can run in parallel behind a load balancer.

## 6.2. Storage layer

PostgreSQL is the system of record. Redis may be used for short-lived technical acceleration but not as the source of truth for payment state.

## 6.3. Future growth

Architecture must allow:
- onboarding additional banks;
- extraction of bank adapters into separate deployables;
- growth in traffic without redesign of the domain model.

---

## 7. Maintainability

## 7.1. Code structure

The codebase must be organized into explicit layers:
- transport;
- application;
- domain;
- infrastructure.

## 7.2. Configuration

The application must support separate configurations for:
- `local`
- `dev`
- `test`
- `prod`

## 7.3. Change safety

Any DB schema change must go through versioned SQL migrations.

## 7.4. Contract stability

External API responses must use a stable response envelope and contract-first discipline.

---

## 8. Observability

## 8.1. Logging

The service must produce structured logs containing:
- timestamp;
- level;
- request id / trace id;
- endpoint;
- merchant id where available;
- payment id where available;
- status / error summary.

## 8.2. Metrics

The service must expose operational metrics at minimum for:
- request count;
- error count;
- callback duplicate count;
- outbound Kaspi status checks;
- request latency;
- payment status transition counts.

## 8.3. Tracing

Each request should be traceable across:
- inbound request;
- domain processing;
- outbound internal calls;
- outbound Kaspi calls.

## 8.4. Health endpoints

The service must expose:
- liveness probe;
- readiness probe.

---

## 9. Auditability

The application must keep sufficient audit trail for:
- inbound requests;
- callback payloads;
- status changes;
- reconciliation attempts;
- technical failures on posting / recovery.

Audit data must allow reconstruction of the payment event timeline.

---

## 10. Testability

## 10.1. Mandatory automated tests

The implementation must include:
- unit tests for signature validation;
- unit tests for status mapping;
- unit tests for idempotency rules;
- integration tests for HTTP handlers;
- integration tests for callback duplicate processing;
- integration tests for fallback status check flow.

## 10.2. Mocking

Kaspi integration must be testable against:
- mock HTTP server;
- sandbox/stub behavior;
- deterministic fixture payloads.

---

## 11. Deployment requirements

## 11.1. Runtime

Recommended runtime stack:
- Go application
- PostgreSQL
- Redis
- containerized deployment

## 11.2. Container behavior

Container must:
- start fast;
- terminate gracefully;
- expose health endpoints;
- read config from env / mounted config only.

## 11.3. Environment isolation

Local and test environments must not use production Kaspi credentials or production merchant secrets.

---

## 12. Data retention and housekeeping

The application should support:
- retention policy for raw request/response logs;
- archival or cleanup of old technical logs;
- preserving payment audit history independently from temporary technical cache records.

---

## 13. Operational alerts

At minimum alerts should exist for:
- spike in signature validation failures;
- high callback failure rate;
- increasing number of payments in pending state;
- status polling failures;
- readiness failures;
- repeated internal posting failures after successful external payment.

---

## 14. Engineering constraints for coding agent

The coding agent must follow these rules:
- do not hardcode secrets;
- do not bypass idempotency checks;
- do not perform direct business side effects from handlers;
- centralize signature validation;
- centralize response envelope formatting;
- make outbound client behavior configurable;
- cover critical branches with tests.