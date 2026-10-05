# Architecture decisions

One ADR per architecture decision the code or documents make evident.

## Pages

- [ADR: Asynchronous Event Notifications for Payment Outcomes](/decisions/async-event-notifications.md) — Direct debits take up to three working days to settle, whereas card charges settle immediately.
- [High-Value Payout Approval Separation of Concerns](/decisions/high-value-payout-approval.md) — The engineering team needed to determine whether multi-stage approval workflows should be enforced inside payments-gateway or handled upstream by claims-management.
- [ADR: Tokenized Card Storage](/decisions/tokenized-card-storage.md) — Handling and persisting raw cardholder data—such as Primary Account Numbers (PANs), CVVs, and expiration dates—introduces significant security risks and brings internal infrastructure into full PCI-DSS compliance scope.
