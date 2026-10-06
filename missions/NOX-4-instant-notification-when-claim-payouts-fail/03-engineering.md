---
mission: NOX-4
title: 'Instant notification when claim payouts fail'
role: engineering
status: ai_drafted
version: 1
author: NoX
ai_drafted: true
---

# Engineering design: Instant notification when claim payouts fail

## Applications changing

| Application | Owning Team | Reason for Change |
| :--- | :--- | :--- |
| `payments-gateway` | `tw-billing` | Must update the payout execution pipeline in `internal/payout/payout.go` and define the failure topic in `internal/events/topics.go` to publish `payments.payout.failed` whenever Faster Payments validation or execution fails. |

### Applications not changing
- `claims-management`: Consumes `payments.payout.failed` in a downstream mission to transition claim status and alert claims handlers; no code changes in this mission.
- `billing-service`: Manages policy payment plans and premium collections (`POST /v1/collections`, `payments.collection.succeeded`); completely unrelated to claim payouts.
- `notifications-hub`: Downstream notification consumer; will subscribe to the failure topic in a separate mission.
- `customer-portal`: Downstream UI consumer; unaffected during gateway-level event introduction.

## Approach

`payments-gateway` currently dispatches claim payouts over the Faster Payments rail and only emits `payments.payout.sent` on successful dispatch ([[kb:payments-gateway/concepts/claim-payouts]]). When validation or execution fails, the caller receives an error response, but no asynchronous event is broadcast to Kafka, leaving downstream claim systems unaware of the failure ([[kb:payments-gateway/decisions/async-event-notifications]]).

To solve this:
1. **Event Topic & Model Definition:** Add `TopicPayoutFailed = "payments.payout.failed"` in `internal/events/topics.go` alongside existing topic definitions. Define the payload structure:
   ```go
   type PayoutFailed struct {
       ClaimID     string `json:"claim_id"`
       AmountPence int64  `json:"amount_pence"`
       Reason      string `json:"reason"`
       FailedAt    string `json:"failed_at"`
   }
   ```
2. **Validation & Failure Emission in `payout.Send`:**
   - In `internal/payout/payout.go`, when recipient bank parameters (such as `SortCode` or `AccountNumber`) fail structure or validation rules, construct the failure payload with a descriptive validation error (e.g. `invalid_sort_code`, `invalid_account_number`) and publish to `TopicPayoutFailed`.
   - When the Faster Payments execution via the PSP client fails or the receiving bank rejects the transfer (e.g. `account_closed`, `transfer_rejected`), map the rejection code into `Reason` and publish to `TopicPayoutFailed`.
3. **Transient Retries vs. Terminal Failures:** Transient network or connectivity errors to the PSP must complete standard automated retries within `internal/psp/client.go`. Only terminal execution failures or exhausted retries trigger `payments.payout.failed`.
4. **Idempotency Bounds:** Honour [[kb:payments-gateway/concepts/idempotency]]. Replayed requests with an identical `idempotency_id` that previously resulted in a failure must return the cached error response without re-publishing duplicate `payments.payout.failed` messages.
5. **Approval Boundary Preservation:** In accordance with [[kb:payments-gateway/decisions/high-value-payout-approval]], `payments-gateway` does not re-verify dual-approval for claims exceeding £25,000; if an approved high-value payout reaches the gateway and fails bank execution, it emits `payments.payout.failed` identically to standard payouts.

## Contracts affected

| Contract | Owner App | Consumers | Status |
| :--- | :--- | :--- | :--- |
| `POST /v1/payouts` | `payments-gateway` | `claims-management` | Unchanged: Request payload (`payout.Request`) and synchronous HTTP response contract remain unchanged ([[kb:payments-gateway/summaries/api-spec]]). |
| `payments.payout.sent` | `payments-gateway` | `claims-management`, `notifications-hub`, `customer-portal` | Unchanged: Schema `{claim_id, amount_pence, sent_at}` continues to emit only on successful Faster Payments dispatch ([[kb:payments-gateway/summaries/api-spec]]). |
| `payments.payout.failed` | `payments-gateway` | `claims-management`, `notifications-hub`, `customer-portal` | Additive: New Kafka event emitted when payout validation or bank execution fails. Schema: `{claim_id, amount_pence, reason, failed_at}`. |

## Must not break

- **Successful Payout Execution:** Payouts that pass validation and bank dispatch must continue publishing `payments.payout.sent` and must never emit `payments.payout.failed` ([[kb:payments-gateway/concepts/claim-payouts]]).
- **Idempotency Guarantees:** Duplicate requests with the same `idempotency_id` must return the recorded response and must not execute duplicate bank transfers or emit duplicate Kafka events ([[kb:payments-gateway/concepts/idempotency]]).
- **Premium Collection Flows:** `POST /v1/collections` and `payments.collection.succeeded` event processing in `internal/collect/collect.go` must remain untouched ([[kb:payments-gateway/concepts/premium-collections]]).
- **Separation of Concerns:** Governance dual-approval logic for high-value payouts (> £25,000) remains strictly within `claims-management` and must not be duplicated inside `payments-gateway` ([[kb:payments-gateway/decisions/high-value-payout-approval]]).

## Architecture, guardrails and standards

- **[[kb:payments-gateway/decisions/async-event-notifications|Async Event Notifications ADR]]:** All domain outcome broadcasts use Kafka topics instead of synchronous outbound HTTP calls. Consumers are expected to handle idempotent processing.
- **[[kb:payments-gateway/decisions/high-value-payout-approval|High-Value Payout Approval ADR]]:** `payments-gateway` operates as an execution service and does not enforce business approval limits.
- **[[kb:payments-gateway/concepts/idempotency|Idempotency Invariant]]:** Every payout request requires an `idempotency_id`. Caching covers both successful and failed outcomes.
- **Layering & Code Organization:**
  - Topic names defined in `internal/events/topics.go`.
  - Payout orchestration isolated within `internal/payout/payout.go`.
  - PSP network communication and error codes isolated within `internal/psp/client.go`.
- **Security and PCI/Data Privacy:** Account numbers and sort codes must not be included in event payloads or logs. Payload strictly includes `claim_id`, `amount_pence`, `reason`, and `failed_at`.

## Test strategy

| Level | What it Proves | AC or Contract Covered |
| :--- | :--- | :--- |
| Unit | Validation logic in `internal/payout` correctly identifies invalid sort codes and account numbers and generates `PayoutFailed` struct. | AC-1, AC-3 |
| Unit | Mapping logic accurately converts PSP / Faster Payments rejection codes into standardized reason strings. | AC-2, AC-3 |
| Integration | `POST /v1/payouts` with invalid bank details emits `payments.payout.failed` to Kafka topic and returns rejection. | AC-1, `payments.payout.failed` |
| Integration | `POST /v1/payouts` encountering bank execution refusal emits `payments.payout.failed` and does not emit `payments.payout.sent`. | AC-2, AC-4, `payments.payout.failed` |
| Integration | Successful Faster Payments disbursement publishes `payments.payout.sent` and never `payments.payout.failed`. | AC-4, `payments.payout.sent` |
| Integration | Replaying a failed payout with the same `idempotency_id` returns the cached failure without publishing duplicate events to Kafka. | AC-5, Idempotency |
| Contract | Contract test validating `payments.payout.failed` schema matches `{claim_id, amount_pence, reason, failed_at}`. | AC-3, `payments.payout.failed` |
| Contract | Contract test verifying `POST /v1/payouts` and `payments.payout.sent` remain backward compatible and unbroken. | AC-4, `POST /v1/payouts`, `payments.payout.sent` |
| End-to-End | End-to-end simulation of failed payout from gateway entry through Kafka topic emission. | AC-1, AC-2 |

## Rollout and rollback

- **Rollout Ordering:** `payments-gateway` is deployed first. Because `payments.payout.failed` is an additive Kafka topic, publishing events introduces no breaking changes or dependencies for existing consumers. Downstream consumers (`claims-management`, `notifications-hub`) can subscribe independently.
- **Feature Flags:** No feature flags required for the additive topic definition, but event emission can be guarded by standard gateway configuration if phased rollout is required.
- **Data Migrations & Backfills:** None. No persistent schema changes or historical backfills are required.
- **Rollback Strategy:** If an issue is encountered, rollback `payments-gateway` to the previous container image. Failed payouts will cease emitting Kafka events, reverting system behavior to previous synchronous failure responses without impacting `payments.payout.sent` or collection pipelines.

## Risks

| Risk | Likelihood | Mitigation |
| :--- | :--- | :--- |
| Transient network errors misclassified as terminal failures, causing premature failure event emission. | Low | Ensure automated gateway retries in `internal/psp/client.go` are completely exhausted before treating the failure as terminal and emitting `payments.payout.failed`. |
| Duplicate failure event emission upon client retry. | Medium | Enforce idempotency store lookups on `idempotency_id` before dispatching transfers and publishing failure events ([[kb:payments-gateway/concepts/idempotency]]). |
| Information leakage of recipient bank account details in event reason strings. | Low | Sanitize and map bank rejection codes to categorized enum strings (e.g. `invalid_sort_code`, `account_closed`) without including raw account details or sensitive customer data. |

## Verification checklist

- [ ] Scope matches the design (only `payments-gateway` is modified in this mission).
- [ ] Each contract marked unchanged (`POST /v1/payouts`, `payments.payout.sent`) is untouched.
- [ ] Each additive contract (`payments.payout.failed`) is backward compatible and matches the payload schema `{claim_id, amount_pence, reason, failed_at}`.
- [ ] Architecture, guardrails and standards named ([[kb:payments-gateway/decisions/async-event-notifications]], [[kb:payments-gateway/decisions/high-value-payout-approval]], [[kb:payments-gateway/concepts/idempotency]]) were followed.
- [ ] Test strategy was delivered at every level named (unit, integration, contract, end-to-end).
- [ ] Rollback was proven or is ready.
