---
mission: NOX-4
title: 'Instant notification when claim payouts fail'
role: business
status: ai_drafted
version: 1
author: NoX
ai_drafted: true
---

# Business requirement: Instant notification when claim payouts fail

## The request
"Update the disbursement flow to publish a payout failed event whenever Faster Payments validation or execution fails, enabling downstream claim services to handle transfer rejections."

## Problem
When a customer's claim is settled and paid by direct bank transfer, the payment can occasionally fail—for example, if the bank account details were mistyped or if the receiving bank rejected the transfer.

Today, when a transfer fails during validation or processing, that failure is not automatically reported back to our claims operations. The claim record can remain stuck in a pending state or appear as though money is on its way, even though the transfer never went through. Claims handlers often only learn about the issue days or weeks later when an unhappy policyholder calls in to ask where their settlement money is. This delays customer payments, increases avoidable customer service calls, and damages policyholder trust during their moment of need.

## Who is affected
- **Policyholders awaiting claim settlements:** Customers who are expecting an agreed payout into their bank account.
- **Claims handlers and operations teams:** Staff who manage claim payouts and need an accurate, real-time picture of whether money reached the customer.
- **Customer service agents:** Staff answering inbound calls from policyholders checking on the status of overdue payments.

This affects every claim payout where bank account verification fails or where the banking network rejects the transfer.

## What should change
- When a Faster Payments bank transfer fails validation or is rejected by the bank, the failure must be communicated immediately to our claims management system.
- The claim record should immediately reflect that the payout failed, along with a clear reason (such as invalid sort code, closed bank account, or bank transfer rejection).
- Claims handlers should see these failed payouts directly in their daily workflow queues so they can promptly contact the policyholder, verify bank details, and re-issue the payment without waiting for the customer to chase us.
- The policyholder's online claim tracker should clearly reflect that there is an issue with the bank details rather than falsely showing that payment is in transit.

## What "done" looks like
- When a bank transfer cannot be completed, the claim status automatically updates to show the payout has failed.
- Claims operations staff have immediate visibility of all rejected and failed bank transfers on their daily claims worklist.
- Policyholders with failed payouts are contacted proactively by claims staff to correct their bank details, eliminating customer chase calls and payment delays.

## Examples

### Example 1: Typo in bank account details
On 12 November, Sarah Green's home insurance claim of £1,450 is approved. When the payout is processed, the bank transfer fails because of an invalid sort code.
- **Before:** The claim remains marked as pending payment. Ten days later, Sarah calls customer support asking why her money has not arrived. A handler has to manually investigate the payment records to discover the error.
- **After:** The claim record instantly updates to show the payout failed due to invalid bank details. Sarah's claims handler receives a task notice within minutes, calls Sarah to confirm her correct sort code, and re-issues the payout the same day.

### Example 2: Receiving bank rejects the transfer
On 18 November, David Patel's vehicle repair claim of £820 is sent via Faster Payments, but his bank rejects the transfer because the account has recently been closed.
- **Before:** The claims screen shows the payout as sent, while David waits for money that never arrives.
- **After:** The claim status immediately updates to show the transfer was rejected by the bank. The claims team is alerted right away, and David receives an update prompting him to provide an active bank account for his payout.

## Verification checklist
- [ ] In the claims management system, when a claim payout is submitted with invalid bank account details, the claim record immediately updates to show the payout has failed.
- [ ] When a bank transfer is rejected during payment processing, the failed transfer appears on the claims team's exception worklist.
- [ ] Claims handlers can view the specific failure reason on the claim summary so they can explain the issue clearly to the policyholder.
- [ ] Customer support staff can see up-to-date payment failure statuses without needing to ask the payments team to check manual bank records.
- [ ] Is it now true that we "Update the disbursement flow to publish a payout failed event whenever Faster Payments validation or execution fails, enabling downstream claim services to handle transfer rejections"?
