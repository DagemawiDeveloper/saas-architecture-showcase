# ADR 0001: Use a Modular Monolith Before Microservices

- **Status:** Accepted
- **Date:** 2026-08-24
- **Context:** Multi-tenant survey and field-data platform

## Context

The product combines campaign configuration, dynamic forms, geofence planning, mobile data collection, authentication, training, analytics, notifications, media handling, and external integrations. These domains are distinct, but the engineering team is small and many user workflows cross several domains in one transaction.

Early microservices would create additional deployment units, network failure modes, distributed tracing requirements, duplicated authorization logic, cross-service data consistency problems, and operational overhead before traffic or team boundaries justify those costs.

## Decision

Use a Laravel modular monolith with explicit domain boundaries inside one deployable application.

Each domain owns its application services, policies, jobs, events, validation, and persistence access. Cross-domain communication should use service interfaces or domain events instead of controllers reaching directly into unrelated tables. Slow or failure-prone work leaves the request path through queues, but queue workers initially remain in the same codebase.

## Alternatives Considered

### Independent microservices from day one

Rejected because the organizational and traffic scale did not justify distributed-system complexity. It would make local development, debugging, deployment, and transactional workflows slower without producing a clear customer benefit.

### One unstructured monolith

Rejected because it would reduce initial ceremony but allow feature code to become tightly coupled. The objective is one deployment unit, not one undifferentiated code module.

## Consequences

### Positive

- One repository and deployment path.
- Simple local development and end-to-end debugging.
- Database transactions remain available for workflows that need atomicity.
- Domain boundaries can be improved without network calls.
- Lower infrastructure cost during product validation.

### Negative

- Boundaries require discipline and automated architecture checks.
- A poorly structured module can still affect the whole application.
- Some workloads may eventually require independent scaling.

## Extraction Criteria

A module becomes a service only when at least one of these is demonstrated:

1. It requires materially different scaling or availability characteristics.
2. It has a stable API and clear data ownership.
3. It creates deployment contention for otherwise independent teams.
4. Its failure isolation is worth the additional operational cost.
5. Measurements—not preference—show the monolith is the limiting factor.

## Verification

- Domain-oriented namespaces and service boundaries.
- No direct cross-module table access without a documented exception.
- Queues for exports, notifications, media processing, and external AI/API calls.
- Architecture tests or static checks added as the codebase grows.
