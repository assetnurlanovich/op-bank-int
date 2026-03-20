
---

## `docs/service-scenarios.md`

Этот файл — сценарная декомпозиция под разработку. Сценарии ежемесячного, частичного/полного погашения и оплаты по счету прямо перечислены в протоколе; отдельная схема подробно описывает основной и ошибочные потоки именно для выставления счета, включая создание платежа, выставление счета в Kaspi, callback, идемпотентность и fallback через статус. :contentReference[oaicite:6]{index=6} :contentReference[oaicite:7]{index=7}

```md
# Service Scenarios

## 1. Purpose

This document describes implementation-oriented service scenarios for the OnePay ↔ Kaspi integration application.

The purpose of the document is to make the expected behavior explicit for:
- backend developers;
- coding agents;
- QA engineers;
- architects.

---

## 2. Actors

- **Client**
- **Kaspi**
- **OnePay Integration Service**
- **Internal OnePay / Lombard services**

---

## 3. Scenario A — Monthly repayment

## 3.1. Business goal

Kaspi requests monthly repayment information and later reports payment completion.

## 3.2. Main flow

1. Kaspi calls `GET /api/kaspi/contracts` with client `iin`.
2. OnePay validates request and returns list of client contracts available for repayment.
3. Kaspi customer chooses a contract.
4. Kaspi calls `GET /api/kaspi/repayment` with:
   - `contract_number`
   - `iin`
   - `repayment_type = 1`
   - `reference_id`
5. OnePay validates request, retrieves repayment info and returns data needed for display.
6. Customer completes payment in Kaspi application.
7. Kaspi sends `POST /api/kaspi/callback`.
8. OnePay validates signature.
9. OnePay checks idempotency.
10. OnePay stores callback identifiers (`reference_id`, `rnn` if present).
11. OnePay updates payment status.
12. If status is `SUCCESS`, OnePay initiates monthly repayment processing in internal systems.
13. OnePay returns technical acknowledgment.
14. If callback is not received or is suspicious, OnePay schedules fallback `status` check.

## 3.3. Result

- payment state is synchronized;
- repayment is posted in internal systems;
- audit record is stored.

---

## 4. Scenario B — Partial or full repayment

## 4.1. Business goal

Kaspi requests repayment information for partial or full closing of debt.

## 4.2. Main flow

1. Kaspi calls `GET /api/kaspi/contracts`.
2. OnePay returns contract list for the provided `iin`.
3. Customer selects contract in Kaspi.
4. Kaspi calls `GET /api/kaspi/repayment` with:
   - `contract_number`
   - `iin`
   - `repayment_type = 2`
   - `reference_id`
5. OnePay returns current repayment context, including full debt / repayment-related values.
6. If Kaspi requires recalculation before actual payment, it calls `GET /api/kaspi/calculate-repayment`.
7. OnePay performs recalculation and returns monetary breakdown:
   - interest;
   - penalty;
   - repayment amount;
   - total amount;
   - fee if applicable.
8. Customer confirms payment in Kaspi app.
9. Kaspi sends callback.
10. OnePay validates callback and updates state.
11. For `SUCCESS`, OnePay initiates partial or full repayment processing according to business rules.
12. If callback is not received, OnePay uses `status`.

## 4.3. Result

- debt is repaid partially or fully;
- payment status is synchronized with Kaspi;
- repayment structure is preserved in audit trail.

---

## 5. Scenario C — Invoice-based payment initiated on OnePay side

## 5.1. Business goal

Client starts payment from Lombard / OnePay side, then finishes it in Kaspi application.

## 5.2. Main flow

1. Client chooses Kaspi as payment method on Lombard / OnePay side.
2. OnePay creates internal payment entity.
3. OnePay calculates:
   - repayment type;
   - contract number;
   - customer IIN;
   - payment amount;
   - expected fee;
   - total amount.
4. OnePay sends request to Kaspi to create invoice / payment entry.
5. Kaspi returns success and Kaspi-side payment identifier.
6. OnePay stores returned external identifier.
7. OnePay informs customer that invoice is available in Kaspi application.
8. Customer opens Kaspi application and pays.
9. Kaspi sends callback with payment result.
10. OnePay validates signature.
11. OnePay checks callback idempotency.
12. OnePay updates status.
13. On `SUCCESS`, OnePay triggers repayment posting.
14. OnePay notifies client via its own channels if required.

## 5.3. Error branch — invoice creation failure

1. OnePay sends invoice creation request to Kaspi.
2. Kaspi returns error code and description.
3. OnePay stores failed attempt.
4. OnePay shows user business-safe error message.
5. Payment remains not paid.

## 5.4. Error branch — payment failure

1. Customer attempts payment in Kaspi.
2. Kaspi sends failed callback.
3. OnePay stores bank error code and message.
4. OnePay changes payment status to failed.
5. Repayment posting is not performed.

---

## 6. Scenario D — Callback processing

## 6.1. Goal

Provide safe, repeatable and auditable processing of Kaspi server notification.

## 6.2. Processing steps

1. Accept HTTP request.
2. Parse request body / params.
3. Validate required fields.
4. Validate `merchant_id`.
5. Validate `request_time`.
6. Validate `sign`.
7. Find payment by `payment_id`.
8. Check whether same callback business key was already processed.
9. If duplicate:
   - do not repeat business side effects;
   - return idempotent acknowledgment.
10. If not duplicate:
   - persist raw callback payload;
   - save `reference_id` / `rnn`;
   - update payment status;
   - write status history record;
   - if `SUCCESS`, trigger internal repayment processing.
11. Return response in unified format.

## 6.3. Business rules

- callback is the main proof of payment;
- redirect is not proof of payment;
- repeated callback must not repeat repayment;
- callback processing must be transactional.

---

## 7. Scenario E — Fallback status check

## 7.1. Goal

Reconcile payment status when callback is missing or unreliable.

## 7.2. Triggers

Fallback `status` check is required when:
- callback was not received within configured SLA window;
- callback processing failed technically;
- callback contents are inconsistent with local state;
- manual reconciliation is requested.

## 7.3. Flow

1. Background job selects payments requiring verification.
2. Service sends `GET /api/payment/status` to Kaspi.
3. Request contains:
   - `merchant_id`
   - `request_time`
   - `sign`
   - one or more identifiers: `payment_id`, `reference_id`, `rnn`
4. Kaspi returns current payment status.
5. OnePay stores raw response.
6. OnePay compares remote state with local state.
7. If reconciliation is possible automatically, local state is updated.
8. If inconsistency remains, payment is marked for manual investigation.

---

## 8. Scenario F — Invalid signature

## 8.1. Goal

Reject unauthorized or corrupted request.

## 8.2. Flow

1. Request is received.
2. Signature validation fails.
3. Business processing is not started.
4. Security log is written.
5. Error response is returned.

## 8.3. Result

- payment state unchanged;
- internal systems are not called;
- incident is observable.

---

## 9. Scenario G — Stale request time / replay risk

## 9.1. Goal

Reduce risk of replay attacks.

## 9.2. Flow

1. Request is received with `request_time`.
2. Service checks allowed time window.
3. If outside allowed window:
   - request is rejected;
   - security log is written;
   - business processing is not started.

---

## 10. Scenario H — Unknown payment

## 10.1. Goal

Handle callback or payment-related requests for non-existing payment safely.

## 10.2. Flow

1. Request passes technical validation.
2. Payment is not found by `payment_id`.
3. Service returns controlled error response.
4. Event is logged for investigation.

---

## 11. Scenario I — Technical failure during internal posting

## 11.1. Goal

Separate external payment fact from internal posting failure.

## 11.2. Flow

1. `SUCCESS` callback is received.
2. Payment status is confirmed externally.
3. Internal repayment posting fails due to internal error.
4. Service:
   - stores confirmed external payment fact;
   - marks operation for retry / reconciliation;
   - does not lose callback payload.
5. Background recovery process retries posting or escalates for manual handling.

---

## 12. Agent implementation notes

For the coding agent:
- each scenario should be traceable to handler, service and repository methods;
- every branch with state change must be covered by tests;
- duplicate callback test is mandatory;
- missing callback + fallback status scenario test is mandatory;
- invalid signature and stale request tests are mandatory.

---

## ВАЖНО ДЛЯ РЕАЛИЗАЦИИ

Источник истины:
1. callback
2. fallback status

return_url НЕ является подтверждением оплаты.