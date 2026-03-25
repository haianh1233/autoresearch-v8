# Module Design

## Overview

11 Maven modules. Codec and handler are **co-located per protocol** — no separate `ivy-codec` module.
`ivy-server` is a thin assembly that auto-discovers protocols via `ProtocolBundleRegistry` (ServiceLoader).

```
ivy-server
  ├── ivy-protocol-kafka
  ├── ivy-protocol-amqp       (0-9-1 + 1.0)
  ├── ivy-protocol-mqtt       (3.1.1 + 5.0)
  ├── ivy-protocol-postgresql
  ├── ivy-protocol-mysql
  ├── ivy-protocol-http       (REST produce/consume)
  ├── ivy-broker
  │     └── ivy-storage
  │           └── ivy-common
  └── ivy-common
```

**Dependency rules (enforced at build time):**
- `ivy-common` — zero external runtime dependencies
- `ivy-storage` — depends only on `ivy-common` + JDBC
- `ivy-broker` — depends on `ivy-common` + `ivy-storage`
- Each `ivy-protocol-*` — depends on `ivy-common` + `ivy-broker` + Netty
- `ivy-server` — depends on all modules; the only module with `main()`
- No protocol module may depend on another protocol module

---

## Module List

| Module | Purpose |
|--------|---------|
| `ivy-common` | Branded types, sealed interfaces, `ProtocolBundle` SPI, `ByteBufCodec` SPI |
| `ivy-storage` | `LogSegment` cache + `PostgresStorageEngine` (source of truth) |
| `ivy-broker` | Engine, write/read paths, clustering, DLQ, consumer groups, transactions |
| `ivy-protocol-kafka` | Kafka wire protocol — codec + all handlers |
| `ivy-protocol-amqp` | AMQP 0-9-1 and AMQP 1.0 — codec + handlers (two sub-packages) |
| `ivy-protocol-mqtt` | MQTT 3.1.1 and 5.0 — codec + handlers (version-negotiated at CONNECT) |
| `ivy-protocol-postgresql` | PgWire read-only SQL view — codec + handlers |
| `ivy-protocol-mysql` | MySQL wire read-only SQL view — codec + handlers |
| `ivy-protocol-http` | HTTP REST API — produce/consume via JSON over HTTP on port 8081 |
| `ivy-server` | Thin assembly: Netty bootstrap + `ProtocolBundleRegistry` wiring + `BrokerMain` |
| `ivy-testing` | Testcontainers fixtures, broker helpers, multi-protocol test utilities |

---

## Core SPI (ivy-common)

### ProtocolBundle

Each protocol module registers one (or two) `ProtocolBundle` implementations via
Java `ServiceLoader` (`META-INF/services/com.ivy.common.protocol.ProtocolBundle`).

```java
// ivy-common/src/main/java/com/ivy/common/protocol/ProtocolBundle.java
public interface ProtocolBundle {
    ProtocolId       protocolId();        // unique enum value
    ByteBufCodec     codec();             // null for server-speaks-first protocols
    DetectionRule    detectionRule();     // how to identify this protocol from first bytes
    ProtocolPorts    defaultPorts();      // default plaintext + TLS ports
    DestinationParser destinationParser(); // topic name resolution strategy
    ChannelHandler   newFrameDecoder();   // Netty ByteToMessageDecoder factory
    ChannelHandler   newRequestHandler(); // Netty ChannelInboundHandlerAdapter factory
}
```

### ByteBufCodec

```java
// ivy-common/src/main/java/com/ivy/common/protocol/ByteBufCodec.java
public interface ByteBufCodec {
    ProtocolId    protocolId();
    int           frameLength(ByteBuf in);                       // detect frame boundary
    InternalRequest  decode(ByteBuf frame, CodecContext ctx);    // bytes → request
    void          encode(InternalResponse resp, ByteBuf out, CodecContext ctx); // response → bytes
    void          encodeError(InternalErrorCode code, ByteBuf out, CodecContext ctx);
}
```

### ProtocolBundleRegistry

```java
// ivy-common/src/main/java/com/ivy/common/protocol/ProtocolBundleRegistry.java
public class ProtocolBundleRegistry {
    // Auto-discovers all ProtocolBundle implementations on the classpath
    public static ProtocolBundleRegistry fromServiceLoader();

    public void register(ProtocolBundle bundle);
    public Optional<ProtocolBundle> lookup(ProtocolId protocolId);
    public Collection<ProtocolBundle> listAll();
}
```

### InternalRequest / InternalResponse (sealed)

```java
// Wire-format-agnostic representation used between codec and broker engine
sealed interface InternalRequest {
    record WriteRequest(TenantId, List<PendingWrite>, SecurityContext)   implements InternalRequest {}
    record FetchRequest(TenantId, PartitionId, Offset, int maxBytes)     implements InternalRequest {}
    record SubscribeRequest(TenantId, GroupId, List<PartitionId>)        implements InternalRequest {}
    record CommitOffsetRequest(TenantId, GroupId, PartitionId, Offset)   implements InternalRequest {}
    record CreateTopicRequest(TenantId, TopicName, int partitions)       implements InternalRequest {}
    record MetadataRequest(TenantId, List<TopicName>)                    implements InternalRequest {}
    record GroupCoordinatorRequest(TenantId, GroupId)                    implements InternalRequest {}
    // ... others
}
```

---

## ivy-common

**Purpose:** Zero-dependency foundation. Types, interfaces, SPIs.

### Packages
```
com.ivy.common/
  types/        — TenantId, PartitionId, TopicId, BrokerId, Offset, TopicName, GroupId,
                  Port, ProducerEpoch, LeaderEpoch, ProducerId
  engine/       — BrokerEngine, StorageEngine, AuthEngine (sealed interfaces)
  protocol/     — ProtocolBundle, ByteBufCodec, ProtocolBundleRegistry,
                  InternalRequest (sealed), InternalResponse (sealed),
                  CodecContext, DetectionRule, ProtocolPorts, DestinationParser
  model/        — PendingWrite, WriteResult, FetchResult, FlushEvent (value records)
  config/       — BrokerConfig, ClusterConfig, StorageConfig
  error/        — ErrorCode (sealed), InternalErrorCode
  env/          — Environment, DefaultEnvironment, SimulatedEnvironment
```

---

## ivy-storage

**Purpose:** Data persistence. LogSegment cache + PostgreSQL source of truth.

**Dependencies:** `ivy-common`, JDBC (postgresql driver + HikariCP)

### Packages
```
com.ivy.storage/
  segment/      — LogSegment, MetadataSegment, SegmentCleaner, LogSegmentStore
  index/        — OffsetIndex (sparse, mmap'd, 8 bytes/entry)
  postgres/     — PostgresStorageEngine (binary COPY write, SELECT fetch, DDL migrations)
  flush/        — StorageFlusher (async 200ms background flush to LogSegment)
  migration/    — SchemaManager (V{n}__*.sql versioned migrations)
```

---

## ivy-broker

**Purpose:** Core engine. Write/read paths, clustering, DLQ, consumer groups, transactions.

**Dependencies:** `ivy-common`, `ivy-storage`, Netty (inter-broker RPC only)

### Packages
```
com.ivy.broker/
  engine/       — DefaultBrokerEngine (implements BrokerEngine)
  write/        — WriteAccumulator, WriteWorker (4 threads, PG-first), OffsetAllocator,
                  DuplicateDetector
  read/         — ReadAccumulator (L1→L2→L3), SubscriptionRegistry, FlushEventDispatcher
  dlq/          — DlqRouter, DlqHeaderBuilder, DlqTopicManager, DlqConfig
  consumer/     — ConsumerGroupCoordinator, MemberManager, PartitionAssignor, OffsetManager,
                  DeliveryTracker
  transaction/  — TransactionCoordinator, ProducerStateManager, TransactionRepository,
                  ControlRecordWriter
  auth/         — DefaultAuthEngine (implements AuthEngine), ScramAuthenticator (tenant-scoped),
                  AclStore (tenant + protocol-scoped, deny-first), AclAuthorizer,
                  TokenBucketQuotaManager (per-tenant/per-user), TenantSqlIsolation
  internal/     — TenantStore (__tenants cache), TenantEntry, CredentialStore, AclStore,
                  QuotaStore (all compacted-topic backed, replayed on startup)
  cluster/      — HRWRouter, MetadataImage, MetadataImageHolder, MetadataPoller,
                  ClusterManager, HeartbeatWriter, HeartbeatMonitor,
                  BrokerFencingPipeline, ForwardWriteManager,
                  InterBrokerRpcServer, InterBrokerRpcClient,
                  InterBrokerMessage (sealed)
```

---

## ivy-protocol-kafka

**Purpose:** Kafka wire protocol — codec, all request handlers, ProtocolBundle.

**Dependencies:** `ivy-common`, `ivy-broker`, Netty, `kafka-clients:4.2.0` (test only)

### Package Structure
```
com.ivy.protocol.kafka/
  KafkaCodec.java              — implements ByteBufCodec; frameLength, decode, encode
  KafkaRequestParser.java      — ApiKey + version deserialization → InternalRequest
  KafkaResponseSerializer.java — InternalResponse → wire bytes per API version
  KafkaFraming.java            — 4-byte big-endian frame length prefix detection
  KafkaVersions.java           — supported API key versions table
  KafkaNamespaceStrategy.java  — implements DestinationParser
  KafkaErrors.java             — Ivy ErrorCode → Kafka error code mapping
  KafkaFetchMetadata.java      — per-fetch session state (epoch, fetch session id)
  KafkaProduceMetadata.java    — per-produce metadata (acks, timeout)

  handler/
    KafkaRequestDispatcher.java    — routes decoded InternalRequest to sub-handlers
    KafkaProduceHandler.java       — Produce v3-v9
    KafkaFetchHandler.java         — Fetch v4-v15
    KafkaMetadataHandler.java      — Metadata v1-v12, DescribeCluster v0-v1
    KafkaOffsetCommitHandler.java  — OffsetCommit v0-v8
    KafkaOffsetFetchHandler.java   — OffsetFetch v0-v8
    KafkaGroupHandler.java         — JoinGroup, SyncGroup, Heartbeat, LeaveGroup, ListGroups, DescribeGroups
    KafkaCoordinatorHandler.java   — FindCoordinator (group + transaction)
    KafkaTransactionHandler.java   — InitProducerId, AddPartitionsToTxn, EndTxn, TxnOffsetCommit
    KafkaAdminHandler.java         — CreateTopics, DeleteTopics, CreatePartitions
    KafkaConfigHandler.java        — DescribeConfigs, AlterConfigs
    KafkaSaslAuthHandler.java      — SaslHandshake v0-v1, SaslAuthenticate v0-v2, ApiVersions
    KafkaListOffsetsHandler.java   — ListOffsets v1-v7
    KafkaProtocolBundle.java       — implements ProtocolBundle
                                     detection: first 4 bytes = request size (MSB ~0)
                                     ports: 9092 (plain), 9093 (TLS)
```

### ServiceLoader Registration
```
META-INF/services/com.ivy.common.protocol.ProtocolBundle:
  com.ivy.protocol.kafka.handler.KafkaProtocolBundle
```

---

## ivy-protocol-amqp

**Purpose:** AMQP 0-9-1 and AMQP 1.0 — both versions in one module, separate sub-packages.

**Dependencies:** `ivy-common`, `ivy-broker`, Netty, `protonj2:1.1.0` (AMQP 1.0 type system)

### Package Structure
```
com.ivy.protocol.amqp/                          ← AMQP 0-9-1
  Amqp091Codec.java            — frame constants, encode/decode methods
  AmqpFrame.java               — sealed: MethodFrame, HeaderFrame, BodyFrame, HeartbeatFrame
  Amqp091NamespaceStrategy.java — implements DestinationParser
  AmqpBasicPublishData.java    — data classes for each method
  AmqpBasicConsumeData.java
  AmqpQueueDeclareData.java
  AmqpExchangeDeclareData.java
  ... (other method data classes)

  handler/
    Amqp091FrameDecoder.java       — ByteToMessageDecoder: splits frames from TCP stream
    Amqp091RequestHandler.java     — main ChannelInboundHandlerAdapter, dispatches by method
    Amqp091ConnectionHandler.java  — Connection.Start/Tune/Open/Close + SASL
    Amqp091ChannelHandler.java     — Channel.Open/Close
    Amqp091ExchangeHandler.java    — Exchange.Declare/Delete
    Amqp091QueueHandler.java       — Queue.Declare/Bind/Unbind/Purge/Delete
    Amqp091PublishHandler.java     — Basic.Publish + publisher confirms
    Amqp091ConsumeHandler.java     — Basic.Consume/Cancel, Basic.Get
    Amqp091AckHandler.java         — Basic.Ack/Nack/Reject → DlqRouter on requeue=false
    Amqp091TxHandler.java          — Tx.Select/Commit/Rollback
    Amqp091SessionState.java       — per-channel: exchanges, queues, consumers, confirm mode
    Amqp091ProtocolBundle.java     — implements ProtocolBundle
                                     detection: "AMQP" + bytes[4-7] = {0,0,9,1}
                                     ports: 5672 (plain), 5671 (TLS)

com.ivy.protocol.amqp10/                        ← AMQP 1.0
  Amqp10Codec.java             — AMQP 1.0 type system encoding/decoding
  Amqp10FrameHeader.java       — [size:4][doff:1][type:1][type-specific:2]
  Amqp10TypeCodec.java         — described types, composite types, primitive encoding
  Amqp10MessageCodec.java      — message sections: header, properties, app-properties, body
  Amqp10NamespaceStrategy.java — implements DestinationParser

  handler/
    Amqp10FrameDecoder.java        — ByteToMessageDecoder for AMQP 1.0 frames
    Amqp10RequestHandler.java      — main dispatcher (SASL frames + AMQP frames)
    Amqp10ConnectionHandler.java   — Open/Close performatives + SASL exchange
    Amqp10SessionHandler.java      — Begin/End, session-level flow
    Amqp10SenderLinkHandler.java   — Attach(role=sender), Transfer ingestion, Disposition reply
    Amqp10ReceiverLinkHandler.java — Attach(role=receiver), Flow credit, Disposition settlement
    Amqp10SessionState.java        — per-session: links, delivery-id tracking, link-credit map
    Amqp10ProtocolBundle.java      — implements ProtocolBundle
                                     detection: "AMQP" + bytes[4-7] = {0,1,0,0} or {3,1,0,0}
                                     ports: 5673 (plain), 5674 (TLS)
```

### ServiceLoader Registration
```
META-INF/services/com.ivy.common.protocol.ProtocolBundle:
  com.ivy.protocol.amqp.handler.Amqp091ProtocolBundle
  com.ivy.protocol.amqp10.handler.Amqp10ProtocolBundle
```

---

## ivy-protocol-mqtt

**Purpose:** MQTT 3.1.1 and 5.0 — one module, version detected from CONNECT packet.

**Dependencies:** `ivy-common`, `ivy-broker`, Netty (no external MQTT library — pure Java)

### Package Structure
```
com.ivy.protocol.mqtt/
  MqttFraming.java             — variable-length integer frame boundary detection
  MqttPacketType.java          — enum: CONNECT=1, PUBLISH=3, SUBSCRIBE=8, ...
  MqttVersion.java             — enum: V3_1_1(4), V5(5); detected from CONNECT byte[9]
  MqttQoS.java                 — enum: AT_MOST_ONCE(0), AT_LEAST_ONCE(1), EXACTLY_ONCE(2)
  MqttConnectCodec.java        — encode/decode CONNECT + CONNACK (both versions)
  MqttPublishCodec.java        — encode/decode PUBLISH + PUBACK/PUBREC/PUBREL/PUBCOMP
  MqttSubscribeCodec.java      — encode/decode SUBSCRIBE + SUBACK, UNSUBSCRIBE + UNSUBACK
  MqttSimpleCodec.java         — PINGREQ/PINGRESP, DISCONNECT
  Mqtt5PropertiesCodec.java    — MQTT 5.0 property set encoding/decoding
  MqttNamespaceStrategy.java   — implements DestinationParser (slash → dot conversion)
  MqttConnectData.java         — CONNECT payload: clientId, will, auth, cleanSession, keepAlive
  MqttPublishData.java         — PUBLISH payload: topic, payload, qos, retain, packetId
  MqttSubscribeData.java       — SUBSCRIBE payload: topicFilters + QoS options

  handler/
    MqttFrameDecoder.java          — ByteToMessageDecoder: splits MQTT packets from TCP stream
    MqttRequestHandler.java        — top-level dispatcher; detects version at CONNECT
    MqttConnectHandler.java        — CONNECT / CONNACK (shared 3.1.1 + 5.0 logic)
    MqttPublishHandler.java        — PUBLISH + full QoS 0/1/2 state machine
    MqttSubscribeHandler.java      — SUBSCRIBE / SUBACK, topic filter wildcard matching
    MqttUnsubscribeHandler.java    — UNSUBSCRIBE / UNSUBACK
    MqttPingHandler.java           — PINGREQ / PINGRESP
    MqttDisconnectHandler.java     — DISCONNECT (3.1.1 + 5.0 reason codes)
    Mqtt5AuthHandler.java          — AUTH packet for enhanced auth (MQTT 5.0 only)
    MqttSharedSubHandler.java      — $share/<group>/<filter> → ConsumerGroupCoordinator
    MqttRetainedMessageStore.java  — retained message storage + on-subscribe replay
    MqttTopicMatcher.java          — wildcard matching: '+' (single-level), '#' (multi-level)
    MqttSessionManager.java        — session state lifecycle (clean vs persistent)
    MqttSessionState.java          — per-connection: will, subscriptions, QoS 2 state,
                                     topic aliases (5.0), session-expiry, receive-maximum
    MqttProtocolBundle.java        — implements ProtocolBundle
                                     detection: first byte 0x10 (CONNECT control packet)
                                     version (3.1.1 vs 5.0) resolved inside CONNECT handler
                                     ports: 1883/1884 (plain), 8883/8884 (TLS)
```

### Version Negotiation Detail

MQTT 3.1.1 and 5.0 share the same `MqttFrameDecoder` and `MqttRequestHandler`.
Version is detected inside `MqttConnectHandler` by reading the protocol-level byte from the CONNECT payload:

```java
// Inside MqttConnectHandler.handle(MqttConnectData connect):
MqttVersion version = switch (connect.protocolLevel()) {
    case 4  -> MqttVersion.V3_1_1;
    case 5  -> MqttVersion.V5;
    default -> throw new UnsupportedProtocolVersionException(connect.protocolLevel());
};
ctx.channel().attr(MQTT_VERSION_KEY).set(version);
```

After CONNECT, `MqttRequestHandler` reads the version from the channel attribute and delegates
to version-appropriate sub-handlers where behaviour differs (properties parsing, reason codes, etc.).

### ServiceLoader Registration
```
META-INF/services/com.ivy.common.protocol.ProtocolBundle:
  com.ivy.protocol.mqtt.handler.MqttProtocolBundle
```

---

## ivy-protocol-postgresql

**Purpose:** PgWire read-only SQL view of broker state.

**Dependencies:** `ivy-common`, `ivy-broker`, Netty

### Package Structure
```
com.ivy.protocol.pg/
  PgWireCodec.java             — PostgreSQL wire protocol encode/decode
  PgStartupMessage.java        — StartupMessage + SSLRequest handling
  PgWireNamespaceStrategy.java — implements DestinationParser
  PgParseData.java             — PARSE (prepared statement)
  PgBindData.java              — BIND + EXECUTE
  PgMessage.java               — sealed: Query, Parse, Bind, Execute, Describe, Sync, Terminate

  handler/
    PgWireFrameDecoder.java        — ByteToMessageDecoder: startup vs regular message framing
    PgWireRequestHandler.java      — main dispatcher
    PgWireStartupHandler.java      — StartupMessage, SSLRequest, Auth (MD5/SCRAM)
    PgWireSimpleQueryHandler.java  — Query message → SqlQueryParser → execute → DataRow response
    PgWirePreparedStmtHandler.java — Parse/Bind/Execute/Describe flow
    PgWireQueryExecutor.java       — SqlQueryParser → BrokerEngine.fetch() or PG metadata query
    SqlQueryParser.java            — minimal SQL parser: SELECT, SHOW TABLES, DESCRIBE
    SqlQuery.java                  — sealed: SelectTopic, ShowTables, DescribeTable,
                                     SelectMetadata, Unsupported
    PgWireSessionState.java        — per-connection: auth state, parameter status map,
                                     prepared statements
    PgWireProtocolBundle.java      — implements ProtocolBundle
                                     detection: first 4 bytes = length, bytes[4-7] = 0x00030000
                                     (protocol version 3.0 = 196608)
                                     ports: 5432 (plain), 5433 (TLS)
```

### ServiceLoader Registration
```
META-INF/services/com.ivy.common.protocol.ProtocolBundle:
  com.ivy.protocol.pg.handler.PgWireProtocolBundle
```

---

## ivy-protocol-mysql

**Purpose:** MySQL wire read-only SQL view of broker state.

**Dependencies:** `ivy-common`, `ivy-broker`, Netty

### Package Structure
```
com.ivy.protocol.mysql/
  MysqlWireCodec.java          — MySQL wire protocol encode/decode
  MySqlNamespaceStrategy.java  — implements DestinationParser
  MySqlColumnDef.java          — column metadata (ColumnDefinition41)
  MysqlAuthResponse.java       — auth handshake response data

  handler/
    MysqlFrameDecoder.java         — ByteToMessageDecoder: [length:3][seq:1][payload] framing
    MysqlRequestHandler.java       — main dispatcher
    MySqlHandshakeHandler.java     — HandshakeV10 → HandshakeResponse41 → OK/ERR
    MySqlComQueryHandler.java      — COM_QUERY text protocol → SqlQueryParser
    MySqlComStmtPrepareHandler.java — COM_STMT_PREPARE (prepared statements)
    MySqlComStmtExecuteHandler.java — COM_STMT_EXECUTE (binary protocol)
    MySqlQueryExecutor.java        — SqlQueryParser → BrokerEngine.fetch() or PG metadata query
    SqlQueryParser.java            — shared with pg: SELECT, SHOW TABLES, DESCRIBE
    MySqlSessionState.java         — per-connection: auth state, capabilities
    MySqlProtocolBundle.java       — implements ProtocolBundle
                                     detection: server-speaks-first HandshakeV10
                                       (seq=0, packet_type=0x0A)
                                     ports: 3306 (plain), 3307 (TLS)
```

### ServiceLoader Registration
```
META-INF/services/com.ivy.common.protocol.ProtocolBundle:
  com.ivy.protocol.mysql.handler.MySqlProtocolBundle
```

---

## ivy-server

**Purpose:** Thin assembly. Auto-discovers protocols via `ProtocolBundleRegistry`. No handler logic.

**Dependencies:** All modules above.

### Package Structure
```
com.ivy.server/
  BrokerMain.java                    — entry point, manual IoC assembly (no DI framework)
  NettyBrokerServer.java             — Netty ServerBootstrap, EventLoopGroup lifecycle
  NettyPipelineFactory.java          — builds per-connection pipeline using ProtocolBundleRegistry
  PerProtocolBootstrap.java          — one ServerBootstrap per port; codec pre-wired, no detection
  FlowControlHandler.java            — channel backpressure (high/low watermarks)
  ServerExceptionHandler.java        — catch-all, log + graceful close
  GracefulShutdown.java              — drain → release partitions → close connections → exit
  BrokerConfigWatcher.java           — hot-reload YAML config

  tenant/
    SniTenantResolver.java           — SNI hostname → deterministic TenantId (no interface, final)
    TenantResolverHandler.java       — Netty handler: TLS SNI extraction → channel attr
    TenantContextHandler.java        — Netty handler: TenantRegistry lookup + status enforcement
    TenantRegistry.java              — dual-indexed cache: by TenantId + by SNI hostname
    TenantRecord.java                — identity + status + config + audit timestamps
    TenantLifecycleManager.java      — create / activate / suspend (8 steps) / delete (10 steps)
    TenantConfig.java                — per-tenant overrides: TLS, auth, quotas, namespace
    TlsTenantConfig.java             — per-tenant cert + key + CA + mTLS mode
    AuthTenantConfig.java            — per-tenant SASL mechanisms + OAuth config
    QuotaTenantConfig.java           — per-tenant byte/request/connection rate limits
    TenantPersistence.java           — interface: save/load from __tenants internal topic
    InMemoryTenantPersistence.java   — implementation backed by TenantStore
    TenantConfigLoader.java          — parses broker.yaml tenants: section

  suspension/
    TenantSuspensionService.java     — interface: 8 suspension operations
    DefaultTenantSuspensionService.java — coordinates drain + flush + purge
    TenantChannelRegistry.java       — tracks active channels per tenant + drainTenant()

  tls/
    SslContextPool.java              — ConcurrentHashMap<TenantId, AtomicReference<SslContext>>
    CertificateWatcher.java          — hot-reload certs without dropping connections
```

### BrokerMain Assembly

```java
public static void main(String[] args) {
    BrokerConfig config = ConfigParser.parse(args);
    Environment  env    = new DefaultEnvironment();

    // --- Storage ---
    var dataSource    = HikariDataSourceFactory.create(config.storage());
    var storageEngine = new PostgresStorageEngine(dataSource);
    storageEngine.migrateSchema();

    var logSegmentStore = new LogSegmentStore(config.storage().dataDir());
    var storageFlusher  = new StorageFlusher(logSegmentStore);

    // --- Broker ---
    var metadataImage    = new MetadataImageHolder(MetadataImage.empty());
    var hrwRouter        = new HRWRouter(config.cluster().clusterSecret(), metadataImage);
    var writeAccumulator = new WriteAccumulator(config.broker().write());
    var writeWorker      = new WriteWorker(storageEngine, storageFlusher, writeAccumulator);
    var readAccumulator  = new ReadAccumulator(logSegmentStore, storageEngine);
    var dlqRouter        = new DlqRouter(config.broker().dlq());
    var groupCoordinator = new ConsumerGroupCoordinator(storageEngine, env);
    var txCoordinator    = new TransactionCoordinator(storageEngine, env);
    var authEngine       = new DefaultAuthEngine(dataSource);
    var clusterManager   = new ClusterManager(config, dataSource, metadataImage, hrwRouter, env);
    var forwardMgr       = new ForwardWriteManager(hrwRouter, clusterManager.rpcClients());

    var brokerEngine = new DefaultBrokerEngine(
        writeAccumulator, writeWorker, readAccumulator, dlqRouter,
        groupCoordinator, txCoordinator, hrwRouter, forwardMgr, storageEngine);

    // --- Protocol auto-discovery ---
    // Each ivy-protocol-* module registers itself in META-INF/services/
    var protocolRegistry = ProtocolBundleRegistry.fromServiceLoader();
    log.info("Loaded {} protocol bundles: {}",
        protocolRegistry.listAll().size(),
        protocolRegistry.listAll().stream().map(b -> b.protocolId().name()).toList());

    // Inject broker dependencies into each bundle's handlers
    protocolRegistry.listAll().forEach(bundle ->
        bundle.injectDependencies(brokerEngine, authEngine, config));

    // --- Netty ---
    var pipelineFactory = new NettyPipelineFactory(protocolRegistry, config);
    var server          = new NettyBrokerServer(pipelineFactory, config);

    // --- Lifecycle ---
    clusterManager.start();
    server.start();
    Runtime.getRuntime().addShutdownHook(new Thread(() -> {
        clusterManager.drainAndStop();
        server.stop();
        dataSource.close();
    }));
}
```

### Netty Pipeline (per connection)

```
[SslHandler]               optional, per-port TLS config
[TenantResolverHandler]    SNI hostname → TenantId (stored as ChannelAttr)
[ConnectionLimitHandler]   max connections per tenant
[Protocol codec]           pre-installed by ServerBootstrap at bind time (port = protocol)
[FlowControlHandler]       high/low watermark backpressure
[ServerExceptionHandler]   catch-all, log + graceful close
```

---

## Protocol Dependencies Summary

| Module | ivy-common | ivy-storage | ivy-broker | Netty | External |
|--------|-----------|------------|-----------|-------|----------|
| ivy-protocol-kafka | ✓ | — | ✓ | ✓ | `kafka-clients:4.2.0` (test only) |
| ivy-protocol-amqp | ✓ | — | ✓ | ✓ | `protonj2:1.1.0` (AMQP 1.0 types) |
| ivy-protocol-mqtt | ✓ | — | ✓ | ✓ | none (pure Java) |
| ivy-protocol-postgresql | ✓ | — | ✓ | ✓ | none (pure Java) |
| ivy-protocol-mysql | ✓ | — | ✓ | ✓ | `mysql-connector-j` (test only) |

---

## Full File Structure

```
autoresearch-v8/
├── pom.xml                              # parent POM, BOM
├── ivy-common/
│   └── src/main/java/com/ivy/common/
│       ├── types/                       # branded value records
│       ├── engine/                      # BrokerEngine, StorageEngine, AuthEngine
│       ├── protocol/                    # ProtocolBundle, ByteBufCodec, ProtocolBundleRegistry,
│       │                                # InternalRequest (sealed), CodecContext
│       ├── model/                       # PendingWrite, WriteResult, FetchResult, FlushEvent
│       ├── config/                      # BrokerConfig, ClusterConfig, StorageConfig
│       └── env/                         # Environment, DefaultEnvironment, SimulatedEnvironment
│
├── ivy-storage/
│   └── src/main/java/com/ivy/storage/
│       ├── segment/                     # LogSegment, MetadataSegment, LogSegmentStore
│       ├── index/                       # OffsetIndex (mmap'd)
│       ├── postgres/                    # PostgresStorageEngine
│       ├── flush/                       # StorageFlusher
│       └── migration/                   # SchemaManager + V{n}__*.sql
│
├── ivy-broker/
│   └── src/main/java/com/ivy/broker/
│       ├── engine/                      # DefaultBrokerEngine
│       ├── write/                       # WriteAccumulator, WriteWorker
│       ├── read/                        # ReadAccumulator, SubscriptionRegistry
│       ├── dlq/                         # DlqRouter, DlqHeaderBuilder, DlqTopicManager
│       ├── consumer/                    # ConsumerGroupCoordinator, OffsetManager
│       ├── transaction/                 # TransactionCoordinator, ProducerStateManager
│       ├── auth/                        # DefaultAuthEngine, ScramAuthenticator, AclAuthorizer
│       └── cluster/                     # HRWRouter, ClusterManager, HeartbeatWriter/Monitor,
│                                        # BrokerFencingPipeline, ForwardWriteManager,
│                                        # InterBrokerRpcServer/Client, MetadataImage
│
├── ivy-protocol-kafka/
│   └── src/main/java/com/ivy/protocol/kafka/
│       ├── KafkaCodec.java
│       ├── KafkaRequestParser.java
│       ├── KafkaResponseSerializer.java
│       ├── KafkaFraming.java
│       ├── KafkaVersions.java
│       ├── KafkaNamespaceStrategy.java
│       └── handler/
│           ├── KafkaRequestDispatcher.java
│           ├── KafkaProduceHandler.java
│           ├── KafkaFetchHandler.java
│           ├── KafkaMetadataHandler.java
│           ├── KafkaGroupHandler.java
│           ├── KafkaTransactionHandler.java
│           ├── KafkaAdminHandler.java
│           ├── KafkaSaslAuthHandler.java
│           └── KafkaProtocolBundle.java
│
├── ivy-protocol-amqp/
│   └── src/main/java/com/ivy/protocol/
│       ├── amqp/                        # AMQP 0-9-1
│       │   ├── Amqp091Codec.java
│       │   ├── AmqpFrame.java
│       │   ├── Amqp091NamespaceStrategy.java
│       │   ├── Amqp*Data.java           # method data classes
│       │   └── handler/
│       │       ├── Amqp091FrameDecoder.java
│       │       ├── Amqp091RequestHandler.java
│       │       ├── Amqp091ConnectionHandler.java
│       │       ├── Amqp091ExchangeHandler.java
│       │       ├── Amqp091QueueHandler.java
│       │       ├── Amqp091PublishHandler.java
│       │       ├── Amqp091ConsumeHandler.java
│       │       ├── Amqp091AckHandler.java
│       │       ├── Amqp091TxHandler.java
│       │       ├── Amqp091SessionState.java
│       │       └── Amqp091ProtocolBundle.java
│       └── amqp10/                      # AMQP 1.0
│           ├── Amqp10Codec.java
│           ├── Amqp10TypeCodec.java
│           ├── Amqp10MessageCodec.java
│           ├── Amqp10FrameHeader.java
│           ├── Amqp10NamespaceStrategy.java
│           └── handler/
│               ├── Amqp10FrameDecoder.java
│               ├── Amqp10RequestHandler.java
│               ├── Amqp10ConnectionHandler.java
│               ├── Amqp10SessionHandler.java
│               ├── Amqp10SenderLinkHandler.java
│               ├── Amqp10ReceiverLinkHandler.java
│               ├── Amqp10SessionState.java
│               └── Amqp10ProtocolBundle.java
│
├── ivy-protocol-mqtt/
│   └── src/main/java/com/ivy/protocol/mqtt/
│       ├── MqttFraming.java
│       ├── MqttPacketType.java
│       ├── MqttVersion.java
│       ├── MqttQoS.java
│       ├── MqttConnectCodec.java
│       ├── MqttPublishCodec.java
│       ├── MqttSubscribeCodec.java
│       ├── MqttSimpleCodec.java
│       ├── Mqtt5PropertiesCodec.java
│       ├── MqttNamespaceStrategy.java
│       ├── Mqtt*Data.java               # packet data classes
│       └── handler/
│           ├── MqttFrameDecoder.java
│           ├── MqttRequestHandler.java  # version-dispatches at CONNECT
│           ├── MqttConnectHandler.java  # shared 3.1.1 + 5.0
│           ├── MqttPublishHandler.java  # QoS 0/1/2 state machine
│           ├── MqttSubscribeHandler.java
│           ├── MqttUnsubscribeHandler.java
│           ├── MqttPingHandler.java
│           ├── MqttDisconnectHandler.java
│           ├── Mqtt5AuthHandler.java    # 5.0 only: AUTH packet
│           ├── MqttSharedSubHandler.java # 5.0: $share/ → ConsumerGroupCoordinator
│           ├── MqttRetainedMessageStore.java
│           ├── MqttTopicMatcher.java
│           ├── MqttSessionManager.java
│           ├── MqttSessionState.java
│           └── MqttProtocolBundle.java
│
├── ivy-protocol-postgresql/
│   └── src/main/java/com/ivy/protocol/pg/
│       ├── PgWireCodec.java
│       ├── PgStartupMessage.java
│       ├── PgWireNamespaceStrategy.java
│       ├── PgMessage.java               # sealed message types
│       ├── Pg*Data.java                 # Parse, Bind, Execute data
│       └── handler/
│           ├── PgWireFrameDecoder.java
│           ├── PgWireRequestHandler.java
│           ├── PgWireStartupHandler.java
│           ├── PgWireSimpleQueryHandler.java
│           ├── PgWirePreparedStmtHandler.java
│           ├── PgWireQueryExecutor.java
│           ├── SqlQueryParser.java
│           ├── SqlQuery.java            # sealed
│           ├── PgWireSessionState.java
│           └── PgWireProtocolBundle.java
│
├── ivy-protocol-mysql/
│   └── src/main/java/com/ivy/protocol/mysql/
│       ├── MysqlWireCodec.java
│       ├── MySqlNamespaceStrategy.java
│       ├── MySqlColumnDef.java
│       ├── MysqlAuthResponse.java
│       └── handler/
│           ├── MysqlFrameDecoder.java
│           ├── MysqlRequestHandler.java
│           ├── MySqlHandshakeHandler.java
│           ├── MySqlComQueryHandler.java
│           ├── MySqlComStmtPrepareHandler.java
│           ├── MySqlComStmtExecuteHandler.java
│           ├── MySqlQueryExecutor.java
│           ├── SqlQueryParser.java
│           ├── MySqlSessionState.java
│           └── MySqlProtocolBundle.java
│
├── ivy-server/
│   └── src/main/java/com/ivy/server/
│       ├── BrokerMain.java
│       ├── NettyBrokerServer.java
│       ├── NettyPipelineFactory.java
│       ├── PerProtocolBootstrap.java
│       ├── TenantResolverHandler.java
│       ├── ConnectionLimitHandler.java
│       ├── FlowControlHandler.java
│       ├── ServerExceptionHandler.java
│       ├── GracefulShutdown.java
│       ├── BrokerConfigWatcher.java
│       └── suspension/
│           └── SuspensionManager.java
│
├── ivy-testing/
│   └── src/main/java/com/ivy/testing/
│       ├── IvyBrokerContainer.java      # Testcontainers singleton
│       ├── ProtocolClientFactory.java   # configured clients for all 7 protocols
│       └── fixtures/                    # common test data builders
│
├── ivy-lint/
│   └── src/main/java/com/ivy/lint/
│       ├── TenantSqlIsolationRule.java  # compile-time: SQL must include tenant_id
│       ├── SealedSwitchExhaustivenessRule.java  # no default on sealed switch
│       ├── NullReturnRule.java          # no null returns from public methods
│       ├── RawPrimitiveRule.java        # no raw UUID/long in public signatures
│       └── EnvironmentUsageRule.java    # no direct System.currentTimeMillis()
│
└── ivy-schema/
    └── src/main/java/com/ivy/schema/
        ├── SchemaRegistry.java          # Confluent SR REST v1 compatible
        ├── AvroSchemaParser.java
        ├── JsonSchemaParser.java
        ├── ProtobufSchemaParser.java
        ├── CompatibilityValidator.java  # BACKWARD, FORWARD, FULL, NONE
        └── SchemaProtocolBundle.java    # ServiceLoader registration
```

---

## Future Modules (Designed, Not Yet Implemented)

| Module | Purpose | Status |
|--------|---------|--------|
| `ivy-lake` | S3-compatible chunk storage (8MB fixed chunks, FileChannel I/O) for object storage protocol | Designed |
| `ivy-codegraph` | Bytecode-level dependency analysis for selective test execution and lint | Designed |
