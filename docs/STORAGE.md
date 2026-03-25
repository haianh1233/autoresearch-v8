# Storage Design

## Two-Tier Storage

```
                    WRITE PATH
                        │
              ┌─────────▼──────────┐
              │  PostgreSQL        │  ← Source of truth
              │  (ACID, HA-ready)  │    ACK fires after COMMIT
              └─────────┬──────────┘
                        │ async (200ms batching)
              ┌─────────▼──────────┐
              │  LogSegment        │  ← Read performance cache
              │  (local NVMe)      │    Rebuilt from PG on restart
              └────────────────────┘

                    READ PATH
                        │
              ┌─────────▼──────────┐
              │  L1: LogSegment    │  → <1ms (owner, warm)
              └─────────┬──────────┘
                    (miss)
              ┌─────────▼──────────┐
              │  L2: Owner cache   │  → ~1ms (inter-broker RPC)
              │  (inter-broker)    │
              └─────────┬──────────┘
                    (miss)
              ┌─────────▼──────────┐
              │  L3: PostgreSQL    │  → ~3ms (SELECT)
              └────────────────────┘
```

---

## PostgresStorageEngine

### Binary COPY Write Path

Standard parameterized INSERT handles one row per round-trip (~0.1ms per message).
Binary COPY streams an entire batch in one syscall — ~100x faster for bulk writes.

```java
// PgCopyOutputStream wraps a PG COPY FROM STDIN in binary format
try (PGConnection pgConn = dataSource.getConnection().unwrap(PGConnection.class);
     PgCopyOutputStream out = new PgCopyOutputStream(pgConn,
         "COPY messages (partition_id, offset_num, tenant_id, key, value, " +
         "headers, timestamp_ms, protocol_id, is_dlq) FROM STDIN WITH (FORMAT binary)")) {

    out.writeHeader();  // PG binary format header
    for (MessageRecord r : records) {
        out.writeShort(9);                  // field count
        out.writeUUID(r.partitionId());
        out.writeLong(r.offset());
        out.writeUUID(r.tenantId());
        out.writeBytes(r.key());            // null-safe
        out.writeBytes(r.value());
        out.writeBytes(r.headers());
        out.writeLong(r.timestampMs());
        out.writeShort(r.protocolId());
        out.writeBoolean(r.isDlq());
    }
    out.writeTrailer();
}
```

The COPY is always executed within the same transaction as the `partition_offsets` UPDATE,
ensuring atomicity between offset allocation and message persistence.

### fetchRange Query

```sql
SELECT offset_num, key, value, headers, timestamp_ms, protocol_id
FROM messages
WHERE partition_id = :partitionId
  AND tenant_id    = :tenantId       -- defense-in-depth
  AND offset_num  >= :fromOffset
  AND offset_num   < :toOffset
ORDER BY offset_num
LIMIT :maxRows;
```

**Why `tenant_id` in the WHERE clause when `partition_id` already implies it?**
Defense-in-depth. A bug that passes the wrong `tenantId` would expose data across tenants
without this clause. The extra index cost is negligible.

### Schema Management

```java
class SchemaManager {
    // Apply pending migrations on startup
    void migrate() {
        int currentVersion = readCurrentVersion();
        for (Migration m : loadMigrations()) {
            if (m.version() > currentVersion) {
                executeSql(m.sql());
                recordVersion(m.version());
            }
        }
    }
}
```

Migration files in `ivy-storage/src/main/resources/db/migration/`:
```
V1__initial_schema.sql       ← all tables, indexes
V2__add_dlq_entries.sql      ← dlq_entries table
V3__add_acl_entries.sql      ← acl_entries, credentials
...
```

---

## LogSegment

### File Layout

Each partition has a directory of segment pairs:
```
data/
  <tenantId>/<partitionId>/
    00000000000000000000.log    ← messages
    00000000000000000000.index  ← sparse offset index
    00000000000000012345.log    ← next segment (sealed when prev exceeds threshold)
    00000000000000012345.index
    ...
```

File names are the **base offset** of the first message in that segment (20-digit zero-padded).

### .log File Format (per record)

```
offset 0:  [size: 4 bytes]        total record size (excluding this field)
offset 4:  [crc32: 4 bytes]       CRC of everything after this field
offset 8:  [offset: 8 bytes]      absolute offset of this record
offset 16: [timestamp_ms: 8 bytes]
offset 24: [key_length: 4 bytes]  -1 if null
offset 28: [key: key_length bytes]
offset ?:  [value_length: 4 bytes]
offset ?:  [value: value_length bytes]
offset ?:  [headers_length: 4 bytes]
offset ?:  [headers: headers_length bytes]
```

### .index File Format (sparse)

```
offset 0:  [relative_offset: 4 bytes]  offset - baseOffset (saves 4 bytes vs absolute)
offset 4:  [file_position: 4 bytes]    byte offset in .log file

... repeated every ~4KB of log data ...
```

**Lookup:**
1. Binary search `.index` for the largest `relative_offset` ≤ `(fetchOffset - baseOffset)`
2. Seek to `file_position` in `.log`
3. Scan forward until `offset >= fetchOffset`

**Why sparse?**
- Dense index = 12 bytes per message × 1M messages = 12MB (too large for mmap)
- Sparse (1 entry per 4KB) = 12MB log → 1 entry = tiny mmap footprint
- Worst case: scan 4KB of log after index hit (acceptable)

### Segment Lifecycle

```
OPEN (current)
  ↓ (exceeds 512MB or 60 minutes)
SEALED (immutable, read-only)
  ↓ (exceeds retention period, default 7 days)
DELETED
```

`SegmentCleaner` runs every 60 seconds and deletes eligible segments.
Before deleting, verifies the segment's data is persisted in PG (or not needed per retention policy).

### LogSegment vs PG: Correctness Invariant

> A LogSegment record MAY be missing from disk (e.g. after crash).
> A PG record that has been COMMITted is ALWAYS the truth.
> On restart: rebuild LogSegment from PG for recent offsets (last 60 minutes).

**Rebuild on startup:**
```java
// In ClusterManager.start():
for each ownedPartition:
    Offset pgHighWatermark = storageEngine.highWatermark(partitionId)
    Offset segHighWatermark = logSegmentStore.highWatermark(partitionId)
    if (segHighWatermark < pgHighWatermark):
        records = storageEngine.fetchRange(partitionId, segHighWatermark, pgHighWatermark, MAX_BYTES)
        logSegmentStore.append(partitionId, records)
```

---

## OffsetIndex

```java
class OffsetIndex {
    // mmap'd file — OS manages page cache
    MappedByteBuffer indexBuffer;

    // Binary search for largest entry with relativeOffset <= target
    long filePositionFor(long absoluteOffset) {
        long relativeOffset = absoluteOffset - baseOffset;
        int lo = 0, hi = entryCount - 1;
        while (lo < hi) {
            int mid = (lo + hi + 1) / 2;
            if (relativeOffsetAt(mid) <= relativeOffset) lo = mid;
            else hi = mid - 1;
        }
        return filePositionAt(lo);
    }

    // Append entry (called by LogSegment.append() every ~4KB)
    void append(long absoluteOffset, long filePosition) {
        indexBuffer.putInt((int)(absoluteOffset - baseOffset));
        indexBuffer.putInt((int)filePosition);
    }
}
```

---

## StorageFlusher

Background service that drains the post-ACK LogSegment writes.

```java
class StorageFlusher {
    // Per-partition queue of pending log writes
    Map<PartitionId, Queue<List<MessageRecord>>> pendingFlushes;

    // Called by WriteWorker after PG COMMIT
    void schedule(PartitionId partition, List<MessageRecord> records) {
        pendingFlushes.get(partition).add(records);
    }

    // Background thread: runs every 200ms
    void flushAll() {
        for (var entry : pendingFlushes.entrySet()) {
            PartitionId partitionId = entry.getKey();
            Queue<List<MessageRecord>> queue = entry.getValue();
            List<MessageRecord> batch = drain(queue);
            if (!batch.isEmpty()) {
                logSegmentStore.append(partitionId, batch);
            }
        }
    }
}
```

**Why 200ms delay?**
- LogSegment is a cache — small staleness is acceptable
- Batching reduces fsync overhead (one fsync per 200ms vs per batch)
- Readers that need very recent data fall back to PG (L3) until LogSegment catches up

---

## Storage Configuration

```yaml
storage:
  postgresql:
    jdbc-url: jdbc:postgresql://localhost:5432/ivy
    username: ivy
    password: ${IVY_PG_PASSWORD}
    pool:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout-ms: 3000
  log-segments:
    data-dir: /var/lib/ivy/data
    segment-max-bytes: 536870912    # 512MB
    segment-max-ms: 3600000         # 60 minutes
    retention-ms: 604800000         # 7 days
    index-interval-bytes: 4096      # 1 index entry per 4KB
    flush-interval-ms: 200          # StorageFlusher interval
    rebuild-on-startup: true        # rebuild LogSegment cache from PG on startup
```
