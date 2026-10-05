---
type: Concept
title: Claim Payouts
description: In the payments-gateway service, the claim payout workflow handles disbursements to policyholders for settled insurance claims via the Faster Payments network.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/concepts/claim-payouts.md
tags:
- payments-gateway
- concepts
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

# Claim Payouts

In the [[index|payments-gateway]] service, the claim payout workflow handles disbursements to policyholders for settled insurance claims via the Faster Payments network. Payout requests are initiated upstream by `claims-management` via the REST API and concluded by publishing transaction outcome events to Kafka.

---

## Payout Request Data Structure

Claim payouts are requested through the `POST /v1/payouts` endpoint (see [[summaries/api-spec]]). The payload is mapped directly to the `payout.Request` struct defined in `internal/payout/payout.go`:

| Field | JSON Key | Type | Description |
|---|---|---|---|
| `ClaimID` | `claim_id` | `string` | Unique identifier for the settled insurance claim. |
| `AmountPence` | `amount_pence` | `int64` | Disbursement amount represented in integer pence. |
| `PayeeName` | `payee_name` | `string` | Full name of the recipient bank account holder. |
| `SortCode` | `sort_code` | `string` | Recipient UK bank sort code. |
| `AccountNumber` | `account_number` | `string` | Recipient UK bank account number. |
| `IdempotencyID` | `idempotency_id` | `string` | Unique client-generated key preventing duplicate disbursements. |

For detailed request structure specifications, see [[entities/payout-request]].

---

## Disbursement Lifecycle

```
[claims-management] (Secondary approval if > £25,000)
        │
        ▼ POST /v1/payouts (with idempotency_id)
[payments-gateway (payout.Send)]
        │
        ├──> Dispatches via Faster Payments
        │
        └──> Publishes 'payments.payout.sent'
                 │
                 ├──> [claims-management]
                 ├──> [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub]]
                 └──> [[ap:kb-tw-digital-customer-portal/concepts/claim-tracking-flow#payments-payout-sent|customer-portal]]
```

### 1. High-Value Dual Approval Upstream
Before reaching `payments-gateway`, high-value claims exceeding **£25,000** must undergo secondary approval within `claims-management`. `payments-gateway` assumes any payout request arriving at `POST /v1/payouts` has satisfied all required governance controls. See [[decisions/high-value-payout-approval]] for architectural details.

### 2. Processing & Faster Payments Dispatch
The endpoint invokes `payout.Send(r)` (`internal/payout/payout.go`). The payout is submitted for bank transfer over the Faster Payments rail using the provided `SortCode` and `AccountNumber`.

### 3. Idempotency & Safety
Every disbursement request includes an `idempotency_id`. If a network interruption occurs or `claims-management` retries the request, `payments-gateway` ensures the disbursement is executed exactly once without duplicating money transfers (see [[concepts/idempotency]]).

### 4. Asynchronous Event Notification
Upon successful dispatch, `payments-gateway` emits the [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]] event on the Kafka topic `TopicPayoutSent` (`internal/events/topics.go`):

- **Topic Name:** [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]]
- **Payload Schema:** `{claim_id, amount_pence, sent_at}`
- **Subscribers:**
  - `claims-management` (updates claim payment status)
  - [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]] (alerts the policyholder)
  - [[ap:kb-tw-digital-customer-portal/concepts/claim-tracking-flow#payments-payout-sent|customer-portal (payments.payout.sent)]] (updates customer-facing claim tracker)

For details on the event publishing model, see [[decisions/async-event-notifications]].
