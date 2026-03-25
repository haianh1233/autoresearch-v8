# Consumer Group Coordination

> **Related:** [PROTOCOLS.md](PROTOCOLS.md) §Kafka Consumer Groups, [TRANSACTIONS.md](TRANSACTIONS.md),
> [READ_PATH.md](READ_PATH.md) §Push Delivery, [POSTGRES_SCHEMA.md](POSTGRES_SCHEMA.md) §consumer_groups

---

## Overview

Consumer groups provide coordinated, parallel consumption across multiple consumers and multiple
partitions. Ivy implements the Kafka consumer group protocol (classic rebalance) as the unified
coordination mechanism for all protocols.

---

## State Machine

```
EMPTY
  │ (first JoinGroup received)
  ↓
PREPARING_REBALANCE
  │ (all members joined OR rebalance timeout)
  ↓
COMPLETING_REBALANCE
  │ (leader sends SyncGroup with assignments)
  ↓
STABLE
  │ (member leaves, joins, or heartbeat timeout)
  ↓ back to PREPARING_REBALANCE
  │
  │ (all members leave OR group expires)
  ↓
DEAD
```

### State Transitions

| From | To | Trigger |
|------|----|---------|
| EMPTY | PREPARING_REBALANCE | First JoinGroup request |
| PREPARING_REBALANCE | COMPLETING_REBALANCE | All known members have joined, OR `rebalance.timeout.ms` elapsed |
| COMPLETING_REBALANCE | STABLE | Leader sends SyncGroup with partition assignments |
| STABLE | PREPARING_REBALANCE | Member joins, leaves, or heartbeat timeout |
| STABLE | DEAD | All members leave AND offsets expire |
| PREPARING_REBALANCE | EMPTY | All members leave before completion |
| Any | DEAD | Group marked for deletion (admin API) |

---

## Kafka Protocol Integration

### JoinGroup (API Key 11)

```
C→S: JoinGroupRequest(
       groupId, sessionTimeoutMs, rebalanceTimeoutMs,
       memberId,          // empty string on first join
       groupInstanceId,   // nullable (static membership KIP-345)
       protocolType,      // "consumer"
       protocols[{name="range", metadata=<assignment strategy>}])

S→C: JoinGroupResponse(
       generationId,      // monotonically increasing
       protocolName,      // selected strategy (e.g., "range")
       leader,            // memberId of elected leader
       memberId,          // assigned memberId (if first join)
       members[{memberId, metadata}])   // only sent to leader
```

**Leader election:** first member to join becomes leader. Leader receives full member list
and is responsible for computing partition assignments.

### SyncGroup (API Key 14)

```
Leader→S: SyncGroupRequest(
            groupId, generationId, memberId,
            assignments[{memberId, assignment=<partition list>}])

S→All:  SyncGroupResponse(
            assignment=<this member's partition list>)
```

### Heartbeat (API Key 12)

```
C→S: HeartbeatRequest(groupId, generationId, memberId)
S→C: HeartbeatResponse(errorCode)
     errorCode = REBALANCE_IN_PROGRESS → client must rejoin
```

**Session timeout:** if no heartbeat within `session.timeout.ms` (default 45s), member is
removed and rebalance triggered.

### OffsetCommit / OffsetFetch (API Keys 8/9)

Committed offsets stored in `consumer_offsets` table (and `__consumer_offsets` internal topic):

```sql
INSERT INTO consumer_offsets (group_id, partition_id, tenant_id, committed_offset, metadata)
VALUES (:groupId, :partitionId, :tenantId, :offset, :metadata)
ON CONFLICT (group_id, partition_id)
DO UPDATE SET committed_offset = :offset, metadata = :metadata, committed_at = now();
```

---

## Cross-Protocol Consumer Groups

### MQTT 5.0 Shared Subscriptions

```
SUBSCRIBE $share/<group-name>/<topic-filter>
```

Maps to a Kafka-style consumer group:
- `group-name` → `groupId`
- `topic-filter` → resolved partitions
- Messages distributed round-robin across group members
- `MqttRequestHandler.subscribe()` detects `$share/` prefix → delegates to `ConsumerGroupCoordinator`
- MQTT clients do NOT send JoinGroup/SyncGroup — the broker manages group state internally

### AMQP 0-9-1 Competing Consumers

Multiple AMQP consumers on the same queue form an implicit consumer group:
- Queue name → `groupId` (deterministic)
- Each `Basic.Consume` on the queue = member join
- Messages dispatched round-robin (prefetch-count via `Basic.Qos`)
- Consumer cancel (`Basic.Cancel`) = member leave → rebalance

### AMQP 1.0 Competing Receivers

Multiple receiver links with the same source address:
- `source.address` → resolved topic → implicit group
- Link credit (`flow`) controls per-consumer prefetch
- Detach = member leave

### HTTP Long-Poll Consumers

HTTP consumers do NOT participate in consumer groups. Each `GET /topics/{t}/messages?offset=N`
is an independent stateless read. Offset tracking is client-side.

---

## ConsumerGroupCoordinator

```java
class ConsumerGroupCoordinator {
    // Group state: replayed from __consumer_groups internal topic on startup
    ConcurrentHashMap<GroupKey, GroupMetadata> groups;

    record GroupKey(TenantId tenantId, String groupId) {}

    record GroupMetadata(
        String groupId,
        TenantId tenantId,
        GroupState state,
        int generationId,
        String leaderId,
        String protocolName,
        Map<String, MemberMetadata> members,
        Map<String, byte[]> assignments,    // memberId → assignment bytes
        long stateTimestamp
    ) {}
}
```

### FindCoordinator (API Key 10)

Every consumer group is coordinated by a single broker — the **group coordinator**:

```
coordinator(groupId) = HRWRouter.ownerOf(
    UUID.nameUUIDFromBytes(("__consumer_groups:" + tenantId + ":" + groupId).getBytes(UTF_8)))
```

The coordinator partition is derived from the group ID, not the data partitions. This ensures
all group state operations are serialized on a single broker.

### Partition Assignment Strategies

| Strategy | Description |
|----------|-------------|
| `range` | Partition ranges assigned sequentially (Kafka default) |
| `roundrobin` | Partitions distributed round-robin across members |
| `sticky` | Minimize partition movement on rebalance |
| `cooperative-sticky` | Incremental rebalance (KIP-429) — revoke only moved partitions |

Strategy selected by the consumer in JoinGroup; leader picks strategy supported by all members.

---

## Offset Management

### Transactional Offset Commits

When a Kafka consumer is part of a transaction (`read-process-write` pattern):

```
Producer: AddOffsetsToTxn(producerId, groupId)
Producer: TxnOffsetCommit(producerId, groupId, offsets)
          → offsets staged in pendingOffsets (NOT committed yet)
Producer: EndTxn(COMMIT)
          → pendingOffsets flushed to consumer_offsets
          → LSO advances → READ_COMMITTED consumers see committed messages
```

### Offset Expiry

Offsets for empty groups (no active members) expire after `offsets.retention.minutes`
(default 7 days). Expired offsets are deleted from `consumer_offsets` table and
`__consumer_offsets` internal topic (tombstone).

---

## Recovery

On broker restart, `ConsumerGroupCoordinator` replays the `__consumer_groups` internal topic:

```
1. Scan __consumer_groups partition (log-compacted)
2. Rebuild GroupMetadata from latest entries per groupId
3. For each STABLE group: schedule heartbeat timeout checks
4. For each PREPARING_REBALANCE group: resume rebalance timer
5. For each DEAD group: skip (tombstone)
```

Active members re-send JoinGroup after reconnecting to the new coordinator.

---

## Configuration

```yaml
consumer-groups:
  session-timeout-ms: 45000           # heartbeat timeout
  rebalance-timeout-ms: 300000        # max time for all members to rejoin
  heartbeat-interval-ms: 3000         # client heartbeat frequency
  max-poll-interval-ms: 300000        # max time between polls
  offsets-retention-minutes: 10080    # 7 days
  max-groups-per-tenant: 1000
```

---

*Last updated: 2026-03-25*
