# ADR 0003: Make Mobile Survey Submissions Idempotent

- **Status:** Accepted
- **Date:** 2026-08-24
- **Context:** Field collection over unreliable mobile networks

## Context

Field workers may submit data through slow, intermittent, or duplicated network requests. A client can time out after the server commits a submission, then retry because it never received the response. Without idempotency, one interview can become multiple records, distort analytics, duplicate media processing, and consume campaign quotas incorrectly.

A transport retry must be safe, while a genuinely different submission must not be hidden as a duplicate.

## Decision

Each draft/submission receives a client-generated UUID before its first synchronization attempt. The API treats the tuple `(tenant_id, campaign_id, client_submission_id)` as unique.

For a completed request, the server stores:

- client submission ID;
- authenticated collector ID;
- campaign and tenant IDs;
- a canonical payload fingerprint;
- processing status;
- server submission ID;
- timestamps and correlation ID.

Behavior:

1. A new key with valid content creates the submission once.
2. An identical retry returns the original server resource and status.
3. The same key with a different payload returns a conflict; it never mutates the accepted submission silently.
4. Side effects such as quota updates, analytics events, notifications, and media jobs are tied to the persisted submission and are themselves idempotent.
5. The client keeps the local record until it receives a durable server identifier or a terminal validation error.

## Alternatives Considered

### Deduplicate by timestamp, phone number, or payload similarity

Rejected because valid respondents may share those values, and fuzzy matching cannot guarantee correctness.

### Let the server generate the only identifier

Rejected because the client needs a stable identity before the first request completes.

### Accept duplicate rows and clean them later

Rejected because downstream quotas, analytics, notifications, and audit history may already be corrupted.

## Consequences

### Positive

- Safe retries after timeouts and app restarts.
- Clear distinction between replay and conflicting reuse.
- Better support diagnostics through stable correlation identifiers.
- Accurate quotas and downstream processing.

### Negative

- Canonical payload hashing must be deterministic.
- Client IDs must survive local edits and restarts.
- Retention policy is required for idempotency records.

## Verification

- Unique database constraint on the idempotency tuple.
- Concurrent retry tests prove only one submission is created.
- Conflict tests reuse a key with changed content.
- Side-effect handlers store or derive stable idempotency keys.
- Mobile integration tests simulate timeout after commit, reconnect, and retry.
