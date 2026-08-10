# ADR 0003: Object storage for uploaded/generated files

**Status:** Accepted

## Decision

Application instances do not persist durable user files on local disk. Uploads, generated reports and media artifacts use object storage with tenant-aware keys and authorized access.

## Why

This keeps application servers stateless, supports horizontal scaling and enables lifecycle/retention policies independently from compute.
