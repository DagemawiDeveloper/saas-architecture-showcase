# Failure Modes

Architecture quality shows up when dependencies fail.

| Failure | Bad outcome | Preferred response |
|---|---|---|
| External API returns 503 | User request hangs/fails | Queue + bounded retry/backoff |
| Duplicate webhook | Duplicate side effect | Idempotency key + persisted receipt |
| Worker crashes mid-job | Lost work | Durable queue + retry-safe job |
| Mobile client retries submit | Duplicate record | Client/request idempotency key |
| Object upload succeeds but DB transaction fails | Orphaned file | Compensating cleanup / lifecycle rule |
| DB succeeds but notification provider fails | User data rolled back unnecessarily | Commit transaction, notify async |
| Tenant scope omitted | Cross-tenant exposure | Central tenant scoping + isolation tests |
| Secret accidentally logged | Credential exposure | Structured redaction + log policy |
| Large export in HTTP request | Timeouts/memory spikes | Background export job + signed download |
| Deploy changes schema incompatibly | Rolling deploy errors | Expand/migrate/contract deployment pattern |

## External integration circuit thinking

Not every failure should be retried immediately. Distinguish:

- permanent 4xx validation/auth failures
- transient 429/5xx/network failures
- application bugs

Retry only what can plausibly succeed later.
