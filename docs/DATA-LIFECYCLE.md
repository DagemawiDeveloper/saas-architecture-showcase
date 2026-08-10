# Data Lifecycle

## Categories

Different data needs different lifecycle rules:

- core transactional records
- user-generated uploads
- generated exports
- logs
- audit events
- analytics aggregates
- temporary processing artifacts

## Retention

Define retention by business/security need rather than keeping everything forever. Temporary exports and processing artifacts are strong candidates for automatic expiry.

## Export pattern

```mermaid
sequenceDiagram
    participant U as User
    participant A as API
    participant Q as Queue
    participant W as Worker
    participant S as Object Storage

    U->>A: Request export
    A->>Q: Queue export job
    A-->>U: 202 export_id
    Q->>W: Process export
    W->>S: Upload generated file
    W-->>A: Mark ready
    U->>A: Get export status
    A-->>U: Short-lived signed URL
```

## Deletion

Deletion workflows should account for database records, object storage, search indexes, cached representations, analytics copies and downstream integrations.
