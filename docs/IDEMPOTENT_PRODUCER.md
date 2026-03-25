# Idempotent Producer Design

> **Related:** [WRITE_PATH.md](WRITE_PATH.md) §ProducerStateManager, [TRANSACTIONS.md](TRANSACTIONS.md),
> [POSTGRES_SCHEMA.md](POSTGRES_SCHEMA.md) §producer_state

---

## Overview

Idempotent producers guarantee exactly-once delivery per partition by tracking a sliding window
of recent batches per `(producerId, partitionId)`. Duplicate writes return the cached offset
instead of writing again. This is the foundation for Kafka's exactly-once semantics (EOS).

---

## InitProducerId (API Key 22)

Allocates a `producerId` and `producerEpoch` for idempotent/transactional producers.

### Request/Response

```
C→S: InitProducerIdRequest(transactionalId=null, transactionTimeoutMs=60000)
S→C: InitProducerIdResponse(
       errorCode=0,
       producerId=42,            // monotonically increasing BIGINT
       producerEpoch=0)          // 0 for new producer
```

### Epoch Bumping

When a producer restarts with the same `transactionalId`:

```
// First init:   producerId=42, epoch=0
// Restart #1:   producerId=42, epoch=1  (same ID, bumped epoch)
// Restart #2:   producerId=42, epoch=2
```

**Epoch bumping fences zombie producers:** any in-flight writes with `epoch < 2` are rejected
with `INVALID_PRODUCER_EPOCH`. This prevents duplicate writes from stale producer instances.

### ProducerId Allocation

```sql
-- Allocate new producerId (monotonic)
SELECT COALESCE(MAX(producer_id), 0) + 1 FROM producer_state;
```

No reuse — `producerId` is a monotonically increasing BIGINT. Each call to `InitProducerId`
with a new `transactionalId` (or `null` for non-transactional) allocates a fresh ID.

---

## ProducerStateManager (Sliding Window)

### Data Structure

```java
class ProducerStateManager {
    // Per (producerId, partitionId) → state
    ConcurrentHashMap<ProducerKey, ProducerState> states;

    record ProducerKey(PartitionId partitionId, long producerId) {}

    record ProducerState(
        short epoch,
        ArrayList<BatchMetadata> window,      // max 5 entries (FIFO)
        long lastTimestampMs                   // for LRU eviction
    ) {}

    record BatchMetadata(
        int baseSequence,
        int lastSequence,
        long offset,                           // cached offset for duplicate response
        long timestampMs
    ) {}
}
```

### Sequence Validation Algorithm

```
checkSequence(producerId, partitionId, epoch, baseSequence, lastSequence):

  state = states.get(ProducerKey(partitionId, producerId))

  if (state == null):
    // New producer: must start at sequence 0
    if (baseSequence != 0) → OUT_OF_ORDER_SEQUENCE
    accept, create new state with epoch and window[0] = batch

  if (epoch < state.epoch):
    → INVALID_PRODUCER_EPOCH (zombie producer, fenced out)

  if (epoch > state.epoch):
    // Producer restarted with new epoch: reset window
    if (baseSequence != 0) → OUT_OF_ORDER_SEQUENCE
    clear window, set epoch, accept

  if (epoch == state.epoch):
    // Check sliding window for duplicate
    for (BatchMetadata batch : state.window):
      if (batch.baseSequence == baseSequence && batch.lastSequence == lastSequence):
        → DUPLICATE: return cached WriteResult(batch.offset)

    // Check monotonicity
    expectedNext = state.lastSequence() + 1
    if (baseSequence == expectedNext):
      → ACCEPT: add to window, advance lastSequence
    if (baseSequence > expectedNext):
      → OUT_OF_ORDER_SEQUENCE (gap detected)
    if (baseSequence < expectedNext):
      → OUT_OF_ORDER_SEQUENCE (too old, evicted from window)
```

### Sliding Window (Size = 5)

**Why 5?** Kafka allows max 5 in-flight batches per producer (`max.in.flight.requests.per.connection`).
A properly behaving producer never has more than 5 unacknowledged batches.

```java
void addBatch(ProducerState state, BatchMetadata batch) {
    state.window.add(batch);
    if (state.window.size() > WINDOW_SIZE) {
        state.window.remove(0);   // FIFO eviction of oldest
    }
    state.lastTimestampMs = batch.timestampMs;
}
```

### Multi-Record Batches

A single batch may contain multiple records:
- `baseSequence = 100`, `lastSequence = 109` → 10 records
- Next batch must have `baseSequence = 110`
- Sequence arithmetic must handle ranges, not just single values

---

## Epoch Fencing (Zombie Prevention)

### Write Path Integration

In `WriteWorker.processBatch()`, after offset allocation:

```java
// 1. In-memory epoch check (fast)
SequenceCheckResult result = producerStateManager.checkSequence(
    producerId, partitionId, epoch, baseSequence, lastSequence);

switch (result) {
    case ACCEPT   -> { /* proceed with write */ }
    case DUPLICATE -> { future.complete(cachedWriteResult); return; }
    case FENCED   -> { future.completeExceptionally(new InvalidProducerEpochException()); return; }
    case OUT_OF_ORDER -> { future.completeExceptionally(new OutOfOrderSequenceException()); return; }
}

// 2. PG epoch fence (safety net)
// UPSERT producer_state WHERE producer_epoch <= :epoch
// Prevents stale epoch from overwriting newer state
```

### PG Upsert with Epoch Guard

```sql
INSERT INTO producer_state (producer_id, partition_id, producer_epoch, last_sequence, last_offset)
VALUES (:pid, :partId, :epoch, :seq, :offset)
ON CONFLICT (producer_id, partition_id)
DO UPDATE SET
    producer_epoch = :epoch,
    last_sequence  = :seq,
    last_offset    = :offset,
    updated_at     = now()
WHERE producer_state.producer_epoch <= :epoch;   -- never downgrade epoch
```

---

## State Recovery (3 Strategies)

On broker restart, producer state is rebuilt. Strategies tried in priority order:

### Strategy 1: Binary Snapshot (Fastest)

```
File format: MAGIC(4B) + VERSION(2B) + COUNT(4B) + entries(46B each)

Per entry (46 bytes):
  producerId:    8 bytes
  partitionId:  16 bytes (UUID)
  epoch:         2 bytes
  lastSequence:  4 bytes
  lastOffset:    8 bytes
  timestampMs:   8 bytes
```

Written atomically via `.tmp` rename. Skips segment scan entirely.
**Trade-off:** brief blind spot for out-of-window duplicates (5-batch window not fully captured).

### Strategy 2: PostgreSQL (Authoritative)

```sql
SELECT producer_id, producer_epoch, last_sequence, last_offset
FROM producer_state
WHERE partition_id IN (:ownedPartitions);
```

Restored via `ProducerStateManager.restore()` (bypasses validation).

### Strategy 3: Segment Scan (Complete)

Replay all log segment entries in base-offset order:
- Skip non-transactional entries (`producerId <= 0`)
- Track active (uncommitted) transaction producers
- Rebuild full sliding window history
- O(total entries) — slowest path

---

## LRU Eviction

Prevents unbounded memory growth from transient producers:

```java
static final int MAX_TRACKED_PRODUCERS = 100_000;

void maybeEvict() {
    if (states.size() <= MAX_TRACKED_PRODUCERS) return;

    // Single-pass scan for oldest by lastTimestampMs
    ProducerKey oldest = null;
    long oldestTs = Long.MAX_VALUE;
    for (var entry : states.entrySet()) {
        if (entry.getValue().lastTimestampMs < oldestTs) {
            oldestTs = entry.getValue().lastTimestampMs;
            oldest = entry.getKey();
        }
    }
    if (oldest != null) states.remove(oldest);
}
```

**Memory:** ~100 bytes per entry → 100K producers ≈ 10 MB.

Evicted producers that retry get `UNKNOWN_PRODUCER_ID` → client calls `InitProducerId` for fresh state.

---

## Dual-Layer Deduplication

| Layer | Mechanism | Latency | Scope |
|-------|-----------|---------|-------|
| In-memory | Sliding window (5 batches) | ~1 µs | Recent duplicates |
| PostgreSQL | `ON CONFLICT (partition_id, offset_num) DO NOTHING` | ~2 ms | All-time (survives restart) |

Combined = exactly-once semantics across producer retries and broker restarts.

---

## Configuration

```yaml
producer:
  max-tracked-producers: 100000      # LRU eviction threshold
  snapshot-interval-ms: 60000         # binary snapshot write frequency
  snapshot-path: /var/lib/ivy/data/producer-state.snapshot
```

---

*Last updated: 2026-03-25*
