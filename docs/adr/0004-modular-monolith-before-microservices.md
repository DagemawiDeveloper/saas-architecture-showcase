# ADR 0004: Use a modular monolith before introducing microservices

**Status:** Accepted

## Context

The product combines tenant administration, campaign configuration, dynamic forms, geofenced field operations, mobile data collection, OTP workflows, training content, analytics, exports, media handling, notifications and external AI/API integrations.

These capabilities have different responsibilities, but they share important transactions and data relationships. The engineering team is small, requirements are still evolving, and operational simplicity matters more than independent service deployment at this stage.

The architecture must make boundaries visible without creating a distributed system before there is a demonstrated need for one.

## Decision drivers

- Keep deployments, local development, migrations and debugging straightforward.
- Preserve database transactions across closely related workflows.
- Avoid duplicating authentication, authorization, tenant context, observability and deployment infrastructure.
- Allow modules to evolve independently at the code boundary.
- Move slow or failure-prone work out of web requests without requiring separate services.
- Retain a credible path to extract components if scale or ownership later justifies it.

## Considered options

### Option A: Begin with independent microservices

Potential benefits:

- independent deployment and scaling;
- stronger process-level isolation;
- service-specific technology choices.

Costs at the current stage:

- network and message failure modes appear in ordinary feature work;
- transactions become eventual-consistency workflows;
- authentication, tenant context and observability must be propagated across services;
- local development, testing and deployment become materially heavier;
- a small team spends more time operating boundaries than learning from users.

### Option B: One unstructured application

Potential benefit:

- fastest initial implementation.

Cost:

- business logic, controllers, jobs and integrations become tightly coupled;
- ownership and future extraction boundaries remain unclear;
- tests require too much application context;
- changes become risky as the product grows.

### Option C: Modular monolith with asynchronous boundaries

A single Laravel application and relational database are retained, but capabilities are separated by domain-oriented modules, services, policies, jobs and events. Long-running work—exports, notifications, media processing, AI calls and external integrations—runs through queues.

## Decision

Use **Option C**.

The application remains one deployable unit, while domain boundaries are enforced through code organization and explicit interfaces rather than separate network services.

Examples of module boundaries include:

- identity, membership and tenant context;
- campaigns and form definitions;
- field assignments and geofences;
- submissions and validation;
- media and document processing;
- training content;
- analytics and exports;
- notifications and external integrations.

Modules may read shared reference data through deliberate application services. Cross-module writes should occur through a small number of orchestration paths or domain events rather than arbitrary model access.

## Consequences

### Positive

- One deployment and rollback path.
- Straightforward local development and end-to-end testing.
- Relational transactions remain available for workflows that require them.
- Debugging can follow one request/job trace without crossing multiple services.
- Shared security controls and tenant context are easier to apply consistently.
- Queue workers provide independent throughput for slow workloads.
- Future extraction seams are visible in code.

### Negative

- The whole application generally deploys together.
- Poor discipline can allow module boundaries to erode.
- Database load cannot be scaled independently by module without additional work.
- A badly behaved module can still affect the shared process or database.

## Boundary enforcement

- Resolve tenant context before domain logic runs.
- Keep controllers thin and put workflow logic in application/domain services.
- Keep external APIs behind integration adapters.
- Dispatch slow work to named queues.
- Use module-focused tests in addition to end-to-end tests.
- Record cross-module architectural decisions as ADRs.
- Avoid adding a shared helper layer that becomes an unowned dependency sink.

## Signals that justify extraction

A module may become a separate service when multiple signals are present, such as:

- its workload needs independent scaling or isolation;
- it has a stable contract and low transactional coupling;
- its deployment cadence repeatedly conflicts with the rest of the application;
- a separate team owns and operates it;
- regulatory or data-residency requirements demand physical separation;
- its failure domain must not share the main application process;
- observed production data shows that extraction solves a real bottleneck.

Extraction is therefore a response to evidence, not a prerequisite for appearing scalable.
