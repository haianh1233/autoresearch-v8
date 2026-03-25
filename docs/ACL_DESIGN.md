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
    ResourceType resourceType,     // see Per-Protocol Resource Types table
    String resourceName,           // "orders", "payment-*", "*"
    AclPatternType patternType,    // LITERAL or PREFIXED
    String principal,              // "User:alice", "Group:admins", "*"
    String host,                   // "10.0.1.42", "10.0.1.0/24", "*"
    AclOperation operation,        // READ, WRITE, CREATE, DELETE, ALTER, DESCRIBE, ALL
    AclPermission permission,      // ALLOW or DENY
    ProtocolId protocol            // KAFKA, MQTT, AMQP, etc. (protocol-scoped)
)
```

### Per-Protocol Resource Types

| Protocol | Resource Types | Notes |
|----------|---------------|-------|
| Kafka | TOPIC, GROUP, CLUSTER, TRANSACTIONAL_ID, DELEGATION_TOKEN | Standard Kafka ACL model |
| MQTT | TOPIC, SUBSCRIPTION | SUBSCRIPTION for `$share/group/topic` shared subs |
| AMQP 0-9-1 | EXCHANGE, QUEUE, VHOST | VHOST maps to tenant; EXCHANGE and QUEUE are separate resources |
| AMQP 1.0 | TOPIC, LINK | LINK for per-link Attach authorization (sender vs receiver) |
| HTTP | TOPIC | REST endpoints map to topic operations |
| MySQL/PgWire | TOPIC | SELECT on topic = READ |

`ResourceType` is an **open enum** — new protocol resource types can be added without modifying
existing ACL entries. Unknown resource types are rejected at creation time.

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

**Host-based cache key rationale:** The `clientAddress` in `AclCacheKey` is REQUIRED to prevent
host-based ACL bypass. Without it, an ALLOW for `10.0.0.1` gets cached, and clients from
`10.0.0.2` incorrectly receive the cached ALLOW. Including the address ensures each IP gets
its own cache entry.

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

### Cross-Protocol Isolation Rules

1. **No cross-protocol inheritance:** A Kafka WRITE on `payments` does NOT grant MQTT PUBLISH
   on `payments`. Each protocol has independent ACL entries.

2. **Same user, different protocols:** When `alice` authenticates via Kafka AND MQTT, she gets
   the same `ResolvedIdentity` (username, groups) but different `AclCacheKey`s:
   - Kafka: `(acme, alice, TOPIC, payments, WRITE, KAFKA, 10.0.1.1)`
   - MQTT: `(acme, alice, TOPIC, payments, WRITE, MQTT, 10.0.1.1)`
   These are independent cache entries with independent ACL evaluations.

3. **Protocol wildcard (`protocol=*`):** A wildcard protocol entry matches ALL protocols.
   Use sparingly — typically only for DENY rules (e.g., "deny User:bob ALL on TOPIC:* across
   all protocols"). Wildcard ALLOW rules are checked AFTER protocol-specific rules.

4. **AMQP 0-9-1 resource type mismatch:** AMQP 0-9-1 uses EXCHANGE and QUEUE resource types,
   not TOPIC. A Kafka ACL on `TOPIC:orders` does NOT apply to AMQP `EXCHANGE:orders` even
   if the underlying Ivy topic is the same.

5. **Error code mapping:** Each protocol returns its native error when ACL denies:

   | Protocol | Error on ACL DENY |
   |----------|------------------|
   | Kafka | `TOPIC_AUTHORIZATION_FAILED` (error code 29) |
   | AMQP 0-9-1 | `Channel.Close(reply-code=403, reply-text="ACCESS_REFUSED")` |
   | AMQP 1.0 | `Detach(error={condition="amqp:unauthorized-access"})` |
   | MQTT 3.1.1 | `SUBACK(returnCode=0x80 Failure)` or connection close |
   | MQTT 5.0 | `CONNACK/SUBACK/PUBACK(reasonCode=0x87 Not Authorized)` |
   | HTTP | `403 Forbidden` with JSON error body |
   | MySQL | `ERROR 1045 (28000): Access denied for user` |
   | PgWire | `ErrorResponse(severity=ERROR, code=42501, message="permission denied")` |

### Complete Wire-Operation → ACL Mapping

**Kafka:**

| Wire Operation | ACL Operation | ACL Resource | Notes |
|---------------|---------------|--------------|-------|
| Produce | WRITE | TOPIC | Per-topic, per-partition |
| Fetch | READ | TOPIC | |
| JoinGroup | READ | GROUP | Consumer group membership |
| SyncGroup | READ | GROUP | |
| OffsetCommit | READ | GROUP | |
| OffsetFetch | DESCRIBE | GROUP | |
| CreateTopics | CREATE | TOPIC | |
| DeleteTopics | DELETE | TOPIC | |
| AlterConfigs | ALTER | TOPIC or CLUSTER | |
| DescribeConfigs | DESCRIBE | TOPIC or CLUSTER | |
| CreatePartitions | ALTER | TOPIC | |
| InitProducerId | WRITE | TRANSACTIONAL_ID | Transactional producers |
| AddPartitionsToTxn | WRITE | TRANSACTIONAL_ID | |
| EndTxn | WRITE | TRANSACTIONAL_ID | |
| CreateDelegationToken | CREATE | DELEGATION_TOKEN | |
| DescribeAcls | DESCRIBE | CLUSTER | Admin |
| CreateAcls | ALTER | CLUSTER | Admin |
| DeleteAcls | ALTER | CLUSTER | Admin |

**AMQP 0-9-1:**

| Wire Operation | ACL Operation | ACL Resource | Notes |
|---------------|---------------|--------------|-------|
| Basic.Publish | WRITE | EXCHANGE | Checked against exchange name |
| Basic.Consume | READ | QUEUE | Checked against queue name |
| Basic.Get | READ | QUEUE | Synchronous pull |
| Exchange.Declare | CREATE | EXCHANGE | |
| Exchange.Delete | DELETE | EXCHANGE | |
| Exchange.Bind | CREATE | EXCHANGE | Binding requires CREATE on both source and destination |
| Queue.Declare | CREATE | QUEUE | |
| Queue.Delete | DELETE | QUEUE | |
| Queue.Bind | CREATE | QUEUE | |
| Queue.Unbind | DELETE | QUEUE | |
| Queue.Purge | DELETE | QUEUE | Destructive operation |
| Connection.Open(vhost) | READ | VHOST | Vhost maps to tenant namespace |
| Tx.Select/Commit/Rollback | — | — | No ACL (transaction boundary, not resource operation) |
| Confirm.Select | — | — | No ACL (feature flag) |

**AMQP 1.0:**

| Wire Operation | ACL Operation | ACL Resource | Notes |
|---------------|---------------|--------------|-------|
| Attach(role=sender, target) | WRITE | TOPIC | Checked at Attach time against `target.address` |
| Attach(role=receiver, source) | READ | TOPIC | Checked at Attach time against `source.address` |
| Transfer | — | — | No re-check (authorized at Attach) |
| Disposition | — | — | No re-check (ack/nack/reject) |
| Begin (session) | — | — | No ACL (session lifecycle) |
| Detach / End / Close | — | — | No ACL |

**MQTT 3.1.1 / 5.0:**

| Wire Operation | ACL Operation | ACL Resource | Notes |
|---------------|---------------|--------------|-------|
| PUBLISH | WRITE | TOPIC | Topic filter checked |
| SUBSCRIBE | READ | TOPIC | Each topic filter in SUBSCRIBE checked independently |
| SUBSCRIBE `$share/group/topic` | READ | SUBSCRIPTION | Shared subscriptions use SUBSCRIPTION resource |
| CONNECT (will topic) | WRITE | TOPIC | Will message authorization at CONNECT time |
| AUTH (5.0 only) | — | — | Re-auth, not resource access |

**HTTP REST:**

| Wire Operation | ACL Operation | ACL Resource | Notes |
|---------------|---------------|--------------|-------|
| POST /topics/{t}/messages | WRITE | TOPIC | |
| POST /topics/{t}/messages/batch | WRITE | TOPIC | |
| GET /topics/{t}/messages | READ | TOPIC | Long-poll supported |
| GET /topics | DESCRIBE | CLUSTER | Metadata |
| GET /topics/{t} | DESCRIBE | TOPIC | |

**MySQL / PgWire (Read-Only):**

| Wire Operation | ACL Operation | ACL Resource | Notes |
|---------------|---------------|--------------|-------|
| SELECT FROM topic | READ | TOPIC | |
| SHOW TABLES | DESCRIBE | CLUSTER | Metadata |
| SHOW CREATE TABLE topic | DESCRIBE | TOPIC | |

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
