---
type: Architecture Decision
title: 'ADR: Tokenized Card Storage'
description: Handling and persisting raw cardholder data—such as Primary Account Numbers (PANs), CVVs, and expiration dates—introduces significant security risks and brings internal infrastructure into full PCI-DSS compliance scope.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/decisions/tokenized-card-storage.md
tags:
- payments-gateway
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

# ADR: Tokenized Card Storage

## Status
Accepted

## Context
`payments-gateway` is responsible for collecting insurance premiums on behalf of Tidewell Mutual when requested by `billing-service` via `POST /v1/collections` (see [[entities/collection-request]]). These collections support both direct debits and card payments.

Handling and persisting raw cardholder data—such as Primary Account Numbers (PANs), CVVs, and expiration dates—introduces significant security risks and brings internal infrastructure into full PCI-DSS compliance scope. To minimize compliance overhead and prevent potential data exposure, the system requires a payment processing strategy that avoids storing or transmitting raw card data within internal databases.

## Decision
We delegate card capture and tokenization entirely to the Payment Service Provider (PSP). 

- `payments-gateway` and its persistent stores hold no card numbers.
- The PSP securely captures card details and returns a payment method token.
- When processing card collections (`method: "card"` in [[concepts/premium-collections]]), `payments-gateway` interacts with the outbound [[entities/psp-client]] using the stored card token.
- Card transactions settle immediately upon charging the token, at which point the gateway publishes the [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]] event to notify downstream consumers like `billing-service` (see [[decisions/async-event-notifications]] and [[ap:kb-tw-billing-billing-service/summaries/events-spec#payments-collection-succeeded|billing-service (payments.collection.succeeded)]]).

## Consequences

### Positive
- **PCI-DSS Scope Reduction:** Eliminates internal exposure to raw cardholder data, reducing the regulatory and compliance burden on Tidewell Mutual.
- **Enhanced Security:** Even in the event of an internal system breach, no sensitive card numbers or CVVs can be compromised from `payments-gateway` storage.
- **Immediate Settlement:** Tokenized card payments settle at once via the PSP integration.

### Negative & Trade-offs
- **PSP Dependency:** Recurring charges rely entirely on the PSP's token lifecycle, requiring stable communication through the [[entities/psp-client]].
- **Dual Flow Complexity:** The service must maintain separate workflows and settlement lifecycles for tokenized card collections (immediate settlement) versus direct debit collections (settles in 3 working days). All operations must be safeguarded with an `idempotency_id` to prevent duplicate charges (see [[concepts/idempotency]]).
