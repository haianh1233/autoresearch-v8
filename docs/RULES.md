# Rules and Invariants

Design rules enforced throughout the codebase. Some are enforced by lint at compile time,
others by code review convention. These exist to prevent entire categories of bugs.

---

## Type System Rules

**R1. No raw primitives across module boundaries.**
`UUID`, `String`, `int`, `long` must never appear in public method signatures across modules.
Always use branded types: `TenantId`, `PartitionId`, `TopicId`, `BrokerId`, `Offset`, etc.

```java
// WRONG
void write(UUID tenantId, UUID partitionId, byte[] value)

// RIGHT
void write(TenantId tenantId, PartitionId partitionId, byte[] value)
```

**R2. All sealed interfaces must be exhaustively switched.**
No `default` case allowed on `switch` over sealed types.

```java
// WRONG
switch (query) {
    case ShowTables s -> ...
    default -> throw new RuntimeException("unexpected");
}

// RIGHT
switch (query) {
    case ShowTables s    -> ...
    case SelectTopic s   -> ...
    case Unsupported s   -> ...
    // compiler error if any case is missing
}
```

**R3. No null returns from public methods.**
Use `Optional<T>` for nullable returns. Use `throws` for error cases.
Never return `null` from a public method.

**R4. Protocol IDs are append-only.**
`ProtocolId` values are stored in `messages.protocol_id` and segment trailers.
Never reassign, reorder, or delete an existing protocol ID.

---

## Multi-Tenancy Rules

**R5. TenantId must be present in all operations.**
Every public method in `BrokerEngine` and `StorageEngine` that touches messages,
partitions, topics, or consumer groups takes a `TenantId` parameter.

**R6. Defense-in-depth SQL filtering.**
All SQL queries that touch messages or partitions must filter on BOTH `partition_id` AND `tenant_id`,
even when `partition_id` is already tenant-scoped.

```sql
-- WRONG (partition_id alone is sufficient for correctness, but not for defense-in-depth)
SELECT * FROM messages WHERE partition_id = :pid

-- RIGHT
SELECT * FROM messages WHERE partition_id = :pid AND tenant_id = :tenantId
```

**R7. SecurityContext is never null.**
Use `SecurityContext.ANONYMOUS` (with `TenantId.SYSTEM`) for pre-auth operations.
Never pass `null` as a security context.

---

## Storage Rules

**R8. ACK fires after PG COMMIT, never before.**
In `WriteWorker.processBatch()`, the client ACK is sent only after the PG transaction
has been committed. LogSegment write is always async (post-ACK).

**R8a. `markFlushed()` fires after PG COMMIT, never before.**
In `StorageFlusher.flush(segment)`, the segment is marked FLUSHED only after the COPY
transaction has been committed. Calling `markFlushed()` before commit would allow the
segment to be deleted while PG still lacks the data.

**R9. LogSegment is a cache, not the source of truth.**
Any code that needs to know the true high watermark or partition state must query PG.
LogSegment reads are acceptable for performance; do not rely on LogSegment completeness.

**R10. Binary COPY for bulk writes.**
All bulk message writes to PG use binary COPY, not parameterized INSERT.
This is a performance invariant: INSERT-based writes are forbidden in the write path.

**R11. Epoch fencing on every write.**
Every PG UPDATE of `partition_offsets` must include `AND leader_epoch = :expectedEpoch`.
If 0 rows are updated, the broker must throw `WrongEpochException` and not ACK.

**R11a. WriteWorker threads are platform threads, not virtual threads.**
Each `WriteWorker` holds a `ThreadLocal<Connection>` for a dedicated PG connection.
Virtual threads do not have stable `ThreadLocal` affinity across park/unpark cycles.
This rule prevents connection leaks and pool corruption from misplaced virtual-thread use
in the write hot path. The read path uses virtual threads freely.

**R11b. `AbortedRange` is registered before LSO advances.**
`TransactionCoordinator.endTransaction(ABORT)` must call
`abortedTxTracker.recordAbort(pid, range)` before calling `lsoTracker.onTxnEnd(pid, ...)`.
Reversing the order would create a window where the LSO has advanced past the aborted range
but the range is not yet in the tracker — consumers could read aborted records.

---

## Clustering Rules

**R12. HRW is the sole partition leader election mechanism.**
No advisory locks, no Raft, no external coordinator.
`HRWRouter.ownerOf(partitionId)` is the definitive answer.

**R13. Forwarded writes have a hop limit of 2.**
`ForwardWriteRequest` carries a `hopCount`. If `hopCount >= 2`, the broker must reject
with `TOO_MANY_HOPS` error (prevents routing loops on stale MetadataImage).

**R14. MetadataImage must be updated before acting on stale ownership.**
If a `WrongEpochException` or `WrongLeaderException` is received, the broker must refresh
MetadataImage from PG before retrying. No retry without refresh.

**R15. Fencing is idempotent.**
Multiple brokers may attempt to fence the same stale broker simultaneously.
`BrokerFencingPipeline` uses CAS (`UPDATE ... WHERE status = 'ACTIVE'`) to ensure
exactly one broker succeeds. All others silently no-op.

---

## DLQ Rules

**R16. DLQ topic names are reserved.**
Topics starting with `__dlq.` are managed exclusively by `DlqRouter`.
No external producer may write to them directly via protocol (enforced by ACL).

**R17. DLQ partition count matches original.**
`__dlq.<topic>` always has the same `partition_count` as the original topic.
Messages are routed to the same `partition_num` as the original.

**R18. DLQ headers are immutable after injection.**
Once `x-dlq-*` headers are injected by `DlqHeaderBuilder`, they must not be modified.
They represent the state at the time of DLQ routing.

---

## Determinism Rules (for testing)

**R19. No direct clock or ID calls.**
Never call `System.currentTimeMillis()`, `Instant.now()`, `UUID.randomUUID()`,
`LocalDateTime.now()`, or `Thread.sleep()` in production code paths.
Always use `Environment.clock()`, `Environment.idGenerator()`, `Environment.scheduler()`.

**R20. No static mutable state.**
No static fields that can change at runtime. Configuration is always injected.
This enables parallel test execution without state leakage.

---

## Protocol Rules

**R21. Protocol handlers are stateless singletons.**
Per-connection state lives in `ChannelAttribute` (Netty) or a session object
keyed by channel ID. Handler classes themselves hold no mutable state.

**R22. Codecs have no business logic.**
`ivy-codec` modules only encode/decode wire formats.
Routing decisions, validation, authorization — all belong in `ivy-broker` or `ivy-server` handlers.

**R23. SQL protocols never write messages.**
`MySqlRequestHandler` and `PgWireRequestHandler` are read-only.
Any DML that would produce messages (INSERT, UPDATE, DELETE) must return an error:
```
ERROR: ivy read-only protocol adapter does not support writes. Use Kafka, AMQP, or MQTT.
```

---

## Security Rules

**R24. Deny by default.**
`AclAuthorizer` denies all operations by default. Explicit ALLOW entries are required.
There is no implicit ALLOW for unauthenticated or partially-authorized requests.

**R25. Credentials are never logged.**
The `Credentials` type overrides `toString()` → `***REDACTED***`.
Passwords, tokens, keys, and certificates must never appear in logs, exceptions, or metrics.

**R26. TLS is required for any port exposed outside localhost.**
Non-TLS ports are for local development only. Production deployments must use TLS ports.

---

## Re-Authentication Rules

**R27. Re-auth MUST NOT change the tenant.**
If the new credentials resolve to a different `TenantId` than the connection's current
`TenantId`, the broker MUST close the connection immediately with `ILLEGAL_SASL_STATE`.
This is checked in `ReAuthManager` before accepting the `IN_PROGRESS → AUTHENTICATED` transition.

**R28. ACL cache is invalidated on every successful re-auth.**
`ReAuthManager.completeReAuth()` atomically clears all cached ACL decisions for the
connection. The next authorization check will re-evaluate from `acl_entries`.

**R29. Idle connections are NOT forcibly closed at session expiry (Kafka).**
Per KIP-368: the session lifetime is checked lazily on the next incoming request.
A sleeping producer is not disconnected mid-sleep. Only active connections that send
a request after expiry receive a `RE_AUTHENTICATION_REQUIRED` error.

**R30. HTTP auth is stateless — no session, no re-auth.**
`HttpRequestHandler` evaluates credentials on every request. There is no `ReAuthManager`
instance for HTTP connections. Token expiry returns `401 Unauthorized`; the client
obtains a new token and retries the request.
