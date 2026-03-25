# Protocol Design

## Overview

All five protocols share the same underlying broker engine, storage, and clustering.
They differ only in wire encoding and protocol-specific semantics.

```
Protocol Wire Format → Protocol Handler → BrokerEngine (shared)
                    ← Protocol Handler ← WriteResult / FetchResult
```

**Unified message model:**
- Everything is a (key, value, headers, timestamp) record stored in a partition
- Partition = unit of ordering and parallelism
- Consumer group = stateful cursor over one or more partitions
- Topic = named collection of N partitions

---

## Protocol ID Registry (append-only)

| ID | Protocol | Notes |
|----|----------|-------|
| 1  | Kafka | Primary protocol |
| 2  | AMQP 0-9-1 | Exchange/queue model |
| 3  | MQTT 3.1.1 | Pub/sub, IoT |
| 4  | MySQL wire | Read-only SQL view |
| 5  | PgWire | Read-only SQL view |

Protocol IDs are stored in the `messages.protocol_id` column and segment trailers.
They are **append-only** — never reuse or reorder an ID.

---

## Protocol Detection (magic bytes)

`ProtocolDetector` peeks at the first 4 bytes without consuming them:

```
Bytes 0-3          → Protocol
─────────────────────────────────────────────────────────
0x00 0x?? ...      → Kafka (MSB-first 4-byte request length, starts near 0)
0x10 - 0xF0        → MQTT  (first byte = control packet type; CONNECT = 0x10)
0x41 0x4D 0x51 0x50 → AMQP  ("AMQP" ASCII magic)
<len:3><seq:1>0x0A → MySQL (handshake packet, length-prefixed, type=0x0A for server greeting)
                            Note: detect by 4-byte frame with byte[4]=0x0A
0x00 0x00 0x00 ??  → PgWire (first 4 bytes = message length, next 4 = 196608 = protocol 3.0)
                            Disambiguate from Kafka: byte[4..7] = 0x00 0x03 0x00 0x00
```

**Netty pipeline installation:**
```
ProtocolDetector peeks 8 bytes →
  match → remove self → install protocol-specific codec → fire channelRead
  no match → send error and close
```

---

## Kafka Protocol (port 9092 / 9093 TLS)

### Implemented Request Types

| API Key | Request | Notes |
|---------|---------|-------|
| 0 | Produce | v3-v9, idempotent, transactional |
| 1 | Fetch | v4-v15, follower fetching |
| 2 | ListOffsets | v1-v7 |
| 3 | Metadata | v1-v12, cluster awareness |
| 8 | OffsetCommit | v0-v8 |
| 9 | OffsetFetch | v0-v8 |
| 10 | FindCoordinator | group + transaction |
| 11 | JoinGroup | v0-v9 (classic + KIP-848) |
| 12 | Heartbeat | v0-v4 |
| 13 | LeaveGroup | v0-v5 |
| 14 | SyncGroup | v0-v5 |
| 15 | DescribeGroups | v0-v5 |
| 16 | ListGroups | v0-v4 |
| 17 | SaslHandshake | v0-v1 |
| 18 | ApiVersions | v0-v3 |
| 19 | CreateTopics | v0-v7 |
| 20 | DeleteTopics | v0-v6 |
| 36 | SaslAuthenticate | v0-v2 |
| 37 | CreatePartitions | v0-v3 |
| 50 | DescribeConfigs | v0-v4 |
| 51 | AlterConfigs | v0-v2 |
| 65 | DescribeCluster | v0-v1 (cluster-aware) |

### Partition Mapping
- Kafka topic → Ivy topic (1:1 name mapping, tenant-scoped)
- Kafka partition index → Ivy `partition_num`
- Kafka leader ID → Ivy `leader_id` (from `partition_offsets`)
- Kafka `Metadata` response returns actual `leader_id` for each partition

### Consumer Groups
- Classic rebalance protocol (JoinGroup/SyncGroup) — fully supported
- KIP-848 new protocol (ConsumerGroupHeartbeat) — supported
- Assignment strategies: `range`, `roundrobin`, `sticky`
- Coordinator election: broker with `hash(groupId) % N` from active brokers

### Transactions
- `InitProducerId` → allocate `producerId` + `producerEpoch`, store in `producer_state`
- `AddPartitionsToTxn` → register partitions in `transactions` table
- `EndTxn(COMMIT)` → write control records to each partition, update `transactions` state
- `EndTxn(ABORT)` → write abort control records, mark `transactions` as `COMPLETE_ABORT`
- `TxnOffsetCommit` → commit offsets atomically within transaction

### Auth
- SASL/PLAIN (username:password, use with TLS)
- SASL/SCRAM-SHA-256

---

## AMQP 0-9-1 Protocol (port 5672 / 5671 TLS)

### Supported Operations

| Class | Method | Notes |
|-------|--------|-------|
| Connection | Start, StartOk, Tune, TuneOk, Open, OpenOk, Close, CloseOk | Full handshake |
| Channel | Open, OpenOk, Close, CloseOk | Multi-channel |
| Exchange | Declare, DeclareOk, Delete, DeleteOk | 4 exchange types |
| Queue | Declare, DeclareOk, Bind, BindOk, Unbind, UnbindOk, Purge, Delete | Full lifecycle |
| Basic | Publish, Deliver, Get, GetOk, GetEmpty, Ack, Nack, Reject, Consume, ConsumeOk, Cancel, CancelOk, Qos, QosOk | Full publish/consume |
| Confirm | Select, SelectOk | Publisher confirms |
| Tx | Select, SelectOk, Commit, CommitOk, Rollback, RollbackOk | AMQP transactions |

### Exchange Types

| Type | Routing Logic | Ivy Mapping |
|------|--------------|------------|
| `direct` | exact match on routing key | route to partition by key hash |
| `fanout` | all bound queues | broadcast to all partitions of topic |
| `topic` | wildcard match (`*`, `#`) | match `routingKey` against `bindingKey` patterns |
| `headers` | match on message headers | match header map against binding args |

### AMQP → Ivy Mapping

```
AMQP Exchange  →  Ivy topic prefix (exchange.name)
AMQP Queue     →  Ivy topic (exchange.name + "." + queue.name)
AMQP Binding   →  routing rule stored in-memory per channel
AMQP Message   →  Ivy PendingWrite (headers + body)
AMQP Consumer  →  Ivy subscription on partition(s)
AMQP Ack       →  Ivy consumer offset commit
AMQP Nack/Reject (requeue=false) → Ivy DLQ route
```

### DLQ via DLX

Queue declared with `x-dead-letter-exchange` argument:
```
Queue.Declare(
  arguments = {
    "x-dead-letter-exchange"   : "dlx",
    "x-dead-letter-routing-key": "failed-orders",
    "x-message-ttl"            : 300000,
    "x-delivery-limit"         : 5
  }
)
```

`x-death` header added to DLQ messages for RabbitMQ client compatibility:
```
x-death: [{
  "count": 1,
  "exchange": "orders",
  "queue": "order-processing",
  "reason": "rejected",
  "routing-keys": ["order.new"],
  "time": <timestamp>
}]
```

### Publisher Confirms

```
Channel.ConfirmSelect → broker tracks unconfirmed deliveryTags
Basic.Publish → write to broker → on PG COMMIT → Basic.Ack(deliveryTag, multiple=false)
             → on failure → Basic.Nack(deliveryTag) → client retries
```

---

## MQTT 3.1.1 Protocol (port 1883 / 8883 TLS)

### Packet Types

| Type | Packet | Direction | Notes |
|------|--------|-----------|-------|
| 1 | CONNECT | C→S | clientId, keepAlive, will, auth |
| 2 | CONNACK | S→C | sessionPresent, returnCode |
| 3 | PUBLISH | C↔S | QoS 0/1/2, retain, dup |
| 4 | PUBACK | C↔S | QoS 1 ack |
| 5 | PUBREC | C↔S | QoS 2 phase 1 |
| 6 | PUBREL | C↔S | QoS 2 phase 2 |
| 7 | PUBCOMP | C↔S | QoS 2 phase 3 |
| 8 | SUBSCRIBE | C→S | topicFilter, QoS |
| 9 | SUBACK | S→C | returnCode per filter |
| 10 | UNSUBSCRIBE | C→S | topicFilter |
| 11 | UNSUBACK | S→C | |
| 12 | PINGREQ | C→S | keepAlive |
| 13 | PINGRESP | S→C | |
| 14 | DISCONNECT | C→S | graceful close |

### QoS Semantics

| QoS | Guarantee | DLQ support |
|-----|-----------|-------------|
| 0 | At most once (fire and forget) | No (not tracked) |
| 1 | At least once (PUBACK) | Yes (on PUBACK failure N times) |
| 2 | Exactly once (PUBREC/PUBREL/PUBCOMP) | Yes (on PUBCOMP failure) |

### MQTT → Ivy Mapping

```
MQTT topic        →  Ivy topic (slash → dot for internal naming)
                     "home/living/temperature" → "home.living.temperature"
MQTT clientId     →  Ivy consumer group member ID
MQTT subscription →  Ivy partition subscription
MQTT QoS 1 PUBACK →  Ivy consumer offset commit
MQTT retain=true  →  Ivy compacted partition (latest value per key retained)
MQTT will message →  stored on CONNECT, published on unexpected disconnect
```

**Topic filter wildcards:**
- `+` (single-level): matches exactly one level → regex `[^/]+`
- `#` (multi-level): matches remaining levels → match all partitions of prefix

**Clean session:**
- `cleanSession=true` → new subscription, offsets start at LATEST
- `cleanSession=false` → resume from committed offset (stored in `consumer_offsets`)

### Retained Messages

MQTT `RETAIN=1` is mapped to Ivy's log compaction:
- Messages with `retain=true` are written with `cleanup_policy=compact`
- On subscribe, the last retained message for matching topics is delivered immediately
- Implemented by querying the compacted LogSegment or PG for the latest record per key

### Will Messages

Stored in `MqttSessionState` on CONNECT:
```
willTopic, willPayload, willQos, willRetain
```
Published to broker on unexpected disconnect (no DISCONNECT packet received).
`clientId` prefix is used as the will message key for tracking.

---

## MySQL Wire Protocol (port 3306 / 3307 TLS) — Read-Only

### Handshake

```
Server → Client: HandshakeV10 (capabilities, auth plugin = mysql_native_password)
Client → Server: HandshakeResponse41 (username, auth_response)
Server → Client: OK_Packet (auth success) or ERR_Packet
```

### Supported SQL

```sql
-- List topics for current tenant
SHOW TABLES;
SHOW TABLES LIKE 'order%';

-- Fetch messages from a topic
SELECT key, value, offset_num, timestamp_ms, protocol_id
FROM <topic_name>
WHERE offset_num > 100
LIMIT 50;

SELECT key, value
FROM <topic_name>
WHERE timestamp_ms > UNIX_TIMESTAMP('2024-01-01') * 1000
LIMIT 100;

-- Inspect topic metadata
DESCRIBE <topic_name>;
-- Returns: Field, Type, Null, Key, Default, Extra
-- Columns: offset_num(BIGINT), key(BLOB), value(BLOB),
--          headers(BLOB), timestamp_ms(BIGINT), protocol_id(SMALLINT)

-- Cluster state
SELECT * FROM __broker_registry;
SELECT * FROM __consumer_groups;
SELECT * FROM __partitions;
SELECT * FROM __dlq_entries WHERE original_topic = 'orders' LIMIT 20;

-- Topic info
SELECT topic_id, name, partition_count, retention_ms FROM __topics;
```

### Result Set Encoding

MySQL ResultSet wire format:
```
[column_count: LengthEncoded]
[ColumnDefinition41 × column_count]
[EOF_Packet]
[row × N rows]  ← each row: [LengthEncodedString × column_count]
[EOF_Packet]
```

### SQL Parser (minimal)
- Not a full SQL parser — uses pattern matching for the supported subset
- `SqlQueryParser.parse(sql)` → `SqlQuery` sealed record:
  ```java
  sealed interface SqlQuery {
    record ShowTables(String pattern)                  implements SqlQuery {}
    record DescribeTable(String tableName)             implements SqlQuery {}
    record SelectTopic(String topic, long fromOffset,
                       long fromTimestamp, int limit) implements SqlQuery {}
    record SelectMetadata(MetadataTable table,
                          String filter, int limit)   implements SqlQuery {}
    record Unsupported(String reason)                  implements SqlQuery {}
  }
  ```

---

## PostgreSQL Wire Protocol (port 5432 / 5433 TLS) — Read-Only

### Startup Sequence

```
Client → Server: StartupMessage (protocol=196608, user, database)
Server → Client: AuthenticationMD5Password or AuthenticationSASL (SCRAM-SHA-256)
Client → Server: PasswordMessage or SASLInitialResponse + ...
Server → Client: AuthenticationOk, ParameterStatus×N, BackendKeyData, ReadyForQuery
```

### Simple Query Protocol

```
Client → Server: Query('SELECT ...')
Server → Client: RowDescription, DataRow×N, CommandComplete, ReadyForQuery
             or: ErrorResponse, ReadyForQuery
```

### Supported SQL

```sql
-- List topics
SELECT topic_id, name, partition_count, retention_ms
FROM topics
WHERE tenant_id = current_setting('ivy.tenant_id')::uuid;

-- Fetch messages from topic
SELECT offset_num, key, value, headers, timestamp_ms
FROM <topic_name>
WHERE offset_num > $1
ORDER BY offset_num
LIMIT $2;

-- Fetch by timestamp
SELECT offset_num, key, value
FROM <topic_name>
WHERE timestamp_ms > $1
LIMIT $2;

-- Partition info
SELECT partition_id, partition_num, state
FROM partitions
WHERE topic_id = (SELECT topic_id FROM topics WHERE name = $1);

-- Consumer group state
SELECT group_id, state, generation, leader_member
FROM consumer_groups;

-- Consumer committed offsets
SELECT co.group_id, p.partition_num, co.committed_offset
FROM consumer_offsets co
JOIN partitions p ON co.partition_id = p.partition_id
WHERE co.group_id = $1;

-- Cluster state
SELECT broker_id, host, port, status, last_heartbeat
FROM broker_registry;

-- DLQ inspection
SELECT * FROM dlq_entries
WHERE tenant_id = current_setting('ivy.tenant_id')::uuid
ORDER BY failed_at DESC
LIMIT $1;
```

### Type Encoding

PgWire text format for column types:
```
offset_num   → INT8   (OID 20)
key          → BYTEA  (OID 17)
value        → BYTEA  (OID 17)
headers      → BYTEA  (OID 17)
timestamp_ms → INT8   (OID 20)
protocol_id  → INT2   (OID 21)
topic_id     → UUID   (OID 2950)
```

### Error Response

PG ErrorResponse packet with fields:
- `S` Severity: ERROR
- `C` Code: standard PG SQLSTATE code
- `M` Message: human-readable
- `D` Detail (optional)

---

## Cross-Protocol Message Consumption

A message produced via Kafka can be consumed via AMQP or MQTT (and vice versa).

**Compatibility rules:**
- `key` and `value` are raw bytes — all protocols pass through without modification
- Headers:
  - Kafka headers → AMQP message headers (key-value map)
  - AMQP headers → Kafka headers
  - MQTT: no headers in 3.1.1 → prepended as binary metadata prefix (if present)
- Timestamps: all stored as Unix milliseconds
- Protocol-specific metadata (MQTT QoS, AMQP deliveryTag) is not persisted across protocols

**Example: MQTT producer → Kafka consumer**
```
MQTT PUBLISH(topic="orders/new", payload="...", QoS=1)
  → stored as messages row: (partitionId, offset, key=null, value=payload, protocol_id=3)

Kafka FETCH(topic="orders.new", partition=0, offset=42)
  → returns Record(key=null, value=payload, headers=[], timestamp=...)
```
