# Read Path

## Overview

Reads use a three-tier cache hierarchy to minimize latency while maintaining correctness.
The cache is a performance optimization — all three tiers return the same data.

```
Client
  │  (fetch request: partitionId, fetchOffset, maxBytes)
  ▼
Protocol Handler
  │  decode → FetchRequest
  ▼
DefaultBrokerEngine.fetch(tenantId, partitionId, fetchOffset, maxBytes)
  │
  ▼
ReadAccumulator.fetch(partitionId, fetchOffset, maxBytes)
  │
  ├── L1: LogSegment (owner's in-memory read cache)
  │     hit  → return records immediately (<1ms)
  │     miss ↓
  │
  ├── L2: Owner's cache via InterBrokerRpc (non-owner broker only)
  │     send ForwardFetchRequest to owner
  │     hit  → return records (0.5ms)
  │     miss ↓
  │
  └── L3: PostgreSQL (authoritative)
           SELECT key, value, headers, timestamp_ms, protocol_id
           FROM messages
           WHERE partition_id = ? AND tenant_id = ?
             AND offset_num >= ? AND offset_num < ?
           ORDER BY offset_num
           LIMIT ? (bytes-limited, not count-limited)
           (~2ms on local PG)
  │
  ▼
FetchResult(records[], highWatermark, lastStableOffset)
  │
  ▼
Protocol Handler encodes in wire format → Client
```

---

## L1: LogSegment Cache

The `LogSegment` is a local, append-only binary file maintained by the partition owner.
It is populated **asynchronously after ACK** by `StorageFlusher`.

**Structure:**
```
LogSegment file (e.g. 00000000000000012345.log):
  [size:4][crc:4][offset:8][timestamp:8][key_len:4][key:N][value_len:4][value:N][headers_len:4][headers:N]
  ... repeated ...

OffsetIndex file (e.g. 00000000000000012345.index):
  [relative_offset:4][file_position:4]  -- sparse: every ~4KB of log
  ... repeated ...
  (mmap'd for O(log N) binary search)
```

**Lookup algorithm:**
1. Binary search `OffsetIndex` for largest entry ≤ fetchOffset
2. Seek to file position in `.log` file
3. Scan forward until offset matches or end of segment

**Segment lifecycle:**
- New segment created when current segment exceeds 512MB or 60 minutes
- Old segments are eligible for deletion after retention period (default 7 days)
- Compacted segments keep only the latest value per key (for metadata topics)

---

## L2: Owner Fetch via InterBrokerRpc

When a non-owner broker receives a fetch request, it first checks its own LogSegment
(which may be warm if it was recently the owner). On miss, it fetches from the owner.

```
non-owner broker:
  InterBrokerRpcClient.send(ForwardFetchRequest(partitionId, fetchOffset, maxBytes))
    ↓
owner broker:
  ReadAccumulator.fetch() → L1 LogSegment (always warm on owner)
  → ForwardFetchResponse(records[])
    ↓
non-owner broker:
  (optionally populate own LogSegment cache from response)
  → return to client
```

**Why L2?**
The owner's LogSegment is always warm for recent offsets. This avoids a PG query for the
common case of a consumer reading near the tip of the partition.

---

## L3: PostgreSQL Fetch

Used when LogSegment cache miss on both the receiving broker and the owner.
This happens on:
- Cold start (LogSegment not yet rebuilt from PG)
- Seek to old offsets (beyond LogSegment retention window)
- Consumer group catching up from a large lag

```sql
SELECT
  offset_num, key, value, headers, timestamp_ms, protocol_id
FROM messages
WHERE partition_id  = :partitionId
  AND tenant_id     = :tenantId
  AND offset_num   >= :fetchOffset
ORDER BY offset_num
LIMIT :maxRecords;   -- computed from maxBytes / avg message size
```

**Pagination:** Uses `offset_num` cursor (not SQL OFFSET) for O(log N) index seek.

---

## FetchResult (value record)

```java
value class FetchResult {
    PartitionId    partitionId;
    List<Record>   records;
    long           highWatermark;    // partition_offsets.next_offset - 1
    long           lastStableOffset; // partition_offsets.lso
    FetchErrorCode errorCode;
}

value class Record {
    long   offset;
    long   timestampMs;
    byte[] key;
    byte[] value;
    byte[] headers;
    short  protocolId;
}
```

---

## Push Delivery (Subscribe)

For protocols that use server-push (MQTT, AMQP basic.deliver, Kafka fetch with long-poll):

```
Consumer subscribes:
  BrokerEngine.subscribe(tenantId, partitionId, fetchOffset, listener)
    → SubscriptionRegistry.register(partitionId, ConsumerHandle)

Producer commits (after PG COMMIT):
  FlushEventDispatcher.dispatch(FlushEvent(partitionId, newHighWatermark))
    → notify all registered ConsumerHandles for partitionId
    → each handle: ReadAccumulator.fetch(pendingOffset, ...) → push to client
```

**Inter-broker push notification:**
When the owner commits a batch, it also broadcasts `MetadataUpdateBroadcast` to peers.
Peers with subscribers to the same partition wake up their listeners.

---

## Offset Semantics

| Offset | Meaning |
|--------|---------|
| `EARLIEST` (-2) | Fetch from offset 0 (beginning of partition) |
| `LATEST` (-1) | Fetch from `highWatermark` (new messages only) |
| `TIMESTAMP` (-3) | Binary search `messages.timestamp_ms` for approximate offset |
| `N` (≥0) | Exact offset to fetch from |

---

## Read Configuration

```yaml
broker:
  read:
    max-fetch-bytes: 52428800    # 50MB per fetch response
    max-partition-fetch-bytes: 1048576  # 1MB per partition
    fetch-min-bytes: 1           # minimum bytes before returning
    fetch-max-wait-ms: 500       # max long-poll wait (Kafka-style)
    log-segment-max-bytes: 536870912   # 512MB per segment
    log-segment-max-ms: 3600000        # 60 minutes per segment
    log-retention-ms: 604800000        # 7 days retention
```

---

## SQL Read Path (MySQL / PgWire)

For the read-only SQL protocol adapters, the read path bypasses BrokerEngine
and queries PostgreSQL directly for metadata views:

```
Client (psql / JDBC):
  "SELECT key, value FROM my_topic WHERE offset_num > 100 LIMIT 50"
    ↓
SqlQueryParser.parse(sql) → SqlQuery(type=TOPIC_FETCH, topicName="my_topic", ...)
    ↓
TopicFetchExecutor:
  1. Resolve topicName → topicId + partitionId(s) for tenant
  2. BrokerEngine.fetch(tenantId, partitionId, offset, maxBytes)
  3. Convert FetchResult → ResultSet rows
    ↓
PgWireEncoder / MySqlEncoder → DataRow packets → Client
```

Metadata queries go directly to PG:
```sql
-- "SELECT * FROM broker_registry"
-- "SHOW TABLES" / "SELECT * FROM topics"
-- "SELECT * FROM consumer_groups"
-- "SELECT * FROM partitions"
```
