---
type: Architecture Decision
title: 'ADR: Asynchronous Event Notifications for Payment Outcomes'
description: Direct debits take up to three working days to settle, whereas card charges settle immediately.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/decisions/async-event-notifications.md
tags:
- payments-gateway
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

# ADR: Asynchronous Event Notifications for Payment Outcomes

## Status
Accepted

## Context
The `payments-gateway` handles money movements for Tidewell Mutual across two primary workflows:
1. Collecting insurance premiums initiated via `POST /v1/collections` by `billing-service` (see [[concepts/premium-collections]] and [[entities/collection-request]]).
2. Dispatching settled claim disbursements via `POST /v1/payouts` by `claims-management` (see [[concepts/claim-payouts]] and [[entities/payout-request]]).

Direct debits take up to three working days to settle, whereas card charges settle immediately. Additionally, multiple downstream domains require notification when money moves:
- `billing-service` needs confirmation of premium collections to update policy instalment schedules.
- `claims-management` requires confirmation when Faster Payments disbursements are dispatched.
- `notifications-hub` requires payout signals to send outbound notifications to policyholders.

Coupling `payments-gateway` directly to downstream services via synchronous HTTP callbacks would create tight runtime coupling, cascade downstream outages into payment execution, and complicate the addition of new observers.

## Decision
We publish domain events to Kafka topics for payment outcome notifications instead of relying on synchronous outbound HTTP calls:

1. **[[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]]** (`TopicCollectionSucceeded` in `internal/events/topics.go`):
   - **Published by:** `internal/collect/collect.go` (`Collect`) after charging a stored card token or submitting a Direct Debit instruction.
   - **Payload schema:** `{plan_id, instalment, amount_pence, collected_at}`.
   - **Consumers:** [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]].

2. **[[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]]** (`TopicPayoutSent` in `internal/events/topics.go`):
   - **Published by:** `internal/payout/payout.go` (`Send`) after dispatching funds via Faster Payments.
   - **Payload schema:** `{claim_id, amount_pence, sent_at}`.
   - **Consumers:** `claims-management`, [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]], and downstream tracking components like [[ap:kb-tw-digital-customer-portal/concepts/claim-tracking-flow#payments-payout-sent|customer-portal (payments.payout.sent)]].

All REST requests to `payments-gateway` continue to require an `idempotency_id` to guarantee that retry attempts do not duplicate payment execution or emit duplicate events (see [[concepts/idempotency]]).

## Consequences

### Positive
- **Decoupled Architecture:** `payments-gateway` does not need direct network access or schema awareness of downstream notification mechanisms, customer communication channels, or claim ledger internals.
- **Extensibility:** Additional consumers (such as audit loggers or customer portals) can subscribe to payment outcome events without requiring code or configuration changes in `payments-gateway`.
- **Fault Isolation:** Downstream processing failures or message delivery delays in `notifications-hub` or `billing-service` do not fail the core payment execution transaction.

### Negative / Trade-offs
- **Eventual Consistency:** Downstream systems update their internal states asynchronously rather than within the synchronous boundary of the incoming REST request.
- **Consumer Idempotency Required:** Downstream event consumers must handle potential duplicate event deliveries gracefully using event identifiers and payload fields like `plan_id`, `instalment`, or `claim_id`.

## Related Documentation
- [[summaries/api-spec]]
- [[concepts/premium-collections]]
- [[concepts/claim-payouts]]
- [[decisions/tokenized-card-storage]]
- [[decisions/high-value-payout-approval]]
