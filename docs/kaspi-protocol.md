# Kaspi Protocol Summary for Implementation

## 1. Purpose

This document is an implementation-oriented summary of the agreed interaction protocol between OnePay and Kaspi.

It is intended to be used by engineers and coding agents as the primary functional contract for the OnePay ↔ Kaspi integration service.


## 2. Participants

- **Client** — borrower / customer of the Lombard.
- **Lombard** — source system of contracts, debt and repayment facts.
- **OnePay** — integration layer and payment orchestration side.
- **Kaspi** — banking system / client application.


## 3. Supported business scenarios

The protocol covers the following scenarios:

1. **Monthly repayment**
   - full customer journey is performed inside Kaspi application.

2. **Partial or full repayment**
   - full customer journey is performed inside Kaspi application.

3. **Invoice-based payment**
   - customer starts payment on Lombard / OnePay side, then pays in Kaspi.


## 4. General protocol rules

- Transport: **HTTPS**
- Payload format: **JSON**
- OnePay response time to external Kaspi request: **up to 15 seconds**
- Expected initial load profile: **10+ payments per minute**, OnePay should support **10–15 concurrent connections**
- Multi-merchant support is mandatory via `merchant_id`
- All requests contain `request_time`
- All requests contain `sign`
- Signature is based on a service-specific field + `request_time` + shared `secret`


## 5. Main identifiers and fields

### Core identifiers
- `merchant_id` — merchant / Lombard identifier in OnePay
- `payment_id` — payment identifier in OnePay
- `reference_id` — Kaspi operation identifier
- `rnn` — Kaspi reference number
- `contract_number` — contract / pawn ticket number
- `iin` — client IIN
- `repayment_type` — repayment type:
  - `1` = monthly repayment
  - `2` = partial or full repayment

### Money fields
- `monthly_due_amount` — monthly due amount
- `full_due_amount` — full due amount
- `payment_amount` — repayment amount without fee
- `fee_amount` — fee amount
- `total_amount` — total charged amount
- `interest_amount` — accrued interest amount
- `penalty_amount` — accrued penalty amount
- `repayment_amount` — amount directed to loan repayment after fee logic

### Technical fields
- `request_time` — request timestamp in `YYYYMMDDHHMMSS`
- `return_url` — redirect URL after invoice-based payment flow


## 6. Payment statuses

Kaspi sends / returns payment status as integer:

- `1` — SUCCESS
- `2` — PENDING
- `5` — CANCELLED
- `6` — FAILED

These statuses must be mapped to the internal payment status model of the integration service.


## 7. Unified OnePay response format

All OnePay responses to Kaspi use the same JSON envelope:

```json
{
  "success": true,
  "data": {},
  "error_code": 0,
  "message": "OK"
}
```

Rules:
- success — operation result flag
- data — payload object
- error_code — 0 on success
- message — processing result description


## 8. Signature rules

Current protocol rule: sign = SHA512(field1 + request_time + secret)

Where field1 depends on the service:
- contracts → iin
- repayment → contract_number
- calculate-repayment → payment_id
- payment → payment_id
- callback → payment_id
- status → payment_id

Implementation notes:
- use exact concatenation without separators unless protocol is уточнен otherwise;
- signature validation must be deterministic;
- signature failures must be logged and rejected;
- replay window must be restricted by timestamp validation.

Note: scheme diagrams also mention a possible future joint decision to use JWT-based signature format. The current implementation baseline remains SHA512 as defined above.


## 9. OnePay APIs called by Kaspi

### 9.1. GET /api/kaspi/contracts

Purpose:
get client contract list by IIN.

Required query params:
- merchant_id
- iin
- request_time
- sign

Signature:
sha512(iin + request_time + secret)

### 9.2. GET /api/kaspi/repayment

Purpose:
get repayment information for selected contract for receipt display or amount entry form.

Required query params:
- merchant_id
- contract_number
- iin
- repayment_type
- reference_id
- request_time
- sign

Signature:
sha512(contract_number + request_time + secret)

### 9.3. GET /api/kaspi/calculate-repayment

Purpose:
calculate partial/full repayment amount before payment creation.

Expected baseline:
- request is linked to a concrete payment_id
- signature is based on payment_id

Signature:
sha512(payment_id + request_time + secret)

### 9.4. GET /api/kaspi/payment

Purpose:
return payment information / invoice-related payment parameters for a previously prepared payment scenario.

Signature:
sha512(payment_id + request_time + secret)

### 9.5. POST /api/kaspi/callback

Purpose:
server-to-server notification from Kaspi about payment result.

Important fields:
- merchant_id
- payment_id
- reference_id
- rnn
- contract_number (optional)
- iin (optional)
- repayment_type (optional)
- payment_amount
- fee_amount
- total_amount
- status
- error_code (optional)
- message (optional)
- request_time
- sign

Signature:
sha512(payment_id + request_time + secret)

Mandatory processing rules:
- validate signature;
- locate payment;
- check idempotency;
- save reference_id and rnn;
- update payment status;
- on SUCCESS initiate repayment processing.


## 10. Kaspi API called by OnePay

### 10.1. GET /api/payment/status

Purpose:
fallback status retrieval if:
- callback was not received;
- callback was received with error;
- disputed operation needs reconciliation.

Request params:
- merchant_id — required
- payment_id — optional
- rnn — optional
- reference_id — optional
- request_time — required
- sign — required

Rule:
- at least one of reference_id, rnn, payment_id must be provided
- preferred: reference_id + rnn

Signature:
sha512(payment_id + request_time + secret)

Known response status values:
- 1 SUCCESS
- 2 PENDING
- 5 CANCELLED
- 6 FAILED


## 11. Invoice-based payment flow specifics

High-level flow:
- customer selects Kaspi payment option on Lombard / OnePay side;
- OnePay creates payment and sends invoice creation request to Kaspi;
- Kaspi returns success and its payment id;
- OnePay informs customer that invoice is available in Kaspi app;
- customer pays in Kaspi app;
- Kaspi sends callback;
- OnePay validates signature, checks idempotency, updates status and triggers repayment processing.

Important rules:
- redirect / return URL is not proof of payment;
- callback is the primary confirmation source;
- status API is a fallback confirmation source.


## 12. Error handling rules

Invoice creation error
- Kaspi returns error code and description
- OnePay shows business-friendly message to the customer
- payment remains non-successful and should not be treated as paid

Payment error
- Kaspi sends failed callback with error details
- OnePay moves payment to failed state

Repeated callback
- repeated callback must not lead to duplicate repayment
- repeated callback should return technical success or idempotent acknowledgment


## 13. Agent implementation rules

For the coding agent:
- all APIs must return the unified envelope;
- all status transitions must be auditable;
- callback handling must be transactional and idempotent;
- fallback polling via status must be implemented as background job / recovery mechanism;
- signature validation logic must be centralized, not duplicated across handlers;
- external contract values should be stored as-is and mapped to internal domain models.

