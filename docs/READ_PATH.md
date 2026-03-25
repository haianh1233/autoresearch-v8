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

### 10-Step Per-Partition Read Pipeline

```
fetchPartition(partitionId, fetchOffset, maxRecords, maxBytes, isolation):

  Step 1.  HWM bound check — if fetchOffset > highWatermark → return EMPTY
  Step 2.  Tier 1: LogSegment lookup (ConcurrentSkipListMap.floorEntry)
  Step 3.  OffsetIndex binary search within segment → file position
  Step 4.  FileChannel.read() from position → raw entries
  Step 5.  HWM re-check — filter entries with offset >= hwm (may have advanced mid-read)
  Step 6.  IsolationFilter (4-gate chain): tenant, control, LSO, aborted ranges
  Step 7.  Record count limit (maxRecords)
  Step 8.  Byte size limit (maxBytes)
  Step 9.  If empty → Tier 3: PG fetchRange + populate Tier 1 (read-through caching)
  Step 10. Assemble ReadResult (entries, hwm, lso, hitTier)
```

```java
class ReadAccumulator {
    final LogSegmentStore logSegments;              // Tier 1
    final PostgresStorageEngine pgStorage;          // Tier 3
    final ComposedIsolationFilter filter;           // 4-gate chain
    final LsoTracker lsoTracker;                    // per-partition LSO
    final ConcurrentHashMap<PartitionId, AtomicLong> hwmMap;

    ReadResult fetchPartition(PartitionId pid, long fetchOffset,
            int maxRecords, int maxBytes, IsolationLevel isolation) {

        long hwm = hwmMap.getOrDefault(pid, new AtomicLong(0)).get();
        if (fetchOffset > hwm) return ReadResult.EMPTY;            // Step 1

        var entries = logSegments.read(pid, fetchOffset, maxRecords); // Steps 2-4

        hwm = hwmMap.get(pid).get();                               // Step 5
        entries = entries.stream().filter(e -> e.offset() < hwm).toList();

        long lso = lsoTracker.getLso(pid);
        entries = filter.apply(entries, pid, tenantId, isolation, lso); // Step 6
        entries = applyLimits(entries, maxRecords, maxBytes);        // Steps 7-8

        if (entries.isEmpty() && fetchOffset < hwm) {              // Step 9
            entries = pgStorage.fetchRange(pid, fetchOffset, maxRecords);
            logSegments.populateCache(pid, entries);  // read-through: Tier 3 → Tier 1
            entries = filter.apply(entries, pid, tenantId, isolation, lso);
            entries = applyLimits(entries, maxRecords, maxBytes);
        }

        return new ReadResult(entries, hwm, lso, computeSize(entries),
            entries.isEmpty() ? Tier.EMPTY : Tier.TIER_1);         // Step 10
    }
}
```

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

### SubscriptionManager

Per-partition registry of active push subscribers with error-isolated broadcast:

```java
class SubscriptionManager {
    // Partition-specific subscriptions
    ConcurrentHashMap<PartitionId, ConcurrentHashMap<String, Subscription>> subscriptions;
    // Global subscriptions (MQTT # wildcard, AMQP fanout)
    ConcurrentHashMap<String, Subscription> globalSubscriptions;

    void subscribe(PartitionId pid, Subscription sub) {
        subscriptions.computeIfAbsent(pid, k -> new ConcurrentHashMap<>())
            .put(sub.subscriberId(), sub);
    }

    void subscribeAll(Subscription sub) {  // MQTT # wildcard
        globalSubscriptions.put(sub.subscriberId(), sub);
    }

    void broadcast(PartitionId pid, List<SegmentEntry> entries) {
        // Partition-specific subscribers
        var map = subscriptions.get(pid);
        if (map != null) {
            for (Subscription sub : map.values()) {
                try { sub.callback().accept(entries); }
                catch (Exception e) { log.warn("Subscriber {} failed", sub.subscriberId(), e); }
            }
        }
        // Global subscribers
        for (Subscription sub : globalSubscriptions.values()) {
            try { sub.callback().accept(entries); }
            catch (Exception e) { log.warn("Global subscriber {} failed", sub.subscriberId(), e); }
        }
    }
}

record Subscription(String subscriberId, Consumer<List<SegmentEntry>> callback) {}
```

**Error isolation:** one subscriber failure does NOT cascade to others — each callback
wrapped in try-catch. Callbacks must be **fast** (enqueue to protocol channel buffer, not
process inline) since they run on the dispatcher thread.

### Two-Path Write-Read Notification

After PG COMMIT, consumers are notified via two complementary paths:

```
Path 1 (FAST, direct, ~0ms):
  WriteWorker.processBatch() calls directly:
    → pendingFetchRegistry.notifyWaiters(partitionId)   // wake PULL consumers
    → subscriptionManager.broadcast(partitionId, entries) // push to MQTT/AMQP

Path 2 (ASYNC, backup, 1-5ms):
  WriteWorker publishes FlushEvent → MpscArrayQueue
    → FlushEventDispatcher drains → notifyWaiters (backup)
    → clusterNotifier.notifyPeers (cluster-wide push)
```

**Why two paths?**
- Path 1: sub-millisecond latency for local consumers (critical for Kafka Streams)
- Path 2: cluster-wide notification + safety net if Path 1 consumer wasn't registered yet
- Path 2 is the ONLY path that notifies **peer brokers**
- `notifyWaiters()` is idempotent — calling twice finds no waiters on second call

### Owner-Notifies Model (Cluster Mode)

Non-owner brokers with local subscribers register interest with the partition owner:

```
SETUP (one-time):
  ivy-3 has MQTT subscriber for "sensor/temp" → owner is ivy-2
  ivy-3 → InterBrokerRpc: InterestAdd(partitionId=sensor/temp:0, brokerId=ivy-3)
  ivy-2 subscription registry: sensor/temp:0 → interested peers: [ivy-3]

RUNTIME (on each write):
  Producer writes to sensor/temp:0 on ivy-2 (owner)
  → PG COMMIT
  → FlushNotification RPC to ivy-3: { partitionId, startOffset, endOffset }

  ivy-3 receives FlushNotification:
  → Fetch from ivy-2's cache (Tier 2, ~0.5ms inter-broker RPC)
  → Populate local cache (read-through)
  → Push to local MQTT subscriber → PUBLISH
  → Total latency: ~1-3ms after COMMIT
```

**Interest lifecycle:**
- `InterestAdd` sent on first local subscriber for a remote partition
- `InterestRemove` sent when last local subscriber disconnects
- Owner crash → re-register with new owner after HRW recomputation

### Cross-Protocol Fan-Out

Multiple protocols can subscribe to the same partition:

```
Kafka producer writes to "events:0" (owner: ivy-1)
  → SubscriptionManager on ivy-1:
      events:0 → [mqtt-sub-1 (MQTT PUBLISH), amqp-consumer-1 (basic.deliver)]
  → broadcast delivers to BOTH:
      mqtt-sub-1:     entries → MqttPublishEncoder → PUBLISH QoS 1
      amqp-consumer-1: entries → AmqpBasicDeliverEncoder → basic.deliver
```

---

## Backfill Integration

When a read misses Tier 1 and falls through to Tier 3 (PG), the `BackfillScheduler` is notified
to proactively populate the gap in Tier 1. See [STORAGE.md](STORAGE.md) §BackfillScheduler.

```
ReadAccumulator Step 9 (Tier 3 fallback):
  records = pgStorage.fetchRange(pid, offset, maxRecords)
  logSegments.populateCache(pid, records)           // immediate read-through
  backfillScheduler.request(pid, offset, hwm)       // schedule wider gap fill
```

The backfill runs asynchronously on virtual threads with semaphore-gated concurrency (max 16
concurrent PG queries). This ensures that subsequent reads for the same offset range hit Tier 1.

---

## Cache Warming After Leadership Transfer

When a partition changes ownership, the new owner's Tier 1 cache is cold:

```
t=0s     New leader claims partition (HRW recompute, CAS claim in PG)
t=0-3s   COLD: all reads hit Tier 3 (PG, ~2-5ms)
t=3s+    New writes → LogSegment populated post-ACK (hot tip warms)
t=10s+   Consumer reads → read-through caching fills gaps in Tier 1
t=60s+   FULLY WARM: all active consumer offset ranges cached (~0.1ms)
```

**Proactive warming** (optional): on leadership claim, prefetch last N records from PG:

```yaml
storage:
  log-segments:
    proactive-warm-on-leadership: true
    warm-ahead-records: 10000
    max-warm-bytes-per-partition: 67108864   # 64 MB cap
```

---

## Read Metrics

```
ivy_read_latency_ns              Histogram   Per-tier latency (tags: tier={segment|owner_rpc|pg})
ivy_read_records_total           Counter     Records read per tier
ivy_read_bytes_total             Counter     Bytes read
ivy_cache_hit_ratio              Gauge       Tier 1 hit rate (0.0-1.0)
ivy_longpoll_wait_ms             Histogram   Long-poll duration (tags: outcome={data|timeout})
ivy_longpoll_active              Gauge       Currently registered long-poll waiters
ivy_push_notification_latency_ms Histogram   PG COMMIT to subscriber delivery
ivy_push_subscribers_active      Gauge       Active push subscribers per partition
ivy_consumer_lag                 Gauge       HWM - committed offset per (group, partition)
ivy_backfill_requests_total      Counter     BackfillScheduler requests
ivy_read_through_cache_fills     Counter     Tier 3 → Tier 1 cache population events
```

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
