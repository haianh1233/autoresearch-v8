# Protocol Design

## Overview

All seven protocols share the same underlying broker engine, storage, and clustering.
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
| 2  | AMQP 0-9-1 | Exchange/queue model, classic RabbitMQ wire |
| 3  | AMQP 1.0 | ISO standard, Azure Service Bus / ActiveMQ Artemis wire |
| 4  | MQTT 3.1.1 | Pub/sub, IoT, most widely deployed MQTT version |
| 5  | MQTT 5.0 | Enhanced MQTT: user properties, shared subscriptions, reason codes |
| 6  | MySQL wire | Read-only SQL view |
| 7  | PgWire | Read-only SQL view |

Protocol IDs are stored in the `messages.protocol_id` column and segment trailers.
They are **append-only** — never reuse or reorder an ID.

---

## Protocol Detection (magic bytes)

`ProtocolDetector` peeks at the first 4 bytes without consuming them:

```
Bytes 0-3               → Protocol
──────────────────────────────────────────────────────────────────────────────
0x00 0x?? ...           → Kafka  (MSB-first 4-byte request length, starts near 0)
0x10 - 0xEF             → MQTT 3.1.1  (first byte = control packet type; CONNECT = 0x10)
                                       Note: MQTT 5.0 also starts with 0x10 (CONNECT)
                                       → version detected inside CONNECT payload (byte[9])
                                         0x04 = MQTT 3.1.1, 0x05 = MQTT 5.0
0x41 0x4D 0x51 0x50     → AMQP  ("AMQP" ASCII — both 0-9-1 and 1.0 share this magic)
    [0x41 0x4D 0x51 0x50 0x00 0x00 0x09 0x01] → AMQP 0-9-1 (protocol header bytes 4-7 = 0,0,9,1)
    [0x41 0x4D 0x51 0x50 0x00 0x01 0x00 0x00] → AMQP 1.0   (bytes 4-7 = 0,1,0,0)
    Detect by peeking 8 bytes: byte[5] = 0x00 means 1.0, byte[5] = 0x09 means 0-9-1
<len:3><seq:1>0x0A      → MySQL (handshake packet, server greeting type=0x0A)
0x00 0x00 0x00 ??       → PgWire (first 4 bytes = message length, next 4 = 196608 = proto 3.0)
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

## AMQP 1.0 Protocol (port 5673)

AMQP 1.0 is a fundamentally different wire protocol from 0-9-1 — not a version upgrade
but a separate ISO/IEC 19464 standard. Used by Azure Service Bus, ActiveMQ Artemis,
Apache Qpid, and RabbitMQ (via plugin).

### Frame Structure

```
┌──────────────────────────────────────────────────────┐
│ Frame Header (8 bytes)                               │
│  [size:4][doff:1][type:1][type-specific:2]           │
│  type=0x00: AMQP frame                               │
│  type=0x01: SASL frame                               │
├──────────────────────────────────────────────────────┤
│ Extended Header (doff*4 - 8 bytes, usually 0)        │
├──────────────────────────────────────────────────────┤
│ Performative (AMQP type-encoded descriptor + body)   │
└──────────────────────────────────────────────────────┘
```

### Connection / Session / Link Hierarchy

```
Connection (TCP)
  └── Session (1..N per connection, bidirectional)
        └── Link (1..N per session, unidirectional)
              Sender Link  → messages flow to broker  (producer)
              Receiver Link ← messages flow to client (consumer)
```

### Performative Types (key subset)

| Descriptor | Name | Direction | Purpose |
|------------|------|-----------|---------|
| 0x10 | open | C↔S | Connection-level negotiation (max-frame-size, channel-max) |
| 0x11 | begin | C↔S | Open a session |
| 0x12 | attach | C↔S | Attach a link (source, target, role=sender/receiver) |
| 0x13 | flow | C↔S | Flow control (link-credit, delivery-count) |
| 0x14 | transfer | C→S | Deliver a message on a sender link |
| 0x15 | disposition | C↔S | Settle deliveries (accepted, rejected, released, modified) |
| 0x16 | detach | C↔S | Detach a link (with optional error) |
| 0x17 | end | C↔S | End a session |
| 0x18 | close | C↔S | Close connection |

### Message Format (AMQP Value Sections)

```
[header]          — durable, priority, ttl, first-acquirer, delivery-count
[delivery-annotations]   — per-delivery metadata (map)
[message-annotations]    — per-message metadata (map)
[properties]      — message-id, user-id, to, subject, reply-to, content-type, ...
[application-properties] — user-defined key/value map  ← maps to Ivy headers
[body]            — amqp-value | amqp-sequence | data section ← maps to Ivy value
[footer]          — delivery metadata applied after body
```

### AMQP 1.0 → Ivy Mapping

```
AMQP 1.0 target address    → Ivy topic name (from link Attach.target.address)
AMQP 1.0 source address    → Ivy topic name (from link Attach.source.address)
AMQP 1.0 transfer payload  → Ivy PendingWrite(value = body, headers = application-properties)
AMQP 1.0 disposition(accepted) → Ivy consumer offset commit
AMQP 1.0 disposition(rejected) → Ivy DlqRouter (requeue=false semantics)
AMQP 1.0 disposition(released) → requeue (no DLQ)
AMQP 1.0 disposition(modified{undeliverable=true}) → Ivy DlqRouter
```

### Flow Control

AMQP 1.0 uses **link credit** for flow control (unlike 0-9-1 which uses Qos.prefetch-count):
```
Receiver → broker: flow(link-credit=100)   ← "I can accept 100 more messages"
Broker sends up to 100 transfer frames
Receiver → broker: flow(link-credit=50)    ← replenish credit
```

`Amqp10SessionHandler` tracks `linkCredit` per link and buffers outgoing transfers.

### Settlement Modes

| Mode | Sender | Receiver | Semantics |
|------|--------|----------|-----------|
| `at-most-once` | settled=true | — | Fire and forget |
| `at-least-once` | settled=false | settles on disposition | At least once (default) |
| `exactly-once` | settled=false | coordinates with sender | Two-phase settlement |

### SASL (connection-level)

AMQP 1.0 uses a SASL exchange before the AMQP open frame:
```
S→C: sasl-mechanisms([PLAIN, SCRAM-SHA-256])
C→S: sasl-init(mechanism=PLAIN, initial-response=\0user\0pass)
S→C: sasl-outcome(code=OK)
     — AMQP open frame follows
```

### DLQ in AMQP 1.0

`disposition(rejected, error={condition="amqp:rejected"})` triggers `DlqRouter`.
`disposition(modified, delivery-failed=true, undeliverable-here=true)` also triggers DLQ.

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

## MQTT 5.0 Protocol (port 1884 / 8884 TLS, or shared 1883 via version byte)

MQTT 5.0 adds significant features over 3.1.1 while keeping backward wire compatibility.
The version byte in CONNECT (byte[9]) distinguishes them: `0x04` = 3.1.1, `0x05` = 5.0.

### New Packet Types vs 3.1.1

| Type | Packet | Notes |
|------|--------|-------|
| 15 | AUTH | New in 5.0 — enhanced authentication exchange |
| DISCONNECT now has Reason Code + Properties | — | Client can now send reason code |

All 3.1.1 packet types exist in 5.0 with extended properties sections.

### Key New Features

**1. User Properties (application-level headers)**
Every packet type in MQTT 5.0 can carry a list of user properties (key/value UTF-8 string pairs).
These map directly to Ivy message headers:
```
MQTT 5.0 PUBLISH user-property "order-id" = "12345"
  → stored in Ivy headers: packed [key_len:2]["order-id"][val_len:4]["12345"]
  → available when consumed via Kafka or AMQP (header round-trip)
```

**2. Message Expiry Interval**
```
PUBLISH property: Message-Expiry-Interval = 300  (seconds)
  → broker stores expiry as timestamp = publish_time + 300s
  → if consumer fetches after expiry → message routed to DLQ (TTL_EXPIRED)
  → on re-delivery, remaining expiry is decremented in the delivered PUBLISH
```

**3. Subscription Options (SUBSCRIBE)**
```
subscription-identifier  → correlate received PUBLISH back to a subscription
retain-as-published      → forward the RETAIN flag as-is to subscriber
retain-handling: 0/1/2   → 0=send retained on subscribe, 1=only if new sub, 2=never
no-local                 → don't receive own publishes (per-connection flag)
```

**4. Shared Subscriptions**
```
SUBSCRIBE $share/<group-name>/<topic-filter>
  → maps to Ivy consumer group (group-name) on the resolved partitions
  → messages distributed round-robin across group members
  → equivalent to Kafka consumer group semantics
```
`$share/` prefix is detected in `MqttRequestHandler.subscribe()` and routed to `ConsumerGroupCoordinator`.

**5. Request/Response Pattern**
```
PUBLISH properties:
  response-topic  = "replies/order-status"
  correlation-data = <binary id>

Responder:
  PUBLISH to "replies/order-status" with same correlation-data
```
Ivy stores `response-topic` and `correlation-data` as headers and passes them through unchanged.

**6. Reason Codes (all packets)**
Every acknowledgement in 5.0 carries a reason code byte (not just success/fail):
```
PUBACK reason codes: 0x00 Success, 0x10 No matching subscribers, 0x80 Unspecified error, ...
SUBACK reason codes: 0x00 QoS0, 0x01 QoS1, 0x02 QoS2, 0x80 Not authorized, ...
DISCONNECT reason codes: 0x00 Normal, 0x81 Malformed packet, 0x89 Keep-alive timeout, ...
```

**7. Topic Aliases**
Client can assign a short integer alias to a long topic name:
```
PUBLISH topic="very/long/topic/name" topic-alias=5
→ subsequent PUBLISH topic="" topic-alias=5  (empty topic = use alias)
```
`MqttSessionState` maintains a `Map<Short, TopicName> topicAliasMap` per connection.

**8. Enhanced Authentication (AUTH packet)**
```
CONNECT auth-method="SCRAM-SHA-256" auth-data=<client-first>
S→C: AUTH reason=0x18 (continue) auth-data=<server-first>
C→S: AUTH reason=0x18 auth-data=<client-final>
S→C: CONNACK reason=0x00 auth-data=<server-final>
```

**9. Session Expiry Interval**
```
CONNECT session-expiry-interval = 3600   (seconds; 0 = clean session, 0xFFFFFFFF = persistent)
DISCONNECT session-expiry-interval = 0   (can override on disconnect)
```
Stored in `consumer_groups.config` and enforced by `MqttSessionManager`.

**10. Flow Control (Receive Maximum)**
```
CONNECT receive-maximum = 20    ← client tells broker: max 20 in-flight QoS 1/2 messages
→ broker tracks per-session in-flight count, pauses delivery at limit
```

### MQTT 5.0 → Ivy Mapping (additions to 3.1.1 mapping)

```
MQTT 5.0 user-properties      → Ivy headers (added alongside MQTT 3.1.1 headers)
MQTT 5.0 message-expiry       → Ivy DLQ on expiry (TTL_EXPIRED)
MQTT 5.0 shared subscriptions → Ivy ConsumerGroupCoordinator
MQTT 5.0 response-topic       → stored as header "mqtt5-response-topic"
MQTT 5.0 correlation-data     → stored as header "mqtt5-correlation-data"
MQTT 5.0 content-type         → stored as header "mqtt5-content-type"
MQTT 5.0 topic-alias          → resolved to full topic name before storage (aliases not persisted)
```

### Handler Structure

`Mqtt5RequestHandler` extends `Mqtt311RequestHandler`:
- Shares all QoS 0/1/2 logic, retained messages, will messages
- Overrides: CONNECT parsing (version=5, session-expiry, receive-maximum, auth-method)
- Adds: AUTH packet handling, topic alias resolution, user-property injection into headers
- Adds: shared subscription detection (`$share/`) → ConsumerGroupCoordinator
- Adds: reason codes in all PUBACK/SUBACK/UNSUBACK/DISCONNECT responses

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
