# ADR 0001: Shared-schema tenancy as the default

**Status:** Accepted

## Context

The platform needs multi-tenant isolation while preserving straightforward migrations, analytics and operational cost for many small/medium tenants.

## Decision

Use a shared relational database/schema with explicit `tenant_id` on tenant-owned records. Tenant context is resolved from authenticated membership and applied centrally.

## Consequences

### Positive
- Simple deployment and migrations
- Efficient resource utilization
- Easier cross-tenant operational analytics

### Negative
- Application-level isolation must be rigorously enforced
- Very large/regulatory tenants may later require stronger physical isolation
