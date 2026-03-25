# Audit Logging

> **Related:** [SECURITY.md](SECURITY.md) §Layer 8, [RE_AUTH.md](RE_AUTH.md) §Audit Events

---

## Overview

All security-relevant events are logged as structured JSON to the `audit_log` PostgreSQL table.
Audit events are append-only, partitioned by timestamp, and retained according to configurable
tier policies.

---

## Event Types

| Event Type | Trigger | Key Fields |
|------------|---------|------------|
| `AUTHENTICATE` | Initial auth success/failure | mechanism, principal, clientIp, protocol |
| `RE_AUTHENTICATE` | Re-auth success/failure | mechanism, principal, newPrincipal, reAuthReason, durationMs |
| `IDENTITY_MAP` | PrincipalResolverChain completes | rawPrincipal, resolvedPrincipal, groups, resolver |
| `AUTHORIZE` | ACL check (logged only on DENY or when audit-all enabled) | principal, resource, operation, protocol, decision |
| `QUOTA_EXCEEDED` | Token bucket throttle applied | principal, quotaType, limit, actual, throttleMs |
| `TLS_HANDSHAKE` | TLS negotiation complete | cipherSuite, tlsVersion, sniHostname, clientCertCN |
| `CREDENTIAL_CHANGE` | Credential created/updated/deleted | principal, changeType, mechanism |
| `ACL_CHANGE` | ACL entry created/deleted | principal, resource, operation, permission, changeType |
| `TENANT_CREATE` | Tenant provisioned | tenantId, sniHostname, provisionMode |
| `TENANT_SUSPEND` | Tenant suspended | tenantId, reason |
| `TENANT_DELETE` | Tenant deleted | tenantId |
| `CONNECTION_CLOSE` | Connection terminated | principal, protocol, reason, durationMs, bytesIn, bytesOut |

---

## Event Structure

```json
{
  "eventId": "01913e4b-7c8a-7000-8000-000000000001",
  "eventType": "AUTHENTICATE",
  "timestamp": "2026-03-25T14:32:01.123Z",
  "tenantId": "550e8400-e29b-41d4-a716-446655440000",
  "principal": "alice",
  "clientIp": "10.0.1.42",
  "protocol": "KAFKA",
  "sessionId": "01913e4b-7c8a-7000-8000-000000000002",
  "brokerId": "b1-uuid",
  "result": "SUCCESS",
  "details": {
    "mechanism": "SCRAM-SHA-512",
    "sessionLifetimeMs": 3600000
  }
}
```

---

## Field Masking (Rule R25)

The following fields are **never** included in audit events:

| Masked Field Pattern | Example | Replacement |
|---------------------|---------|-------------|
| `password` | SASL password bytes | not logged |
| `token` | JWT token value | not logged |
| `secret` | HMAC secret, client secret | not logged |
| `key` (credential context) | SCRAM StoredKey, ServerKey | not logged |
| `credential` | raw credential bytes | not logged |
| `hmac` | delegation token HMAC | not logged |

The `mechanism` field identifies **how** auth was performed; the actual credential is never logged.
`Credentials.toString()` → `"***REDACTED***"`.

---

## PostgreSQL Storage

```sql
CREATE TABLE audit_log (
    event_id    UUID        NOT NULL DEFAULT gen_random_uuid(),
    tenant_id   UUID        NOT NULL,
    event_type  TEXT        NOT NULL,
    timestamp   TIMESTAMPTZ NOT NULL DEFAULT now(),
    principal   TEXT,
    client_ip   INET,
    protocol    TEXT,
    session_id  UUID,
    broker_id   UUID,
    result      TEXT,
    details     JSONB       NOT NULL DEFAULT '{}',
    PRIMARY KEY (timestamp, event_id)
) PARTITION BY RANGE (timestamp);

-- Partition per month (created automatically by SchemaManager)
CREATE TABLE audit_log_2026_03 PARTITION OF audit_log
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

CREATE INDEX idx_audit_tenant_time ON audit_log (tenant_id, timestamp DESC);
CREATE INDEX idx_audit_principal   ON audit_log (tenant_id, principal, timestamp DESC);
CREATE INDEX idx_audit_event_type  ON audit_log (event_type, timestamp DESC);
```

---

## Retention Tiers

| Tier | Age | Storage | Indexes |
|------|-----|---------|---------|
| **Hot** | 0–30 days | Primary PG table, full indexes | All |
| **Warm** | 30–365 days | Archived partitions (read-only tablespace) | tenant_id + timestamp only |
| **Cold** | 1–7 years | Exported to S3/Iceberg (optional) | None (query via Athena/Trino) |

Partition rotation is handled by `AuditPartitionManager`:
- Creates next month's partition 7 days in advance.
- Detaches partitions older than hot tier.
- Optionally exports to S3 before detach.

---

## Write Path

Audit events are **buffered** to minimize hot-path impact:

```
Security event → AuditWriter.emit(event)
                    │
                    ▼
              MpscArrayQueue<AuditEvent> (capacity: 16K)
                    │
                    ▼ (dedicated platform thread, 100ms flush cycle)
              COPY audit_log FROM STDIN (binary, batched)
                    │
                    ▼
              PG COMMIT (separate from message writes)
```

**Key properties:**
- Audit writes are **async** and never block the request hot path.
- Queue-full behaviour: drop event (counter incremented; alert if sustained).
- Dedicated PG connection (not from HikariCP pool — prevents audit from starving writes).
- Flush on shutdown: `AuditWriter.drainAll()` called during graceful shutdown step 3.5.

---

## Configuration

```yaml
audit:
  enabled: true
  log-authorize-allow: false      # only log DENY by default (reduce volume)
  log-connection-close: true
  buffer-capacity: 16384
  flush-interval-ms: 100
  retention:
    hot-days: 30
    warm-days: 365
    cold-export: false            # set true to export to S3 before detach
    cold-s3-bucket: ""
```

---

*Last updated: 2026-03-25*
