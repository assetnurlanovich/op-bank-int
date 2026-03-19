# AGENTS.md

## Project context
This repository implements OnePay server-side integration with Kaspi for loan repayment flows.

## Architecture rules
- Language: Go
- API style: REST JSON for external APIs, gRPC for internal service boundaries
- Database: PostgreSQL
- Cache: Redis
- Keep code production-oriented, modular, testable
- Do not use mock business logic in final service layer without clear TODO markers

## Security rules
- Never commit real secrets
- Never hardcode credentials
- All callback handlers must validate signatures
- All external operations must be idempotent

## Functional scope
Implement Kaspi-facing services:
- contracts
- repayment
- calculate-repayment
- payment
- callback
- status

## Engineering rules
- Add migrations for every DB schema change
- Add unit tests for domain logic
- Add integration tests for HTTP handlers
- Keep OpenAPI spec in sync with handlers