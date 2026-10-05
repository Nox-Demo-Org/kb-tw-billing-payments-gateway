---
type: Concept
title: Premium Collections
description: payments-gateway handles incoming premium payments requested by billing-service for insurance policies across Tidewell Mutual.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/concepts/premium-collections.md
tags:
- payments-gateway
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

# Premium Collections

`payments-gateway` handles incoming premium payments requested by `billing-service` for insurance policies across Tidewell Mutual. Premium collections support two payment methods: tokenized card charges and Direct Debit instructions.

## Collection Workflow

Collections are initiated by `billing-service` calling `POST /v1/collections` (handled by `internal/collect/collect.go`).

```
[ billing-service ]
        │
        │ POST /v1/collections { plan_id, instalment, amount_pence, method, idempotency_id }
        ▼
[ payments-gateway (Collect) ]
        │
        ├──► (card) ────────► PSP (Tokenized charge) ──► Settles immediately
        │
        └──► (direct_debit) ─► PSP (Debit instruction) ─► Settles in 3 working days
        │
        ▼
[ Emit Event ] ──► Topic: payments.collection.succeeded
```

### Request Structure
A collection request receives the `collect.Request` payload (defined in `internal/collect/collect.go`):

| Field | Type | Description |
| --- | --- | --- |
| `plan_id` | `string` | Unique identifier of the billing payment plan |
| `instalment` | `int` | Sequence number of the instalment being collected |
| `amount_pence` | `int64` | Collection amount in pence |
| `method` | `string` | Payment method: `"card"` or `"direct_debit"` |
| `idempotency_id` | `string` | Unique client-supplied idempotency key |

For detailed entity specifications, see [[entities/collection-request]].

---

## Settlement Lifecycle by Method

1. **Card (`card`)**:
   - Executes a charge against a stored card token held by the Payment Service Provider (PSP).
   - Raw card data is never stored locally; the gateway references PSP tokens (see [[decisions/tokenized-card-storage]]).
   - **Settlement:** Settles immediately ("at once").

2. **Direct Debit (`direct_debit`)**:
   - Submits a Direct Debit instruction to the PSP.
   - **Settlement:** Settles in **3 working days**.

---

## Event Notifications & Success Handling

Upon successful collection, `Collect` publishes an event to Kafka:
- **Topic:** [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]] (`events.TopicCollectionSucceeded` in `internal/events/topics.go`)
- **Payload:** `{plan_id, instalment, amount_pence, collected_at}`
- **Consumers:** Consumed asynchronously by [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]] to update policy instalment schedules.

For architecture context regarding event-driven notifications, see [[decisions/async-event-notifications]].

---

## Failure Handling and Retries

- **Idempotency Protection:** Every request requires an `idempotency_id`. Repeat requests using the same ID return the original result without duplicating charges (see [[concepts/idempotency]]).
- **PSP Retries:** Outbound PSP client requests enforce timeouts and retry logic (see [[entities/psp-client]]).
- **Arrears Workflow:** If a collection attempt fails:
  - `payments-gateway` retries the collection after **3 working days**.
  - If a second consecutive failure occurs, downstream processes trigger [[ap:kb-tw-billing-billing-service/summaries/events-spec#billing-payment-missed|billing-service (billing.payment.missed)]] to initiate customer notifications and arrears management.
