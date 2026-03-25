# Kafka Streams Compatibility Quirks

> **Related:** [KAFKA_PIPELINING_POSTMORTEM.md](KAFKA_PIPELINING_POSTMORTEM.md),
> [PROTOCOLS.md](PROTOCOLS.md) §Kafka, [CONSUMER_GROUPS.md](CONSUMER_GROUPS.md),
> [TRANSACTIONS.md](TRANSACTIONS.md)

---

## Overview

Kafka Streams is the most demanding client of Ivy's Kafka protocol implementation. It exercises
producer idempotency, transactions, consumer groups, internal topic management, and state store
restoration. Several subtle compatibility issues were discovered through Streams upgrade tests.

---

## NULL Key/Value Semantics

Kafka Streams relies on `null` values as **tombstones** (delete markers) and `null` keys for
certain operations. Ivy must preserve nullness through the entire produce → store → fetch cycle.

### The Problem

```
Produce: key=null, value=null (tombstone)
  → Store: stored as byte[0] (no flag for null)
  → Fetch: returns byte[0]
  → Client: SUPPRESS operator checks value == null → gets byte[0] → crash
```

### The Fix

Bit flags in `TrailerMetadata` (59-byte segment trailer):

```java
static final byte FLAG_NULL_KEY   = 0x08;
static final byte FLAG_NULL_VALUE = 0x04;

// Write: if (key == null) flags |= FLAG_NULL_KEY;
// Read:  byte[] key = trailer.isNullKey() ? null : entry.key();
```

### Why It Matters for Streams

| Streams Feature | Null Semantics |
|----------------|----------------|
| SUPPRESS operator | `value == null` = tombstone → don't emit |
| KTable delete | `value == null` = delete from state store |
| Changelog compaction | `value == null` = tombstone → remove key on compact |
| Left join | `value == null` on right side → emit null join result |

---

## SUPPRESS Operator

SUPPRESS suppresses intermediate/duplicate records within a time window:

```java
stream.suppress(Suppressed.untilTimeLimit(
    Duration.ofMinutes(5),
    Suppressed.BufferConfig.unbounded()));
```

**Interaction with broker:**
- SUPPRESS expects `record.value() == null` for tombstones
- If broker returns `byte[0]` instead of `null`, SUPPRESS tries to deserialize empty bytes
- Result: `BufferUnderflowException` in KafkaAvroDeserializer

---

## Intermediate Topics (Repartition & Changelog)

Kafka Streams auto-creates internal topics via the broker:

| Topic Type | Naming | Partition Count | Cleanup Policy |
|-----------|--------|-----------------|----------------|
| Repartition | `{appId}-{storeName}-repartition` | max(source partitions) | `delete` |
| Changelog | `{appId}-{storeName}-changelog` | = source partitions | **`compact`** |

### Auto-Creation Requirement

The broker MUST auto-create internal topics before the consumer group coordinator assigns
partitions. The KIP-848 `ConsumerGroupHeartbeat` request carries topology metadata:

```java
StreamsGroupHeartbeatRequest {
  topology: {
    subtopologies: [{
      subtopologyId: "0",
      sourceTopics: ["input-topic"],
      repartitionSourceTopics: [{name: "...-repartition", partitions: 3}],
      stateChangelogTopics: [{name: "...-changelog"}]
    }]
  }
}
```

**Handler must:**
1. Extract ALL topic names (source + repartition + changelog)
2. Auto-create internal topics BEFORE calling coordinator
3. Use `repartitionSourceTopics.partitions` for repartition count
4. Use source topic partition count for changelog count

### Subscription Extraction (Bug Fix)

The handler must extract topics from ALL three fields, not just `sourceTopics`:

```java
for (var subtopology : topology.subtopologies()) {
    subscribedTopics.addAll(subtopology.sourceTopics());
    for (var t : subtopology.repartitionSourceTopics())
        subscribedTopics.add(t.name());
    for (var t : subtopology.stateChangelogTopics())
        subscribedTopics.add(t.name());
}
```

---

## KTable Joins Across Processor Generations

### The Problem

```
Generation 1: processor v0 builds min/max state stores (10s runtime)
Generation 2: processor v1 rebalances, restores state from changelog
Generation 3: processor v2 joins maxTable ∩ minTable → dif topic
```

If either state store is not fully restored before the join, `dif` produces wrong results.

### Root Cause

- ALOS (at-least-once) commits state stores every `commit.interval.ms` (1000ms in test)
- With 3 generations running 1-4 seconds each, state may not be committed before shutdown
- Kafka's own default is 30s commit interval

### Test Results

| Mode | Crash | Protocol | Stability |
|------|-------|----------|-----------|
| **EOS** | No | Classic | 100% stable |
| **EOS** | No | Streams (KIP-848) | 100% stable |
| **EOS** | Yes | Classic | 100% stable |
| **EOS** | Yes | Streams | 100% stable |
| ALOS | No | Classic | Mostly stable |
| ALOS | No | Streams | Flaky (state store timing) |
| ALOS | Yes | Classic | Flaky |
| ALOS | Yes | Streams | Flaky |

**All EOS subsets are 100% stable.** ALOS flakiness is inherent to the at-least-once model
under tight timing constraints.

---

## Produce Latency (Linger Timer Issue)

### The Problem

```
Client: ProduceRequest(acks=1, single record)
  → Broker: WriteAccumulator (linger timer = 5ms)
  → Timer fires after 5ms → batch drained → PG write (~3ms)
  → Total: ~8ms per record
  → 11K records × 8ms = ~88s → exceeds 180s test timeout
```

### The Fix

`writeImmediate()` bypasses the linger timer for acks != 0:

```java
CompletableFuture<WriteResult> writeFuture = (acks != 0)
    ? engine.writeImmediate(write)   // bypass linger, flush immediately
    : engine.write(write);           // normal batching path
```

---

## Heartbeat Topology Consistency

### The Problem

KIP-848 heartbeat #1 includes full topology. Heartbeat #2 sends `topology=null` (unchanged).
If broker treats null topology as "no subtopologies", task assignment collapses.

### The Fix

```java
boolean hasTopology = data.topology() != null
    && data.topology().subtopologies() != null
    && !data.topology().subtopologies().isEmpty();

if (hasTopology) {
    // Store topology, build tasks
    storedTopology = data.topology();
} else if (storedTopology != null) {
    // Use previously stored topology (epoch > 0)
}
```

Task assignments must be consistent across heartbeats. Client treats inconsistent assignments
as fatal → never reaches RUNNING state.

---

## State Store Changelog Requirements

For Streams state store restoration to work correctly:

1. **Cleanup policy must be `compact`** — retains latest value per key
2. **Partition count must match source topic** — co-partitioning invariant
3. **Offset tracking must survive generations** — stored in consumer group offset commits

If any of these are violated:
- State store restoration is incomplete
- KTable joins produce wrong results
- SUPPRESS operator misses tombstones

---

## Known Audit Findings (KIP-848)

| ID | Severity | Finding |
|----|----------|---------|
| F-05 | CRITICAL | `subscribedTopicRegex` completely missing from heartbeat handler |
| F-06 | HIGH | No sticky assignment (partition movement not minimized) |
| F-11 | HIGH | No heartbeat session timeout enforcement (dead members not fenced) |
| F-15 | HIGH | No classic-to-KIP-848 migration path |
| F-19 | MEDIUM | Changelog partition count uses `max(sourcePartitions)` without co-partitioning validation |

---

*Last updated: 2026-03-25*
