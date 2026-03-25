# Read Path

## Overview

Reads use a three-tier cache hierarchy to minimise latency while maintaining correctness.
All three tiers return the same data; the cache is a pure performance optimisation.

```
Client
  │  (fetch request: partitionId, fetchOffset, maxBytes, maxWaitMs)
  ▼
Protocol Handler                                 [Netty event loop — non-blocking]
  │  decode → FetchRequest
  ▼
DefaultBrokerEngine.read(tenantId, partitionId, fetchOffset, maxBytes, isolation)
  │                                              [offloaded to virtual thread]
  ▼
ReadAccumulator.fetch(partitionId, fetchOffset, maxBytes, isolation)
  │
  ├─ HWM pre-check: fetchOffset >= hwm(partitionId) → register long-poll waiter (§Long-Poll)
  │
  ├─ L1: LogSegment (owner's local read cache)
  │     hit  → IsolationFilter → return records  (<1 ms)
  │     miss ↓
  │
  ├─ L2: Owner's cache via InterBrokerRpc  (non-owner broker only)
  │     ForwardFetchRequest to owner → L1 hit on owner (~1 ms)
  │     miss ↓
  │
  └─ L3: PostgreSQL (authoritative)
         SELECT … FROM messages WHERE …   (~2–5 ms)
         → read-through: populate L1 from result before returning
  │
  ▼
IsolationFilter.apply(records, tenantId, isolation, partitionId)
  ▼
FetchResult(records[], highWatermark, lastStableOffset, errorCode)
  ▼
Protocol Handler encodes in wire format → Client
```

---

## Threading Model

```
Netty event loop threads (2 × CPU cores)
  │   Decode FetchRequest, call engine.read() — NEVER blocks.
  │   The call is synchronous only to the point of handing off to a virtual thread.
  ▼
Virtual threads (unbounded, JEP 444 / Java 21+)
  │   One virtual thread per read call. Blocks freely on:
  │     • LogSegment FileChannel.read()  (OS-managed, not the event loop)
  │     • InterBrokerRpc future.get()
  │     • PG JDBC Connection.execute()
  │     • CompletableFuture.join() (long-poll wait)
  │   Virtual threads are cheap — 10K concurrent long-poll waiters = 10K parked carriers.
  ▼
FlushEventDispatcher (1 platform thread)
      Notifies waiting CompletableFutures when new data arrives.
      1 ms park on empty queue — kept on a platform thread to avoid carrier pinning.
```

**Invariant**: Netty event loop threads never call any blocking operation. `engine.read()`
immediately submits to a virtual thread executor and returns a `CompletableFuture<FetchResult>`.

---

## ReadAccumulator

### Multi-Partition Parallel Reads

A Kafka `FetchRequest` can cover dozens of partitions. Reads are executed in parallel:

```java
FetchResult fetch(Map<PartitionId, FetchParams> partitions) {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Map<PartitionId, StructuredTaskScope.Subtask<PartitionFetchResult>> tasks =
            partitions.entrySet().stream().collect(toMap(
                Map.Entry::getKey,
                e -> scope.fork(() -> fetchPartition(e.getKey(), e.getValue()))
            ));

        scope.joinUntil(Instant.now().plusMillis(fetchTimeoutMs));  // default: 5000 ms
        // Partial results are acceptable: return whatever completed
        return assembleResult(tasks);
    }
}
```

Timed-out subtasks return an empty partition response; the client retries on the next poll.

### HWM Invariant (Applied Twice)

The high-water mark can advance while a read is in flight (another batch just committed).
HWM is read both before and after to ensure we don't serve records past it:

```
Before fetch:   hwm1 = hwmTracker.get(partitionId)
Fetch records from L1/L2/L3 up to min(fetchOffset + maxRecords, hwm1)
After fetch:    hwm2 = hwmTracker.get(partitionId)   ← may have advanced
Filter result:  drop any record with offset >= hwm2   ← trim to definitive HWM
Return hwm2 as the FetchResult.highWatermark
```

`hwmTracker` is a `ConcurrentHashMap<PartitionId, AtomicLong>` updated by `FlushEventDispatcher`
on every commit (`accumulateAndGet(newHwm, Math::max)`).

### Read-Through Caching (L3 → L1)

When a cache miss reaches L3 (PostgreSQL), the fetched records are written back to L1:

```java
List<Record> records = pgStorageEngine.fetchRange(partitionId, fetchOffset, fetchOffset + limit);
logSegmentStore.appendBatch(partitionId, records);   // warm the cache for future reads
return records;
```

This means consumers catching up from large lag automatically warm the cache as they read.

---

## L1: LogSegment Cache

The partition owner maintains an append-only binary file per segment.
Full structure defined in `STORAGE.md`.

**Lookup (within one segment):**
1. `LogSegmentStore.floorSegment(partitionId, fetchOffset)` — `ConcurrentSkipListMap.floorEntry(fetchOffset)` → segment with largest `baseOffset ≤ fetchOffset` (O(log N) over segment count)
2. `OffsetIndex.filePositionFor(fetchOffset)` — binary search in sparse mmap'd index (O(log N) over index entries)
3. `FileChannel.read()` starting at file position → scan forward until `record.offset >= fetchOffset`
4. Continue reading until `maxBytes` or `maxRecords` or end of segment
5. If `maxBytes` not satisfied and more segments exist → advance to next segment, repeat from step 1

**Cached segment for non-owner:**
A non-owner broker may have a warm LogSegment from a previous leadership term. `ReadAccumulator`
checks its own L1 before going to L2 — a stale owner's cache is still valid for old offsets.

---

## L2: Owner Fetch via InterBrokerRpc

When the local L1 misses, a non-owner broker fetches from the partition owner:

```
non-owner broker:
  InterBrokerRpcClient.send(ForwardFetchRequest(partitionId, fetchOffset, maxBytes, tenantId))
    ↓  (persistent Netty connection, per-peer)
owner broker:
  ReadAccumulator.fetchPartition(partitionId, fetchOffset, maxBytes)
    → L1 hit (owner is always warm for recent offsets)
    → ForwardFetchResponse(records[], highWatermark)
    ↓
non-owner broker:
  logSegmentStore.appendBatch(partitionId, response.records)  // optionally warm own cache
  return records to client
```

**Why L2?** The owner's LogSegment is always warm for the tip of the partition
(StorageFlusher runs every 200 ms). A non-owner avoids a full PG round-trip for the
common consumer-at-tip case.

---

## L3: PostgreSQL Fetch

Used when both L1 and L2 miss. Typical cases:
- Cold start (LogSegment rebuilding from PG)
- Consumer reading historical offsets (beyond LogSegment retention)
- Large consumer lag (segments already cleaned)

```sql
SELECT offset_num, key, value, headers, timestamp_ms, protocol_id
FROM   messages
WHERE  partition_id  = :partitionId
  AND  tenant_id     = :tenantId        -- defense-in-depth (see STORAGE.md)
  AND  offset_num   >= :fetchOffset
  AND  offset_num    < :fetchOffset + :maxRecords
ORDER BY offset_num
LIMIT  :maxRecords;
```

**Cursor pagination**: uses `offset_num >=` (index range scan), not SQL `OFFSET`, so cost
is O(log N) in the btree, not O(N).

**maxRecords** is computed from `maxBytes` using the average message size estimate (or
a conservative 1 KB/message if no estimate is available). Slightly over-fetching and
discarding is cheaper than issuing a second query.

---

## Long-Poll Mechanism

Kafka and MQTT consumers frequently request records that do not yet exist. Rather than
returning an empty response immediately, the broker parks the request until data arrives
or a timeout fires.

### Components

```
PendingFetchRegistry
  ConcurrentHashMap<PartitionId, CopyOnWriteArrayList<CompletableFuture<Boolean>>>

FlushEventDispatcher (see WRITE_PATH.md)
  Calls pendingFetchRegistry.notifyWaiters(partitionId) on each FlushEvent
```

### Full Flow

```
1.  FetchRequest arrives → ReadAccumulator.fetch() → empty result (partition at HWM)
2.  maxWaitMs > 0 → create CompletableFuture<Boolean> waiter
3.  Register: pendingFetchRegistry.register(partitionId, waiter)
4.  Schedule timeout: scheduler.schedule(() -> waiter.complete(false), maxWaitMs)
5.  Virtual thread parks: boolean newData = waiter.join()
                                         ↑ blocks here until step 7 or timeout

    [In parallel, WriteWorker commits a batch to this partition]

6.  WriteWorker: FlushEventQueue.offer(FlushEvent(partitionId, ...))
7.  FlushEventDispatcher: pendingFetchRegistry.notifyWaiters(partitionId)
      → for each waiter: waiter.complete(true)    // wakes virtual thread
8.  Virtual thread wakes: newData == true
9.  Retry fetch → L1 hit (StorageFlusher ran during the wait period)
10. Return FetchResult with records to client

    [On timeout at step 4]
8'. waiter.complete(false); virtual thread wakes
9'. Retry fetch → still empty → return empty FetchResult (client retries next poll)
```

### Cluster Push Notifications

When the partition owner commits a batch, it notifies peer brokers that have long-poll
waiters on the same partition:

```java
// In FlushEventDispatcher, cluster mode only:
Set<BrokerId> peers = metadataImage.brokersWithSubscribers(event.partitionId());
for (BrokerId peer : peers) {
    rpcClient.send(peer, PushNotification(event.partitionId(), event.endOffset()));
}
```

The peer wakes its local `PendingFetchRegistry` for that partition without a PG round-trip.

---

## IsolationFilter (4-Gate Chain)

Applied to every record fetched from any tier before returning to the caller.

```
Gate 1 — Tenant isolation
  record.tenantId == readingTenantId
  → cross-tenant contamination guard; defense-in-depth beyond partition UUID isolation
  → fail: skip record

Gate 2 — Control record filter
  if (record.isControl):
    only include if isolation == READ_COMMITTED and record is an END_TXN marker
    (clients need this to advance their LSO tracking)
  → fail: skip record

Gate 3a — READ_UNCOMMITTED
  pass all non-control records unconditionally

Gate 3b — READ_COMMITTED only:

  Gate 3b-i — LSO gate
    record.offset < lsoTracker.getLso(partitionId)
    → only return records below the Last Stable Offset
    → records >= LSO are part of an in-progress transaction; not yet visible
    → fail: skip record

  Gate 3b-ii — Aborted transaction gate
    if (record.isTransactional && !record.isControl):
      if (abortedTxTracker.isAborted(partitionId, record.producerId, record.offset)):
        → record is part of an aborted transaction; skip
```

**Applied per-record** during segment scan (not as a post-fetch pass), so aborted records
never enter the result buffer.

---

## Transaction LSO Tracking

### TransactionLsoTracker

```java
class TransactionLsoTracker {
    // partitionId → (txn start offset → set of in-progress producerIds)
    ConcurrentSkipListMap<PartitionId,
        ConcurrentSkipListMap<Long, Set<Long>>> activeTxns;

    // O(1): first key in sorted map = smallest start offset of any active txn
    long getLso(PartitionId pid) {
        var inner = activeTxns.get(pid);
        if (inner == null || inner.isEmpty()) return hwmTracker.get(pid);  // no txns = LSO = HWM
        return inner.firstKey();
    }

    void onTxnBegin(PartitionId pid, long startOffset, long producerId) {
        activeTxns.computeIfAbsent(pid, k -> new ConcurrentSkipListMap<>())
                  .computeIfAbsent(startOffset, k -> ConcurrentHashMap.newKeySet())
                  .add(producerId);
    }

    void onTxnEnd(PartitionId pid, long startOffset, long producerId) {
        var inner = activeTxns.get(pid);
        if (inner == null) return;
        var producers = inner.get(startOffset);
        if (producers != null) {
            producers.remove(producerId);
            if (producers.isEmpty()) inner.remove(startOffset);
        }
    }
}
```

### AbortedTransactionTracker

```java
record AbortedRange(long startOffset, long endOffset, long producerId) {}

class AbortedTransactionTracker {
    // Read-heavy, write-rare: CopyOnWriteArrayList per partition
    ConcurrentHashMap<PartitionId, CopyOnWriteArrayList<AbortedRange>> aborted;

    boolean isAborted(PartitionId pid, long producerId, long offset) {
        List<AbortedRange> ranges = aborted.getOrDefault(pid, List.of());
        for (AbortedRange r : ranges) {
            if (r.producerId() == producerId && offset >= r.startOffset() && offset <= r.endOffset())
                return true;
        }
        return false;
    }

    // For Kafka client-side filtering (FetchResponse.aborted_transactions field)
    List<AbortedRange> getAbortedRanges(PartitionId pid, long fromOffset, long toOffset) {
        return aborted.getOrDefault(pid, List.of()).stream()
            .filter(r -> r.endOffset() >= fromOffset && r.startOffset() <= toOffset)
            .toList();
    }

    void recordAbort(PartitionId pid, AbortedRange range) {
        aborted.computeIfAbsent(pid, k -> new CopyOnWriteArrayList<>()).add(range);
    }
}
```

**Ordering invariant**: `recordAbort()` must be called **before** `onTxnEnd()`. If the
aborted range is registered after LSO advances past it, consumers could miss the aborted
records and serve incomplete data.

---

## Compaction Filter

Applied when `partitionId ∈ compactedPartitions` (topics with `cleanup.policy=compact`).
This includes all internal topics (`__consumer_offsets`, `__credentials`, `__acl_entries`,
`__tenants`) and any user topic configured for compaction.

```java
List<Record> applyCompactionFilter(List<Record> raw, long deleteRetentionMs) {
    // Pass 1: find the latest offset for each key
    HashMap<ByteBuffer, Long> latestOffset = new HashMap<>();
    for (Record r : raw) {
        if (r.key() != null) {
            latestOffset.merge(ByteBuffer.wrap(r.key()), r.offset(), Math::max);
        }
    }

    // Pass 2: keep only latest per key; handle tombstones
    List<Record> result = new ArrayList<>();
    for (Record r : raw) {
        if (r.key() == null) {
            result.add(r);  // null-key records always pass (not compacted)
            continue;
        }
        if (!r.offset().equals(latestOffset.get(ByteBuffer.wrap(r.key())))) {
            continue;  // not the latest value for this key — skip
        }
        if (r.value() == null || r.value().length == 0) {
            // Tombstone: keep if within deleteRetentionMs (catching-up consumers need it)
            // Drop if older than deleteRetentionMs (cleanup complete)
            if (System.currentTimeMillis() - r.timestampMs() < deleteRetentionMs) {
                result.add(r);
            }
        } else {
            result.add(r);
        }
    }
    return result;
}
```

`compactedPartitions` is a `ConcurrentHashMap.newKeySet()` updated when `AlterConfigs`
changes a topic's `cleanup.policy`.

---

## FetchResult (value record)

```java
value record FetchResult(
    PartitionId         partitionId,
    List<Record>        records,
    long                highWatermark,      // partition_offsets.next_offset (inclusive end)
    long                lastStableOffset,   // lowest start offset of any active transaction
    List<AbortedRange>  abortedTransactions,// for READ_COMMITTED client-side filtering
    FetchErrorCode      errorCode
)

value record Record(
    long   offset,
    long   timestampMs,
    byte[] key,           // nullable
    byte[] value,
    byte[] headers,       // packed: [keyLen:2][key][valLen:4][val]...
    short  protocolId,
    boolean isControl     // true for transaction control records
)
```

---

## Push Delivery (Subscribe Mode)

For protocols with server-push delivery (MQTT, AMQP `basic.deliver`, Kafka push-mode):

```
Consumer subscribes:
  BrokerEngine.subscribe(tenantId, partitionId, fromOffset, consumerHandle)
    → SubscriptionRegistry.register(partitionId, consumerHandle)

Producer commits (after PG COMMIT):
  FlushEventDispatcher:
    subscriptionManager.broadcast(partitionId, baseOffset, endOffset)
      → for each ConsumerHandle subscribed to partitionId:
           ReadAccumulator.fetchPartition(partitionId, handle.pendingOffset, maxBytes)
           handle.deliver(fetchResult)   // push records to client
```

For cluster mode: the partition owner's `FlushEventDispatcher` broadcasts a
`PushNotification` RPC to peer brokers. Each peer wakes its local `SubscriptionRegistry`
and pushes from its own read cache.

---

## Offset Semantics

| Offset value | Meaning |
|-------------|---------|
| `EARLIEST` (−2) | Start from offset 0 (beginning of partition) |
| `LATEST` (−1) | Start from `highWatermark` (only new messages) |
| `TIMESTAMP` (−3) | Binary search `messages.timestamp_ms` for closest offset |
| `N` (≥ 0) | Start from exact offset N |

**TIMESTAMP resolution** (`ListOffsets` / MQTT retain-from-time):

```sql
SELECT MIN(offset_num)
FROM   messages
WHERE  partition_id = :partitionId
  AND  tenant_id    = :tenantId
  AND  timestamp_ms >= :targetTimestampMs
LIMIT  1;
```

Uses the `(partition_id, timestamp_ms)` composite index for efficient range scan.

---

## SQL Read Path (MySQL / PgWire)

SQL protocol adapters bypass `BrokerEngine` and translate SQL into `BrokerEngine.read()` calls:

```
Client (psql / JDBC):
  SELECT key, value FROM my_topic WHERE offset_num > 100 LIMIT 50
    ↓
SqlQueryParser.parse(sql) → SqlQuery(type=TOPIC_FETCH, topicName="my_topic", ...)
    ↓
TopicFetchExecutor:
  1. Resolve topicName → (topicId, partitionId[]) for current tenant
  2. BrokerEngine.read(tenantId, partitionId, fetchOffset, maxBytes, READ_COMMITTED)
  3. Convert FetchResult.records → ResultSet rows (one row per record)
    ↓
PgWireRowEncoder / MySqlRowEncoder → DataRow packets → Client
```

Metadata queries (SHOW TABLES, SELECT * FROM broker_registry) go directly to PG and bypass
`BrokerEngine` entirely.

---

## Read Configuration

```yaml
broker:
  read:
    max-fetch-bytes: 52428800          # 50 MB per fetch response
    max-partition-fetch-bytes: 1048576 # 1 MB per partition per fetch
    fetch-min-bytes: 1                 # minimum bytes before returning (Kafka consumer.min.fetch.bytes)
    fetch-max-wait-ms: 500             # max long-poll wait (Kafka-style)
    fetch-timeout-ms: 5000             # StructuredTaskScope per-partition timeout
    log-segment-max-bytes: 536870912   # 512 MB per segment (also in storage config)
    log-segment-max-ms: 3600000        # 60 minutes per segment
    log-retention-ms: 604800000        # 7 days retention
    delete-retention-ms: 86400000      # tombstone retention for compacted topics (1 day)
```
