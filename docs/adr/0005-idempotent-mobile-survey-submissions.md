# ADR 0005: Make mobile survey submissions idempotent

**Status:** Accepted

## Context

Field-data collection happens on mobile devices that may experience slow, intermittent or changing network connectivity. A user can submit once, see a timeout, and retry even though the first request reached the server. The mobile operating system or application may also retry queued work after a restart.

Creating a second response for the same completed survey can corrupt counts, analytics, quotas, payment calculations and operational follow-up. At the same time, the API must still allow a legitimate second response when the campaign rules permit it.

HTTP success alone is therefore not a sufficient identity model for a submission.

## Decision drivers

- A retry of the same logical submission must not create a duplicate response.
- Different legitimate submissions must remain distinguishable.
- The server must remain the authority on acceptance, campaign rules and final identifiers.
- The design must work across application restarts and offline queues.
- Operators need enough traceability to diagnose repeated or conflicting requests.
- Media upload retries must not duplicate attachments or leave an accepted response in an ambiguous state.

## Considered options

### Option A: Trust one tap in the user interface

Disable the submit button after the first tap.

This improves the interface but does not handle timeouts, process restarts, background retries, duplicate HTTP delivery or malicious clients.

### Option B: Deduplicate by respondent, form and time window

Reject submissions that look similar within a short period.

This can suppress legitimate repeated responses and still misses duplicates outside the guessed time window. Similar content is not reliable identity.

### Option C: Client-generated idempotency key with server-side request fingerprint

The mobile app creates a stable UUID when a draft becomes a submission attempt. That key is persisted with the local queued item and reused for every retry of the same logical submission.

The server stores the key inside the tenant/campaign identity boundary together with a canonical request fingerprint and the accepted response ID.

## Decision

Use **Option C**.

### Client behavior

1. Keep a local draft identifier while the user edits.
2. Generate a new submission idempotency UUID when the user commits the draft for sending.
3. Persist the UUID, payload version, campaign/form identifiers and attachment references in the local queue.
4. Reuse the same UUID for network retries, background retries and application restarts.
5. Generate a different UUID only for a genuinely new logical submission.
6. Do not treat a timeout as proof that the server rejected the request.

### API behavior

The request includes an idempotency key, authenticated actor/device context and the payload.

The server:

1. resolves tenant and campaign context;
2. validates the key format and payload limits;
3. canonicalizes the identity-relevant payload;
4. computes a fingerprint;
5. atomically creates or loads the idempotency record;
6. returns the original accepted response when the key and fingerprint match;
7. returns a conflict when the same key is reused for different content;
8. performs campaign, authorization, geofence and validation checks server-side;
9. commits the response and idempotency result in the same transaction where practical.

## Identity scope

The unique constraint should include the smallest boundary in which a key must be unique, for example:

```text
tenant_id + campaign_id + idempotency_key
```

The authenticated user or device may also be included when campaign semantics require it. A raw key should not be globally trusted without tenant context.

## Canonical fingerprint

The fingerprint covers fields that define the logical submission, such as:

- campaign and form version;
- normalized answers;
- relevant respondent identity fields;
- captured coordinates and timestamps when semantically required;
- stable attachment identifiers.

Associative key ordering, transport formatting and other non-semantic JSON differences must not change the fingerprint.

Sensitive raw payloads do not need to be duplicated in the idempotency table. A bounded status record, hash, response ID and timestamps are normally sufficient for reconciliation.

## Response behavior

| Situation | Result |
|---|---|
| First valid request | Create response and return accepted response ID |
| Identical retry | Return the original response ID without creating a second response |
| Same key, changed logical content | Return an explicit idempotency conflict |
| Validation or authorization failure | Return the failure; do not record a successful result |
| Processing still in progress | Return a retryable/in-progress state rather than starting a second workflow |

The exact HTTP status may depend on API conventions, but the semantic states must remain distinct.

## Media handling

Large media should use stable attachment/upload identifiers rather than embedding repeated binaries in every submission retry.

A practical flow is:

1. create or resume an upload slot with its own idempotent identity;
2. upload the object directly or in resumable parts;
3. verify size, type and checksum;
4. reference the verified attachment ID in the response payload;
5. finalize the survey response only when required attachments are in an acceptable state.

Cleanup jobs remove abandoned uploads after a retention window. Reconciliation jobs can detect accepted responses whose attachment processing later failed.

## Consequences

### Positive

- Network retries do not inflate response counts.
- The mobile queue can retry safely after restart.
- Conflicting client behavior becomes explicit instead of silently overwriting data.
- Operators can trace one logical submission across attempts.
- Analytics and campaign quotas receive stable response identity.

### Negative

- The client must persist submission identity correctly.
- The server needs an idempotency table, unique constraint and retention policy.
- Canonicalization rules must evolve carefully with form versions.
- Multi-step media workflows require separate identities and reconciliation.
- In-progress and failed states add API complexity.

## Failure and recovery considerations

- Use a database uniqueness constraint rather than a read-then-create check alone.
- Keep response creation and successful idempotency completion transactionally aligned.
- Allow stale `processing` claims to be recovered through a bounded lease or reconciliation job.
- Record bounded error/status metadata without logging sensitive survey answers.
- Retain idempotency records for at least the maximum retry/offline period plus an operational safety margin.
- Test concurrent identical requests and same-key/different-payload conflicts.

This design provides retry safety; it does not replace campaign rules such as one response per respondent, device limits, OTP verification or fraud controls. Those are separate policy decisions.
