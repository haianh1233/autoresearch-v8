# Storage Design

## Two-Tier Storage

```
                    WRITE PATH
                        │
              ┌─────────▼──────────┐
              │   PostgreSQL        │  ← Source of truth
              │   (ACID, HA-ready)  │    ACK fires after COMMIT
              └─────────┬──────────┘
                        │  async (StorageFlusher, ~200 ms)
              ┌─────────▼──────────┐
              │   LogSegment        │  ← Read performance cache
              │   (local NVMe)      │    Rebuilt from PG on restart
              └────────────────────┘

                    READ PATH
                        │
              ┌─────────▼──────────┐
              │  L1: LogSegment     │  → <1 ms  (owner, warm)
              └─────────┬──────────┘
                   (miss)
              ┌─────────▼──────────┐
              │  L2: Owner cache    │  → ~1 ms  (inter-broker RPC)
              └─────────┬──────────┘
                   (miss)
              ┌─────────▼──────────┐
              │  L3: PostgreSQL     │  → ~2–5 ms (SELECT)
              └────────────────────┘
```

**Correctness invariant**: LogSegment may be absent or incomplete. A PG-committed record is
always authoritative. If LogSegment and PG disagree, PG wins.

---

## LogSegment

### Directory Layout

```
{data-dir}/
  {tenantId}/{partitionId}/
    00000000000000000000.log      ← segment with baseOffset=0
    00000000000000000000.index
    00000000000000012345.log      ← sealed segment, baseOffset=12345
    00000000000000012345.index
    00000000000000098000.log      ← active segment (current writes land here)
    00000000000000098000.index
```

File names are the **base offset** of the first record in the segment, zero-padded to 20
digits. This allows lexicographic sort = chronological sort, and enables O(log N) lookup
of "which segment contains offset X?" via a `ConcurrentSkipListMap<Long baseOffset, LogSegment>`.

### Multi-Segment Registry

```java
class LogSegmentStore {
    // One per partition: sorted map of baseOffset → segment file pair
    ConcurrentHashMap<PartitionId,
        ConcurrentSkipListMap<Long, LogSegment>> segments;

    // Find the segment that may contain `targetOffset`
    // → largest baseOffset ≤ targetOffset
    LogSegment floorSegment(PartitionId pid, long targetOffset) {
        var map = segments.get(pid);
        if (map == null) return null;
        Map.Entry<Long, LogSegment> entry = map.floorEntry(targetOffset);
        return entry == null ? null : entry.getValue();
    }

    // Open a new active segment (called when current exceeds size/time limit)
    LogSegment openNewSegment(PartitionId pid, long baseOffset) { ... }
}
```

Cross-segment reads (consumer spanning multiple segments) step through `floorEntry` +
`higherEntry` until `maxBytes` is satisfied or the active segment is reached.

### .log File Format (per record)

```
offset  0: [size:          4 bytes]  total size of this record (excluding the size field itself)
offset  4: [crc32c:        4 bytes]  CRC32C of all fields from offset 8 onward
offset  8: [offset:        8 bytes]  absolute offset of this record
offset 16: [timestamp_ms:  8 bytes]  wall-clock time in milliseconds
offset 24: [flags:         1 byte]   bit 0 = isControl, bit 1 = isDlq, bit 2 = isTransactional
offset 25: [producer_id:   8 bytes]  -1 if non-idempotent
offset 33: [producer_epoch:2 bytes]  -1 if non-idempotent
offset 35: [key_length:    4 bytes]  -1 if null
offset 39: [key:           key_length bytes]
offset  ?: [value_length:  4 bytes]
offset  ?: [value:         value_length bytes]
offset  ?: [headers_length:4 bytes]
offset  ?: [headers:       headers_length bytes]
```

CRC32C is computed over bytes 8 onward (not over the size or the CRC itself).
On read, if `crc32c(bytes[8..end]) != stored_crc` → skip the record and log a warning
(segment may be partially written after a crash; PG is authoritative).

### .index File Format (sparse)

```
[relative_offset: 4 bytes]  = absoluteOffset - baseOffset
[file_position:   4 bytes]  = byte offset within the .log file
... one entry per ~4 KB of log data ...
```

**Why sparse**: Dense indexing at 12 bytes/record × 1M records = 12 MB just for the index.
Sparse at 4 KB intervals: a 512 MB segment produces at most ~128K index entries = ~1 MB.
Maximum scan cost per lookup: 4 KB forward scan in the .log file (one or two disk pages).

**Pre-allocation**: The `.index` file is pre-allocated to its maximum size
(`segment-max-bytes / index-interval-bytes * 8`) on segment creation to avoid fragmentation
and eliminate `ftruncate` calls during writes. Sealed on close to actual size.

### Warm Section Cache

The first 1,024 index entries (~8 KB) of each `.index` file are copied into a `long[]` heap
array on segment open. Binary searches over recent offsets hit this heap array instead of
triggering OS page faults into the mmap region.

For the active segment (which grows continuously), the warm section covers the most recently
appended records, which are the hottest for consumers at the tip of the partition.

### LazyIndex

Sealed segments' `.index` files are **not** mmap'd at startup. The `MappedByteBuffer` is
created on first access (`synchronized` lazy-init). This reduces startup time significantly
when hundreds of segments exist for a long-running partition.

```java
class OffsetIndex {
    private volatile MappedByteBuffer indexBuffer;  // null until first use

    long filePositionFor(long absoluteOffset) {
        ensureMapped();   // double-checked locking
        // ... binary search
    }

    private synchronized void ensureMapped() {
        if (indexBuffer == null) {
            FileChannel ch = FileChannel.open(indexPath, READ);
            indexBuffer = ch.map(READ_ONLY, 0, ch.size());
        }
    }
}
```

---

## Segment Lifecycle (5 States)

```
ACTIVE
  │  (size > 512 MB OR age > 60 min)
  ↓
SEALED  ──── FileChannel.force(true) called here (fsync)
  │  (StorageFlusher copies to PG and commits)
  ↓
FLUSHED ──── segment.markFlushed() called ONLY AFTER PG COMMIT
  │  (age > retention period, AND no active reader within 30 s)
  ↓
EXPIRED ──── SegmentCleaner marks markedForDeletion = true
  │  (30-second deletion delay elapsed)
  ↓
DELETED ──── Files.delete(logPath); Files.delete(indexPath)
```

### Why FLUSHED Is Separate from SEALED

A SEALED segment has been fsync'd to local disk but may not yet be persisted in PG.
StorageFlusher runs asynchronously every 200 ms. If the broker crashes between SEALED
and FLUSHED, the segment still exists on disk and will be flushed on restart.

**Critical ordering in `StorageFlusher.flush(segment)`:**

```
1.  COPY segment records into PG messages table (binary COPY)
2.  COMMIT the PG transaction
3.  segment.markFlushed()        ← ONLY after confirmed COMMIT, never before

If crash occurs between step 2 and 3:
  - PG has the data (committed)
  - segment.state = SEALED (not FLUSHED)
  - On restart: StorageFlusher re-flushes the segment
  - PG deduplication via PRIMARY KEY (partition_id, offset_num) rejects duplicates silently
  → No data loss, no corruption
```

---

## OffsetIndex

```java
class OffsetIndex {
    final long     baseOffset;        // first offset in this segment
    final Path     indexPath;
    volatile MappedByteBuffer indexBuffer;  // lazily initialized
    final long[]   warmSection;       // heap copy of first 1024 entries
    int            warmSectionSize;   // entries actually populated
    int            entryCount;        // total entries so far

    // O(log N) binary search
    long filePositionFor(long absoluteOffset) {
        long relativeOffset = absoluteOffset - baseOffset;

        // Check warm section first (heap array, no page fault)
        if (warmSectionSize > 0 && relativeOffset <= warmSection[(warmSectionSize-1)*2]) {
            return binarySearchWarm(relativeOffset);
        }

        ensureMapped();
        return binarySearchMmap(relativeOffset);
    }

    // Called during segment.append() — once per ~4 KB of log data
    synchronized void append(long absoluteOffset, int filePosition) {
        ensureMapped();
        indexBuffer.putInt(entryCount * 8,     (int)(absoluteOffset - baseOffset));
        indexBuffer.putInt(entryCount * 8 + 4, filePosition);
        entryCount++;

        if (entryCount <= 1024) {
            warmSection[(entryCount-1)*2]   = absoluteOffset - baseOffset;
            warmSection[(entryCount-1)*2+1] = filePosition;
            warmSectionSize = entryCount;
        }
    }

    // Called when segment is sealed
    void truncateToActualSize() {
        indexBuffer.force();
        FileChannel ch = FileChannel.open(indexPath, WRITE);
        ch.truncate((long) entryCount * 8);
        ch.close();
    }
}
```

---

## StorageFlusher

Background service that copies committed records from WriteWorker queues into LogSegment
files. Runs every 200 ms on a single dedicated thread.

### Thread Model

```java
ScheduledExecutorService scheduler =
    Executors.newSingleThreadScheduledExecutor(
        Thread.ofPlatform().name("storage-flusher").factory());

scheduler.scheduleAtFixedRate(this::flushAll, 0, flushIntervalMs, MILLISECONDS);
```

A platform thread (not virtual) because this thread runs a tight loop and owns a
`ThreadLocal<Connection>` for the PG COPY operations inside `flushAll()`.

### ThreadLocal PG Connection

StorageFlusher performs its own PG COPY operations (separate from WriteWorker).
It uses a dedicated `ThreadLocal<Connection>` — one persistent PG connection for all
COPY-from-segment operations:

```java
ThreadLocal<Connection> pgConn = ThreadLocal.withInitial(dataSource::getConnection);
```

This avoids pool contention with the 4 WriteWorker threads and gives the flusher its own
query pipeline. If the connection drops, it is re-acquired on the next flush cycle.

### Flush Logic

```java
void flushAll() {
    for (PartitionId pid : logSegmentStore.allPartitions()) {
        for (LogSegment seg : logSegmentStore.sealedUnflushed(pid)) {
            try {
                flush(seg);
            } catch (Exception e) {
                log.warn("Flush failed for {}, will retry next cycle", seg.path(), e);
                // segment stays SEALED, retried on next 200 ms cycle
            }
        }
    }
}

void flush(LogSegment seg) throws Exception {
    List<MessageRecord> records = seg.readAll();          // read from .log file
    Connection conn = pgConn.get();
    // 1. COPY records to PG messages table (binary format)
    pgCopyInsert(conn, records);
    // 2. COMMIT
    conn.commit();
    // 3. ONLY NOW mark flushed — after confirmed commit
    seg.markFlushed();
}
```

### drainAll() on Shutdown

Called during graceful shutdown before WriteWorker threads are stopped:

```java
void drainAll(Duration timeout) {
    acceptingSchedules = false;              // stop accepting new schedules
    Instant deadline = Instant.now().plus(timeout);
    while (Instant.now().isBefore(deadline)) {
        flushAll();
        if (logSegmentStore.sealedUnflushedCount() == 0) return;
        Thread.sleep(50);
    }
    log.warn("{} segments still unflushed at shutdown", logSegmentStore.sealedUnflushedCount());
}
```

Unflushed records at shutdown are safe: they are in PG (WriteWorker committed before ACK).
The local LogSegment file will be re-flushed on next startup from PG.

---

## SegmentCleaner

Runs every 60 seconds on a background thread. Manages the FLUSHED → EXPIRED → DELETED
lifecycle and enforces disk budget.

### Disk Budget Enforcement

```
DiskBudgetMonitor.usagePercent(dataDir):
  > 85%  → aggressive mode: delete oldest FLUSHED segments even within retention window
  70–85% → normal mode: delete only FLUSHED segments past retention period
  < 70%  → safe mode: no deletions (disk pressure relieved)
```

**Minimum segment guarantee**: at least 1 SEALED or FLUSHED segment per partition is always
kept, even under disk pressure. This ensures the LogSegment store is never completely empty
(a completely empty L1 forces every read through L3/PG until StorageFlusher rebuilds it).

### Deletion with 30-Second Delay

A FLUSHED, EXPIRED segment is not deleted immediately:

```java
// Phase 1: mark for deletion
void cleanExpired(LogSegment seg) {
    if (seg.state() == FLUSHED && seg.isOlderThan(retentionMs)) {
        seg.markForDeletion(clock.instant());  // state → EXPIRED
    }
}

// Phase 2: actual deletion (next cleaner cycle, if 30 s have elapsed)
void deleteMarked(LogSegment seg) {
    if (seg.state() == EXPIRED &&
        Duration.between(seg.markedForDeletionAt(), clock.instant()).toSeconds() >= 30) {
        Files.deleteIfExists(seg.logPath());
        Files.deleteIfExists(seg.indexPath());
        logSegmentStore.remove(seg.partitionId(), seg.baseOffset());
        seg.setState(DELETED);
    }
}
```

**Why 30 seconds?** A `ReadAccumulator` thread may have called `floorSegment()`, obtained
a reference to this segment, and be mid-read when the cleaner runs. The 30-second window
ensures any in-flight read using the old segment finishes before the file is deleted.
An open `FileChannel` (POSIX) keeps the inode alive past `Files.delete()`, but the explicit
delay makes the timing deterministic and avoids relying on GC finalization of the channel.

### Full Cleaner Cycle

```
1. Seal any ACTIVE segments past max-size or max-age
2. Identify SEALED segments not yet FLUSHED → StorageFlusher handles these
3. For each FLUSHED segment per partition (sorted oldest-first):
   a. Skip if it is the only SEALED/FLUSHED segment for this partition (minimum guarantee)
   b. Skip if disk usage < 70% AND segment is within retention window
   c. Mark EXPIRED (start 30 s timer)
4. Delete any EXPIRED segments whose 30 s timer has elapsed
5. Log metrics: active/sealed/flushed/expired counts per partition
```

---

## PostgresStorageEngine

### Binary COPY Write Path

Standard parameterized INSERT = 1 PG round-trip per row. Binary COPY = entire batch in one
syscall. The difference is ~100× for 1,000-message batches.

```java
// Inside WriteWorker.processBatch()
try (PGCopyOutputStream out = new PGCopyOutputStream(
        connection.unwrap(PGConnection.class),
        "COPY messages (partition_id, offset_num, tenant_id, key, value, " +
        "headers, timestamp_ms, protocol_id, is_dlq) FROM STDIN WITH (FORMAT binary)")) {

    out.writeHeader();  // 11-byte PG binary COPY header
    for (int i = 0; i < records.size(); i++) {
        MessageRecord r = records.get(i);
        out.writeShort(9);                   // field count
        out.writeUUID(r.partitionId());
        out.writeLong(baseOffset + i);       // offset_num
        out.writeUUID(r.tenantId());
        out.writeBytes(r.key());             // null writes -1 as length
        out.writeBytes(r.value());
        out.writeBytes(r.headers());
        out.writeLong(r.timestampMs());
        out.writeShort(r.protocolId().id());
        out.writeBoolean(r.isDlq());
    }
    out.writeTrailer();                      // 2-byte 0xFFFF terminator
}
```

The COPY command is always within the same transaction as the `UPDATE partition_offsets`,
guaranteeing atomicity between offset allocation and record persistence.

### fetchRange Query

```sql
SELECT offset_num, key, value, headers, timestamp_ms, protocol_id, is_dlq,
       producer_id, producer_epoch, flags
FROM   messages
WHERE  partition_id  = :partitionId
  AND  tenant_id     = :tenantId        -- defense-in-depth (never rely on partition_id alone)
  AND  offset_num   >= :fromOffset
  AND  offset_num    < :toOffset
ORDER  BY offset_num
LIMIT  :maxRows;
```

**Why `AND tenant_id = :tenantId`?**
A bug that passes the wrong `TenantId` would expose records across tenants without this
clause. The extra filter cost on an already-selective (`partition_id, offset_num`) index
scan is negligible — it's an additional column check on already-retrieved rows.

### highWatermark Query

```sql
SELECT next_offset FROM partition_offsets WHERE partition_id = :partitionId;
```

Called during startup rebuild and by non-owner brokers to validate their cached HWM.

### Schema Management

```java
class SchemaManager {
    void migrate(DataSource dataSource) {
        int current = readCurrentVersion(dataSource);
        for (Migration m : loadMigrationsFromClasspath("db/migration/")) {
            if (m.version() > current) {
                executeScript(dataSource, m.sql());
                recordVersion(dataSource, m.version());
                log.info("Applied migration V{}", m.version());
            }
        }
    }
}
```

Migration files in `ivy-storage/src/main/resources/db/migration/`:
```
V1__initial_schema.sql        ← messages, partitions, partition_offsets, topics, tenants, ...
V2__add_dlq_entries.sql       ← dlq_entries table
V3__add_acl_entries.sql       ← acl_entries, credentials
V4__add_delegation_tokens.sql ← delegation_tokens table
...
```

Migrations are idempotent within a version (guarded by the `schema_version` table).

---

## Startup Rebuild

On broker startup, for each partition this broker owns, the LogSegment cache is rebuilt
from PG for the most recent records (those that might be missing from disk after a crash):

```java
// In ClusterManager.claimPartitions():
for (PartitionId pid : ownedPartitions) {
    long pgHwm  = storageEngine.highWatermark(pid);
    long segHwm = logSegmentStore.highWatermark(pid);

    if (segHwm < pgHwm) {
        // LogSegment is behind PG — fill the gap
        List<MessageRecord> missing =
            storageEngine.fetchRange(pid, segHwm, pgHwm, MAX_REBUILD_BYTES);
        logSegmentStore.appendBatch(pid, missing);
        log.info("Rebuilt {} records for {} (seg={}, pg={})", missing.size(), pid, segHwm, pgHwm);
    }
}
```

`MAX_REBUILD_BYTES` defaults to 64 MB per partition. If the gap is larger (extended outage),
it is filled lazily via read-through caching as consumers catch up.

---

## Storage Configuration

```yaml
storage:
  postgresql:
    jdbc-url: jdbc:postgresql://localhost:5432/ivy
    username: ivy
    password: ${IVY_PG_PASSWORD}
    pool:
      maximum-pool-size: 6          # 4 WriteWorkers + 1 StorageFlusher + 1 read pool
      minimum-idle: 4
      connection-timeout-ms: 3000
      keepalive-time-ms: 30000

  log-segments:
    data-dir: /var/lib/ivy/data
    segment-max-bytes: 536870912    # 512 MB — seal when exceeded
    segment-max-ms: 3600000         # 60 minutes — seal when exceeded
    retention-ms: 604800000         # 7 days — mark EXPIRED after this age
    delete-retention-ms: 86400000   # 1 day — keep tombstones for compacted topics
    index-interval-bytes: 4096      # one .index entry per 4 KB of .log data
    flush-interval-ms: 200          # StorageFlusher cycle
    rebuild-on-startup: true        # fill LogSegment gap from PG on broker start

  disk-budget:
    reject-threshold-percent: 85    # aggressive eviction above this
    resume-threshold-percent: 70    # stop eviction below this
    cleaner-interval-ms: 60000      # SegmentCleaner cycle
    deletion-delay-ms: 30000        # wait after EXPIRED before DELETED
```

---

## Pluggable Storage Engine SPI (Future)

The current implementation uses PostgreSQL as the sole permanent storage engine.
The architecture supports a pluggable `PermanentStorageEngine` SPI for future extensibility:

```java
sealed interface PermanentStorageEngine {
    void writeBatch(PartitionId partitionId, List<MessageRecord> records);
    List<MessageRecord> fetchRange(PartitionId partitionId, long startOffset, int maxRecords);
    long highWatermark(PartitionId partitionId);
}
```

### Planned Storage Backends

| Backend | Latency | Use Case | Status |
|---------|---------|----------|--------|
| **PostgreSQL** (current) | ~2–5 ms | SQL-queryable, ACID, replication via Patroni/RDS | Implemented |
| **Standalone** (fsync-only) | ~0 ms | Single-node, dev/test, no external dependency | Designed |
| **Kafka** | ~5 ms | Fan-out to upstream Kafka cluster, multi-DC | Designed |
| **FoundationDB** | ~10 ms | Ordered KV, deterministic, multi-region | Designed |
| **S3 + Iceberg** | ~100 ms | Cost-optimized cold storage, analytics via Trino/Athena | Designed |

### Standalone Mode

When no external storage is configured, segments with `fsync=on` are the sole durability layer.
ACK fires after `FileChannel.force(true)` instead of PG COMMIT. This eliminates the PG
dependency entirely but loses SQL queryability and cross-broker data sharing.

### Storage Engine Selection

```yaml
storage:
  engine: postgresql           # postgresql | standalone | kafka | foundationdb | s3-iceberg
  postgresql:
    jdbc-url: jdbc:postgresql://localhost:5432/ivy
    # ... (current config)
```

Only one engine is active per broker. Changing engines requires data migration (export/import
via segment replay).
