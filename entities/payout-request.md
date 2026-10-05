---
type: Component
title: Payout Request
description: The payout.Request entity defines the data model and execution contract for claim disbursements.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/entities/payout-request.md
tags:
- payments-gateway
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/internal/payout/payout.go
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/internal/events/topics.go
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

<!-- anchor: internal/payout/payout.go:L1-L15 -->
<!-- anchor: internal/events/topics.go:L1-L6 -->
<!-- anchor: README.md:L1-L14 -->

# Payout Request

The `payout.Request` entity defines the data model and execution contract for claim disbursements. Claim payouts are submitted to `payments-gateway` via `POST /v1/payouts` by `claims-management` and executed over the Faster Payments network.

## Data Model

Defined in `internal/payout/payout.go`, the `Request` struct represents the payload received by `POST /v1/payouts`:

```go
type Request struct {
	ClaimID       string `json:"claim_id"`
	AmountPence   int64  `json:"amount_pence"`
	PayeeName     string `json:"payee_name"`
	SortCode      string `json:"sort_code"`
	AccountNumber string `json:"account_number"`
	IdempotencyID string `json:"idempotency_id"`
}
```

### Field Reference

| Field | Type | JSON Key | Description |
| --- | --- | --- | --- |
| `ClaimID` | `string` | `claim_id` | Identifier of the settled claim originating in `claims-management`. |
| `AmountPence` | `int64` | `amount_pence` | Payout amount expressed in pence. |
| `PayeeName` | `string` | `payee_name` | Name of the bank account holder receiving funds. |
| `SortCode` | `string` | `sort_code` | 6-digit UK bank sort code. |
| `AccountNumber` | `string` | `account_number` | 8-digit UK bank account number. |
| `IdempotencyID` | `string` | `idempotency_id` | Unique key to ensure transactions are executed at most once. |

## Execution Structure

Payout execution is handled by `payout.Send`:

```go
func Send(r Request) error
```

1. **Dual-Approval Verification:** Payout requests exceeding £25,000 must complete a secondary approver workflow within `claims-management` before the request is submitted to this endpoint (see [[decisions/high-value-payout-approval]]).
2. **Disbursement via Faster Payments:** `Send` transfers funds directly to the recipient's bank account using the provided `SortCode` and `AccountNumber`.
3. **Idempotency Guarantee:** The `IdempotencyID` prevents duplicate payments. Repeating a request with the same `idempotency_id` returns the initial response without initiating a second bank transfer (see [[concepts/idempotency]]).
4. **Event Emission:** Upon dispatch, the service publishes the [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]] event (`TopicPayoutSent` defined in `internal/events/topics.go`) with the payload schema `{claim_id, amount_pence, sent_at}` (see [[decisions/async-event-notifications]] and [[concepts/claim-payouts]]).

## Responsibilities

- **Disbursement Processing:** Encapsulate bank routing information and monetary amounts required to execute settled insurance payouts via Faster Payments.
- **Duplicate Prevention:** Enforce at-most-once transfer semantics using `IdempotencyID`.
- **Domain Event Triggering:** Facilitate downstream event emission to notify services when funds have been dispatched.

## Dependencies

- **Inbound Caller:** `claims-management` calls `POST /v1/payouts` to trigger claim settlements.
- **Outbound Network:** Faster Payments scheme for bank transfers.
- **Outbound Events:** Emits [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]] on `TopicPayoutSent`, consumed by:
  - `claims-management` (to update claim settlement status)
  - [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]]
  - [[ap:kb-tw-digital-customer-portal/concepts/claim-tracking-flow#payments-payout-sent|customer-portal (payments.payout.sent)]]
- **Internal Modules:**
  - `internal/payout/payout.go`
  - `internal/events/topics.go`
  - [[summaries/api-spec]]
  - [[concepts/claim-payouts]]
  - [[concepts/idempotency]]
  - [[decisions/high-value-payout-approval]]
