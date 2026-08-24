# ADR 0002: Resolve Tenant Context Explicitly and Scope Access Centrally

- **Status:** Accepted
- **Date:** 2026-08-24
- **Context:** Shared-database multi-tenant SaaS

## Context

Most product records belong to an organization: campaigns, forms, groups, routes, submissions, training content, reports, media, and team members. Relying on every controller or developer to remember `where tenant_id = ...` creates an unacceptable risk of cross-tenant data exposure.

Tenant identity may come from an authenticated membership, a verified API token, or an installation-specific host. It must never be accepted from an untrusted request field without authorization.

## Decision

Resolve a tenant once at the start of every authenticated request and job, then make that context explicit to the application layer.

- HTTP middleware resolves the tenant from trusted identity and membership data.
- Authorization confirms that the actor can operate within that tenant.
- Tenant-owned models use centralized query scoping or tenant-aware repositories.
- Create operations assign `tenant_id` from the resolved context, not request input.
- Route-model binding and policies verify tenant ownership.
- Queue payloads carry an immutable tenant identifier and restore context before business logic runs.
- Administrative cross-tenant operations use separate, audited code paths rather than disabling scope casually.

## Alternatives Considered

### Controller-level filters

Rejected because one missed filter can become a data breach, and reviewers must repeatedly verify the same invariant.

### Database per tenant from the beginning

Deferred. It offers strong isolation but increases provisioning, migration, backup, connection-pool, and reporting complexity. It remains an option for regulatory or very large tenants.

### Tenant ID supplied by the client

Rejected as an authority source. It may be used as a routing hint only after matching it to the authenticated actor's allowed memberships.

## Consequences

### Positive

- Tenant isolation becomes a platform invariant rather than a coding convention.
- Policies, logs, jobs, and metrics can include tenant context consistently.
- Security reviews have one resolution path to inspect.

### Negative

- Background jobs and console commands require deliberate context restoration.
- Global administration and aggregate analytics need explicit escape hatches.
- Central scopes can hide query behavior if poorly documented.

## Verification

- Tests prove one tenant cannot read, update, delete, export, or enumerate another tenant's records.
- Factories create multiple tenants in authorization tests.
- Jobs fail closed when tenant context is missing or invalid.
- Audit events include tenant ID, actor ID, action, target, and correlation ID.
- Code review rejects tenant-owned queries that bypass the approved boundary without documented justification.
