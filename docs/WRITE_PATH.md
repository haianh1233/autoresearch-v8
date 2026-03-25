# Write Path

## Overview

The write path is **PG-first**: ACK is sent to the client only after PostgreSQL COMMIT.
LogSegment is populated asynchronously after the ACK — it is a read performance cache, not
the source of truth.

```
Client
  │  (produce request, any protocol)
  ▼
Protocol Handler                            [Netty event loop thread — never blocks]
  │  decode → List<PendingWrite>
  ▼
DefaultBrokerEngine.write(tenantId, pendingWrites, securityContext)
  │
  ├── [partition leader check via HRWRouter]
  │
  ├── IS owner → WriteAccumulator.accumulate(pendingWrites)
  │                │
  │                │  (flush trigger: 1K msgs OR 1MB OR 5ms linger)
  │                ▼
  │             WriteWorker.processBatch()   [platform thread, PG-affined]
  │                │
  │                ├─ Layer 1: check MetadataImage.epoch(partitionId) == batch.epoch
  │                ├─ BEGIN TRANSACTION
  │                │    UPDATE partition_offsets                       ← Layer 2 epoch fence
  │                │      SET next_offset = next_offset + batch_size
  │                │      WHERE partition_id = ? AND leader_epoch = ?
  │                │      RETURNING next_offset - batch_size AS base;
  │                │    COPY messages FROM STDIN BINARY;
  │                │    UPSERT producer_state;
  │                │    COMMIT;
  │                ├─ ◀── ACK to client fires HERE (after COMMIT, never before)
  │                ├─ FlushEventDispatcher.publish(FlushEvent)         [non-blocking]
  │                └─ StorageFlusher.schedule(partitionId, records)    [non-blocking]
  │
  └── NOT owner → ForwardWriteManager.forward(ownerBrokerId, pendingWrites)
                     │
                     ▼
                  InterBrokerRpcClient.send(ForwardWriteRequest)
                     │  (Netty, persistent connection, exponential-backoff reconnect)
                     ▼
                  Owner Broker → same path as above
                     ▼
                  ForwardWriteResponse(baseOffset, errorCode)
                     ▼
                  ACK to original client
```

---

## Threading Model

Understanding which thread does what is essential for correctness and performance.

```
Netty event loop threads (2 × CPU cores)
  │   Role: decode protocol frames, call engine.write() — NEVER blocks
  │   Key invariant: returns as soon as batch is handed to WriteAccumulator
  ▼
WriteAccumulator
  │   Non-blocking append. If flush triggers fire, the linger scheduler
  │   (not the event loop) drains the batch.
  ▼
WriteWorker threads (4 platform threads, NOT virtual threads)
  │   Role: run PG transactions (UPDATE + COPY + COMMIT)
  │   Why platform threads: each worker owns a ThreadLocal<Connection>,
  │   bypassing HikariCP lock acquisition on every batch. Virtual threads
  │   do not have stable ThreadLocal affinity across park/unpark, making
  │   dedicated connection ownership unreliable.
  │   Thread affinity: partitionId.hashCode() % 4 — one partition always
  │   uses the same worker, the same connection, the same PG statement cache.
  ▼
FlushEventDispatcher (1 platform thread)
  │   Role: drain MpscArrayQueue<FlushEvent>, notify waiters and push subscribers
  │   Why platform: tight 1 ms park loop — virtual thread overhead not worth it
  ▼
StorageFlusher (1 background thread, 200 ms cycle)
      Role: copy committed records from WriteWorker queues into LogSegment files
```

---

## WriteAccumulator

One accumulator per partition. Batches pending writes until a flush trigger fires.

### Flush Triggers (whichever fires first)

| Trigger | Default |
|---------|---------|
| Message count | 1,000 |
| Accumulated bytes | 1 MB |
| Linger time (time since first message in batch) | 5 ms |

### Swap-Drain Pattern (Lock-Free)

```java
class PartitionBatch {
    AtomicReference<List<PendingWrite>> current = new AtomicReference<>(new ArrayList<>());

    // Called from Netty event loop — must not block
    void add(PendingWrite pw) {
        while (true) {
            List<PendingWrite> batch = current.get();
            List<PendingWrite> next = new ArrayList<>(batch);
            next.add(pw);
            if (current.compareAndSet(batch, next)) {
                checkFlushTriggers(next);
                return;
            }
        }
    }

    // Called by linger scheduler or flush trigger
    List<PendingWrite> drain() {
        return current.getAndSet(new ArrayList<>());
    }
}
```

The linger scheduler runs a `ScheduledExecutorService` with one timer per active partition,
reset whenever a new message arrives. On timer fire → `drain()` → hand batch to WriteWorker.

### Tenant Suspension

```java
Set<TenantId> suspendedTenants = ConcurrentHashMap.newKeySet();

void accumulate(TenantId tenantId, List<PendingWrite> writes) {
    if (suspendedTenants.contains(tenantId)) {
        throw new TenantSuspendedException(tenantId);
    }
    // ... normal accumulation
}

// Called by TenantLifecycleManager during DRAINING or SUSPENDED transition
void suspendTenant(TenantId tenantId)   { suspendedTenants.add(tenantId); }
void unsuspendTenant(TenantId tenantId) { suspendedTenants.remove(tenantId); }
```

### submitImmediate (Bypass Linger)

For single-message writes where the client has acks=-1 and cannot wait for the linger timer:

```java
void submitImmediate(TenantId tenantId, PendingWrite pw) {
    // Drain any existing batch for the partition (consolidate), then flush
    List<PendingWrite> combined = drain(pw.partitionId());
    combined.add(pw);
    writeWorker(pw.partitionId()).submit(combined);
}
```

Use cases: MQTT QoS 2 final release, AMQP publisher-confirm with mandatory=true.

---

## WriteWorker (PG-First Transaction)

Four platform threads, each dedicated to one quarter of the partition space via hash affinity.

```
processBatch(List<PendingWrite> batch):

LAYER 1 — LOCAL EPOCH CHECK (fast, no PG round-trip)
  expected = metadataImage.epoch(partitionId)
  if (batch.leaderEpoch != expected) → throw WrongEpochException

BEGIN TRANSACTION

LAYER 2 — ATOMIC OFFSET ALLOCATION + EPOCH FENCE
  UPDATE partition_offsets
  SET    next_offset = next_offset + :batchSize,
         leader_id   = :myBrokerId             -- write our broker_id atomically
  WHERE  partition_id  = :partitionId
    AND  leader_epoch  = :expectedEpoch         -- PG-side epoch fence
  RETURNING next_offset - :batchSize AS baseOffset;

  → 0 rows updated means epoch changed between Layer 1 and here → WrongEpochException

WRITE MESSAGES
  COPY messages (partition_id, offset_num, tenant_id, key, value,
                 headers, timestamp_ms, protocol_id, is_dlq)
  FROM STDIN WITH (FORMAT binary);

  Each PendingWrite[i] → offset_num = baseOffset + i

UPSERT PRODUCER STATE (idempotency)
  INSERT INTO producer_state
    (producer_id, partition_id, producer_epoch, last_sequence, last_offset)
  VALUES (:pid, :partId, :epoch, :sequence, :lastOffset)
  ON CONFLICT (producer_id, partition_id) DO UPDATE
    SET producer_epoch = :epoch,
        last_sequence  = :sequence,
        last_offset    = :lastOffset
    WHERE producer_state.producer_epoch <= :epoch;  -- never downgrade epoch

COMMIT

RETURN WriteResult(baseOffset, batchSize, partitionId, logAppendTimeMs=now())

[non-blocking, post-commit]
  flushEventQueue.offer(FlushEvent(partitionId, baseOffset, baseOffset+batchSize-1, batchSize))
  storageFlusher.schedule(partitionId, records)
```

### ThreadLocal PG Connection

Each WriteWorker holds a dedicated connection: `ThreadLocal<Connection> pgConnection`.

```java
Connection getConnection() {
    Connection conn = pgConnection.get();
    if (conn == null || conn.isClosed()) {
        conn = dataSource.getConnection();
        pgConnection.set(conn);
    }
    return conn;
}
```

This eliminates HikariCP pool lock acquisition per batch. The pool is sized to exactly
`worker-threads` (4) to ensure each thread always gets a connection without queueing.

### PG COMMIT Timeout Recovery

If the `COMMIT` call times out (network blip), the WriteWorker re-reads `partition_offsets`
before retrying:

```
SELECT next_offset FROM partition_offsets WHERE partition_id = ?
  → if next_offset == expectedBase + batchSize  → commit succeeded; proceed as normal
  → if next_offset == expectedBase              → commit failed; retry
  → otherwise                                  → another broker took over; WrongEpochException
```

This makes the write idempotent: a double-commit is caught by the `(partition_id, offset_num)`
PRIMARY KEY on the `messages` table and silently discarded.

---

## ProducerStateManager (Idempotency)

Enforces exactly-once semantics for producers using a sliding window of the last 5 batches.

```
checkAndRecord(producerId, partitionId, producerEpoch, sequence, batchSize):

  state = producerStates.get(producerId, partitionId)

  if state is null:
      → new producer, accept, record initial state

  if producerEpoch > state.epoch:
      → new epoch (producer restarted), reset window, accept

  if producerEpoch < state.epoch:
      → INVALID_PRODUCER_EPOCH (zombie producer, fenced out)

  if producerEpoch == state.epoch:
      for each cached batch in sliding window (last 5):
          if batch.sequence == sequence:
              → duplicate, return cached WriteResult (not an error)
      if sequence == state.nextExpected:
          → in-order, accept, advance window
      if sequence > state.nextExpected:
          → OUT_OF_ORDER_SEQUENCE (gap detected, reject)
      if sequence < state.nextExpected and not in cache:
          → duplicate but evicted from window → treat as out-of-order or accept based on
             fence: if offset already committed, duplicate; if gap, error

Sliding window: fixed-size ring buffer of 5 BatchState records per (producerId, partitionId)
```

**Why 5?** Kafka uses 5 as the maximum number of in-flight batches. A properly behaving
producer never has more than 5 unacknowledged batches outstanding at once.

---

## FlushEventDispatcher

Decouples the write commit path from consumer notification. Runs on one dedicated
platform thread to avoid contention.

### Queue

```
MpscArrayQueue<FlushEvent>  capacity = 65,536  (power of 2, JCTools lock-free MPSC)
```

Four WriteWorker threads publish concurrently (multiple producers). One dispatcher thread
consumes (single consumer). The MPSC invariant allows lock-free operation on both sides.

**Queue full behaviour**: `offer()` returns `false`. The event is dropped. This is safe:
long-poll consumers will wake up via their `maxWaitMs` timeout instead. The FlushEvent is
a performance optimisation — dropping one event causes at most one extra timeout wait.

### FlushEvent Record

```java
value record FlushEvent(
    PartitionId partitionId,
    long        baseOffset,    // first offset in this batch (inclusive)
    long        endOffset,     // last offset in this batch (inclusive)
    int         entryCount,    // number of records
    long        timestampMs    // wall-clock time of PG COMMIT
)
```

### Dispatcher Loop

```java
// Runs on dedicated platform thread
void run() {
    while (!shutdown) {
        FlushEvent event = queue.poll();
        if (event == null) {
            LockSupport.parkNanos(1_000_000L);  // 1 ms park when idle
            continue;
        }
        // Action 1: wake long-poll consumers waiting on this partition
        pendingFetchRegistry.notifyWaiters(event.partitionId());

        // Action 2: push to active subscribers (MQTT, AMQP, Kafka push-fetch)
        subscriptionManager.broadcast(event.partitionId(), event.baseOffset(), event.endOffset());

        // Action 3 (cluster mode): notify peer brokers that have subscribers
        clusterNotifier.notifyPeers(event.partitionId(), event.endOffset());
    }
}
```

---

## PendingWrite (value record)

```java
value record PendingWrite(
    TenantId      tenantId,
    PartitionId   partitionId,
    byte[]        key,              // nullable
    byte[]        value,
    byte[]        headers,          // packed: [keyLen:2][key][valLen:4][val]...
    long          timestampMs,
    long          producerId,       // -1 if non-idempotent
    short         producerEpoch,    // -1 if non-idempotent
    int           sequence,         // -1 if non-idempotent
    boolean       isTransactional,
    boolean       isDlq,
    ProtocolId    protocolId
)
```

---

## WriteResult (value record)

```java
value record WriteResult(
    PartitionId partitionId,
    long        baseOffset,
    int         recordCount,
    long        logAppendTimeMs,
    ErrorCode   errorCode      // NONE on success
)
```

---

## DLQ Integration in Write Path

`DlqRouter` is invoked **before** `WriteAccumulator` — the core write path knows nothing
about DLQ routing:

```
DefaultBrokerEngine.write(tenantId, pendingWrites, ctx):
  for each write:
    if DlqRouter.shouldRoute(write, deliveryAttempts):
      dlqWrite = DlqRouter.route(write, reason, retryCount, consumerGroup)
        → inject x-dlq-* headers, set isDlq=true, target DLQ partition
      accumulate(dlqWrite)             ← normal write path
      INSERT INTO dlq_entries (...)    ← metadata sidecar
    else:
      accumulate(write)                ← normal write path
```

---

## Write Path Shutdown

During graceful shutdown (`GracefulShutdown.drain()`):

```
1. suspendTenant(*) on all tenants          → reject new writes
2. WriteAccumulator.drainAll(30s)           → flush all pending batches to WriteWorkers
3. WriteWorker.awaitIdle(30s)               → wait for in-flight PG transactions
4. StorageFlusher.drainAll(30s)             → flush all pending LogSegment writes
5. FlushEventDispatcher.drainAndStop()      → notify remaining waiters, then stop
```

If any step times out, a warning is logged and shutdown proceeds. Outstanding batches
that did not reach PG are nacked to their callers.

---

## Error Handling

| Error | Action |
|-------|--------|
| `WrongEpochException` (Layer 1 — local) | Refresh MetadataImage, forward to new owner via ForwardWriteManager |
| `WrongEpochException` (Layer 2 — PG fence) | Same as above; PG is authoritative |
| `PG connection lost` | Reacquire via HikariCP, retry up to 3× with exponential backoff, then fail batch |
| `PG COMMIT timeout` | Re-read `partition_offsets` to detect if commit succeeded (idempotent check) |
| `INVALID_PRODUCER_EPOCH` | Reject write, return error to client; do not close connection |
| `OUT_OF_ORDER_SEQUENCE` | Reject write, return error to client |
| `DuplicateSequence` (in window) | Return success with original offset; do not re-write |
| `TenantSuspendedException` | Return `TOPIC_AUTHORIZATION_FAILED` to client |
| `MpscArrayQueue full` | FlushEvent dropped; not a write error, only a notification miss |

---

## Batching Configuration

```yaml
broker:
  write:
    batch-size: 1000          # max messages per batch
    batch-bytes: 1048576      # max bytes per batch (1MB)
    linger-ms: 5              # max linger wait before forced flush
    worker-threads: 4         # WriteWorker thread count (= PG connection pool size)
    ack-timeout-ms: 5000      # client-facing timeout before LEADER_NOT_AVAILABLE
    submit-immediate: false   # true = bypass linger for all writes (latency-optimised)
```

---

## Binary COPY Format

Each message row in PG binary COPY format:

```
[field_count: 2 bytes = 9]
[partition_id_len: 4][partition_id: 16 bytes UUID]
[offset_num_len:  4][offset_num:    8 bytes big-endian long]
[tenant_id_len:   4][tenant_id:    16 bytes UUID]
[key_len:         4][key:           N bytes]   (key_len = -1 if NULL)
[value_len:       4][value:         N bytes]
[headers_len:     4][headers:       N bytes]
[timestamp_len:   4][timestamp_ms:  8 bytes big-endian long]
[protocol_id_len: 4][protocol_id:   2 bytes big-endian short]
[is_dlq_len:      4][is_dlq:        1 byte (0 or 1)]
```

Stream is prefixed with the 11-byte PG binary COPY header and suffixed with a 2-byte
trailer (`0xFFFF`). Written via `PGCopyOutputStream` from the pgjdbc driver for
zero-copy streaming into PG (no intermediate buffering in the JVM).

---

## Throughput Targets

| Configuration | Target |
|---------------|--------|
| Single broker, 1 partition, unbatched | ~20K msg/s |
| Single broker, 4 partitions, batched (1K/batch) | ~80K msg/s |
| 3 brokers, 12 partitions | ~200K msg/s |
| Batched COPY (1K msgs/batch) per broker | ~500K msg/s |
| Latency p99 (PG local) | < 10 ms |
| Latency p99 (PG remote, same AZ) | < 30 ms |

Throughput is bounded by PG transaction throughput (~2K TPS per PG core), not network.
Batching amortises transaction overhead across messages, which is why batch size
dominates the throughput number above everything else.

---

## Transformation Pipeline (Pre-Write)

Before messages enter the `WriteAccumulator`, an optional transformation pipeline runs.
Transformations may change the message key, value, or headers — and since the key determines
the partition, **transformation MUST happen BEFORE partition resolution**.

```
Protocol Handler (decode wire → RawMessage)
    ↓
TransformationPipeline (optional, per-tenant config)
    ↓ (may change key → changes partition assignment)
PartitionRouter (key hash → partition selection)
    ↓
HRWRouter (partition → owner broker)
    ↓
WriteAccumulator (standard write path)
```

### Transform Stages (Executed In Order)

| Stage | Description | May Change Key? |
|-------|-------------|-----------------|
| **Schema Validation** | Validate against registered Avro/JSON/Protobuf schema | No |
| **Data Masking** | Mask PII fields (e.g., `email` → `***@***.com`) | No |
| **Field-Level Encryption** | Encrypt sensitive fields with tenant KMS key | No |
| **JSONata Transform** | Apply JSONata expression to restructure payload | **Yes** |
| **Header Injection** | Add metadata headers (source protocol, timestamp, tracing) | No |

### Payload Abstraction (Dual-View)

Messages may arrive as binary (Kafka, MQTT) or JSON (HTTP, PgWire). The transformation
pipeline operates on a unified `Payload` interface:

```java
sealed interface Payload {
    byte[] asBytes();       // for wire encoding (Kafka, MQTT)
    JsonNode asJson();      // for transformation, masking, encryption
    int sizeBytes();        // for quota accounting
}

record BinaryPayload(byte[] data) implements Payload {
    // asJson() parses on first access (lazy)
}

record JsonPayload(JsonNode node) implements Payload {
    // asBytes() serializes on first access (lazy)
}
```

### Critical Ordering Invariant

**Transform BEFORE Resolve.** If a JSONata transform changes the message key, the partition
assignment must reflect the **transformed** key, not the original. Reversing this order would
route messages to the wrong partition.
