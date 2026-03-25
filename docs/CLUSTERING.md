# Clustering Design

## Philosophy

> "PostgreSQL decides who may write. HRW decides who should own. Brokers route accordingly."

No Raft. No Zookeeper. No separate coordinator.
PostgreSQL is already the single source of truth — we leverage its ACID transactions
as a distributed consensus primitive via epoch-fenced CAS updates.

---

## HRW Partition Leadership

### Algorithm

Rendezvous Hashing (HRW) deterministically assigns each partition to one broker.
Every broker independently computes the same result — no coordination required.

```
score(brokerId, partitionId) = HMAC-SHA-256(clusterSecret, partitionId + ":" + brokerId)
leader(partitionId)          = broker with HIGHEST score among ACTIVE brokers
```

**Properties:**
- Deterministic: all brokers independently compute the same leader
- ~1/(N+1) partitions re-assign when a broker joins or leaves
- No central coordinator, no leader election round-trips
- `clusterSecret` prevents external manipulation of ownership

```java
// HRWRouter.java
BrokerId ownerOf(PartitionId partitionId) {
    return metadataImage.activeBrokers()
        .stream()
        .max(Comparator.comparingLong(
            broker -> hmacSha256Score(clusterSecret, partitionId, broker.brokerId())))
        .orElseThrow(NoActiveBrokerException::new);
}

long hmacSha256Score(byte[] secret, PartitionId partitionId, BrokerId brokerId) {
    Mac mac = Mac.getInstance("HmacSHA256");
    mac.init(new SecretKeySpec(secret, "HmacSHA256"));
    mac.update(partitionId.bytes());
    mac.update((byte) ':');
    mac.update(brokerId.bytes());
    byte[] hash = mac.doFinal();
    return Longs.fromBytes(hash[0], hash[1], hash[2], hash[3],
                           hash[4], hash[5], hash[6], hash[7]);
}
```

---

## Broker Lifecycle

```
                    startup
                       │
                  STARTING  (register in broker_registry, claim partitions)
                       │
              heartbeat begins
                       │
                    ACTIVE  ←──────────────────────────────────┐
                       │                                        │
              ┌────────┴────────┐                              │
        graceful drain      heartbeat stops                    │
              │            (crash/network)                     │
           DRAINING             │                              │
              │           HeartbeatMonitor detects stale       │
           SHUTDOWN        BrokerFencingPipeline:              │
                             1. CAS UPDATE status='FENCED'     │
                             2. UPDATE partition_offsets        │
                                  SET leader_id = NULL         │
                             3. MetadataUpdateBroadcast         │
                             4. Other brokers recompute HRW    │
                             5. Winners claim via CAS ─────────┘
```

### Status Values

| Status | Meaning |
|--------|---------|
| `STARTING` | Registering, claiming partitions, warming LogSegment cache |
| `ACTIVE` | Fully operational, accepting writes and reads |
| `DRAINING` | Graceful shutdown: finish in-flight writes, transfer ownership |
| `SHUTDOWN` | Cleanly exited |
| `FENCED` | Missed heartbeats; partitions have been re-elected |

---

## HeartbeatWriter

Runs on a `ScheduledExecutorService` every 3 seconds (configurable).

```sql
UPDATE broker_registry
SET last_heartbeat = now()
WHERE broker_id = :brokerId
  AND status = 'ACTIVE';
```

If the UPDATE affects 0 rows (broker was fenced), trigger `SelfFencedException`:
- Stop accepting new writes
- Flush pending writes
- Close connections
- Exit process (let supervisor restart)

---

## HeartbeatMonitor

Runs every 5 seconds. Checks for stale brokers.

```sql
SELECT broker_id
FROM broker_registry
WHERE status = 'ACTIVE'
  AND last_heartbeat < now() - interval '10 seconds';
```

For each stale broker → triggers `BrokerFencingPipeline`.

**Stale threshold:** 10 seconds (configurable). Should be > 2x heartbeat interval.

---

## BrokerFencingPipeline

Executed by any broker that detects a stale peer.
Multiple brokers may race to fence the same broker — CAS ensures idempotency.

```
1. CAS FENCE (atomic, only one broker wins)
   UPDATE broker_registry
   SET status = 'FENCED'
   WHERE broker_id = :staleBrokerId
     AND status = 'ACTIVE'
     AND last_heartbeat < now() - interval '10 seconds';
   -- 0 rows = someone else already fenced it, stop here

2. RELEASE PARTITIONS (if CAS won)
   UPDATE partition_offsets
   SET leader_id = NULL, leader_epoch = leader_epoch + 1
   WHERE leader_id = :staleBrokerId;

3. BROADCAST (fire and forget)
   MetadataUpdateBroadcast → all active peers via InterBrokerRpc
   (peers also poll PG periodically as fallback)

4. RE-ELECT (run locally — all brokers do the same independently)
   For each partitionId where leader_id = NULL:
     newOwner = HRWRouter.ownerOf(partitionId)
     if newOwner == self:
       UPDATE partition_offsets
       SET leader_id = :selfBrokerId, leader_epoch = :newEpoch
       WHERE partition_id = :partitionId AND leader_id IS NULL;
       -- CAS: only one broker wins the claim

5. UPDATE MetadataImage
   Reload from PG: activeBrokers, ownership, epochs
```

---

## Inter-Broker RPC

A separate Netty server listens on `inter_broker_port` (default: main port + 2).
Each broker maintains persistent outbound connections to all peers.

### Message Types (sealed interface)

```java
sealed interface InterBrokerMessage {

    // Write forwarding
    record ForwardWriteRequest(
        PartitionId partitionId,
        int         leaderEpoch,
        List<PendingWrite> writes
    ) implements InterBrokerMessage {}

    record ForwardWriteResponse(
        long       baseOffset,
        int        recordCount,
        ErrorCode  errorCode
    ) implements InterBrokerMessage {}

    // Read forwarding (non-owner fetching from owner's LogSegment)
    record ForwardFetchRequest(
        PartitionId partitionId,
        long        fetchOffset,
        int         maxBytes
    ) implements InterBrokerMessage {}

    record ForwardFetchResponse(
        List<Record> records,
        long         highWatermark,
        ErrorCode    errorCode
    ) implements InterBrokerMessage {}

    // Metadata sync
    record MetadataUpdateBroadcast(
        Map<PartitionId, BrokerId> ownership,
        Map<PartitionId, Integer>  epochs,
        Set<BrokerId>              activeBrokers,
        long                       version
    ) implements InterBrokerMessage {}

    // Heartbeat (liveness check between peers)
    record HeartbeatPing(BrokerId brokerId, UUID incarnationId) implements InterBrokerMessage {}
    record HeartbeatPong(BrokerId brokerId)                     implements InterBrokerMessage {}
}
```

### RPC Protocol (binary, over Netty)

```
Frame format:
[total_length: 4 bytes]
[message_type: 1 byte]   -- discriminator for sealed interface
[payload: N bytes]       -- message-specific binary encoding
```

### InterBrokerRpcClient

- One persistent Netty connection per peer
- Reconnect with exponential backoff (100ms → 30s)
- Request/response correlation via `requestId` (long)
- Virtual thread offload for write/fetch operations
- Max in-flight requests per connection: 1000

### Write Forwarding Flow

```
non-owner receives produce request:
  1. Load owner from MetadataImage: owner = HRWRouter.ownerOf(partitionId)
  2. If owner == self: process locally (MetadataImage was stale, now caught up)
  3. If owner != self:
       response = InterBrokerRpcClient.send(ForwardWriteRequest(partitionId, epoch, writes))
       if response.errorCode == WRONG_LEADER:
         refresh MetadataImage from PG
         retry once (hop_count < 2 to prevent loops)
       else:
         return response to client
```

---

## MetadataImage

In-memory snapshot of cluster state. Updated atomically via `VarHandle`.

```java
record MetadataImage(
    Map<BrokerId, BrokerInfo>   activeBrokers,   // broker_id → {host, port, inter_broker_port}
    Map<PartitionId, BrokerId>  ownership,       // partitionId → current owner
    Map<PartitionId, Integer>   epochs,          // partitionId → leader_epoch
    long                        version          // monotonically increasing
) {}
```

**Update triggers:**
1. `MetadataUpdateBroadcast` received from peer (fast path, ~1ms)
2. PG polling every 30s (reliable fallback):
   ```sql
   SELECT p.partition_id, po.leader_id, po.leader_epoch
   FROM partition_offsets po
   JOIN partitions p ON po.partition_id = p.partition_id
   WHERE p.tenant_id = :tenantId;
   ```

**Atomic update:**
```java
// VarHandle CAS — no locks
MetadataImage current = imageHolder.get();
if (incoming.version() > current.version()) {
    imageHolder.compareAndSet(current, incoming);
}
```

---

## Cluster Configuration

```yaml
cluster:
  enabled: true
  broker-id: ${BROKER_ID}        # unique UUID, stable across restarts
  cluster-secret: ${CLUSTER_SECRET}  # shared HMAC secret for HRW scoring
  seeds:                         # other brokers for initial metadata load
    - host: broker-1
      inter-broker-port: 9094
    - host: broker-2
      inter-broker-port: 9094
  heartbeat-interval-ms: 3000
  stale-threshold-ms: 10000
  metadata-refresh-ms: 30000
  rpc:
    max-inflight: 1000
    request-timeout-ms: 5000
    reconnect-backoff-ms: 100
    reconnect-max-ms: 30000
```

---

## Cluster Limits and Scaling

| Dimension | Limit | Bottleneck |
|-----------|-------|-----------|
| Max brokers | ~50 | PG connection pool (20 conns × 50 brokers = 1000 PG connections) |
| Max partitions | ~10,000 | MetadataImage memory + HRW computation |
| Write latency (p50) | ~3ms | PG transaction (UPDATE + COPY + COMMIT) |
| Write latency (p99) | ~15ms | PG + network |
| Read latency (L1 hit) | <1ms | LogSegment + OS page cache |
| Read latency (L2 hit) | ~1ms | Inter-broker fetch |
| Read latency (L3 PG) | ~3ms | PG SELECT |
| Partition rebalance on broker join | ~1/(N+1) partitions | HRW re-hash |
| Partition rebalance on broker loss | ~1/(N-1) partitions | HRW re-hash |

---

## No-Raft Rationale

| Feature | Raft approach | Our approach |
|---------|--------------|-------------|
| Leader election | Raft election rounds | HRW: deterministic, no rounds |
| Log replication | ISR + ack quorum | PG COMMIT (PG handles replication) |
| State storage | Raft log | PostgreSQL tables |
| Failure detection | Heartbeat + election | PG heartbeat + CAS fencing |
| Consistency | Raft consensus | PG ACID transactions |
| Complexity | High | Low |

PG already provides ACID. Adding Raft on top would duplicate its guarantees with
more complexity. We use PG as the distributed consensus layer directly.
