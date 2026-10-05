---
type: Architecture Decision
title: High-Value Payout Approval Separation of Concerns
description: The engineering team needed to determine whether multi-stage approval workflows should be enforced inside payments-gateway or handled upstream by claims-management.
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/decisions/high-value-payout-approval.md
tags:
- payments-gateway
- decisions
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

# High-Value Payout Approval Separation of Concerns

## Status
Accepted

## Context
When an insurance claim is settled, `payments-gateway` executes disbursement to the customer's bank account via Faster Payments through the `POST /v1/payouts` endpoint (see [[concepts/claim-payouts]] and [[entities/payout-request]]). Disbursing high-value claim settlements (exceeding £25,000) carries heightened financial risk and regulatory governance requirements, mandating a secondary approver before funds leave Tidewell Mutual accounts.

The engineering team needed to determine whether multi-stage approval workflows should be enforced inside `payments-gateway` or handled upstream by `claims-management`.

## Decision
We decided to enforce dual-approval workflows upstream in `claims-management` rather than within `payments-gateway`.

Specifically:
- `claims-management` is responsible for claim validation, policy limit checks, and holding payouts over £25,000 until a second authorized approver signs off.
- `payments-gateway` acts as a pure money-movement execution service. When `POST /v1/payouts` receives a [[entities/payout-request|payout request]] containing `ClaimID`, `AmountPence`, `PayeeName`, `SortCode`, `AccountNumber`, and `IdempotencyID`, it assumes all business-level approval policies have been satisfied upstream.
- `payout.Send` (`internal/payout/payout.go`) executes the transfer directly via Faster Payments and emits the [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]] event for consumption by `claims-management` and [[ap:kb-tw-customer-platform-notifications-hub/summaries/api-spec#payments-payout-sent|notifications-hub (payments.payout.sent)]].

## Consequences
### Positive
- **Clear Domain Boundaries:** Business authorization, claims lifecycle logic, and claim-level audit trails remain isolated within `claims-management`.
- **Service Simplicity:** `payments-gateway` remains stateless with respect to approval lifecycles, focusing strictly on payment rails, PSP communication, timeout/retry handling, and [[concepts/idempotency|idempotency guarantees]].
- **Uniform Execution Interface:** The interface contract for `POST /v1/payouts` remains identical regardless of payout amount, avoiding complex async approval states or multi-stage webhook handshakes in `payments-gateway`.

### Negative & Trade-offs
- **Trust Assumption:** `payments-gateway` does not independently block or verify approval signatures on payouts exceeding £25,000; it trusts that any caller authorized to access `POST /v1/payouts` has completed the required approval checks.
- **Upstream Enforcement Dependency:** Any defect or misconfiguration in `claims-management` approval flows could result in an unapproved high-value payout being immediately dispatched once sent to the gateway.
