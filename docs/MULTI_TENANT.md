# Multi-Tenancy Design

## Overview

Ivy Broker is multi-tenant by default. Every operation is scoped to a `TenantId`.
Tenants are isolated at every layer: partition UUID, SQL, TLS, credentials, ACLs, quotas.

**Core invariant:** Two tenants with the same topic name "orders" get different partition UUIDs.
No query, no routing call, no storage operation can cross tenant boundaries — even if the
application layer has a bug.

---

## Tenant Resolution Pipeline

```
Client TLS handshake (SNI extension)
  │
  ▼ TenantResolverHandler (Netty, after SslHandler)
  Extract SNI hostname: "acme.broker.example.com"
  │
  ▼ SniTenantResolver
  subdomain = hostname.split(".")[0]  → "acme"
  tenantId  = UUID.nameUUIDFromBytes(subdomain.getBytes(UTF-8))  ← deterministic
  │
  ▼ TenantRegistry.resolve(hostname) → TenantRecord
  Validate status == ACTIVE → close connection if SUSPENDED/DELETED/unknown
  │
  ▼ TenantContextHandler
  TenantContext ctx = TenantContext.of(tenantId, sniHostname, protocol, connectionId)
  channel.attr(TENANT_ATTR).set(ctx)         ← fallback for Netty event loop threads
  │
  ▼ Virtual thread (WriteWorker, ReadAccumulator, etc.)
  TenantContext.callScoped(ctx, () → brokerEngine.write(...))
  ScopedValue<TenantContext>.get()            ← primary propagation (JEP 506)
```

**SNI is mandatory.** Bare hostnames (no subdomain) are rejected.
Non-TLS connections use a configurable `defaultTenant` or are rejected.

---

## Core Types

### TenantId

```java
// ivy-common — value record, zero allocation, structural equality
value record TenantId(UUID id) {
    static final TenantId SYSTEM = new TenantId(new UUID(0L, 0L));  // pre-auth ops only

    static TenantId of(UUID id)    { return new TenantId(Objects.requireNonNull(id)); }
    static TenantId of(String sni) { return new TenantId(UUID.nameUUIDFromBytes(sni.getBytes(UTF_8))); }
}
```

### TenantContext

```java
// ivy-common — carried via ScopedValue through virtual threads
record TenantContext(
    TenantId    tenantId,       // required
    String      sniHostname,    // required (e.g. "acme.broker.example.com")
    ProtocolId  protocol,       // known at bind time (port = protocol)
    UUID        connectionId    // nullable: set per-connection
) {
    // JEP 506 ScopedValue (primary mechanism for virtual threads)
    private static final ScopedValue<TenantContext> SCOPED = ScopedValue.newInstance();

    // Netty channel attribute (fallback for event loop threads)
    static final AttributeKey<TenantContext> TENANT_ATTR = AttributeKey.valueOf("ivy.tenantContext");

    static TenantContext current()                              // throws if unbound
    static Optional<TenantContext> currentOptional()
    static boolean isBound()
    static void runScoped(TenantContext ctx, Runnable action)
    static <T> T callScoped(TenantContext ctx, Callable<T> action)
}
```

### TenantMode

```java
enum TenantMode {
    SINGLE,   // all connections use a single implicit tenant (local dev)
    SNI       // tenant resolved from TLS SNI subdomain (production)
}
```

---

## Partition Isolation (the Primary Security Layer)

The most important isolation mechanism: partition UUIDs are derived from `tenantId + topicName + partitionNum`.
Two tenants with the same topic name "orders" get **different UUIDs** at every level.

```java
// MetadataManager — partition creation
UUID.nameUUIDFromBytes(
    (tenantId.id().toString() + ":" + topicName + ":" + partitionNum)
    .getBytes(UTF_8))

// Example:
// tenant "acme" + "orders" + 0  → e7b0c2a1-...
// tenant "globex" + "orders" + 0 → 3d9f1b2c-...  (completely different UUID)
```

**Effect:** Storage, routing, HRW leadership, consumer groups, transactions — all use partition UUIDs.
Cross-tenant access is impossible without knowing the other tenant's UUID.

Defense-in-depth layer: `tenant_id` column in every table + `WHERE tenant_id = ?` in every query.

---

## Tenant Registry

```java
// ivy-server — thread-safe dual-indexed cache
final class TenantRegistry {
    // Pre-sized for 64 tenants, concurrencyLevel=16
    ConcurrentHashMap<TenantId, TenantRecord>  byId;
    ConcurrentHashMap<String, TenantRecord>    byHostname;

    Optional<TenantRecord> resolve(String sniHostname)  // O(1) — hot path
    Optional<TenantRecord> get(TenantId id)             // O(1)
    synchronized void register(TenantRecord record)      // updates both maps atomically
    synchronized void unregister(TenantId id)            // removes from both maps atomically
    Collection<TenantRecord> listTenants()
}
```

### TenantRecord

```java
// ivy-server
record TenantRecord(
    TenantId     id,
    String       sniHostname,     // e.g. "acme.broker.example.com"
    TenantStatus status,          // ACTIVE | SUSPENDED | DELETED
    TenantConfig config,          // nullable → broker defaults
    Instant      createdAt,
    Instant      updatedAt
) {
    // Factory methods
    static TenantRecord forAutoProvision(TenantId, String hostname, Instant)
    static TenantRecord fromDefinition(TenantDefinition, Instant)
}
```

### TenantStatus

```java
enum TenantStatus {
    ACTIVE,    // all operations allowed
    SUSPENDED, // all operations rejected, data retained
    DELETED    // terminal; data retained for 7 days then purged
}
// Transitions: ACTIVE ↔ SUSPENDED,  ACTIVE → DELETED,  SUSPENDED → DELETED
// DELETED is terminal (no transition out)
```

---

## Tenant Lifecycle

### Create

```
TenantLifecycleManager.createTenant(TenantRecord):
  1. persistence.save(record)    → write to __tenants internal topic
  2. registry.register(record)   → in-memory cache
  → status = ACTIVE
```

### Provision Modes

| Mode | Trigger | Config |
|------|---------|--------|
| **Static (YAML)** | broker.yaml `tenants:` section | `TenantDefinition` list |
| **Auto-provision** | First SNI connection from unknown host | `provisionMode: AUTO` |
| **Admin API** | `POST /admin/tenants` | REST endpoint |

Auto-provisioning bootstrap path:
```java
// TenantLifecycleManager
Optional<TenantRecord> autoProvision(String sniHostname):
    tenantId = SniTenantResolver.resolveOrThrow(sniHostname)
    record = TenantRecord.forAutoProvision(tenantId, sniHostname, clock.instant())
    createTenant(record)
    return Optional.of(record)
```

### Suspend (8-Step Flow)

```
TenantLifecycleManager.suspendTenant(tenantId):

Step 1: Mark SUSPENDED
  UPDATE tenant record → SUSPENDED
  persistence.save(updated)
  registry.register(updated)

Step 2: Reject new connections
  (TenantContextHandler reads status on every channelActive → closes SUSPENDED tenants)

Step 3: Reject new writes
  WriteAccumulator.suspendTenant(tenantId)
  → new produce requests get TENANT_SUSPENDED error

Step 4: Drain existing connections (30s grace, then force-close)
  TenantChannelRegistry.drainTenant(tenantId, Duration.ofSeconds(30))

Step 5: Flush write accumulator
  suspensionService.flushWriteAccumulator(tenantId)
  → pending batches committed to PG before closing

Step 6: Flush log segments to PG
  StorageFlusher.flush(tenantId)

Step 7: Clean eligible segments
  LogSegmentStore.cleanExpired(tenantId)

Step 8: Flush metadata segments
  suspensionService.flushMetadata(tenantId)

→ return SuspendResult(drainedConnections, flushedSegments)
```

### Activate (4-Step Flow)

```
TenantLifecycleManager.activateTenant(tenantId):

Step 1: Mark ACTIVE
  persistence.save(withStatus(ACTIVE))
  registry.register(withStatus(ACTIVE))

Step 2: Accept connections (automatic — TenantContextHandler sees ACTIVE)

Step 3: Backfill LogSegment cache lazily (on first fetch)

Step 4: Resume producers (WriteAccumulator.activateTenant(tenantId))
```

### Delete (10-Step Flow)

```
TenantLifecycleManager.deleteTenant(tenantId):

Steps 1–8: Full suspension (if not already SUSPENDED)

Step 9: Mark DELETED, remove from registry
  persistence.save(withStatus(DELETED))
  registry.unregister(tenantId)      ← removed from both maps

Step 10: Schedule async data purge after retention window (default 7 days)
  suspensionService.schedulePurge(tenantId, Duration.ofDays(7))

→ return DeleteResult(drainedConnections, purgeScheduledAt)
```

---

## TenantConfig (Per-Tenant Overrides)

```java
// ivy-server
record TenantConfig(
    TlsTenantConfig   tls,               // per-tenant TLS + mTLS
    AuthTenantConfig  auth,              // per-tenant auth mechanisms
    QuotaTenantConfig quotas,            // per-tenant rate limits
    NamespaceMode     namespaceStrategy  // PREFIX | SCHEMA | NATIVE | SESSION
) {
    static TenantConfig DEFAULTS = /* broker-level defaults */;
}
```

### TlsTenantConfig

```java
record TlsTenantConfig(
    Path        certPath,        // PEM certificate file
    Path        keyPath,         // PEM private key file
    Path        caPath,          // CA trust chain for mTLS validation
    MtlsMode    mtlsMode,        // NONE | OPTIONAL | REQUIRED
    List<String> cipherSuites,   // null → broker defaults
    List<String> tlsVersions     // null → broker defaults (TLS 1.2 + 1.3)
)
```

**TLS per-tenant implementation:**
- `SslContextPool` — `ConcurrentHashMap<TenantId, AtomicReference<SslContext>>`
- One heavy `SslContext` per tenant, created once on first connection
- Hot-reload via `CertificateWatcher` — atomic `AtomicReference` swap without dropping connections

### AuthTenantConfig

```java
record AuthTenantConfig(
    List<SaslMechanism> mechanisms,    // PLAIN | SCRAM-SHA-256 | SCRAM-SHA-512 | OAUTHBEARER
    boolean             requireAuth,   // false = anonymous allowed (dev only)
    String              oauthIssuer,   // JWT issuer URL (nullable)
    String              oauthAudience  // JWT audience claim (nullable)
)
```

### QuotaTenantConfig

```java
record QuotaTenantConfig(
    long  produceByteRate,    // default: 10MB/s
    long  consumeByteRate,    // default: 10MB/s
    long  requestRate,        // default: 1000 req/s
    int   maxConnections,     // default: 1000 per broker
    int   maxTopics,          // default: 1000
    int   maxPartitions       // default: 10,000
)
```

Quota precedence (highest wins):
1. Per-user override (`SET QUOTA FOR USER 'alice' AT TENANT 'acme'`)
2. Per-tenant override (`TenantConfig.quotas`)
3. Broker-level default

---

## Namespace Strategies

Controls how logical resource names map per-protocol to physical partition names.
Selected per-tenant via `TenantConfig.namespaceStrategy`.

| Mode | Example | Use Case |
|------|---------|---------|
| `PREFIX` | `acme_orders` | Simple prefix isolation (default for SNI mode) |
| `SCHEMA` | `acme.orders` (PG schema) | PostgreSQL/MySQL schema-per-tenant |
| `NATIVE` | `USE acme; SELECT FROM orders` | Protocol-level command |
| `SESSION` | `SET tenant = 'acme'` | Connection-level variable |

**In SNI multi-tenant mode:** Namespace strategy is mostly redundant because partition UUID
isolation already prevents cross-tenant access. Namespace enforcement adds a human-readable
layer and provides defense-in-depth against misconfigured topic names.

```java
interface NamespaceStrategy {
    String  namespaceResource(TenantId, String resourceName);   // "orders" → "acme_orders"
    String  denamespaceResource(TenantId, String resourceName); // reverse
    boolean belongsToTenant(TenantId, String resourceName);     // ACL check
    Optional<String> connectionNamespaceCommand(TenantId);      // NATIVE only
    boolean isBlockedQuery(String query);                       // NATIVE: prevent schema escape
}
```

---

## SQL Isolation (Defense-in-Depth)

All SQL that touches messages, partitions, topics, consumer groups, or transactions must:
1. Use `partition_id` which is already tenant-scoped (primary isolation)
2. **Also** include `AND tenant_id = ?` (defense-in-depth)

```sql
-- CORRECT: double filtering
SELECT offset_num, key, value
FROM messages
WHERE partition_id = :pid
  AND tenant_id    = :tenantId      ← defense-in-depth
  AND offset_num  >= :fromOffset
ORDER BY offset_num LIMIT :limit;

-- WRONG: missing tenant_id (violates RULES.md R6)
SELECT offset_num, key, value
FROM messages
WHERE partition_id = :pid           ← missing AND tenant_id = ?
```

**`TenantSqlIsolation.validateContainsTenantFilter(sql)`** throws at compile time
if a SQL string missing the tenant filter is passed to any storage method.

---

## Internal Topics (Tenant-Scoped)

All internal metadata topics use `tenantId + ":" + topicName` as their logical name:

| Topic | Key Format | Scope | Content |
|-------|------------|-------|---------|
| `__tenants` | `tenantId` | Global (system) | TenantEntry records |
| `__consumer_offsets` | `tenantId:groupId:partitionId` | Per-tenant | Committed offsets |
| `__consumer_groups` | `tenantId:groupId` | Per-tenant | Group metadata |
| `__transactions` | `tenantId:txnId` | Per-tenant | Transaction state |
| `__producer_states` | `tenantId:producerId:partitionId` | Per-tenant | Idempotent state |
| `__credentials` | `tenantId:username` | Per-tenant | SCRAM hashes |
| `__acl_entries` | `tenantId:resourceType:resourceName:principal:op` | Per-tenant | ACL rules |
| `__schemas` | `tenantId:subject:version` | Per-tenant | Schema definitions |
| `__dlq.*` | — | Per-tenant | Dead letter queue partitions |

All keys include `tenantId` prefix. No bare topic names for metadata. This prevents
the `__consumer_offsets` scoping bug found in conduktor-ivy (Phase -1).

---

## Connection Tracking & Draining

```java
// ivy-server — tracks all active channels per tenant
final class TenantChannelRegistry {
    ConcurrentHashMap<UUID, CopyOnWriteArrayList<Channel>> channelsByTenant;

    void register(TenantId tenantId, Channel channel)      // on channelActive
    void deregister(TenantId tenantId, Channel channel)    // on channelInactive
    int  connectionCount(TenantId tenantId)                // current active connections
    int  drainTenant(TenantId tenantId, Duration deadline) // graceful then force-close
}
```

`drainTenant` flow:
1. Send protocol-appropriate "connection will close" message to each active channel
   - Kafka: `DisconnectResponse`
   - AMQP: `Connection.Close(200, "tenant suspended")`
   - MQTT: `DISCONNECT(reasonCode=0x8B sessionTakenOver)`
2. Wait for graceful close (up to `deadline`)
3. Force-close remaining channels after deadline
4. Return count of drained connections

---

## Credentials (Tenant-Scoped)

```java
// ivy-broker/auth — SCRAM credential store
final class ScramCredentialStore {
    // Key: TenantId + username (never shared across tenants)
    void createUser(TenantId, String username, char[] password)  // password zeroed after derivation
    Optional<ScramCredential> lookup(TenantId, String username, ScramAlgorithm)
    boolean verify(TenantId, String username, char[] password)
    List<String> listUsers(TenantId)
    void deleteUser(TenantId, String username)
}
```

Stored in `__credentials` internal topic. Password arrays zeroed after use.

---

## ACL Isolation

```java
// ivy-broker/auth
final class AclStore {
    // Per-tenant, deny-first algorithm, protocol-scoped
    void allow(TenantId, String principal, ResourceType, String resourceName, Operation, ProtocolId)
    void deny(TenantId, String principal, ResourceType, String resourceName, Operation, ProtocolId)
    boolean isAllowed(TenantId, SecurityContext, ResourceType, String resourceName, Operation)
}
```

**Protocol-scoped ACLs** (from master-planv8):
- Kafka ACLs (`protocol=KAFKA`) do NOT affect MQTT clients for the same topic
- Each protocol has independent ACL namespace
- "Deny by default" — no ACL entry for a protocol = deny all

---

## Metrics (Tenant-Tagged)

All Prometheus metrics carry a `tenant` label:

```
ivy_connections_active{protocol, tenant}
ivy_connections_rejected_total{reason, tenant}
ivy_write_latency_ms{tenant, partition}
ivy_read_latency_ms{tenant, partition}
ivy_dlq_messages_total{topic, reason, tenant}
ivy_quota_throttled_total{type, tenant, user}
ivy_quota_exceeded_total{type, tenant}
ivy_auth_success_total{mechanism, tenant}
ivy_auth_failure_total{mechanism, reason, tenant}
ivy_tls_handshake_ms{protocol, tenant}
ivy_tenant_status{tenant}                 ← 1=ACTIVE, 0=SUSPENDED
```

---

## Configuration Example (broker.yaml)

```yaml
broker:
  tenantMode: SNI               # SINGLE | SNI

  tenants:                      # static provisioning (bootstrap)
    provisionMode: MANUAL       # MANUAL | AUTO
    definitions:
      - id: 550e8400-e29b-41d4-a716-446655440000
        sniHostname: acme.broker.example.com
        namespaceStrategy: PREFIX
        tls:
          certPath: /certs/acme.crt
          keyPath:  /certs/acme.key
          caPath:   /certs/acme-ca.crt
          mtlsMode: OPTIONAL
        auth:
          mechanisms: [SCRAM-SHA-256]
          requireAuth: true
        quotas:
          produceByteRate: 10485760   # 10MB/s
          consumeByteRate: 20971520   # 20MB/s
          maxConnections: 500
          maxTopics: 200

      - id: 6ba7b810-9dad-11d1-80b4-00c04fd430c8
        sniHostname: globex.broker.example.com
        auth:
          mechanisms: [SCRAM-SHA-256, SCRAM-SHA-512, OAUTHBEARER]
          oauthIssuer: https://auth.globex.com
          oauthAudience: ivy-broker
        # tls, quotas → broker defaults
```

---

## Security Layer Responsibilities

| Layer | Module | Tenant Role |
|-------|--------|-------------|
| TLS + SNI | ivy-server | Extract SNI → TenantId; per-tenant SslContext |
| Tenant resolution | ivy-server | TenantRegistry lookup, status check, reject SUSPENDED |
| Protocol binding | ivy-server | Port = protocol; `TenantContext.protocol` set at ServerBootstrap bind time |
| Auth | ivy-broker | `ScramCredentialStore(tenantId, username)` |
| Authorization | ivy-broker | `AclStore(tenantId, principal, resource, op, protocol)` |
| Quota | ivy-broker | `TokenBucketQuotaManager(tenantId, principal)` |
| Partition routing | ivy-broker | `MetadataManager`: tenantId in partition UUID seed |
| Storage write | ivy-broker | `WriteWorker`: tenantId in every COPY row |
| Storage read | ivy-broker | `ReadAccumulator`: `AND tenant_id = ?` in every SELECT |
| Inter-broker RPC | ivy-broker | `InterBrokerMessage` carries tenantId in header |

**ivy-broker itself has zero auth awareness** — it receives pre-authenticated `SecurityContext`
from protocol handlers. Auth enforcement happens in handlers before calling `BrokerEngine`.

---

## Key Classes Per Module

### ivy-common
| Class | Purpose |
|-------|---------|
| `TenantId` | Branded UUID value record |
| `TenantContext` | ScopedValue container + channel attr fallback |
| `TenantStatus` | ACTIVE / SUSPENDED / DELETED enum |
| `TenantMode` | SINGLE / SNI enum |
| `TenantsConfig` | Mode + provisionMode + definitions |
| `TenantDefinition` | YAML-loadable per-tenant spec |

### ivy-server
| Class | Purpose |
|-------|---------|
| `SniTenantResolver` | SNI hostname → deterministic TenantId |
| `TenantResolverHandler` | Netty handler: TLS SNI extraction |
| `TenantContextHandler` | Netty handler: registry lookup + status validation |
| `TenantRegistry` | Dual-indexed in-memory cache |
| `TenantRecord` | Registry entry with config + lifecycle |
| `TenantLifecycleManager` | create/suspend/activate/delete flows |
| `TenantConfig` | Per-tenant TLS + auth + quotas + namespace |
| `TlsTenantConfig` | Per-tenant certificate + mTLS config |
| `AuthTenantConfig` | Per-tenant auth mechanisms |
| `QuotaTenantConfig` | Per-tenant rate limits |
| `TenantPersistence` | Interface: save/load/delete from `__tenants` |
| `TenantSuspensionService` | Interface: 8-step suspension operations |
| `DefaultTenantSuspensionService` | Implementation |
| `TenantChannelRegistry` | Active channel tracking + graceful drain |
| `SslContextPool` | Per-tenant `AtomicReference<SslContext>` + hot-reload |

### ivy-broker
| Class | Purpose |
|-------|---------|
| `TenantStore` | In-memory cache of `__tenants` topic (replayed on startup) |
| `TenantEntry` | Serialized tenant record stored in `__tenants` |
| `ScramCredentialStore` | Tenant-scoped credential lookup + SCRAM derivation |
| `AclStore` | Tenant + protocol-scoped ACL evaluation (deny-first) |
| `TokenBucketQuotaManager` | Per-tenant/per-principal quota enforcement |
| `TenantSqlIsolation` | Compile-time SQL validation (`AND tenant_id = ?`) |

---

## Invariants (from RULES.md)

- **R5:** TenantId present in all `BrokerEngine` and `StorageEngine` method signatures
- **R6:** SQL defense-in-depth — `WHERE partition_id = ? AND tenant_id = ?` always
- **R7:** `SecurityContext` is never null — use `SecurityContext.ANONYMOUS` for pre-auth
- **R17:** DLQ topics are `__dlq.<original-topic>` — always tenant-scoped via partition UUID
- No `System.currentTimeMillis()` in tenant lifecycle code — use `Environment.clock()`
- Internal topic keys always prefixed with `tenantId + ":"` — no bare topic names
