# Concepts and flows

Flows, lifecycles and cross-cutting mechanisms.

## Pages

- [Claim Payouts](/concepts/claim-payouts.md) — In the payments-gateway service, the claim payout workflow handles disbursements to policyholders for settled insurance claims via the Faster Payments network.
- [Idempotency and Double-Charge Prevention](/concepts/idempotency.md) — In payments-gateway, all state-mutating financial operations enforce strict idempotency end-to-end to prevent duplicate charges or double payouts during network partitions, client retries, or downstream timeout scenarios.
- [Premium Collections](/concepts/premium-collections.md) — payments-gateway handles incoming premium payments requested by billing-service for insurance policies across Tidewell Mutual.
