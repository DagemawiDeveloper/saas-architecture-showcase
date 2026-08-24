# ADR 0005: Move Slow and Failure-Prone Work Outside the Request Path

- **Status:** Accepted
- **Date:** 2026-08-24
- **Context:** Exports, notifications, media, analytics, and external AI/API calls

## Context

Several platform operations depend on large datasets or external systems whose latency and availability the application does not control. Running them synchronously makes user requests slow, increases timeout risk, and couples platform availability to partners.

However, sending every operation to a queue would make simple workflows harder to reason about and introduce eventual consistency without benefit.

## Decision

Keep authorization, validation, core state transitions, and the minimal durable record in the synchronous request. Dispatch asynchronous work only after the required database transaction commits.

Queue candidates include:

- report and data exports;
- bulk notifications and email/SMS delivery;
- media scanning, conversion, thumbnails, and transcription;
- AI suggestions and analytics enrichment;
- external CRM, payment, or partner synchronization;
- archive creation and retention workflows;
- expensive aggregate recalculation.

Every queued operation must define:

- a stable job identity or idempotency key;
- tenant and actor context;
- bounded attempts and backoff;
- per-attempt timeout;
- retryable versus terminal failures;
- observable status and correlation ID;
- a dead-letter or operator-recovery path;
- a safe response when the original resource has changed or been deleted.

## Alternatives Considered

### Perform everything synchronously

Rejected because a slow partner or large export would consume request workers and create user-visible failures.

### Queue every write

Rejected because it adds unnecessary eventual consistency to simple operations and makes validation feedback less predictable.

### Retry indefinitely

Rejected because permanent failures would create unbounded cost, noise, and duplicate side effects.

## Consequences

### Positive

- Predictable request latency.
- External outages do not immediately become platform outages.
- Workers can scale independently from web traffic.
- Failed work becomes visible and recoverable.

### Negative

- Users need status indicators for long-running work.
- Jobs can run more than once and therefore must be idempotent.
- Deployments must preserve compatibility with queued payloads already in flight.
- Operational dashboards and alerting become required product infrastructure.

## Verification

- Jobs are dispatched after commit when they depend on newly written data.
- Tests execute each side-effecting job at least twice and verify one logical outcome.
- Queue dashboards expose age, throughput, retries, failure rate, and dead-letter count by job type.
- Alerts distinguish partner outages from application defects.
- Runbooks document retry, cancellation, replay, and data-reconciliation procedures.
