# ADR 0002: External side effects are asynchronous by default

**Status:** Accepted

## Context

Email, SMS, webhooks, media processing and external APIs have variable latency and availability.

## Decision

Commit core application state first and dispatch external side effects to durable queues unless the business transaction truly requires synchronous confirmation.

## Consequences

- Lower user-facing latency
- Better outage isolation
- Requires idempotent jobs, monitoring and retry/dead-letter operations
