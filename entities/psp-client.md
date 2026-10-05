---
type: Component
title: PSP Client
description: The PSP Client provides the HTTP communication layer between payments-gateway and the external Payment Service Provider (PSP).
resource: https://github.com/Nox-Demo-Org/kb-tw-billing-payments-gateway/blob/main/entities/psp-client.md
tags:
- payments-gateway
- entities
sources:
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/internal/psp/client.go
- resource: https://github.com/Nox-Demo-Org/payments-gateway/blob/HEAD/README.md
generated:
  by: nox/gemini-3.7-flash
  at: '2026-10-05T12:45:47Z'
---

<!-- anchor: internal/psp/client.go:L1-L10 -->
<!-- anchor: README.md:L1-L14 -->

# PSP Client

The `PSP Client` provides the HTTP communication layer between `payments-gateway` and the external Payment Service Provider (PSP). It executes tokenized card charges and Direct Debit instructions while enforcing timeouts and retry safety mechanisms.

## Responsibilities

- **PSP Communication:** Dispatches outbound HTTP requests to the Payment Service Provider API to charge tokenized cards and submit Direct Debit payment instructions (see [[concepts/premium-collections]]).
- **Card Tokenization Security:** Interfaces with the PSP using payment method tokens rather than handling or storing raw credit card details directly (see [[decisions/tokenized-card-storage]]).
- **Timeout Management:** Enforces a strict HTTP client timeout to prevent long-lived hanging network calls.
- **Idempotent Retries:** Reuses the transaction idempotency key across up to three retry attempts, preventing duplicate charges across transient network failures (see [[concepts/idempotency]]).

## Client Configuration

The client constructor is defined in `internal/psp/client.go`:

```go
package psp

import (
	"net/http"
	"time"
)

// Client talks to the payment service provider. Timeout 10s; three retries with the same
// idempotency key so a retry can never charge twice.
func NewClient() *http.Client { return &http.Client{Timeout: 10 * time.Second} }
```

### Key Parameters

| Setting | Value | Description |
| --- | --- | --- |
| **Timeout** | `10 * time.Second` | Configured on `http.Client.Timeout` to terminate slow or stalled connections. |
| **Max Retries** | `3` | Attempts up to three retries upon transient failures. |
| **Idempotency Propagation** | Same idempotency key | Ensures retried HTTP requests reuse the same idempotency key so the PSP will not process a transaction twice. |

## Dependencies

- **Go Standard Library:**
  - `net/http`: Configured via `http.Client` to issue outbound requests.
  - `time`: Configures the 10-second request timeout duration.
- **External Services:**
  - **Payment Service Provider API:** Outbound REST interface for card token charges and Direct Debit instructions.
- **Internal Domain Components:**
  - Used during the execution of [[entities/collection-request]] and [[concepts/premium-collections]].
  - Governed by the system-wide idempotency strategy in [[concepts/idempotency]].
