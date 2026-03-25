# Kafka Pipelining Race Conditions Postmortem

> **Related:** [PROTOCOLS.md](PROTOCOLS.md) §Kafka, [WRITE_PATH.md](WRITE_PATH.md),
> [CONSUMER_GROUPS.md](CONSUMER_GROUPS.md)

---

## Overview

Four distinct bugs discovered through Kafka Streams upgrade tests. All share a root cause:
**stateful mutable fields in `KafkaCodec` are overwritten by pipelined requests before the
dispatch thread reads them.**

---

## Bug #1: Null Key/Value Lost (Streams SUPPRESS Crash)

**Symptom:** `BufferUnderflowException` in Kafka Streams SUPPRESS deserializer.

**Data flow:**
```
Produce: key=null, value=null (tombstone)
  → Store: TrailerMetadata stores byte[0] (no null flag)
  → Fetch: returns byte[0] instead of null
  → Client: SUPPRESS expects null → deserializer crashes
```

**Root cause:** Storage layer had no way to distinguish `null` (tombstone) from `byte[0]` (empty).

**Fix:** Bit flags in TrailerMetadata:

```java
static final byte FLAG_NULL_VALUE = 0x04;
static final byte FLAG_NULL_KEY   = 0x08;

// Write path:
if (record.key() == null)   flags |= FLAG_NULL_KEY;
if (record.value() == null) flags |= FLAG_NULL_VALUE;

// Read path:
byte[] key   = trailer.isNullKey()   ? null : entry.key();
byte[] value = trailer.isNullValue() ? null : entry.value();
```

**Why hard to find:** Symptom in *client* code, not broker. Stack trace points to SUPPRESS operator.
Silent semantic loss — bytes are present but nullness is lost.

---

## Bug #2: ListOffsets Pipelining Race

**Symptom:** Wrong offsets returned to clients during burst metadata fetches.

**The race:**
```
Netty Event Loop (decode thread):
  T=0µs   Decode Request 1 → lastListOffsetsAllTopics = [topic-A]
  T=10µs  Decode Request 2 → lastListOffsetsAllTopics = [topic-B]  ← OVERWRITES

Virtual Thread (dispatch thread):
  T=1ms   Dispatch Request 1 → reads lastListOffsetsAllTopics
           Expected: [topic-A]
           Actual: [topic-B]     ← WRONG
```

**Root cause:** `KafkaCodec` stores decoded data in single mutable field. Pipelined requests
(client sends request N+1 before response N arrives) cause decode to overwrite before dispatch reads.

**Fix:** Queue-based FIFO:
```java
ConcurrentLinkedQueue<List<ListOffsetsTopicPartition>> queue = new ConcurrentLinkedQueue<>();
// Decode: queue.add(parsed)
// Dispatch: queue.poll()
```

---

## Bug #3: Fetch Pipelining Race (Same Pattern)

Three fields must be read atomically but are overwritten independently:

```java
private List<FetchTopicPartitions> lastFetchAllTopics;
private byte lastFetchIsolationLevel;      // READ_COMMITTED vs READ_UNCOMMITTED
private int lastFetchMaxWaitMs;            // long-poll timeout
```

**Fix:** Bundle into atomic record + queue:
```java
record FetchRequestData(List<FetchTopicPartitions> topics, byte isolation, int maxWaitMs) {}
ConcurrentLinkedQueue<FetchRequestData> queue = new ConcurrentLinkedQueue<>();
```

---

## Bug #4: Consumer Group Coordinator State Machine (5 Fixes)

### Fix 1: SyncGroup State Guard

```java
// Before: accepts SyncGroup in ANY state
// After: only valid in COMPLETING_REBALANCE
if (group.state() != COMPLETING_REBALANCE) return error(REBALANCE_IN_PROGRESS);
```

### Fix 2: Member Existence Check

```java
// Member could be evicted via session timeout between request arrival and dispatch
if (!group.hasMember(memberId)) return error(UNKNOWN_MEMBER_ID);
```

### Fix 3: Fail Pending Futures on Rebalance

```java
// Before: joinFutures leaked (never completed) → threads hang forever
// After: complete with REBALANCE_IN_PROGRESS error
group.completeJoinFuturesWith(REBALANCE_IN_PROGRESS);
group.completeSyncFuturesWith(REBALANCE_IN_PROGRESS);
```

### Fix 4: Remove Stale Members During PREPARING_REBALANCE

```java
removeNonJoinedMembers();
if (allMembersJoined()) completeJoinBarrier();   // don't wait for dead members
```

### Fix 5: rebalanceTimeoutMs Propagation

```java
// Before: hardcoded 10s (session timeout)
// After: use client's rebalanceTimeoutMs (300s for Kafka Streams)
group.setRebalanceTimeoutMs(req.rebalanceTimeoutMs());
```

---

## Netty Pipelining Architecture (Why Races Happen)

```
TCP Connection (client sends Request 1, Request 2 before waiting)
  ↓
Netty EventLoop (single-threaded decode):
  T=0µs   KafkaCodec.decode(frame1) → writes lastXxx fields
  T=10µs  KafkaCodec.decode(frame2) → OVERWRITES lastXxx fields
  ↓
Virtual Thread Pool (dispatch, may be delayed):
  T=1ms   dispatch(request1) → reads lastXxx → gets request2's data  ← BUG
```

**Key insight:** Decode thread and dispatch thread are different threads. When client pipelines,
decode runs twice before dispatch runs once.

---

## Architectural Debt: ~50 Vulnerable Fields

```java
// All of these are pipelining-vulnerable:
private List<ListOffsetsTopicPartition> lastListOffsetsAllTopics;
private List<FetchTopicPartitions> lastFetchAllTopics;
private byte lastFetchIsolationLevel;
private int lastFetchMaxWaitMs;
private JoinGroupRequestData lastJoinGroupData;
// ... ~45 more lastXxx fields ...
```

**Proper fix: RequestContext abstraction** (not yet implemented):
```java
// Codec becomes stateless — each request carries its own data
record RequestContext(RequestHeader header, AbstractRequest request) {}

CompletableFuture<ByteBuf> decode(ByteBuf in) {
    var ctx = new RequestContext(parseHeader(in), parseRequest(in));
    return dispatcher.dispatch(ctx);   // immutable, no shared mutable state
}
```

---

## Kafka Streams Compatibility Summary

| Test Subset | EOS (Exactly-Once) | ALOS (At-Least-Once) |
|------------|--------------------|--------------------|
| No crash | 100% stable | Flaky (state store timing) |
| With crash | 100% stable | Flaky (state store timing) |
| Classic protocol | 100% stable | Mostly stable |
| Streams protocol (KIP-848) | 100% stable | Flaky |

**All EOS subsets pass 3/3 runs.** ALOS flakiness caused by KTable join requiring both state
stores fully restored within tight timing windows.

---

*Last updated: 2026-03-25*
