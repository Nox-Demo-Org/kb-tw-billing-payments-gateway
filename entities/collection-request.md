---
type: Component
title: Collection Request
description: The collect.Request struct represents the payload accepted by the POST /v1/collections endpoint in payments-gateway.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/entities/collection-request.md
tags:
- payments-gateway
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/internal/collect/collect.go
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/internal/events/topics.go
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

<!-- anchor: internal/collect/collect.go:L1-L14 -->
<!-- anchor: internal/events/topics.go:L1-L6 -->
<!-- anchor: README.md:L1-L14 -->

# Collection Request

The `collect.Request` struct represents the payload accepted by the `POST /v1/collections` endpoint in `payments-gateway`. It encapsulates all parameters required to charge an insurance premium instalment against an existing payment method or schedule.

## Data Model

Defined in `internal/collect/collect.go`, `collect.Request` defines the following schema:

| Field | Go Type | JSON Key | Description |
| --- | --- | --- | --- |
| `PlanID` | `string` | `plan_id` | Identifier for the billing plan associated with the insurance policy. |
| `Instalment` | `int` | `instalment` | Sequence number of the instalment being collected. |
| `AmountPence` | `int64` | `amount_pence` | Amount to collect in pence (e.g., GBP). |
| `Method` | `string` | `method` | Payment rail/instrument: `"card"` or `"direct_debit"`. |
| `IdempotencyID` | `string` | `idempotency_id` | Unique key to guarantee end-to-end payment [[concepts/idempotency|idempotency]]. |

```go
type Request struct {
	PlanID        string `json:"plan_id"`
	Instalment    int    `json:"instalment"`
	AmountPence   int64  `json:"amount_pence"`
	Method        string `json:"method"` // card | direct_debit
	IdempotencyID string `json:"idempotency_id"`
}
```

## Execution Flow

The collection process is executed via `collect.Collect(r Request)`:

1. **Idempotency Verification:** The incoming `IdempotencyID` ensures identical collection requests return their initial result without duplicating transactions (see [[concepts/idempotency]]).
2. **Payment Execution:**
   - **`card`:** Charges the customer's stored card token via the PSP client (see [[decisions/tokenized-card-storage]]). Card collections settle immediately.
   - **`direct_debit`:** Submits a Direct Debit collection instruction. Direct debits settle in 3 working days.
3. **Event Notification:** Upon successful execution, the gateway publishes the [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]] event to notify downstream consumers (see [[decisions/async-event-notifications]] and [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]]).

## Responsibilities

- Represent and validate the payload for `POST /v1/collections`.
- Provide the execution context for dispatching payment instructions to the PSP.
- Differentiate handling between instant card settlements and 3-day Direct Debit clearing lifecycles (see [[concepts/premium-collections]]).
- Enable downstream tracking by capturing `PlanID`, `Instalment`, and `AmountPence`.

## Dependencies

- **Upstream Callers:** Invoked via REST `POST /v1/collections` by `billing-service`.
- **PSP Integration:** Interacts with the Payment Service Provider API via [[entities/psp-client]] to execute tokenized card charges and Direct Debit instructions.
- **Event Bus:** Emits event `TopicCollectionSucceeded` ([[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]]) containing `{plan_id, instalment, amount_pence, collected_at}` defined in `internal/events/topics.go`.
