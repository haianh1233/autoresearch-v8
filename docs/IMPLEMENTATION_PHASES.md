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
- [ ] `PostgresStorageEngine` — binary COPY append, SELECT fetchRange, highWatermark
- [ ] `SchemaManager` — V1__initial_schema.sql migration
- [ ] `LogSegment` — append, read, seal, delete
- [ ] `OffsetIndex` — mmap'd, append, filePositionFor
- [ ] `MetadataSegment` — compacted variant of LogSegment
- [ ] `LogSegmentStore` — lifecycle management (open/seal/delete)
- [ ] `StorageFlusher` — async 200ms background flush
- [ ] HikariCP pool setup

### Tests
- [ ] `PostgresStorageEngineIT` — Testcontainers PG: append 10K, fetchRange, highWatermark
- [ ] `LogSegmentTest` — append, read, seal, OffsetIndex lookup
- [ ] `SchemaManagerIT` — migration idempotency

---

## Phase 1 — Single-Broker + Kafka Protocol

**Goal:** Fully functional single-broker with Kafka compatibility.

### ivy-broker (engine only, no clustering)
- [ ] `WriteAccumulator` — per-partition batching (1K/1MB/5ms)
- [ ] `WriteWorker` — 4 threads, PG-first transaction (UPDATE + COPY + COMMIT + ACK)
- [ ] `ReadAccumulator` — L1 LogSegment + L3 PG fallback (no L2 yet)
- [ ] `FlushEventDispatcher` — push notify subscribers after write
- [ ] `SubscriptionRegistry` — register/unregister consumer handles
- [ ] `DuplicateDetector` — producer_state idempotency check
- [ ] `DefaultBrokerEngine` — single-broker mode (no HRW yet)
- [ ] `ConsumerGroupCoordinator` — EMPTY→PREPARE_REBALANCE→COMPLETING→STABLE→DEAD
- [ ] `TransactionCoordinator` — InitProducerId, AddPartitions, EndTxn
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
- [ ] `KafkaProducerConsumerE2E` — produce 10K, consume, verify all offsets
- [ ] `KafkaConsumerGroupE2E` — join group, rebalance, commit offsets
- [ ] `KafkaTransactionE2E` — produce transactionally, commit + abort
- [ ] `KafkaAdminE2E` — create topic, describe, delete
- [ ] `WriteWorkerIT` — PG-first write, epoch validation, duplicate detection

---

## Phase 2 — AMQP + MQTT + DLQ

**Goal:** Two more protocols + dead letter queues working end to end.

### ivy-codec (AMQP + MQTT)
- [ ] `AmqpFrameDecoder` / `AmqpFrameEncoder`
- [ ] `AmqpMethodCodec` — all AMQP 0-9-1 methods
- [ ] `MqttDecoder` / `MqttEncoder` — all MQTT 3.1.1 packet types

### ivy-broker (DLQ)
- [ ] `DlqRouter` — trigger evaluation, header injection, DLQ partition routing
- [ ] `DlqHeaderBuilder` — x-dlq-* headers
- [ ] `DlqTopicManager` — auto-create `__dlq.<topic>`
- [ ] `DlqConfig` — per-topic configuration
- [ ] V2__add_dlq_entries.sql migration

### ivy-server (AMQP + MQTT handlers)
- [ ] `AmqpRequestHandler` — Connection, Channel, Exchange, Queue, Basic, Confirm, Tx
- [ ] `AmqpSessionState` — per-channel state (exchanges, queues, consumers)
- [ ] `MqttRequestHandler` — Connect, Publish, Subscribe, Unsubscribe, Disconnect
- [ ] `MqttSessionState` — cleanSession, will, subscriptions
- [ ] Update `ProtocolDetector` for AMQP + MQTT
- [ ] Update `NettyPipelineFactory` for all 3 protocols

### Tests
- [ ] `AmqpPublishConsumeE2E` — exchange declare, queue bind, publish, ack
- [ ] `AmqpDlqE2E` — nack(requeue=false) → verify message in `__dlq.<topic>`
- [ ] `AmqpTtlDlqE2E` — message TTL expires → DLQ with reason=TTL_EXPIRED
- [ ] `MqttQos0E2E`, `MqttQos1E2E`, `MqttQos2E2E`
- [ ] `MqttRetainedE2E` — retained message delivered on subscribe
- [ ] `MqttWillE2E` — unexpected disconnect → will published
- [ ] `DlqRouterTest` — all 5 trigger conditions unit-tested
- [ ] `CrossProtocolE2E` — produce via Kafka, consume via AMQP

---

## Phase 3 — MySQL + PgWire SQL Protocols (Read-Only)

**Goal:** SQL-based read-only view of broker state.

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
- [ ] Update `NettyPipelineFactory` for all 5 protocols

### Tests
- [ ] `MySqlBrokerE2E` — SHOW TABLES, SELECT from topic, SELECT cluster state via JDBC
- [ ] `PgWireBrokerE2E` — SELECT from topic, SELECT partitions, consumer_groups via psql
- [ ] `SqlQueryParserTest` — all supported SQL patterns, unsupported → SqlQuery.Unsupported

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

## Phase 5 — Auth, Observability, Polish

**Goal:** Production-ready authentication, metrics, and operational tooling.

### ivy-broker (auth complete)
- [ ] `ScramAuthenticator` — SCRAM-SHA-256 full handshake
- [ ] `AclStore` — acl_entries CRUD + evaluation
- [ ] `AclAuthorizer` — deny by default, resource+operation matching
- [ ] `TokenBucketQuotaManager` — per-principal produce/consume quota
- [ ] V4__add_auth_tables.sql — credentials, acl_entries

### ivy-server (metrics + health)
- [ ] Micrometer registry + Prometheus scrape endpoint (`/metrics`)
- [ ] Key metrics:
  - `ivy_write_latency_ms` (p50, p99 histogram)
  - `ivy_read_latency_ms`
  - `ivy_dlq_messages_total` (by topic, reason)
  - `ivy_broker_status` (ACTIVE=1, else 0)
  - `ivy_partition_ownership_count`
  - `ivy_consumer_group_lag` (by group, partition)
- [ ] Health check endpoint (`GET /health` → `{"status":"UP","broker":"ACTIVE"}`)
- [ ] Readiness endpoint (`GET /ready` → 200 once PG schema migrated + cluster joined)
- [ ] `ConnectionMetrics` — active connections per protocol

### Tests
- [ ] `AuthE2E` — SCRAM-SHA-256 Kafka auth, PLAIN AMQP auth, MQTT username/password
- [ ] `AclE2E` — produce denied, consume denied, admin denied
- [ ] `QuotaE2E` — produce throttled at configured rate
- [ ] `MetricsE2E` — verify Prometheus metrics populated after produce/consume

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
