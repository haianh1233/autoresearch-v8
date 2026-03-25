# Module Design

## Overview

5 Maven modules with strict dependency discipline. No circular dependencies.
Assembly happens only in `ivy-server`. No module knows about `ivy-server` except itself.

```
ivy-server
  ├── ivy-codec        (protocol wire formats)
  ├── ivy-broker       (engine, clustering, DLQ)
  │     └── ivy-storage  (segments, PG)
  │           └── ivy-common
  └── ivy-common       (foundation types)
```

---

## ivy-common

**Purpose:** Zero-dependency foundation. All domain types and core interfaces.

**Allowed dependencies:** None (no external runtime deps, not even Netty)

### Branded Types (value records, Java 26)

```java
// All are value records — no object header, stack-allocatable, ==  is structural equality
value record TenantId(UUID id)         { static TenantId of(UUID id) {...} }
value record PartitionId(UUID id)      {}
value record TopicId(UUID id)          {}
value record BrokerId(UUID id)         {}
value record ProducerId(long id)       {}
value record Offset(long value)        { static final Offset EARLIEST = new Offset(-2L);
                                         static final Offset LATEST   = new Offset(-1L); }
value record TopicName(String value)   { /* validates: non-blank, max 249 chars */ }
value record GroupId(String value)     {}
value record Port(int value)           { /* validates: 1-65535 */ }
value record ProducerEpoch(short value){}
value record LeaderEpoch(int value)    {}
```

**Why branded types?**
- `PartitionId` and `TopicId` are both UUIDs but are incompatible at compile time
- No raw `UUID`, `String`, `int`, `long` cross module API boundaries
- Eliminates entire categories of type confusion bugs

### Core Interfaces (sealed)

```java
// Messaging contract — implemented by DefaultBrokerEngine in ivy-broker
sealed interface BrokerEngine permits DefaultBrokerEngine {
    WriteResult       write(TenantId, List<PendingWrite>, SecurityContext);
    FetchResult       fetch(TenantId, PartitionId, Offset, int maxBytes);
    SubscriptionHandle subscribe(TenantId, PartitionId, Offset, Consumer<FlushEvent>);
    void              commitOffset(TenantId, GroupId, PartitionId, Offset);
    TopicMetadata     describeTopic(TenantId, TopicName);
    List<TopicMetadata> listTopics(TenantId);
    void              createTopic(TenantId, TopicName, int partitions, SecurityContext);
}

// Storage contract — implemented by PostgresStorageEngine in ivy-storage
sealed interface StorageEngine permits PostgresStorageEngine {
    void               append(PartitionId, List<MessageRecord>);
    List<MessageRecord> fetchRange(PartitionId, Offset from, Offset to, int maxBytes);
    Offset             highWatermark(PartitionId);
    void               createSchema();
    void               migrateSchema();
}

// Auth contract — implemented by DefaultAuthEngine in ivy-broker
sealed interface AuthEngine permits DefaultAuthEngine {
    SecurityContext authenticate(Credentials, TenantId);
    void            authorize(SecurityContext, Operation, ResourceType, String resourceName);
}
```

### Value Records (hot-path data types)

```java
// Java 26 value class — GC-free on hot path
value class PendingWrite {
    TenantId    tenantId;
    PartitionId partitionId;
    byte[]      key;
    byte[]      value;
    byte[]      headers;
    long        timestampMs;
    long        producerId;
    short       producerEpoch;
    int         sequence;
    boolean     isTransactional;
    byte        protocolId;
    boolean     isDlq;
}

value class WriteResult {
    PartitionId partitionId;
    long        baseOffset;
    int         recordCount;
    long        logAppendTimeMs;
    ErrorCode   errorCode;
}

value class FlushEvent {
    PartitionId partitionId;
    long        highWatermark;
    long        lastStableOffset;
}
```

### Config Types

```java
record BrokerConfig(
    BrokerId        brokerId,
    List<Port>      kafkaPorts,
    Port            amqpPort,
    Port            mqttPort,
    Port            mysqlPort,
    Port            pgwirePort,
    Port            httpPort,
    ClusterConfig   cluster,
    StorageConfig   storage,
    AuthConfig      auth
) {}

record ClusterConfig(
    boolean          enabled,
    byte[]           clusterSecret,
    List<SeedBroker> seeds,
    Duration         heartbeatInterval,
    Duration         staleThreshold,
    Duration         metadataRefreshInterval
) {}

record StorageConfig(
    String    jdbcUrl,
    String    username,
    String    password,
    int       maxPoolSize,
    Duration  connectionTimeout
) {}
```

### Environment Abstraction (deterministic testing)

```java
interface Environment {
    Clock       clock();
    IdGenerator idGenerator();   // UUID generation
    Scheduler   scheduler();     // ScheduledExecutorService wrapper
}

// Production: real clock, random UUIDs, real scheduler
class DefaultEnvironment implements Environment { ... }

// Test: controlled clock, deterministic IDs, manual scheduler
class SimulatedEnvironment implements Environment { ... }
```

---

## ivy-storage

**Purpose:** Data persistence layer. LogSegment read cache + PostgreSQL source of truth.

**Allowed dependencies:** `ivy-common`, JDBC (PostgreSQL driver, HikariCP)

### LogSegment

Append-only binary file. One per partition. Populated asynchronously after PG COMMIT.

```
Key methods:
  append(List<MessageRecord>)       → write records to end of current segment
  read(Offset from, int maxBytes)   → read records starting at offset
  highWatermark()                   → last appended offset
  seal()                            → mark segment as immutable, start new one
  delete()                          → remove segment file after retention expires
```

**File format:**
```
[message_size: 4 bytes]
[crc32: 4 bytes]
[offset: 8 bytes]
[timestamp_ms: 8 bytes]
[key_length: 4 bytes, -1 = null]
[key: N bytes]
[value_length: 4 bytes]
[value: N bytes]
[headers_length: 4 bytes]
[headers: N bytes]
```

### OffsetIndex

Sparse mmap'd index. One per LogSegment file.

```
Entry: [relative_offset: 4 bytes][file_position: 4 bytes]
Density: 1 entry per 4KB of log data
Lookup: binary search → O(log N)
Max size: 10MB (10MB / 8 bytes = 1.25M entries = ~5GB of log data)
```

### MetadataSegment

Same as LogSegment but uses log compaction — only the latest value per key is retained.
Used for internal topics (`__consumer_offsets`, `__consumer_groups`, `__transactions`).

### PostgresStorageEngine

Implements `StorageEngine`. Primary persistence layer.

```
Key methods:
  append(partitionId, records)    → binary COPY to messages table (within write transaction)
  fetchRange(partitionId, from, to, maxBytes)  → SELECT from messages
  highWatermark(partitionId)      → SELECT next_offset FROM partition_offsets
  createSchema()                  → execute V1__initial_schema.sql
  migrateSchema()                 → apply pending migration files

Internal:
  PgCopyOutputStream binaryWriter → streams records in PG binary COPY format
  HikariDataSource  pool          → connection pool (max 20 connections per broker)
  SchemaManager     migrations    → V{n}__{description}.sql files
```

### StorageFlusher

Background service. Moves records from write path into LogSegment after PG COMMIT.

```
On WriteWorker.processBatch() success:
  StorageFlusher.schedule(partitionId, baseOffset, records)
    → added to per-partition queue
    → background thread flushes every 200ms (configurable)
    → LogSegment.append(records)
```

---

## ivy-broker

**Purpose:** Broker engine, write/read paths, clustering, DLQ, consumer groups, transactions.

**Allowed dependencies:** `ivy-common`, `ivy-storage`, Netty (for inter-broker RPC only)

### engine/DefaultBrokerEngine

Main implementation of `BrokerEngine`. Orchestrates all broker operations.

```java
// Routing: check ownership, write locally or forward
WriteResult write(TenantId, List<PendingWrite>, SecurityContext):
  for each partitionGroup(pendingWrites):
    owner = hrwRouter.ownerOf(partitionId)
    if owner == selfBrokerId:
      writeAccumulator.accumulate(pendingWrites)
    else:
      forwardWriteManager.forward(owner, pendingWrites)
```

### write/

| Class | Responsibility |
|-------|---------------|
| `WriteAccumulator` | Per-partition batch accumulation (1K/1MB/5ms) |
| `WriteWorker` | 4 threads, PG-first transaction, epoch-fenced |
| `OffsetAllocator` | AtomicLong CAS for offset assignment (for single-broker mode) |
| `DuplicateDetector` | `producer_state` lookup for idempotent deduplication |

### read/

| Class | Responsibility |
|-------|---------------|
| `ReadAccumulator` | Three-tier fetch: L1 LogSegment → L2 InterBroker → L3 PG |
| `SubscriptionRegistry` | Map<PartitionId, Set<ConsumerHandle>> for push delivery |
| `FlushEventDispatcher` | Notifies subscribers after WriteWorker PG COMMIT |

### dlq/

| Class | Responsibility |
|-------|---------------|
| `DlqRouter` | Evaluates trigger conditions, injects headers, routes to DLQ partition |
| `DlqHeaderBuilder` | Builds `x-dlq-*` header set for DLQ messages |
| `DlqTopicManager` | Auto-creates `__dlq.<topic>` on first DLQ write |
| `DlqConfig` | Per-topic DLQ configuration (max retries, enabled) |

### consumer/

| Class | Responsibility |
|-------|---------------|
| `ConsumerGroupCoordinator` | Group state machine (EMPTY→PREP_REBALANCE→COMPLETING→STABLE→DEAD) |
| `MemberManager` | Tracks group members, heartbeats, session timeouts |
| `PartitionAssignor` | Range/roundrobin/sticky assignment strategies |
| `OffsetManager` | Commit/fetch offsets from `consumer_offsets` table |
| `DeliveryTracker` | Per-message delivery count tracking for DLQ trigger |

### transaction/

| Class | Responsibility |
|-------|---------------|
| `TransactionCoordinator` | ONGOING→PREPARE_COMMIT/ABORT→COMPLETE_* state machine |
| `ProducerStateManager` | `producer_state` CRUD for idempotency |
| `TransactionRepository` | PG CRUD for `transactions` table |
| `ControlRecordWriter` | Writes COMMIT/ABORT control records to partitions |

### cluster/

| Class | Responsibility |
|-------|---------------|
| `HRWRouter` | HMAC-SHA-256 rendezvous hash, `ownerOf(partitionId)` |
| `MetadataImage` | Immutable snapshot: activeBrokers, ownership, epochs |
| `MetadataImageHolder` | VarHandle atomic reference for MetadataImage |
| `MetadataPoller` | Polls PG every 30s to rebuild MetadataImage |
| `ClusterManager` | Broker lifecycle: STARTING→ACTIVE→DRAINING→SHUTDOWN |
| `HeartbeatWriter` | Periodic `UPDATE broker_registry SET last_heartbeat = now()` |
| `HeartbeatMonitor` | Detects stale brokers, triggers BrokerFencingPipeline |
| `BrokerFencingPipeline` | CAS fence → release partitions → broadcast → re-elect |
| `ForwardWriteManager` | Routes ForwardWriteRequest to owner via RPC client |
| `InterBrokerRpcServer` | Netty server on inter-broker port, dispatches inbound RPCs |
| `InterBrokerRpcClient` | Per-peer Netty client, reconnect with backoff |
| `InterBrokerMessage` | Sealed interface: ForwardWrite, ForwardFetch, MetadataBroadcast, Heartbeat |

### auth/DefaultAuthEngine

```java
// Authentication: verify credentials against credentials table
SecurityContext authenticate(Credentials credentials, TenantId tenantId):
  → ScramAuthenticator.verify(username, password, tenantId)  (SCRAM-SHA-256)
  → or PlainAuthenticator.verify(username, password, tenantId)
  → return SecurityContext(principal, tenantId, roles)

// Authorization: ACL check
void authorize(SecurityContext ctx, Operation op, ResourceType type, String name):
  → AclStore.evaluate(ctx.principal(), tenantId, op, type, name)
  → throw AuthorizationException if denied
```

---

## ivy-codec

**Purpose:** Wire protocol encoding and decoding. No business logic.

**Allowed dependencies:** `ivy-common`, Netty buffers only

### Per-Protocol Codec

Each codec has two responsibilities:
1. **Decoder:** `ByteBuf` → protocol-specific request/frame object
2. **Encoder:** response/frame object → `ByteBuf`

Package structure mirrors `clustering` — one sub-package per protocol:

```
com.ivy.codec/
  kafka/
    KafkaRequestDecoder    — reads request header (apiKey, apiVersion, correlationId) + body
    KafkaResponseEncoder   — writes response header (correlationId) + body
    KafkaApiVersions       — supported API key × version table
    KafkaRequest           — sealed interface for all request types
    KafkaResponse          — sealed interface for all response types

  amqp/                    ← AMQP 0-9-1
    Amqp091FrameDecoder    — reads [frame_type:1][channel:2][payload_size:4][payload:N][0xCE]
    Amqp091FrameEncoder    — writes same
    Amqp091MethodCodec     — encodes/decodes class+method+arguments
    Amqp091Frame           — sealed interface (MethodFrame, HeaderFrame, BodyFrame, HeartbeatFrame)

  amqp10/                  ← AMQP 1.0
    Amqp10FrameDecoder     — reads [size:4][doff:1][type:1][type-specific:2][payload]
    Amqp10FrameEncoder     — writes same
    Amqp10TypeCodec        — AMQP type system (described types, lists, maps, primitives)
    Amqp10Performative     — sealed interface (Open, Begin, Attach, Flow, Transfer, Disposition, Detach, End, Close)
    Amqp10MessageCodec     — message sections (header, properties, application-properties, body, footer)

  mqtt/                    ← MQTT 3.1.1 AND 5.0 share base; 5.0 extends
    MqttDecoder            — reads [fixed_header:1][remaining_length:1-4][payload:N]
    MqttEncoder            — writes same
    Mqtt311Packet          — sealed interface for all MQTT 3.1.1 packet types
    Mqtt5Packet            — sealed interface extending 3.1.1 with 5.0 properties
    Mqtt5PropertiesCodec   — encodes/decodes MQTT 5.0 property sets

  mysql/
    MySqlHandshakeEncoder  — writes server greeting (HandshakeV10)
    MySqlPacketDecoder     — reads [length:3][sequence:1][payload:N]
    MySqlResultSetEncoder  — writes ColumnDefinition41 + rows + EOF
    MySqlOkErrEncoder      — writes OK_Packet and ERR_Packet

  pg/                      ← PgWire
    PgStartupDecoder       — reads StartupMessage (length + protocol version + params)
    PgMessageDecoder       — reads [type:1][length:4][payload:N]
    PgMessageEncoder       — writes same
    PgRowDescEncoder       — writes RowDescription for a given schema
    PgDataRowEncoder       — writes DataRow for each result record
    PgErrorEncoder         — writes ErrorResponse with SQLSTATE fields
```

---

## ivy-server

**Purpose:** Assembly point. Netty pipeline, protocol handlers, `BrokerMain`.

**Allowed dependencies:** All modules. This is the only module with a `main()`.

### BrokerMain

Manual IoC — no DI framework. Pure constructor injection.

```java
public static void main(String[] args) {
    BrokerConfig config = ConfigParser.parse(args);
    Environment env = new DefaultEnvironment();

    // Storage
    var dataSource     = HikariDataSource(config.storage());
    var storageEngine  = new PostgresStorageEngine(dataSource);
    storageEngine.migrateSchema();

    var logSegmentStore = new LogSegmentStore(config.storage().dataDir());
    var storageFlusher  = new StorageFlusher(logSegmentStore);

    // Broker engine
    var metadataImage  = new MetadataImageHolder(MetadataImage.empty());
    var hrwRouter      = new HRWRouter(config.cluster().clusterSecret(), metadataImage);
    var writeAccumulator = new WriteAccumulator(config.broker().write());
    var writeWorker    = new WriteWorker(storageEngine, storageFlusher, writeAccumulator);
    var readAccumulator = new ReadAccumulator(logSegmentStore, storageEngine);
    var dlqRouter      = new DlqRouter(config.broker().dlq());
    var groupCoordinator = new ConsumerGroupCoordinator(storageEngine, env);
    var txCoordinator  = new TransactionCoordinator(storageEngine, env);
    var authEngine     = new DefaultAuthEngine(dataSource);

    // Clustering
    var clusterManager = new ClusterManager(config, dataSource, metadataImage, env);
    var forwardMgr     = new ForwardWriteManager(hrwRouter, /* rpc clients built by clusterManager */);

    var brokerEngine   = new DefaultBrokerEngine(
        writeAccumulator, writeWorker, readAccumulator, dlqRouter,
        groupCoordinator, txCoordinator, hrwRouter, forwardMgr, storageEngine);

    // Protocol codecs (one instance per protocol variant)
    var kafkaCodec     = new KafkaRequestDecoder();
    var amqp091Codec   = new Amqp091FrameDecoder();
    var amqp10Codec    = new Amqp10FrameDecoder();
    var mqtt311Codec   = new MqttDecoder(MqttVersion.V3_1_1);
    var mqtt5Codec     = new MqttDecoder(MqttVersion.V5);
    var mysqlCodec     = new MySqlPacketDecoder();
    var pgwireCodec    = new PgStartupDecoder();

    // Handlers — one per protocol variant, each in its own package
    var kafkaHandler   = new KafkaRequestHandler(brokerEngine, authEngine, kafkaCodec);
    var amqp091Handler = new Amqp091RequestHandler(brokerEngine, authEngine, amqp091Codec);
    var amqp10Handler  = new Amqp10RequestHandler(brokerEngine, authEngine, amqp10Codec);
    var mqtt311Handler = new Mqtt311RequestHandler(brokerEngine, authEngine, mqtt311Codec);
    var mqtt5Handler   = new Mqtt5RequestHandler(brokerEngine, authEngine, mqtt5Codec);
    var mysqlHandler   = new MySqlRequestHandler(brokerEngine, authEngine, mysqlCodec);
    var pgwireHandler  = new PgWireRequestHandler(brokerEngine, authEngine, pgwireCodec);

    // Netty
    var pipelineFactory = new NettyPipelineFactory(
        kafkaHandler, amqp091Handler, amqp10Handler,
        mqtt311Handler, mqtt5Handler, mysqlHandler, pgwireHandler, config);
    var server = new NettyBrokerServer(pipelineFactory, config);

    // Lifecycle
    clusterManager.start();
    server.start();
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
        clusterManager.drainAndStop();
        server.stop();
    }));
}
```

### NettyPipelineFactory

Builds the Netty pipeline per accepted connection:

```
EventLoop thread:
  [TLS SslHandler]             optional, per-port config
  [TenantResolverHandler]      SNI → TenantId (stored in channel attr)
  [ConnectionLimitHandler]     max connections per tenant
  [ProtocolDetector]           peek 8 bytes → identify protocol
  [ProtocolNegotiationHandler] install protocol-specific codec + handler
  [FlowControlHandler]         high/low watermark backpressure
  [ServerExceptionHandler]     catch-all, log + close
```

### Handler Package Structure

Mirrors the `clustering` project: one sub-package per protocol in `com.ivy.server.handler/`:

```
com.ivy.server.handler/
  kafka/
    KafkaRequestHandler        — main dispatcher
    KafkaProduceHandler        — Produce API (v3-v9)
    KafkaFetchHandler          — Fetch API (v4-v15)
    KafkaMetadataHandler       — Metadata, DescribeCluster
    KafkaConsumerGroupHandler  — JoinGroup, SyncGroup, Heartbeat, LeaveGroup, OffsetCommit/Fetch
    KafkaTransactionHandler    — InitProducerId, AddPartitions, EndTxn, TxnOffsetCommit
    KafkaAdminHandler          — CreateTopics, DeleteTopics, DescribeConfigs
    KafkaSaslHandler           — SaslHandshake, SaslAuthenticate

  amqp091/
    Amqp091RequestHandler      — main dispatcher
    Amqp091ConnectionHandler   — Connection.Start/Tune/Open/Close
    Amqp091ChannelHandler      — Channel.Open/Close
    Amqp091ExchangeHandler     — Exchange.Declare/Delete
    Amqp091QueueHandler        — Queue.Declare/Bind/Unbind/Purge/Delete
    Amqp091BasicHandler        — Basic.Publish/Consume/Get/Ack/Nack/Reject/Cancel/Qos
    Amqp091ConfirmHandler      — Confirm.Select, Basic.Ack/Nack for publisher confirms
    Amqp091TxHandler           — Tx.Select/Commit/Rollback
    Amqp091SessionState        — per-channel: exchanges, queues, consumers, confirms

  amqp10/
    Amqp10RequestHandler       — main dispatcher
    Amqp10ConnectionHandler    — Open/Close performatives, SASL exchange
    Amqp10SessionHandler       — Begin/End, link management
    Amqp10SenderLinkHandler    — Attach(sender), Transfer, Disposition settlement
    Amqp10ReceiverLinkHandler  — Attach(receiver), Flow credit management, Disposition
    Amqp10SessionState         — per-session: links, delivery tracking, flow control

  mqtt/
    Mqtt311RequestHandler      — main dispatcher for MQTT 3.1.1
    Mqtt5RequestHandler        — extends Mqtt311RequestHandler with 5.0 features
    MqttConnectHandler         — CONNECT / CONNACK (shared, version-dispatched)
    MqttPublishHandler         — PUBLISH + QoS 0/1/2 flows (PUBACK, PUBREC, PUBREL, PUBCOMP)
    MqttSubscribeHandler       — SUBSCRIBE / SUBACK, topic filter matching
    MqttUnsubscribeHandler     — UNSUBSCRIBE / UNSUBACK
    Mqtt5AuthHandler           — AUTH packet (MQTT 5.0 enhanced auth)
    MqttSessionState           — per-connection: will, subscriptions, QoS 2 state, topic aliases (5.0)
    MqttSharedSubHandler       — $share/ prefix → ConsumerGroupCoordinator (MQTT 5.0)

  mysql/
    MySqlRequestHandler        — handshake + COM_QUERY dispatch
    MySqlQueryExecutor         — SqlQueryParser → BrokerEngine.fetch() or metadata PG query
    MySqlSessionState          — per-connection auth state

  pgwire/
    PgWireRequestHandler       — startup + simple query dispatch
    PgWireQueryExecutor        — SqlQueryParser → BrokerEngine.fetch() or metadata PG query
    PgWireSessionState         — per-connection auth state, parameter status
```

### Per-Protocol Handler Pattern

Each handler implements `ChannelInboundHandlerAdapter`:

```
channelRead(ctx, msg):
  decoded = codec.decode(msg)
  tenantId = ctx.channel().attr(TENANT_ID_KEY).get()
  secCtx = sessionState.getOrCreate(ctx.channel())

  switch (decoded) {
    // Each handler delegates to sub-handlers by request type:
    case Kafka:   kafkaRequestDispatcher.dispatch(decoded, ctx, secCtx)
    case Amqp091: amqp091Dispatcher.dispatch(decoded, ctx, secCtx)
    case Amqp10:  amqp10Dispatcher.dispatch(decoded, ctx, secCtx)
    case Mqtt311: mqttDispatcher.dispatch(decoded, ctx, secCtx)
    case Mqtt5:   mqtt5Dispatcher.dispatch(decoded, ctx, secCtx)
    case MySql:   mysqlQueryExecutor.execute(decoded, ctx, secCtx)
    case PgWire:  pgwireQueryExecutor.execute(decoded, ctx, secCtx)
  }

  response = brokerEngine.write(...) or brokerEngine.fetch(...)
  encoded = codec.encode(response)
  ctx.writeAndFlush(encoded)
```

### GracefulShutdown

```
1. Mark broker status = DRAINING in broker_registry
2. Stop accepting new connections (close server socket)
3. Wait for in-flight WriteWorker batches to complete (max 30s)
4. Flush StorageFlusher (drain async LogSegment writes)
5. Release partition ownership (UPDATE partition_offsets SET leader_id = NULL)
6. Broadcast MetadataUpdateBroadcast
7. Close all client connections
8. Mark broker status = SHUTDOWN
9. Close HikariCP pool
10. Exit
```
