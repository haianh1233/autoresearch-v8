# Write Path

## Overview

The write path is **PG-first**: ACK is sent to the client only after PostgreSQL COMMIT.
LogSegment is populated asynchronously after the ACK — it is a read performance cache, not
the source of truth.

```
Client
  │  (produce request, any protocol)
  ▼
Protocol Handler
  │  decode → List<PendingWrite>
  ▼
DefaultBrokerEngine.write(tenantId, pendingWrites, securityContext)
  │
  ├── [partition leader check via HRWRouter]
  │
  ├── IS owner → WriteAccumulator.accumulate()
  │                │
  │                │ (batch full: 1K msgs OR 1MB OR 5ms linger)
  │                ▼
  │             WriteWorker.processBatch()   [4 threads, partition-affined]
  │                │
  │                ├─ validate producer epoch (idempotency)
  │                ├─ BEGIN TRANSACTION
  │                │    UPDATE partition_offsets
  │                │      SET next_offset = next_offset + batch_size
  │                │      WHERE partition_id = ? AND leader_epoch = ?   ← EPOCH FENCE
  │                │      RETURNING next_offset - batch_size AS base;
  │                │    COPY messages FROM STDIN BINARY;
  │                │    UPSERT producer_state;
  │                │    COMMIT;
  │                ├─ ◀── ACK to client fires HERE (after COMMIT)
  │                ├─ async: StorageFlusher → LogSegment (read cache)
  │                └─ async: FlushEventDispatcher → push subscribers
  │
  └── NOT owner → ForwardWriteManager.forward(ownerBrokerId, pendingWrites)
                     │
                     ▼
                  InterBrokerRpcClient.send(ForwardWriteRequest)
                     │  (Netty, persistent connection, backoff reconnect)
                     ▼
                  Owner Broker → same path as above
                     │
                     ▼
                  ForwardWriteResponse(baseOffset, errorCode)
                     │
                     ▼
                  ACK to original client
```

---

## WriteAccumulator

One accumulator per partition. Batches pending writes until a flush trigger fires.

**Flush triggers (whichever comes first):**
- Message count ≥ 1,000
- Accumulated bytes ≥ 1 MB
- Linger time ≥ 5 ms (configurable per topic)

**Thread model:**
- Accumulator is lock-free: producers append via `AtomicReference` CAS
- 4 `WriteWorker` threads pick up batches: `partitionId.hashCode() % 4` for affinity
- One partition is always processed by the same worker (ordering guarantee)

---

## WriteWorker (PG-First Transaction)

```
processBatch(PartitionId, List<PendingWrite>):

1. VALIDATE EPOCH
   - Load current leader_epoch from MetadataImage
   - If batch.epoch != currentEpoch → throw WrongEpochException (forward to new owner)

2. ALLOCATE OFFSETS
   UPDATE partition_offsets
   SET next_offset = next_offset + :batchSize
   WHERE partition_id = :partitionId
     AND leader_epoch = :expectedEpoch
   RETURNING next_offset - :batchSize AS baseOffset;

   If 0 rows → epoch changed under us → WrongEpochException

3. WRITE MESSAGES (binary COPY)
   COPY messages (partition_id, offset_num, tenant_id, key, value,
                  headers, timestamp_ms, protocol_id, is_dlq)
   FROM STDIN WITH (FORMAT binary);

   For each PendingWrite at index i:
     offset_num = baseOffset + i

4. UPSERT PRODUCER STATE (idempotency)
   INSERT INTO producer_state ...
   ON CONFLICT DO UPDATE
   WHERE producer_epoch <= :newEpoch;

5. COMMIT

6. RETURN WriteResult(baseOffset, batchSize, partitionId)

7. [async] StorageFlusher.schedule(partitionId, baseOffset, records)
8. [async] FlushEventDispatcher.dispatch(FlushEvent(partitionId, highWatermark))
```

---

## PendingWrite (value record)

```java
// Java 26 value class — no object header, stack-allocatable
value class PendingWrite {
    TenantId     tenantId;
    PartitionId  partitionId;
    byte[]       key;           // nullable
    byte[]       value;
    byte[]       headers;       // packed: [keyLen:2][key][valLen:4][val]...
    long         timestampMs;
    int          producerId;
    short        producerEpoch;
    int          sequence;
    boolean      isTransactional;
    ProtocolId   protocolId;    // KAFKA=1, AMQP=2, MQTT=3, MYSQL=4, PGWIRE=5
}
```

---

## WriteResult (value record)

```java
value class WriteResult {
    PartitionId partitionId;
    long        baseOffset;
    int         recordCount;
    long        logAppendTimeMs;
    ErrorCode   errorCode;      // NONE on success
}
```

---

## Batching Configuration

```yaml
broker:
  write:
    batch-size: 1000          # max messages per batch
    batch-bytes: 1048576      # max bytes per batch (1MB)
    linger-ms: 5              # max wait time before forced flush
    worker-threads: 4         # WriteWorker thread count
    ack-timeout-ms: 5000      # max time before client timeout
```

---

## Binary COPY Format

Each message row in COPY binary format:
```
[field_count: 2 bytes = 9]
[partition_id_len: 4][partition_id: 16 bytes UUID]
[offset_num_len: 4][offset_num: 8 bytes]
[tenant_id_len: 4][tenant_id: 16 bytes UUID]
[key_len: 4][key: N bytes]        (key_len = -1 if NULL)
[value_len: 4][value: N bytes]
[headers_len: 4][headers: N bytes]
[timestamp_len: 4][timestamp_ms: 8 bytes]
[protocol_id_len: 4][protocol_id: 2 bytes]
[is_dlq_len: 4][is_dlq: 1 byte]
```

Using `PgCopyOutputStream` from the JDBC driver for zero-copy streaming to PG.

---

## DLQ Integration in Write Path

When `DlqRouter.shouldRoute(pendingWrite, deliveryAttempts)` returns true:

```
Original write → DlqRouter.route(original, reason, retryCount, consumerGroup)
  → inject x-dlq-* headers into new PendingWrite
  → set targetPartition = DLQ partition for original topic
  → set is_dlq = true
  → BrokerEngine.write(dlqWrites)      ← normal write path
  → INSERT INTO dlq_entries (...)      ← metadata record
```

DLQ routing happens **before** the write enters `WriteAccumulator` — the write path itself
does not need to know about DLQ logic.

---

## Error Handling

| Error | Action |
|-------|--------|
| `WrongEpochException` | Update MetadataImage, retry via ForwardWriteManager |
| `PG connection lost` | Retry up to 3x with exponential backoff, then fail batch |
| `PG COMMIT timeout` | Re-read `partition_offsets` to check if commit succeeded (idempotent) |
| `ProducerEpochFenced` | Reject write, return `INVALID_PRODUCER_EPOCH` error code |
| `DuplicateSequence` | Return success with original offset (idempotent deduplication) |
| `PartitionOffline` | Return `LEADER_NOT_AVAILABLE` error code |

---

## Throughput Targets

| Configuration | Target |
|---------------|--------|
| Single broker, single partition | ~20K msg/s (PG-limited) |
| Single broker, 4 partitions | ~80K msg/s |
| 3 brokers, 12 partitions | ~200K msg/s |
| Batched COPY (1K msgs/batch) | ~500K msg/s per broker |
| Latency (p99, PG local) | < 10ms |
| Latency (p99, PG remote) | < 30ms |
