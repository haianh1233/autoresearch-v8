# Implementation Phases

## Phase 0 — Foundation (ivy-common + ivy-storage)

**Goal:** All types, interfaces, and storage working. No broker logic yet.

### ivy-common
- [ ] All branded type value records (`TenantId`, `PartitionId`, `TopicId`, `BrokerId`, `Offset`, `TopicName`, `GroupId`, `Port`, `ProducerEpoch`, `LeaderEpoch`, `ProducerId`)
- [ ] `PendingWrite`, `WriteResult`, `FetchResult`, `FlushEvent` value records
- [ ] `BrokerEngine`, `StorageEngine`, `AuthEngine` sealed interfaces
- [ ] `BrokerConfig`, `ClusterConfig`, `StorageConfig` config records
- [ ] `Environment` abstraction (clock, idGenerator, scheduler)
- [ ] `DefaultEnvironment`, `SimulatedEnvironment`
- [ ] `ErrorCode` sealed interface (all error types)
- [ ] `ProtocolId` enum (KAFKA=1, AMQP=2, MQTT=3, MYSQL=4, PGWIRE=5)

### ivy-storage
- [ ] `PostgresStorageEngine` — binary COPY append, SELECT fetchRange, highWatermark (defense-in-depth: `AND tenant_id = ?`)
- [ ] `SchemaManager` — V1__initial_schema.sql; version table; idempotent
- [ ] `LogSegmentState` — enum: ACTIVE, SEALED, FLUSHED, EXPIRED, DELETED
- [ ] `LogSegment` — append, read, seal (`FileChannel.force(true)`), markFlushed(), markForDeletion(); CRC32C per record
- [ ] `OffsetIndex` — mmap'd (lazy: `ensureMapped()`), sparse (1 entry/4 KB), warm section (first 1024 entries in `long[]` heap cache), append, filePositionFor, truncateToActualSize on seal
- [ ] `LogSegmentStore` — `ConcurrentHashMap<PartitionId, ConcurrentSkipListMap<Long, LogSegment>>`; `floorSegment()`, `openNewSegment()`, `sealedUnflushed()`
- [ ] `MetadataSegment` — compacted variant of LogSegment (used by internal topics)
- [ ] `StorageFlusher` — 200 ms cycle; `ThreadLocal<Connection>` (dedicated PG conn); `flush(seg)`: COPY → COMMIT → `markFlushed()` in that order; `drainAll(timeout)` for shutdown
- [ ] `SegmentCleaner` — 60 s cycle; 5-state lifecycle; disk budget (85 % reject / 70 % resume); 30 s deletion delay; minimum-1-segment guarantee per partition
- [ ] `DiskBudgetMonitor` — `usagePercent(dataDir)` via `FileStore.getUsableSpace()`
- [ ] HikariCP pool setup (pool size = worker-threads + 1 flusher + 1 read)

### Tests
- [ ] `PostgresStorageEngineIT` — Testcontainers PG: append 10K, fetchRange, highWatermark; tenant isolation (wrong tenantId returns empty)
- [ ] `LogSegmentTest` — append, read, OffsetIndex lookup, CRC mismatch detection, warm-section vs mmap path, multi-segment spanning
- [ ] `StorageFlusherTest` — markFlushed called only after COMMIT; crash-between-commit-and-mark is idempotent (duplicate PK rejected)
- [ ] `SegmentCleanerTest` — 30 s delay enforced; min-segment guarantee; disk-budget eviction
- [ ] `SchemaManagerIT` — migration idempotency, re-run is no-op

---

## Phase 1 — Single-Broker + Kafka Protocol

**Goal:** Fully functional single-broker with Kafka compatibility.

### ivy-broker (engine only, no clustering)
- [ ] `WriteAccumulator` — per-partition swap-drain (lock-free `AtomicReference`); flush triggers 1K/1MB/5ms; `suspendTenant()` / `unsuspendTenant()`; `submitImmediate()` bypass for QoS-2 / publisher-confirm
- [ ] `WriteWorker` — 4 **platform** threads (not virtual); `ThreadLocal<Connection>` for PG affinity; 2-layer epoch fence (local MetadataImage + PG `WHERE leader_epoch = ?`); binary COPY; `drainAll(timeout)` on shutdown
- [ ] `ProducerStateManager` — sliding window (last 5 batches) per (producerId, partitionId); duplicate → return cached offset; gap → `OUT_OF_ORDER_SEQUENCE`; stale epoch → `INVALID_PRODUCER_EPOCH`
- [ ] `FlushEventDispatcher` — 1 platform thread; `MpscArrayQueue<FlushEvent>` (JCTools, capacity 65536); 1 ms park when idle; 3 dispatch actions: `pendingFetchRegistry.notifyWaiters`, `subscriptionManager.broadcast`, `clusterNotifier.notifyPeers`; queue-full → drop event (safe)
- [ ] `FlushEvent` — value record: `partitionId, baseOffset, endOffset, entryCount, timestampMs`
- [ ] `PendingFetchRegistry` — `ConcurrentHashMap<PartitionId, CopyOnWriteArrayList<CompletableFuture<Boolean>>>`; `notifyWaiters(partitionId)` called by dispatcher; virtual thread parks on `future.join()`; timeout via scheduler `future.complete(false)`
- [ ] `ReadAccumulator` — L1 LogSegment + L3 PG fallback (no L2 yet); multi-partition via `StructuredTaskScope` (5 s timeout, partial OK); HWM invariant checked before and after; read-through: L3 results populate L1 via `logSegmentStore.appendBatch()`
- [ ] `IsolationFilter` — 4-gate chain: (1) tenant match, (2) control record skip, (3a) READ_UNCOMMITTED pass, (3b) READ_COMMITTED: LSO gate + aborted-transaction gate; applied per-record during segment scan
- [ ] `TransactionLsoTracker` — `ConcurrentSkipListMap<PartitionId, ConcurrentSkipListMap<Long startOffset, Set<Long producerIds>>>`; `getLso()` = `firstKey()` O(1)
- [ ] `AbortedTransactionTracker` — `CopyOnWriteArrayList<AbortedRange>` per partition; `isAborted(pid, producerId, offset)`; `getAbortedRanges(pid, from, to)` for KIP-98 client field
- [ ] `CompactionFilter` — two-pass (build key→maxOffset map, then filter); tombstone retention via `deleteRetentionMs`; enabled when `partitionId ∈ compactedPartitions`
- [ ] `SubscriptionRegistry` — register/unregister `ConsumerHandle`; `broadcast(partitionId, from, to)`
- [ ] `DefaultBrokerEngine` — single-broker mode (no HRW yet); virtual-thread executor for `read()`; validates `SecurityContext` before every call
- [ ] `ConsumerGroupCoordinator` — EMPTY→PREPARE_REBALANCE→COMPLETING_REBALANCE→STABLE→DEAD
- [ ] `TransactionCoordinator` — InitProducerId, AddPartitions, EndTxn (commit advances LSO, abort records `AbortedRange`)
- [ ] `DefaultAuthEngine` — SCRAM-SHA-256, PLAIN, AclStore

### ivy-codec (Kafka only)
- [ ] `KafkaRequestDecoder` — all API keys listed in PROTOCOLS.md
- [ ] `KafkaResponseEncoder`
- [ ] `KafkaApiVersions`

### ivy-server (Kafka only)
- [ ] `BrokerMain` — single-broker assembly
- [ ] `NettyPipelineFactory` — Kafka only
- [ ] `ProtocolDetector` — Kafka magic bytes
- [ ] `KafkaRequestHandler` — Produce, Fetch, Metadata, ListOffsets
- [ ] `KafkaConsumerGroupHandler` — JoinGroup, SyncGroup, Heartbeat, LeaveGroup, OffsetCommit/Fetch
- [ ] `KafkaTransactionHandler` — InitProducerId, AddPartitions, EndTxn, TxnOffsetCommit
- [ ] `KafkaAdminHandler` — CreateTopics, DeleteTopics, DescribeConfigs
- [ ] `GracefulShutdown`

### Tests
**kafka-clients version: `4.2.0` (stable release — not 4.3-SNAPSHOT)**
- [ ] `KafkaProducerConsumerE2E` — produce 10K, consume, verify all offsets
- [ ] `KafkaConsumerGroupE2E` — join group, rebalance, commit offsets
- [ ] `KafkaTransactionE2E` — produce transactionally, commit + abort; READ_COMMITTED hides in-progress records; aborted records not visible after EndTxn(ABORT)
- [ ] `KafkaAdminE2E` — create topic, describe, delete
- [ ] `WriteWorkerIT` — PG-first write, 2-layer epoch fence, duplicate detection (sliding window), stale epoch rejection
- [ ] `LongPollE2E` — fetch with maxWaitMs=2000; producer writes after 500ms; consumer returns in <1s
- [ ] `IsolationFilterTest` — 4-gate chain unit test: tenant mismatch rejected, control records, LSO gate, aborted range gate
- [ ] `CompactionFilterTest` — latest-per-key retained; tombstone within deleteRetentionMs kept; tombstone after deleteRetentionMs dropped
- [ ] `ReadThroughCacheIT` — L3 miss populates L1; second read hits L1

---

## Phase 2 — AMQP 0-9-1 + AMQP 1.0 + MQTT 3.1.1 + MQTT 5.0 + DLQ

**Goal:** Four messaging protocols + dead letter queues working end to end.

### ivy-codec (AMQP + MQTT — all variants)
- [ ] `amqp/Amqp091FrameDecoder` / `Amqp091FrameEncoder` / `Amqp091MethodCodec`
- [ ] `amqp10/Amqp10FrameDecoder` / `Amqp10FrameEncoder` / `Amqp10TypeCodec` / `Amqp10MessageCodec`
- [ ] `mqtt/MqttDecoder` / `MqttEncoder` — MQTT 3.1.1 (version byte = 0x04)
- [ ] `mqtt/Mqtt5PropertiesCodec` — MQTT 5.0 extended properties (version byte = 0x05)

### ivy-broker (DLQ)
- [ ] `DlqRouter` — trigger evaluation, header injection, DLQ partition routing
- [ ] `DlqHeaderBuilder` — x-dlq-* headers
- [ ] `DlqTopicManager` — auto-create `__dlq.<topic>`
- [ ] `DlqConfig` — per-topic configuration
- [ ] V2__add_dlq_entries.sql migration

### ivy-server (AMQP + MQTT handlers — separate packages)
**AMQP 0-9-1** (`handler/amqp091/`):
- [ ] `Amqp091RequestHandler` — main dispatcher
- [ ] `Amqp091ConnectionHandler`, `Amqp091ChannelHandler`
- [ ] `Amqp091ExchangeHandler`, `Amqp091QueueHandler`
- [ ] `Amqp091BasicHandler` — Publish/Consume/Get/Ack/Nack/Reject
- [ ] `Amqp091ConfirmHandler` — publisher confirms
- [ ] `Amqp091TxHandler` — Tx.Select/Commit/Rollback
- [ ] `Amqp091SessionState` — per-channel exchanges, queues, consumers, confirms

**AMQP 1.0** (`handler/amqp10/`):
- [ ] `Amqp10RequestHandler` — main dispatcher
- [ ] `Amqp10ConnectionHandler` — Open/Close + SASL exchange
- [ ] `Amqp10SessionHandler` — Begin/End
- [ ] `Amqp10SenderLinkHandler` — Attach(sender), Transfer, Disposition
- [ ] `Amqp10ReceiverLinkHandler` — Attach(receiver), Flow credit, Disposition
- [ ] `Amqp10SessionState` — per-session links, delivery tracking, flow control

**MQTT 3.1.1** (`handler/mqtt/`):
- [ ] `Mqtt311RequestHandler` — main dispatcher
- [ ] `MqttConnectHandler` — CONNECT/CONNACK (shared with 5.0)
- [ ] `MqttPublishHandler` — PUBLISH + QoS 0/1/2 state machine
- [ ] `MqttSubscribeHandler`, `MqttUnsubscribeHandler`
- [ ] `MqttSessionState` — will, subscriptions, QoS 2 state

**MQTT 5.0** (`handler/mqtt/` — extends 3.1.1):
- [ ] `Mqtt5RequestHandler` — extends `Mqtt311RequestHandler`, version-dispatched at CONNECT
- [ ] `Mqtt5AuthHandler` — AUTH packet exchange
- [ ] `MqttSharedSubHandler` — `$share/` prefix → ConsumerGroupCoordinator
- [ ] MQTT 5.0 user-properties → Ivy headers mapping
- [ ] Message-Expiry-Interval → DLQ (TTL_EXPIRED)
- [ ] Topic alias resolution in `MqttSessionState`

- [ ] Update `ProtocolDetector` — AMQP version discrimination (bytes 4-7), MQTT version read from CONNECT payload
- [ ] Update `NettyPipelineFactory` — 5 messaging protocol handlers

### Tests
- [ ] `Amqp091PublishConsumeE2E` — exchange types (direct/fanout/topic), queue bind, ack
- [ ] `Amqp091DlqE2E` — nack(requeue=false) → `__dlq.<topic>`; x-death header present
- [ ] `Amqp091TtlDlqE2E` — x-message-ttl expires → DLQ reason=TTL_EXPIRED
- [ ] `Amqp091PublisherConfirmsE2E` — Confirm.Select, Basic.Ack per message
- [ ] `Amqp10PublishConsumeE2E` — Attach sender + receiver, Transfer, Disposition(accepted)
- [ ] `Amqp10DlqE2E` — Disposition(rejected) → `__dlq.<topic>`
- [ ] `Amqp10FlowControlE2E` — link-credit exhaustion → broker pauses, credit refresh → resumes
- [ ] `Mqtt311Qos0E2E`, `Mqtt311Qos1E2E`, `Mqtt311Qos2E2E`
- [ ] `Mqtt311RetainedE2E` — retained message delivered on subscribe
- [ ] `Mqtt311WillE2E` — unexpected disconnect → will published
- [ ] `Mqtt5UserPropertiesE2E` — publish with user-properties → consumed via Kafka with headers
- [ ] `Mqtt5SharedSubE2E` — `$share/group/topic` distributes to multiple consumers
- [ ] `Mqtt5MessageExpiryDlqE2E` — message-expiry-interval exceeded → DLQ
- [ ] `Mqtt5EnhancedAuthE2E` — AUTH packet SCRAM-SHA-256 exchange
- [ ] `DlqRouterTest` — all 5 trigger conditions unit-tested
- [ ] `CrossProtocolAmqpKafkaE2E` — produce via AMQP 0-9-1, consume via Kafka
- [ ] `CrossProtocolMqttAmqp10E2E` — produce via MQTT 5.0, consume via AMQP 1.0

---

## Phase 3 — MySQL + PgWire SQL Protocols (Read-Only) + HTTP REST

**Goal:** SQL-based read-only view of broker state + HTTP produce/consume API.

### ivy-codec (MySQL + PgWire)
- [ ] `MySqlHandshakeEncoder` / `MySqlPacketDecoder`
- [ ] `MySqlResultSetEncoder` — ColumnDefinition41, row encoding
- [ ] `PgStartupDecoder` / `PgMessageDecoder` / `PgMessageEncoder`
- [ ] `PgRowDescEncoder` / `PgDataRowEncoder`

### ivy-server (SQL handlers)
- [ ] `MySqlRequestHandler` — handshake, COM_QUERY dispatch
- [ ] `MySqlQueryExecutor` — SqlQueryParser → BrokerEngine.fetch() or PG metadata query
- [ ] `PgWireRequestHandler` — startup, simple query dispatch
- [ ] `PgWireQueryExecutor` — same as MySQL but PgWire encoding
- [ ] `SqlQueryParser` — sealed SqlQuery hierarchy
- [ ] Update `ProtocolDetector` for MySQL + PgWire
- [ ] Update `NettyPipelineFactory` for all 7 protocols (add MySQL, PgWire, HTTP)

### ivy-protocol-http
- [ ] `HttpProtocolBundle` — SPI registration, `codec()` returns null, `httpHandler()` returns dispatcher
- [ ] `HttpRequestDispatcher` — Netty `ChannelInboundHandlerAdapter`, routes on HTTP method + path
- [ ] `HttpRouteTable` — compile-time path → handler mapping
- [ ] `HttpProduceHandler` — `POST /topics/{topic}/messages`
  - JSON body → `PendingWrite` via `HttpMessageCodec`
  - Explicit partition / key-hash / round-robin routing
  - Response: `202 Accepted` with offset + partition
- [ ] `HttpBatchProduceHandler` — `POST /topics/{topic}/messages/batch`
  - Array of messages → `List<PendingWrite>`
  - `207 Multi-Status` on partial failure
- [ ] `HttpConsumeHandler` — `GET /topics/{topic}/messages`
  - `offset`, `limit`, `partition`, `isolation` query params
  - `waitMs` → long-poll via `FlushEventDispatcher`
  - Response: JSON array with base64 key/value + headers + protocolId
- [ ] `HttpTopicMetadataHandler` — `GET /topics`, `GET /topics/{topic}`, `GET /topics/{topic}/partitions`
- [ ] `HttpAuthExtractor` — `Authorization: Bearer` / `X-API-Key` extraction
- [ ] `HttpTenantResolver` — `X-Tenant-Id` header or JWT `tenant` claim
- [ ] `HttpMessageCodec` — JSON ↔ `PendingWrite` / `FetchResult`, base64 encode/decode
- [ ] Dedicated `ServerBootstrap` on port 8081 with `HttpServerCodec + HttpObjectAggregator`
- [ ] Error mapping: `InternalErrorCode` → HTTP status codes

### Tests
- [ ] `MySqlBrokerE2E` — SHOW TABLES, SELECT from topic, SELECT cluster state via JDBC
- [ ] `PgWireBrokerE2E` — SELECT from topic, SELECT partitions, consumer_groups via psql
- [ ] `SqlQueryParserTest` — all supported SQL patterns, unsupported → SqlQuery.Unsupported
- [ ] `HttpProduceConsumeE2E` — produce via HTTP, consume via HTTP, verify offset + value
- [ ] `HttpBatchProduceE2E` — batch produce, verify all offsets returned
- [ ] `HttpLongPollE2E` — waitMs=5000; producer publishes after 1s; consumer returns in ~1s
- [ ] `HttpAuthE2E` — valid Bearer token → 202; expired token → 401; missing → 401
- [ ] `HttpCrossProtocolE2E` — produce via Kafka, consume via HTTP; produce via HTTP, consume via Kafka
- [ ] `KafkaProduceHttpConsumeE2E` — Kafka producer (with re-auth) → HTTP consumer sees messages
- [ ] `AmqpProduceSqlConsumeE2E` — AMQP 0-9-1 publish → MySQL SELECT sees messages (protocol_id=2)
- [ ] `MqttProduceSqlConsumeE2E` — MQTT 5.0 publish → PgWire SELECT sees messages (protocol_id=5)
- [ ] `HttpProduceAmqpConsumeE2E` — HTTP POST → AMQP Basic.Deliver
- [ ] `HttpProduceMqttConsumeE2E` — HTTP POST → MQTT PUBLISH to subscriber

---

## Phase 4 — Clustering

**Goal:** Multi-broker cluster with partition leader election, failover, write forwarding.

### ivy-broker (cluster components)
- [ ] `HRWRouter` — HMAC-SHA-256 rendezvous hash
- [ ] `MetadataImage` + `MetadataImageHolder` — VarHandle atomic update
- [ ] `MetadataPoller` — 30s PG poll to rebuild MetadataImage
- [ ] `ClusterManager` — STARTING→ACTIVE→DRAINING lifecycle
- [ ] `HeartbeatWriter` — 3s periodic `UPDATE broker_registry`
- [ ] `HeartbeatMonitor` — 5s check for stale brokers
- [ ] `BrokerFencingPipeline` — CAS fence → release → broadcast → re-elect
- [ ] `ForwardWriteManager` — ForwardWriteRequest routing
- [ ] `InterBrokerRpcServer` — Netty inbound on inter_broker_port
- [ ] `InterBrokerRpcClient` — per-peer outbound, reconnect backoff
- [ ] `InterBrokerMessage` — sealed: ForwardWrite, ForwardFetch, MetadataBroadcast, Heartbeat
- [ ] Update `WriteWorker` — epoch-fenced PG transaction, `WrongEpochException` handling
- [ ] Update `DefaultBrokerEngine` — cluster-aware write routing
- [ ] Update `ReadAccumulator` — L2 inter-broker fetch
- [ ] V3__add_cluster_tables.sql — (broker_registry already in V1, verify epoch fields)

### Tests
- [ ] `HRWRouterTest` — determinism, ~1/N rebalancing on topology change
- [ ] `ClusterFailoverE2E` — 3-broker cluster, kill leader, verify failover, no message loss
- [ ] `WriteForwardingE2E` — produce to non-owner, verify forwarded and committed
- [ ] `SplitBrainPreventionTest` — epoch fencing prevents stale leader writes
- [ ] `MetadataConvergenceTest` — after topology change, all brokers converge on same MetadataImage
- [ ] `GracefulShutdownE2E` — drain broker under load, verify no message loss

---

## Phase 5 — Auth, Re-Auth, Observability, Polish

**Goal:** Production-ready authentication, re-authentication, metrics, and operational tooling.

### ivy-broker (auth complete)
- [ ] `ScramAuthenticator` — SCRAM-SHA-256 full handshake
- [ ] `AclStore` — acl_entries CRUD + evaluation
- [ ] `AclAuthorizer` — deny by default, resource+operation matching
- [ ] `TokenBucketQuotaManager` — per-principal produce/consume quota
- [ ] V4__add_auth_tables.sql — credentials, acl_entries

### ivy-broker (re-auth — all protocols)
- [ ] `ReAuthManager` — per-connection CAS state machine (AUTHENTICATED → RE_AUTH_REQUIRED → IN_PROGRESS → AUTHENTICATED/FAILED)
- [ ] `ReAuthScheduler` — session lifetime timer with jitter `[0, 5000ms)`, `reAuthBufferMs=30s`
- [ ] `CredentialRevocationHandler` — reactive push on `__credentials` change; `ConnectionRegistry` lookup
- [ ] `ConnectionRegistry` — `(TenantId, username) → Set<ConnectionId>` for revocation fan-out
- [ ] `SessionLifetimeCalculator` — `min(maxReauthMs, tokenRemainingMs)` logic
- [ ] V5__add_reauth_config.sql — per-tenant `max_reauth_ms` in tenants table

**Protocol-specific re-auth:**
- [ ] Kafka: extend `SaslHandshakeHandler` + `SaslAuthenticateHandler` to support re-auth on existing connection (KIP-368)
  - `SaslAuthenticateResponse.sessionLifetimeMs` set from `SessionLifetimeCalculator`
  - New `SaslHandshake` on existing authenticated connection → `ReAuthManager.beginReAuth()`
  - In-flight `Produce` requests queued in `WriteAccumulator` during re-auth round-trips
- [ ] MQTT 5.0: `Mqtt5AuthHandler` — broker-initiated `AUTH(0x19)` re-auth + `AUTH(0x18)` exchange
- [ ] MySQL: `MySqlComChangeUserHandler` — `COM_CHANGE_USER` → `ReAuthManager` transition
- [ ] AMQP 0-9-1: `Amqp091ReAuthHandler` — `Connection.Close(320, "re-authenticate")` on session expiry
- [ ] AMQP 1.0: `Amqp10ReAuthHandler` — `close(error=unauthorized-access, description=session-expired)`
- [ ] MQTT 3.1.1: `MqttReAuthHandler` — send `DISCONNECT` + close TCP on expiry (Category C)

### ivy-server (metrics + health)
- [ ] Micrometer registry + Prometheus scrape endpoint (`/metrics`)
- [ ] Key metrics:
  - `ivy_write_latency_ms` (p50, p99 histogram)
  - `ivy_read_latency_ms`
  - `ivy_dlq_messages_total` (by topic, reason)
  - `ivy_broker_status` (ACTIVE=1, else 0)
  - `ivy_partition_ownership_count`
  - `ivy_consumer_group_lag` (by group, partition)
  - `ivy_reauth_total{protocol, result}` — re-auth attempts by protocol + outcome
  - `ivy_session_lifetime_ms` — histogram of actual session durations
- [ ] Health check endpoint (`GET /health` → `{"status":"UP","broker":"ACTIVE"}`)
- [ ] Readiness endpoint (`GET /ready` → 200 once PG schema migrated + cluster joined)
- [ ] `ConnectionMetrics` — active connections per protocol

### Tests
- [ ] `AuthE2E` — SCRAM-SHA-256 Kafka auth, PLAIN AMQP auth, MQTT username/password
- [ ] `AclE2E` — produce denied, consume denied, admin denied
- [ ] `QuotaE2E` — produce throttled at configured rate
- [ ] `MetricsE2E` — verify Prometheus metrics populated after produce/consume
- [ ] `KafkaReAuthE2E` — produce with short `maxReauthMs`; verify re-auth fires; producer continues without disconnect
- [ ] `KafkaReAuthProduceSqlConsumeE2E` — Kafka producer re-auths mid-stream; MySQL/PgWire SQL client reads all produced messages including those produced during re-auth window
- [ ] `AmqpReAuthGracefulCloseE2E` — AMQP session expires; broker sends `Connection.Close(320)`; client reconnects and resumes
- [ ] `MqttReAuthE2E` — MQTT 5.0 broker-initiated `AUTH(0x19)` re-auth; subscription resumes after success
- [ ] `CredentialRevocationE2E` — revoke user credentials; all active connections for that user are closed within `revocationPollMs`
- [ ] `TenantImmutabilityE2E` — re-auth attempt with different tenant credentials → `ILLEGAL_SASL_STATE` + disconnect
- [ ] `ReAuthSchedulerTest` — session lifetime calculation: token TTL, maxReauthMs, expiry buffer, jitter bounds

---

## Cross-Cutting Concerns (all phases)

### Lint rules (enforced at compile time via annotation processor)
- No `System.currentTimeMillis()` → use `Environment.clock()`
- No `UUID.randomUUID()` → use `Environment.idGenerator()`
- No `Thread.sleep()` → use `Environment.scheduler()`
- No raw `UUID` / `String` across module APIs → use branded types
- All `switch` on sealed interfaces must be exhaustive
- No `null` returns from public methods → use `Optional` or throw
- Tenant ID must be present in all SQL queries that touch messages/partitions

### Testing standards
- Unit tests (`*Test.java`): no Docker, no network, `SimulatedEnvironment`, 5s timeout
- Integration tests (`*IT.java`): Testcontainers PG, real network, 60s timeout
- E2E tests (`*E2E.java`): full broker stack via Testcontainers, 120s timeout
- All tests: deterministic seed for random IDs

### Performance targets

| Metric | Target |
|--------|--------|
| Single-broker write throughput (1 partition) | 20K msg/s |
| Single-broker write throughput (16 partitions) | 200K msg/s |
| p99 write latency (PG local) | < 15ms |
| p99 read latency (LogSegment hit) | < 2ms |
| Cluster failover time | < 15 seconds |
| Time to first message after broker restart | < 5 seconds |
