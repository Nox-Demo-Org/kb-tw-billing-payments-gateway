---
type: Concept
title: Idempotency and Double-Charge Prevention
description: In payments-gateway, all state-mutating financial operations enforce strict idempotency end-to-end to prevent duplicate charges or double payouts during network partitions, client retries, or downstream timeout scenarios.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/concepts/idempotency.md
tags:
- payments-gateway
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

# Idempotency and Double-Charge Prevention

In `payments-gateway`, all state-mutating financial operations enforce strict idempotency end-to-end to prevent duplicate charges or double payouts during network partitions, client retries, or downstream timeout scenarios.

## Inbound Request Idempotency

Both primary REST entrypoints require callers to supply a unique `idempotency_id` in the JSON request payload:

- **Premium Collections:** `POST /v1/collections` accepts a `[[entities/collection-request|collect.Request]]` containing `idempotency_id` (alongside `plan_id`, `instalment`, `amount_pence`, and `method`). See [[concepts/premium-collections]].
- **Claim Disbursements:** `POST /v1/payouts` accepts a `[[entities/payout-request|payout.Request]]` containing `idempotency_id` (alongside `claim_id`, `amount_pence`, `payee_name`, `sort_code`, and `account_number`). See [[concepts/claim-payouts]].

When an endpoint receives a repeat request matching a previously processed `idempotency_id`, `payments-gateway` returns the initial transaction result directly without executing a second money movement.

```
Caller (billing-service / claims-management)
         │
         │ POST /v1/... (idempotency_id: "xyz")
         ▼
┌────────────────────────────────────────┐
│           payments-gateway             │
│                                        │
│  Check idempotency_id:                 │
│  - If already processed: Return result │
│  - If new: Execute payment             │
└──────────────────┬─────────────────────┘
                   │
                   │ HTTP Request with Idempotency Key (up to 3 retries)
                   ▼
       [ Payment Service Provider ]
```

## Outbound PSP Idempotency & Retries

When interacting with the external Payment Service Provider (PSP) via `[[entities/psp-client|psp.NewClient()]]`:

1. **Timeout Configuration:** The HTTP client enforces a strict **10-second timeout** (`Timeout: 10 * time.Second`).
2. **Retry Mechanism:** In the event of network failure or transient errors, the client performs up to **three retries**.
3. **Key Propagation:** All retries reuse the exact same idempotency key when communicating with the PSP API, guaranteeing that network-level retries can never produce duplicate charges or debits at the provider.

## Event Publishing

Upon successful completion of the underlying operation:
- Collections trigger a [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]] event to notify consumers such as [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]].
- Payouts trigger a [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]] event to notify consumers such as [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]].

Idempotency guarantees that duplicate caller requests do not generate duplicate event broadcasts across the system. See [[decisions/async-event-notifications]].
