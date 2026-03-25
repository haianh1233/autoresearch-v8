# Broker Shutdown and Crash Recovery

> **Related:** [WRITE_PATH.md](WRITE_PATH.md) §Graceful Shutdown, [STORAGE.md](STORAGE.md) §StorageFlusher,
> [CLUSTERING.md](CLUSTERING.md) §Broker Lifecycle, [TRANSACTIONS.md](TRANSACTIONS.md) §Coordinator,
> [RULES.md](RULES.md) §R8a, R11a, R11b

---

## Overview

Two distinct scenarios require careful sequencing:

| Scenario | Trigger | Goal |
|---------|---------|------|
| **Graceful shutdown** | SIGTERM / operator `kill` | Zero data loss; clean segment state; inform clients |
| **Crash recovery** | SIGKILL / OOM / power loss | Detect dirty state; replay to consistency; resume as fast as possible |

The fundamental invariant in both cases:

> **PostgreSQL is the single arbiter of durable state.** A write is only ACK'd to the producer
> after the PG transaction commits. The local `LogSegment` is a read cache. On crash, PG is
> authoritative; the segments are only used to refill the cache and detect unflushed tails.

---

## Table of Contents

1. [Graceful Shutdown (12 Steps)](#1-graceful-shutdown-12-steps)
2. [Crash Recovery (3 Paths, 7 Phases)](#2-crash-recovery-3-paths-7-phases)
3. [Startup Sequence](#3-startup-sequence)
4. [Segment Lifecycle on Shutdown](#4-segment-lifecycle-on-shutdown)
5. [Transaction Handling](#5-transaction-handling)
6. [Consumer Group Recovery](#6-consumer-group-recovery)
7. [Cluster Coordination on Shutdown / Restart](#7-cluster-coordination-on-shutdown--restart)
8. [Durability Guarantees](#8-durability-guarantees)
9. [Known Limitations & Open Issues](#9-known-limitations--open-issues)

---

## 1. Graceful Shutdown (12 Steps)

Triggered by: `SIGTERM`, `BrokerMain.shutdown()`, health-check failure, K8s `preStop` hook.

```
Total maximum time: 130 seconds (sum of all per-step timeouts)
Typical time:        2–10 seconds
```

**Lifecycle state machine:**
```
RUNNING
  │ CAS (Step 1)
  ▼
DRAINING                    ← heartbeat continues, partitions still owned
  │ (Steps 2–11)
  ▼
STOPPING                    ← heartbeat stopped (Step 8 deregisters)
  │ (Step 12)
  ▼
STOPPED
```

Every step is wrapped in `swallow()` — one failure never aborts the sequence:
```java
void swallow(String label, Runnable step) {
    try { step.run(); }
    catch (Throwable t) { log.error("Shutdown step [{}] failed — continuing", label, t); }
}
```

---

### Step 1 — CAS State Transition (< 1 ms)

```java
if (!status.compareAndSet(RUNNING, DRAINING)) return;  // already stopping
```

Prevents double-shutdown. Readiness probe immediately returns `503` after this transition (K8s removes from Service endpoints).

---

### Step 2 — Stop Accepting + Drain Connections (30 s)

```java
// 2a. Stop accepting new TCP connections
serverBootstrap.config().group().shutdownGracefully(0, 0, MILLISECONDS);  // close bind

// 2b. Send protocol-specific going-away signals to active connections
for (Channel ch : activeChannels) {
    sendGoingAway(ch);   // MQTT: DISCONNECT 0x8B, AMQP: Connection.Close(320), Kafka: (none)
}

// 2c. Poll until all channels close (100 ms poll interval, 30 s max)
await(connectionCount::isZero, Duration.ofSeconds(30));

// 2d. Force-close any remaining
activeChannels.forEach(Channel::close);
```

**Protocol-specific going-away signals:**

| Protocol | Signal |
|----------|--------|
| Kafka | None — client detects connection reset on next request |
| MQTT 5.0 | `DISCONNECT(0x8B: Server shutting down)` |
| MQTT 3.1.1 | TCP close (no server-side DISCONNECT in spec) |
| AMQP 0-9-1 | `Connection.Close(reply-code=320, reply-text="connection-forced")` |
| AMQP 1.0 | `close(error={condition="amqp:connection-forced"})` |
| HTTP | `Connection: close` header + drain in-flight responses |

---

### Step 2.5 — DeliveryEngine Drain (10 s)

Active push subscriptions (MQTT persistent sessions, AMQP consumers) hold in-flight messages
whose ack/nack callbacks depend on the write path (Steps 4–7) still being active. They must be
drained **before** the write path shuts down.

```java
deliveryEngine.prepareForShutdown();
// For each unacked in-flight delivery:
//   nack(REQUEUE=true)           → re-enqueues message at current position
//   cancelAckDeadlineTimer()     → prevents spurious timeout closes
// Clear SubscriptionRegistry — no new deliveries after this point
```

Without this step, MQTT/AMQP consumers silently lose messages without notification.

---

### Step 3 — Transaction Coordinator Shutdown (5 s)

```java
transactionCoordinator.prepareForShutdown(Duration.ofSeconds(5));
```

| Transaction State | Action |
|------------------|--------|
| `ONGOING` | Write `ABORT` control record to segment; transition → `ABORTED` |
| `PREPARED` | Preserve — external XA transaction manager will decide fate on next start |
| `COMMITTING` | Wait for completion (up to timeout), then treat as COMMITTED |
| `ABORTING` | Wait for completion (up to timeout) |

**ONGOING aborts advance the LSO** (Last Stable Offset), releasing any pinned READ_COMMITTED consumers.

---

### Step 3.5 — Flush Pending Audit Events (10 s)

```java
auditLogger.flush(Duration.ofSeconds(10));
// Ensures BROKER_DRAINING and any in-flight audit events reach PG before broker exits
```

Without this, the last few audit events (including the shutdown event itself) may be lost.

---

### Step 4 — WriteAccumulator Drain (10 s)

```java
writeAccumulator.shutdown(Duration.ofSeconds(10));
// Set shutdown=true (volatile) — blocks new enqueues
// Flush all pending PartitionBatches to WriteWorkers immediately (bypass linger timer)
// Stop linger scheduler
// Shut down write executor
```

After this step, no new writes are accepted. Producers get `LEADER_NOT_AVAILABLE`.

---

### Step 5 — WriteWorkers Drain (10 s)

```java
writeWorkerPool.awaitTermination(Duration.ofSeconds(10));
// If workers still active: shutdownNow() sends interrupts to each platform thread
```

Each `WriteWorker` runs a `while (!shutdown && !interrupted())` loop (see WRITE_PATH.md §WriteWorker).
After interrupt, each worker finishes its current PG transaction or rolls back, then exits.

---

### Step 6 — Metadata Flush (5 s)

```java
metadataFlusher.flushAll(Duration.ofSeconds(5));
```

Flush all internal metadata topics to their log-compacted segments:

| Internal Topic | Contains |
|----------------|---------|
| `__consumer_offsets` | Committed consumer offsets |
| `__consumer_groups` | Group membership, generation IDs |
| `__transactions` | Transaction state log |
| `__producer_states` | ProducerId epoch/sequence snapshots |
| `__acl_entries` | ACL rules |
| `__quotas` | Quota configurations |
| `__schemas` | Schema registry entries |

These are replayed during crash recovery (Phase 5). If this step is skipped (crash), they
are reconstructed from PG on the next startup.

---

### Step 7 — StorageFlusher Drain (30 s)

```java
storageFlusher.drainAll(Duration.ofSeconds(30));
```

Runs repeated flush cycles until `sealedUnflushedCount() == 0` or timeout expires.
See [STORAGE.md §drainAll](STORAGE.md) for implementation. Rule R8a: `markFlushed()` is called
only after the PG COPY transaction commits.

**If timeout expires:** Remaining unflushed segments stay on disk. Recovery will re-flush the
gap on next startup (segment-max-offset > PG high watermark → reconciliation re-flushes).

---

### Step 8 — Background Thread Shutdown (5 s)

Stop in strict order (later threads depend on earlier ones being stopped):

```
1. storageFlusher.stop()        ← no new flush cycles
2. segmentCleaner.stop()        ← no new deletions (Step 9 handles disposition)
3. metadataCompactor.stop()     ← no compaction mid-disposition
4. flushEventDispatcher.stop()  ← drain remaining flush notifications, then stop
5. diskUsageMonitor.stop()
6. heartbeatWriter.stop()       ← MUST stop before broker deregistration (Step 12)
```

**Critical:** `heartbeatWriter.stop()` must happen here, before deregistration. If heartbeat
continues after deregistration, it re-inserts the broker into `broker_registry` and confuses
partition ownership resolution.

---

### Step 9 — Segment Disposition (10 s)

```java
logSegmentStore.dispose(storageMode);
```

**Engine mode (PG as permanent store):**

| Segment state | Action |
|--------------|--------|
| `FLUSHED` (`flushedOffset >= endOffset`) | Rename to `.deleted`, then delete |
| `UNFLUSHED` (`flushedOffset < endOffset`) | Keep on disk (recovery needs it) |
| Active (currently writing) | Seal + keep |
| Metadata segments | Always keep |

**Standalone mode (segments are the only copy):**

All segments kept. Nothing deleted. `fsync()` all data files and index files.

**Rename-before-delete pattern** prevents `ClosedChannelException` on concurrent readers:
```java
Files.move(logPath, logPath.resolveSibling(logPath.getFileName() + ".deleted"),
           StandardCopyOption.ATOMIC_MOVE);
Files.deleteIfExists(deletedPath);
```

---

### Step 10 — Write Clean Shutdown Marker (< 1 ms)

```java
CleanShutdownMarker.write(dataDir);
```

```
.ivy_cleanshutdown contents:
  version=1
  timestamp=<millis>
  broker_id=<uuid>
```

**Critical:** Both the file and the directory are `fsync`'d:
```java
Files.writeString(markerPath, content, CREATE, TRUNCATE_EXISTING, SYNC);
try (FileChannel dir = FileChannel.open(dataDir, READ)) { dir.force(true); }
```

Presence of this marker on next start = NORMAL recovery (skip CRC scan).
Absence = DIRTY recovery (full CRC32C scan of every segment byte).

---

### Step 11 — Netty Event Loop Shutdown (5 s)

```java
bossGroup.shutdownGracefully(2, 5, SECONDS);   // quiet=2s, timeout=5s
workerGroup.shutdownGracefully(2, 5, SECONDS);
```

Quiet period allows any in-progress Netty pipeline operations to complete before forced teardown.

---

### Step 12 — Infrastructure Shutdown (10 s)

```java
authEngine.stop();           // invalidate caches, zero credential arrays
brokerEngine.stop();         // release subscriptions
hikariPool.close();          // drain PG connection pool
status.set(STOPPED);
terminationLatch.countDown();
```

Deregistration from `broker_registry` happens implicitly when heartbeats stop (Step 8) and
the stale detection threshold (10 s) passes. Other brokers will CAS-claim orphaned partitions.

---

## 2. Crash Recovery (3 Paths, 7 Phases)

### 2.1 Recovery Path Selection

On startup, the broker inspects the data directory:

```
.ivy_cleanshutdown EXISTS  →  NORMAL  (clean shutdown, skip CRC)
.ivy_cleanshutdown MISSING
  + segments on disk       →  DIRTY   (crash, must CRC-scan every byte)
  + no segments            →  FULL    (new node or all segments deleted)
```

```java
RecoveryPath path = CleanShutdownMarker.exists(dataDir) ? NORMAL
                  : hasAnySegments(dataDir)               ? DIRTY
                                                          : FULL;
```

**Timing benchmarks:**

| Dataset | NORMAL | DIRTY |
|---------|--------|-------|
| 10K offsets / 100 segments | < 1 s | ~3 s |
| 100K offsets / 1K segments | < 10 s | ~30 s |
| 1M offsets / 10K segments | < 60 s | ~5 min |

### 2.2 Phase 1 — Configuration

Load `BrokerConfig`, validate required fields, freeze (immutable after this phase).
Fail fast if required values (`BROKER_ID`, `DATA_DIR`, PG connection URL) are missing.

### 2.3 Phase 2 — Segment Recovery

```
Scan dataDir/ for partition subdirectories (UUID names)
  For each partition:
    Open *.log files sorted by base offset
    Register lazy OffsetIndex from *.index files
    Initialize OffsetAllocator with recovered offset range
```

**NORMAL path:** OffsetIndex loaded lazily; CRC skipped.

**DIRTY path — CRC32C validation per entry:**
```java
for (LogEntry entry : segment) {
    long computed = CRC32C.compute(entry.keyBytes(), entry.valueBytes(), entry.headers());
    if (computed != entry.storedCrc()) {
        // Partial write at crash: truncate segment to last valid entry
        segment.truncateTo(entry.offset() - 1);
        break;
    }
}
```

After truncation, rebuild the `OffsetIndex` for the truncated segment. Delete any orphaned
`.deleted` files left over from an interrupted rename-before-delete (Step 9).

**FULL path:** No segments to scan. OffsetAllocator seeded from PG `partition_offsets.next_offset`.

### 2.4 Phase 3 — Core Component Initialization

```
Create ProducerStateManager  (empty; populated in Phase 4)
Create AbortedTransactionTracker  (empty; populated in Phase 4)
Create metadata stores  (empty; populated in Phase 5)
Create BrokerEngine
Create SecureEngine (wraps BrokerEngine)
```

### 2.5 Phase 4 — Transaction Recovery

**Must complete before Phase 5** (metadata replay may advance LSO, which depends on
AbortedRanges being registered first — Rule R11b).

```
For each partition in recovery order:
  psm.recover(partitionId, segments)
    → replay segment trailers
    → rebuild (producerId, epoch) → (lastSequence, lastOffset) sliding window

  Scan segment control records:
    ABORT control record  → abortedRanges.add(AbortedRange(pid, startOffset, controlOffset))
    COMMIT control record → clear openTransactionStart[pid]

  After scan:
    For each pid still in openTransactionStart (orphaned = crashed mid-transaction):
      → treat as aborted: abortedRanges.add(AbortedRange(pid, startOffset, startOffset+1))
```

**Ordering invariant (R11b):** `abortedTxTracker.recordAbort()` must be called **before**
`lsoTracker.onTxnEnd()`. Reversing this creates a window where a READ_COMMITTED fetch sees
aborted records.

**PREPARED transactions:** Left as-is. An external XA transaction manager must resolve them.
The broker resumes accepting XA phase-2 calls (`Commit` or `Abort`) after startup.

### 2.6 Phase 5 — Metadata Replay

Replay internal topics in fixed order. Order matters: consumer offsets before consumer groups,
etc.

```
1.  __consumer_offsets   → ConsumerOffsetStore.replay()
2.  __consumer_groups    → ConsumerGroupStore.replay()
3.  __transactions       → TransactionCoordinator.replay()
4.  __producer_states    → ProducerStateManager.merge(snapshotFromPg)
5.  __acl_entries        → AclStore.replay()
6.  __quotas             → QuotaStore.replay()
7.  __schemas            → SchemaStore.replay()
8.  __configs            → ConfigStore.replay()
9.  __credentials        → CredentialStore.replay()
```

**Segment-PG reconciliation (engine mode):**

```
For each partition:
  segMaxOffset  = max offset in local segments
  pgMaxOffset   = SELECT MAX(offset_num) FROM messages WHERE partition_id = ?

  if segMaxOffset > pgMaxOffset:
    // Segments have unflushed data not yet in PG (crash before StorageFlusher committed)
    gap = readSegmentRange(partitionId, pgMaxOffset + 1, segMaxOffset)
    storageFlusher.flushGap(gap)            // re-COPY gap to PG

  else if pgMaxOffset > segMaxOffset:
    // PG has more data than segments (normal after clean shutdown deleted flushed segments)
    offsetAllocator.set(partitionId, pgMaxOffset + 1)

  else:
    // In sync — nothing to do
```

### 2.7 Phase 6 — Cluster Registration (cluster mode only)

```java
// Upsert with NEW incarnation_id — fences out any zombie from previous incarnation
brokerRegistry.upsert(BrokerRecord(
    brokerId       = config.brokerId(),      // stable across restarts
    incarnationId  = UUIDv7.generate(),      // NEW every restart — zombie prevention
    status         = STARTING,
    advertisedHost = config.advertisedHost(),
    lastHeartbeat  = Instant.now()
));

// Load current cluster state from PG
MetadataImage image = metadataStore.loadImage();

// Claim partitions via HRW recompute + CAS
partitionOwnershipManager.claimAll(image);
// For each partition where HRW says this broker should own it:
//   UPDATE partitions
//   SET leader_id = :selfBrokerId, leader_epoch = leader_epoch + 1
//   WHERE partition_id = :pid AND (leader_id IS NULL OR leader_id = :selfBrokerId)
// If CAS fails (another broker claimed first): skip — log and move on

heartbeatWriter.start();
```

**HRW determinism:** Given the same set of active brokers, HRW always assigns the same partition
to the same broker. After a rolling restart, the restarted broker reclaims exactly its original
partitions without any external coordination.

### 2.8 Phase 7 — Handoff to Runtime

```java
// Start background threads
storageFlusher.start();
segmentCleaner.start();
metadataCompactor.start();

// Delete clean shutdown marker — now RUNNING; next crash = DIRTY recovery
CleanShutdownMarker.delete(dataDir);

// Transition to RUNNING
status.compareAndSet(STARTING, RUNNING);

// Accept connections
listenerRegistry.start();   // bind all protocol ports
```

**Marker deletion happens BEFORE `listenerRegistry.start()`** — if the broker crashes between
these two lines, the next restart correctly enters DIRTY recovery (no marker, segments exist).

---

## 3. Startup Sequence

Normal startup follows the same 7 phases as crash recovery (Phase 2 is NORMAL path — fast).

```
Phase 1: Load BrokerConfig                    (fail fast if invalid)
Phase 2: Connect PG + init schema             (HikariCP, idempotent CREATE TABLE IF NOT EXISTS)
Phase 3: Segment recovery (NORMAL path)       (< 1s for typical datasets)
Phase 4: Core component init                  (empty stores)
Phase 5: Transaction recovery                 (from segment trailers + PG producer_state table)
Phase 6: Metadata replay                      (9 internal topics)
Phase 7: Cluster registration + partition claim
Phase 8: Start background threads + delete marker + bind ports
```

**Startup timeout:** 120 seconds total (K8s `initialDelaySeconds=10`, `failureThreshold=12`,
`periodSeconds=10` → 130s before pod restart).

---

## 4. Segment Lifecycle on Shutdown

```
ACTIVE  ──seal()──►  SEALED  ──flush()──►  FLUSHED  ──dispose()──►  DELETED (engine)
                                                     └──────────────►  KEPT    (standalone)
```

See [STORAGE.md §Segment Lifecycle](STORAGE.md) for the full 5-state FSM.

**Key rule on shutdown (Step 9):**
- `FLUSHED` segments → deleted (data is in PG; segment is redundant cache)
- `SEALED` but unflushed → kept (StorageFlusher will re-COPY on next start)
- `ACTIVE` segment → sealed first, then evaluate
- Metadata segments → always kept

**SegmentCleaner is stopped (Step 8) before disposition (Step 9)** to prevent a race where the
cleaner deletes a segment that Step 9 is still evaluating.

---

## 5. Transaction Handling

### On Graceful Shutdown (Step 3)

| State | Action | Effect |
|-------|--------|--------|
| `ONGOING` | Abort: write ABORT control record | Producers retry; idempotent dedup prevents duplicates |
| `PREPARED` | Preserve | XA transaction manager resolves on next broker contact |
| `COMMITTING` | Wait (up to timeout) | If timeout: treat as committed (control record already written) |
| `ABORTING` | Wait (up to timeout) | If timeout: treat as aborted |

### On Crash Recovery (Phase 4)

- Transactions with COMMIT/ABORT control records: reconstructed correctly
- Transactions without control record (broker crashed mid-transaction): treated as **ABORTED**
- Aborted ranges registered in `AbortedTransactionTracker` → filtered from READ_COMMITTED fetches

**Producer idempotent dedup on reconnect:**

After a crash, the producer retries from its last unacked batch. The broker detects duplicates
using the `(producerId, producerEpoch, sequence)` tuple maintained in `ProducerStateManager`.
Duplicate = return cached offset without re-writing. See [WRITE_PATH.md §Idempotent Producer](WRITE_PATH.md).

---

## 6. Consumer Group Recovery

### What Survives Any Crash

| State | Survived? | Stored In |
|-------|-----------|-----------|
| Committed offsets | ✓ | PG `consumer_offsets` table |
| Topic / partition metadata | ✓ | PG `partitions` table |
| Group member list | ✗ | In-memory only |
| Generation ID | ✗ | Reset to 0 on new coordinator |
| Session timers | ✗ | Recreated on rejoin |

### Recovery Timeline

```
T=0s    Coordinator broker crashes (or graceful shutdown)
T=10s   Members: heartbeat session timeout fires (default session.timeout.ms=10s)
T=10s   Members: FindCoordinator → new coordinator (HRW recompute, deterministic)
T=11s   Members: JoinGroup to new coordinator (all members trigger rebalance)
T=12s   New GroupContext (generation=0), all members included → SyncGroup
T=13s   Members: OffsetFetch → PG returns committed positions
T=13s   Members resume consuming from last committed offset. Zero message loss.
```

**Uncommitted in-flight messages** (fetched but not committed before crash) are re-delivered:
the consumer receives them again from the last committed offset. This is at-least-once semantics.
For exactly-once, use transactions (`isolation.level=read_committed`).

---

## 7. Cluster Coordination on Shutdown / Restart

### Graceful Shutdown (controlled)

```
1. Step 1: Status → DRAINING
2. Step 8: heartbeatWriter.stop()
3. Step 12: BrokerRegistry deregistered (heartbeat stale after 10s)
   → Other brokers detect stale heartbeat via leaseMonitor
   → HRW recompute: orphaned partitions CAS-claimed by new owners
   → New leaders announced via partition_ownership changes
```

**Partition handoff latency:** ~10–13 seconds (stale threshold + HRW recompute + CAS claim + metadata reload). This is the unavailability window for each partition during a rolling restart.

**Future improvement:** Explicit partition release before deregistration:
```sql
-- Run during Step 8, before heartbeat stops
UPDATE partitions
SET leader_id = NULL, leader_epoch = leader_epoch + 1
WHERE leader_id = :selfBrokerId;
-- Triggers immediate re-election by other brokers (<1s vs 10s passive detection)
```

### Crash (uncontrolled)

```
T=0s    Broker crashes (no heartbeat update)
T=10s   Stale threshold exceeded: other brokers detect via leaseMonitor
T=10s   HRW recompute excludes crashed broker
T=10s   Surviving brokers CAS-claim orphaned partitions
T=11s   Producers get LEADER_NOT_AVAILABLE, retry → new leader
T=11s   Consumers redirect to new leader
T=~11s  Full recovery. Producers re-try from last unacked sequence.
```

### Rolling Restart Zero-Downtime

Because HRW is deterministic:
- `broker_id` preserved across restarts → same HRW score
- After restart, broker reclaims exactly the same partitions it owned before
- No permanent rebalancing; temporary ownership (10–13s) goes to other brokers then back

---

## 8. Durability Guarantees

### What is Never Lost

| Data | Guarantee | Mechanism |
|------|-----------|-----------|
| Committed producer messages | No loss | PG commit before ACK; idempotent retry on reconnect |
| Committed consumer offsets | No loss | PG `consumer_offsets` table; replayed on recovery |
| ACLs, quotas, schemas, configs | No loss | PG tables + internal topics; replayed on recovery |
| PREPARED transactions | No loss | Preserved across crashes; XA manager resolves |

### What Can Be Retried (Not Lost, But Re-delivered)

| Data | Behaviour | Mechanism |
|------|-----------|-----------|
| In-flight producer batches (not yet ACK'd) | Producer retries; dedup prevents duplicates | ProducerStateManager sliding window |
| Fetched-but-not-committed consumer messages | Re-delivered from committed offset | Consumer rejoins at last committed position |
| ONGOING transactions at crash | Auto-aborted on recovery; producer retries | Transaction recovery Phase 4 |

### What is Lost on Crash

| Data | Loss Condition | Mitigation |
|------|----------------|-----------|
| Audit events buffered in memory | Crash before auditLogger flush | Step 3.5 on graceful shutdown; accept up to 10s of audit loss on crash |
| `CONTINUE_READ_ONLY` re-auth state | In-memory only | Clients re-authenticate on reconnect |
| Re-auth timer schedule | In-memory only | ReAuthScheduler rebuilt from JWT exp claims on reconnect |

---

## 9. Known Limitations & Open Issues

| # | Issue | Severity | Fix |
|---|-------|----------|-----|
| 1 | **Per-step timeout not enforced** — a hanging step blocks the entire sequence | HIGH | Wrap each step in `CompletableFuture.supplyAsync().orTimeout(stepDuration, MILLISECONDS)` |
| 2 | **Explicit partition release on shutdown** — relies on passive stale detection (10s delay) instead of immediate release | MEDIUM | Add explicit `UPDATE partitions SET leader_id=NULL` before heartbeat stops (Step 8) |
| 3 | **DIRTY recovery CRC32C implementation** — algorithm specified but not yet source-implemented | HIGH | Implement in `LogSegment.validateAndTruncate()` |
| 4 | **Dirty recovery OffsetIndex rebuild** — corrupt/missing `.index` file rebuild not specified | MEDIUM | Scan `.log` file and rebuild sparse index if `.index` is absent or corrupt |
| 5 | **DeliveryEngine draining** — Step 2.5 designed but not yet wired in server bootstrap | HIGH | Wire `deliveryEngine.prepareForShutdown()` before `writeAccumulator.shutdown()` |
| 6 | **HeartbeatWriter explicit stop** — currently stopped implicitly in Step 12 via `brokerEngine.stop()` | MEDIUM | Move heartbeat stop to Step 8, explicitly before `context.close()` |
| 7 | **PREPARED transaction XA resolution** — broker preserves PREPARED state but no XA transaction manager is implemented | LOW | If no XA manager, PREPARED transactions should be aborted after a configurable timeout (default: 10 minutes) |
| 8 | **Audit event flush timing** — Step 3.5 has 10s timeout; events after the cut are lost on both clean shutdown and crash | LOW | For compliance: forward audit stream to PG write-ahead log via CDC (Debezium) |
| 9 | **`SegmentCleaner` stop ordering** — must stop before segment disposition; currently relies on documentation rather than code-level ordering enforcement | LOW | Assert `segmentCleaner.isStopped()` at start of Step 9 |
| 10 | **`ProducerStateSnapshotWriter` binary format** — 46-byte per-entry format specified (partitionId 16B + producerId 8B + epoch 2B + sequence 4B + lastOffset 8B + timestamp 8B) but snapshot reader/writer not yet implemented | MEDIUM | Implement for fast Phase 4 recovery (avoids full segment scan) |

---

## Appendix: Shutdown Step Summary

| Step | Action | Timeout | Data impact |
|------|--------|---------|-------------|
| 1 | CAS RUNNING → DRAINING | — | Readiness probe fails (K8s removes from LB) |
| 2 | Stop accepting + drain connections | 30 s | Clients detect on next request |
| 2.5 | DeliveryEngine drain (nack unacked) | 10 s | Push consumers re-receive on reconnect |
| 3 | Abort ONGOING transactions | 5 s | Producers retry; no data loss |
| 3.5 | Flush audit events | 10 s | <10s of audit events may be lost on crash |
| 4 | WriteAccumulator drain | 10 s | No new writes accepted |
| 5 | WriteWorkers drain | 10 s | All pending batches commit or roll back |
| 6 | Metadata flush | 5 s | Internal topics durable |
| 7 | StorageFlusher drain | 30 s | All segments → PG (or kept for recovery) |
| 8 | Background threads stop | 5 s | Heartbeat stops → cluster detects leave |
| 9 | Segment disposition | 10 s | Flushed segments deleted (engine) or kept (standalone) |
| 10 | Write `.ivy_cleanshutdown` | < 1 ms | Next start: NORMAL recovery |
| 11 | Netty event loop shutdown | 5 s | Pipeline fully stopped |
| 12 | Infrastructure shutdown | 10 s | PG pool closed; status = STOPPED |
| **Total** | | **130 s max** | |

---

## Appendix: Recovery Phase Summary

| Phase | Action | Required before |
|-------|--------|----------------|
| 1 | Load config, connect PG, init schema | Everything |
| 2 | Segment scan (CRC if DIRTY) | Phase 3 |
| 3 | Core component init (empty stores) | Phase 4 |
| 4 | Transaction recovery (AbortedRange **before** LSO) | Phase 5 |
| 5 | Metadata replay (9 internal topics) | Phase 6 |
| 6 | Cluster registration (new `incarnationId`) | Phase 7 |
| 7 | Start threads → delete marker → bind ports | Clients |

---

*Last updated: 2026-03-25*
*See also: [CLUSTERING.md §Broker Lifecycle](CLUSTERING.md), [RULES.md §R8a, R11a, R11b](RULES.md),
[STORAGE.md §StorageFlusher](STORAGE.md), [WRITE_PATH.md §GracefulShutdown](WRITE_PATH.md)*
