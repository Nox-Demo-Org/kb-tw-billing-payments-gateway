---
okf_version: '0.2'
title: payments-gateway
description: payments-gateway is the payment integration microservice for Tidewell Mutual.
generated:
  at: '2026-10-05T12:45:47Z'
---

# payments-gateway

`payments-gateway` is the payment integration microservice for Tidewell Mutual. It handles the movement of money across the business by interfacing directly with a Payment Service Provider (PSP) and the Faster Payments network.

### Primary Responsibilities
- **Premium Collections:** Charges stored card tokens or submits direct debit instructions to collect insurance premiums requested by `billing-service`.
- **Claim Payouts:** Dispatches settled claim disbursements via Faster Payments requested by `claims-management`.
- **Event Notifications:** Emits transaction outcome events to Kafka topics for consumption by downstream services like `billing-service`, `claims-management`, and `notifications-hub`.
- **Payment Security & Tokenization:** Interfaces with the PSP using tokenized payment methods, ensuring raw credit card details are never stored or handled directly.

### Architecture & Data Flow
1. **Collections Flow:** `billing-service` calls `POST /v1/collections` providing `plan_id`, `instalment`, `amount_pence`, `method` (`card` or `direct_debit`), and an `idempotency_id`. Cards settle immediately; direct debits settle in 3 working days. Upon successful collection, the gateway emits [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]].
2. **Payouts Flow:** `claims-management` invokes `POST /v1/payouts` with recipient bank account details and `amount_pence` (following secondary approver workflows in claims-management for claims over £25,000). The gateway dispatches the payment via Faster Payments and emits [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]].
3. **Idempotency & Resilience:** All mutation endpoints require an `idempotency_id`. Repeat requests return initial responses without duplicating money transfers. The PSP HTTP client enforces a 10-second timeout with up to three retries reusing the same idempotency key.

### How to Run
`payments-gateway` is written in Go. Entrypoint is located at `cmd/payments/main.go`.

<!-- okf:contents -->

## Contents

- [Concepts and flows](/concepts/index.md) — 3 pages. Flows, lifecycles and cross-cutting mechanisms.
- [Architecture decisions](/decisions/index.md) — 3 pages. One ADR per architecture decision the code or documents make evident.
- [Components and data models](/entities/index.md) — 3 pages. One page per significant component and core data model.
- [Interfaces and references](/summaries/index.md) — 1 page. API, event and module references for the application.
- [Change log](/log.md) — every generation and sync, newest first.

<!-- /okf:contents -->
