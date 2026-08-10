# Observability

A production SaaS platform should make failures diagnosable without reproducing them manually.

## Correlation

Assign a request/correlation ID at the edge or application boundary. Propagate it to:

- application logs
- queue jobs
- outgoing HTTP requests
- audit events
- error responses

## Structured log fields

```json
{
  "level": "error",
  "request_id": "req_01J...",
  "tenant_id": 82,
  "user_id": 304,
  "component": "webhook_delivery",
  "entity_id": 9912,
  "attempt": 3,
  "message": "Partner API returned 503"
}
```

## Metrics that matter

- API p50/p95/p99 latency
- error rate by route
- queue depth and oldest-job age
- job failure rate
- external API latency/error rate
- DB connection saturation
- cache hit ratio
- object-storage failure rate
- authentication failure anomalies

## Audit vs application logs

Application logs are operational and may be rotated quickly. Audit events are product/security records and need deliberate retention, access controls and immutable-enough handling.
