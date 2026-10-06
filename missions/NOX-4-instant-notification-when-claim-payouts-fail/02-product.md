---
mission: NOX-4
title: 'Instant notification when claim payouts fail'
role: product
status: approved
version: 1
author: dev
ai_drafted: false
approved_at: 2026-10-06T02:43:14Z
---

# Product spec: Instant notification when claim payouts fail

## Goal
Ensure immediate visibility of failed claim disbursements by publishing a dedicated payout failed event whenever Faster Payments validation or execution fails, allowing downstream claim systems, operations worklists, and policyholders to be notified without delay.

## User stories
- As a claims handler, I want immediate notification and reason details whenever a claim payout transfer fails, so that I can contact the policyholder, verify bank details, and re-issue the disbursement without waiting for inbound customer chase calls.
- As a policyholder awaiting an agreed claim settlement, I want my claim tracker and notifications to accurately reflect if a bank transfer could not be completed, so that I know right away to provide updated bank details.
- As a customer service agent, I want up-to-date claim payout statuses and failure reasons visible in claim summaries, so that I can explain payment issues to customers without having to initiate manual payment investigations.
- As a downstream claims service (API consumer), I want to receive an asynchronous `payments.payout.failed` event when a Faster Payments execution or validation fails, so that I can automatically transition claim settlement states and trigger remediation workflows.

## Acceptance criteria
- **AC-1: Failure event publication on bank account validation failure**
  - **Given** a claim payout request is submitted to `POST /v1/payouts` with invalid recipient bank details (e.g., invalid sort code or account number structure),
  - **When** the service validates the request,
  - **Then** the service rejects the transfer and publishes a `payments.payout.failed` event containing the claim ID, amount in pence, failure reason indicating validation error, and failure timestamp.

- **AC-2: Failure event publication on bank network execution rejection**
  - **Given** a claim payout request with valid formatting is submitted to `POST /v1/payouts`,
  - **When** the Faster Payments transfer is rejected by the receiving bank or payment network (e.g., beneficiary account closed or transfer refused),
  - **Then** the service terminates the payout attempt and publishes a `payments.payout.failed` event containing the claim ID, amount in pence, the specific rejection reason code, and the failure timestamp.

- **AC-3: Event payload schema consistency**
  - **Given** any payout validation or execution failure occurs,
  - **When** the `payments.payout.failed` event is emitted,
  - **Then** the event payload includes `claim_id` (string), `amount_pence` (integer pence), `reason` (string describing failure cause), and `failed_at` (timestamp), matching standard payout event payload conventions ([[kb:payments-gateway/summaries/api-spec]]).

- **AC-4: Successful disbursement flow preservation (regression prevention)**
  - **Given** a valid claim payout request is submitted to `POST /v1/payouts` and completes successfully across the Faster Payments network,
  - **When** the disbursement is dispatched,
  - **Then** the service emits `payments.payout.sent` as before ([[kb:payments-gateway/concepts/claim-payouts]]) and does not emit a `payments.payout.failed` event.

- **AC-5: Idempotency safety on repeated payout failures**
  - **Given** a payout request with a specific `idempotency_id` has failed and emitted a `payments.payout.failed` event,
  - **When** the client submits a subsequent request with the exact same `idempotency_id`,
  - **Then** the service returns the cached failure outcome without executing a duplicate bank transaction or emitting duplicate failure events ([[kb:payments-gateway/concepts/idempotency]]).

## Edge cases
- **High-value claim payouts failing after approval:** Claims exceeding £25,000 that have already passed secondary governance approval ([[kb:payments-gateway/decisions/high-value-payout-approval]]) must still publish `payments.payout.failed` if the downstream banking network rejects the transfer *(from the map)*.
- **Idempotent duplicate requests:** Replaying a failed payout request with an identical `idempotency_id` must return the prior failure result without broadcasting duplicate failure events *(from the map)*.
- **Transient network timeouts versus permanent rejections:** Transient network connectivity drops to the payment provider versus terminal rejections (such as closed bank accounts) must be distinguished.
  - *Open question:* Should payment provider retries be completely exhausted before emitting the terminal payout failure event? *(suggestion: wait for all automated gateway retries to complete before emitting `payments.payout.failed`)*.
- **Rejection reason classification:** Bank networks return a variety of error and refusal codes.
  - *Open question:* Which bank rejection and payment error codes should be surfaced directly to downstream customer notifications? *(suggestion: categorize failure reasons into distinct actionable categories such as invalid account details versus transient bank processing errors)*.

## Out of scope
- Implementing changes to premium collection failure handling or Direct Debit retries ([[kb:payments-gateway/concepts/premium-collections]]).
- Updating UI screens and workflow queue rendering inside `claims-management` or `customer-portal` (handled within respective downstream service missions).
- Modifying the upstream £25,000 dual-approval workflow in `claims-management` ([[kb:payments-gateway/decisions/high-value-payout-approval]]).
- Direct Debit 3-day clearing lifecycle modifications.

## Success metric
- **Time to payout failure awareness:** Baseline of 10+ business days (discovered only when a customer calls to chase missing funds) → Target of < 5 minutes automated event delivery and claims record update *(suggestion)*.
- **Event publishing reliability:** 100% of rejected or failed Faster Payments disbursements publish a corresponding `payments.payout.failed` event.
- **Measurement source & timing:** Payout event logs in `payments-gateway` cross-referenced with claims exception worklist entries in `claims-management`, measured 30 days after deployment.

## Priority
**P2 (High):** When claim payouts fail silently, customers awaiting settlement funds are left in limbo, leading to escalated complaints and preventable customer support inquiries. Emitting an automated failure event restores immediate operational visibility and enables proactive customer resolution.

## Verification checklist
- [ ] AC-1: A payout submitted with invalid sort code or account format fails and publishes a `payments.payout.failed` event.
- [ ] AC-2: A payout rejected by the receiving bank or Faster Payments rail publishes a `payments.payout.failed` event with the rejection reason.
- [ ] AC-3: The `payments.payout.failed` event contains `claim_id`, `amount_pence`, `reason`, and `failed_at`.
- [ ] AC-4: Successful disbursements continue to emit `payments.payout.sent` and do not emit failure events.
- [ ] AC-5: Submitting a duplicate request with the same `idempotency_id` returns the cached failure without publishing duplicate events.
- [ ] Edge case: High-value payouts exceeding £25,000 emit `payments.payout.failed` if rejected post-approval.
- [ ] Edge case: Duplicate failure requests respect idempotency bounds.
- [ ] Edge case: Transient network retries resolve before terminal failure emission.
- [ ] Edge case: Bank rejection reasons are correctly mapped and categorized.
- [ ] Success metric: Verify automated failure event latency (< 5 minutes) and 100% publishing reliability 30 days post-launch.
