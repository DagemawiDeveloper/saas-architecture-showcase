# ADR 0004: Store Survey Media in Object Storage, Not the Relational Database

- **Status:** Accepted
- **Date:** 2026-08-24
- **Context:** Images, audio, documents, and field evidence

## Context

Survey responses may include media captured on mobile devices. Binary files are significantly larger than the structured response data, can upload slowly, need independent retention policies, and may later be processed into thumbnails, transcripts, or archives.

Placing binary content in the primary relational database would increase backup size, replication pressure, query latency, and restore time. Making a bucket public would expose tenant data and make authorization difficult to revoke.

## Decision

Store media bytes in private object storage and store only metadata and relationships in the relational database.

The database record contains:

- tenant, campaign, submission, and field identifiers;
- object key and storage provider;
- original filename and normalized media type;
- byte size and checksum;
- upload and processing status;
- retention classification;
- timestamps and audit information.

Upload behavior:

1. The authenticated API authorizes the tenant, campaign, submission, field, type, and expected size.
2. The server creates an upload intent and a randomized object key that does not expose customer data.
3. The client uploads through a short-lived signed request or streams through the API when policy requires it.
4. Completion is verified by size, checksum, ownership, and allowed media type.
5. Scanning, metadata extraction, thumbnails, transcription, and archival run asynchronously.
6. Downloads use short-lived signed URLs after authorization; the bucket remains private.

## Alternatives Considered

### Store files in database BLOB columns

Rejected for normal media because it couples large binary growth to transactional storage, backups, replication, and restores.

### Store files on the application server filesystem

Rejected because horizontally scaled or ephemeral application instances cannot reliably share or retain local files.

### Public object URLs

Rejected because object names are not authorization controls and access could outlive user permissions.

## Consequences

### Positive

- Independent scaling and lifecycle management for large objects.
- Smaller relational backups and faster database recovery.
- Direct/resumable mobile upload is possible.
- Processing failures do not block core submission transactions.

### Negative

- The system must reconcile orphaned upload intents and missing objects.
- Deletion must coordinate database records, derived files, archives, and object versions.
- Signed URLs and upload intents add operational complexity.

## Verification

- Buckets are private and deny anonymous access.
- Object keys contain no respondent names, phone numbers, or survey answers.
- Upload intents expire and are scoped to one object.
- Checksums detect incomplete or altered uploads.
- Scheduled reconciliation reports orphaned objects and stale intents.
- Tenant-deletion and retention tests prove all associated derivatives are removed or archived according to policy.
