# ACL Design

> **Related:** [SECURITY.md](SECURITY.md) §Layer 6, [MULTI_TENANT.md](MULTI_TENANT.md) §ACL Isolation,
> [RULES.md](RULES.md) §R24

---

## Overview

Ivy uses a **DENY-first, protocol-scoped** ACL model. Every operation requires an explicit
ALLOW entry. No implicit permissions exist, even for authenticated principals.

---

## ACL Entry Model

```java
record AclEntry(
    TenantId tenantId,
    ResourceType resourceType,     // TOPIC, GROUP, CLUSTER, TRANSACTIONAL_ID
    String resourceName,           // "orders", "payment-*", "*"
    AclPatternType patternType,    // LITERAL or PREFIXED
    String principal,              // "User:alice", "Group:admins", "*"
    String host,                   // "10.0.1.42", "10.0.1.0/24", "*"
    AclOperation operation,        // READ, WRITE, CREATE, DELETE, ALTER, DESCRIBE, ALL
    AclPermission permission,      // ALLOW or DENY
    ProtocolId protocol            // KAFKA, MQTT, AMQP, etc. (protocol-scoped)
)
```

---

## 6-Step Evaluation Algorithm

For each incoming request `(tenantId, principal, groups, resource, operation, protocol, clientIp)`:

```
Step 1 — Protocol filter
   Load ACL entries WHERE tenant_id = :tenantId AND protocol = :protocol

Step 2 — Resource matching
   For each entry, match resource:
     LITERAL:   entry.resourceName == request.resourceName
     PREFIXED:  request.resourceName.startsWith(entry.resourceName)
     Wildcard:  entry.resourceName == "*" (matches all resources of that type)

Step 3 — Principal matching
   Match if:
     entry.principal == "*"                               (any principal)
     entry.principal == "User:" + request.principal       (exact user)
     entry.principal == "Group:" + g  WHERE g ∈ request.groups  (group membership)

Step 4 — Host matching
   Match if:
     entry.host == "*"                                    (any host)
     entry.host == request.clientIp                       (exact IP)
     CIDR match: request.clientIp ∈ entry.host            (subnet)

Step 5 — DENY check (short-circuit)
   If ANY matching entry has permission == DENY → DENIED (immediately, no further checks)

Step 6 — ALLOW check
   If at least one matching entry has permission == ALLOW → APPROVED
   Else → DENIED ("No matching ALLOW ACL")
```

**Key property:** DENY always wins. A single DENY entry overrides any number of ALLOW entries.

---

## Protocol-Scoped ACLs

ACLs are **protocol-scoped**: Kafka ACLs affect only Kafka clients, MQTT ACLs affect only
MQTT clients. This prevents unintended cross-protocol permission leakage.

**Example:**
```
# Alice can produce to "orders" via Kafka but NOT via MQTT
AclEntry(tenant=acme, resource=TOPIC:orders, principal=User:alice, op=WRITE, perm=ALLOW, protocol=KAFKA)
# No MQTT WRITE entry for alice → MQTT publish to "orders" is DENIED
```

### Protocol-to-Operation Mapping

| Protocol | Wire Operation | ACL Operation | ACL Resource |
|----------|---------------|---------------|--------------|
| Kafka | Produce | WRITE | TOPIC |
| Kafka | Fetch | READ | TOPIC |
| Kafka | JoinGroup | READ | GROUP |
| Kafka | CreateTopics | CREATE | TOPIC |
| Kafka | DeleteTopics | DELETE | TOPIC |
| MQTT | PUBLISH | WRITE | TOPIC |
| MQTT | SUBSCRIBE | READ | TOPIC |
| AMQP 0-9-1 | Basic.Publish | WRITE | TOPIC |
| AMQP 0-9-1 | Basic.Consume | READ | TOPIC |
| AMQP 1.0 | Transfer (sender) | WRITE | TOPIC |
| AMQP 1.0 | Transfer (receiver) | READ | TOPIC |
| HTTP | POST /topics/{t}/messages | WRITE | TOPIC |
| HTTP | GET /topics/{t}/messages | READ | TOPIC |
| MySQL/PgWire | SELECT | READ | TOPIC |

---

## Caching

### Caffeine Cache

```java
Cache<AclCacheKey, AclDecision> aclCache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .build();

record AclCacheKey(
    TenantId tenantId,
    String principal,
    ResourceType resourceType,
    String resourceName,
    AclOperation operation,
    ProtocolId protocol,
    InetAddress clientAddress
)
```

### Invalidation Triggers

| Trigger | Scope | Mechanism |
|---------|-------|-----------|
| Successful re-auth | Per-principal (all protocols) | `aclCache.invalidate(principal, *)` |
| ACL entry created/deleted | Per-tenant | `aclCache.invalidateAll()` for tenant |
| Credential revocation | Per-principal | `aclCache.invalidate(principal, *)` |
| Security epoch increment | Per-principal | Implicit (epoch check rejects before cache hit) |

### Cluster Propagation

ACL changes propagate to other brokers via:
1. PG `LISTEN/NOTIFY` on `acl_change` channel (<100 ms).
2. `__acl_entries` internal topic compaction (authoritative).
3. Polling fallback every 60s.

---

## Limits

| Limit | Default | Rationale |
|-------|---------|-----------|
| Max ACL rules per tenant | 10,000 | Prevents cache bloat; enforced via `SELECT COUNT(*) + PG advisory lock` |
| Max resource name length | 255 chars | Prevents excessively long pattern matching |
| Max principal name length | 255 chars | Consistent with Kafka |

---

## Superuser Bypass

Principals listed in `security.superusers` bypass all ACL checks:

```yaml
security:
  superusers:
    - "User:admin"
    - "User:broker-internal"
```

Superuser checks are performed **before** the 6-step algorithm. If the requesting principal
matches any superuser entry, the operation is immediately APPROVED with no ACL evaluation.

---

## Module Placement

| Component | Module | Purpose |
|-----------|--------|---------|
| `AclAuthorizer` | `ivy-auth` | 6-step evaluation engine |
| `AclStore` | `ivy-broker` | In-memory cache of `__acl_entries` topic |
| `AclEntry` | `ivy-common` | Value record |
| `AclCacheKey` / `AclDecision` | `ivy-auth` | Caffeine cache types |
| `AclAdminHandler` | `ivy-codec` (kafka) | DescribeAcls / CreateAcls / DeleteAcls API handlers |

---

## Configuration

```yaml
security:
  acl:
    default-policy: DENY              # DENY or ALLOW (production: always DENY)
    cache-max-size: 10000
    cache-ttl-minutes: 5
    max-rules-per-tenant: 10000
    superusers: ["User:admin"]
```

---

*Last updated: 2026-03-25*
