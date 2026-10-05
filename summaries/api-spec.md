---
type: Interface Reference
title: API and Event Specification
description: payments-gateway exposes HTTP REST endpoints for inbound payment and disbursement operations, and publishes asynchronous outcome events to Kafka topics for downstream systems.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/summaries/api-spec.md
tags:
- payments-gateway
- summaries
sources:
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/cmd/payments/main.go
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/internal/collect/collect.go
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/internal/payout/payout.go
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

<!-- anchor: cmd/payments/main.go:L1-L6 -->
<!-- anchor: internal/collect/collect.go:L1-L14 -->
<!-- anchor: internal/payout/payout.go:L1-L15 -->

# API and Event Specification

`payments-gateway` exposes HTTP REST endpoints for inbound payment and disbursement operations, and publishes asynchronous outcome events to Kafka topics for downstream systems.

For the system-level overview, see [[index]].

---

## Contract Summary

| Contract | Kind | Handler / Location | Consumers | Description |
| :--- | :--- | :--- | :--- | :--- |
| `POST /v1/collections` | REST | `internal/collect/collect.go` | `billing-service` | Collects an insurance premium instalment via card or Direct Debit. |
| `POST /v1/payouts` | REST | `internal/payout/payout.go` | `claims-management` | Dispatches a settled claim disbursement via Faster Payments. |
| [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]] | Event | `internal/events/topics.go` | `billing-service` | Emitted when a collection attempt succeeds. |
| [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]] | Event | `internal/events/topics.go` | `claims-management`, `notifications-hub` | Emitted when a claim payout is successfully sent. |

---

## REST Endpoints

Routes are registered in `cmd/payments/main.go`. All mutation endpoints require an `idempotency_id` to prevent duplicate money movement; see [[concepts/idempotency]].

### 1. Collect an Instalment

* **Route:** `POST /v1/collections`
* **Implementation:** `collect.Collect()` in `internal/collect/collect.go`
* **Consumer:** `billing-service`
* **Related Concepts & Entities:** [[concepts/premium-collections]], [[entities/collection-request]], [[decisions/tokenized-card-storage]]

#### Request Schema (`collect.Request`)

| Field | Type | Description |
| :--- | :--- | :--- |
| `plan_id` | `string` | Unique identifier for the billing plan. |
| `instalment` | `int` | Instalment sequence number. |
| `amount_pence` | `int64` | Transaction amount in pence. |
| `method` | `string` | Payment method: `"card"` or `"direct_debit"`. |
| `idempotency_id` | `string` | Unique key to ensure idempotency. |

#### Processing Behavior
- **Card Payments (`"card"`):** Charges the stored card token via the PSP and settles at once.
- **Direct Debit (`"direct_debit"`):** Submits a direct debit instruction via the PSP; settles in 3 working days.
- **Event Emission:** Publishes [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]] upon successful processing.

---

### 2. Pay a Settled Claim

* **Route:** `POST /v1/payouts`
* **Implementation:** `payout.Send()` in `internal/payout/payout.go`
* **Consumer:** `claims-management`
* **Related Concepts & Entities:** [[concepts/claim-payouts]], [[entities/payout-request]], [[decisions/high-value-payout-approval]]

#### Request Schema (`payout.Request`)

| Field | Type | Description |
| :--- | :--- | :--- |
| `claim_id` | `string` | Unique identifier for the settled claim. |
| `amount_pence` | `int64` | Payout disbursement amount in pence. |
| `payee_name` | `string` | Name on the recipient bank account. |
| `sort_code` | `string` | UK bank sort code. |
| `account_number` | `string` | UK bank account number. |
| `idempotency_id` | `string` | Unique key to ensure idempotency. |

#### Processing Behavior
- Dispatches payout funds via Faster Payments.
- Claims over £25,000 must receive secondary approval in `claims-management` before hitting this endpoint (see [[decisions/high-value-payout-approval]]).
- **Event Emission:** Publishes [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]] once the payout is dispatched.

---

## Published Event Topics

Event topic constants are defined in `internal/events/topics.go`. See [[decisions/async-event-notifications]] for architectural rationale.

### 1. [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]]

Defined by constant `events.TopicCollectionSucceeded = "payments.collection.succeeded"`.

* **Trigger:** Published by `collect.Collect()` when a premium collection succeeds.
* **Target Consumers:**
  * [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]]

#### Payload Schema

| Field | Type | Description |
| :--- | :--- | :--- |
| `plan_id` | `string` | Billing plan identifier. |
| `instalment` | `int` | Instalment number collected. |
| `amount_pence` | `int64` | Total amount collected in pence. |
| `collected_at` | `string` / timestamp | Timestamp when the collection succeeded. |

---

### 2. [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]]

Defined by constant `events.TopicPayoutSent = "payments.payout.sent"`.

* **Trigger:** Published by `payout.Send()` when a Faster Payments claim transfer is sent.
* **Target Consumers:**
  * `claims-management`
  * [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]]
  * [[ap:kb-tw-digital-customer-portal/concepts/claim-tracking-flow#payments-payout-sent|customer-portal (payments.payout.sent)]]

#### Payload Schema

| Field | Type | Description |
| :--- | :--- | :--- |
| `claim_id` | `string` | Identifier of the settled claim. |
| `amount_pence` | `int64` | Disbursed amount in pence. |
| `sent_at` | `string` / timestamp | Timestamp when the disbursement was sent. |
