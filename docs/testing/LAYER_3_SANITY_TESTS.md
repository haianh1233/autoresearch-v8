# Layer 3: Sanity Tests (Fast Integration)

> **Related:** [../TESTING_STRATEGY.md](../TESTING_STRATEGY.md)

---

## Overview

Sanity tests are **fast end-to-end integration tests** running against an in-process broker
with a shared PostgreSQL Testcontainer. They validate core protocol functionality for all 15+
protocols in ~60 seconds total.

**Naming:** `Sanity*IT.java` with `@Tag("sanity")`
**Execution:** `mvn test -Pe2e,sanity -pl e2e`

---

## SanityTestBase Architecture

Singleton pattern: one PG container + one in-process broker per JVM, shared across all test classes.

```java
class SanityTestBase {
    static PostgreSQLContainer<?> pg;
    static TestBrokerLauncher broker;
    static final Object BROKER_LOCK = new Object();
    static final int MESSAGE_COUNT = 1000;

    @BeforeAll
    static void startBroker() {
        synchronized (BROKER_LOCK) {
            if (pg == null || !pg.isRunning()) {
                pg = new PostgreSQLContainer<>("postgres:17-alpine").withReuse(true);
                pg.start();
            }
            if (broker == null) {
                broker = TestBrokerLauncher.start(pg.getJdbcUrl());
                Runtime.getRuntime().addShutdownHook(new Thread(() -> broker.close()));
            }
        }
    }

    String uniqueTopic(String prefix) {
        return "sanity-" + prefix + "-" + UUID.randomUUID().toString().substring(0, 8);
    }
}
```

**Timing:**
- Cold start (first run): ~3-5s (PG pull + broker init)
- Warm start (reuse enabled): ~1-2s (PG already running)
- Total suite (90 tests): ~60s

---

## Complete Test Inventory (31 classes, ~90 tests)

### Core Protocol Tests

| Class | Tests | Key Scenarios |
|-------|-------|---------------|
| `SanityKafkaIT` | 8 | Produce/consume 1K msgs, compression (gzip/snappy/lz4/zstd), admin CRUD, consumer groups, offset commit/fetch |
| `SanityMqttIT` | 4 | QoS 0/1, single-level wildcard (`+`), multi-level wildcard (`#`) |
| `SanityAmqp091IT` | 6 | Direct/fanout/topic exchange, publisher confirms, DLX, multi-channel |
| `SanityAmqp10IT` | 4 | Link credit flow control, multiple sessions, at-least-once/exactly-once settlement |
| `SanityNatsIT` | 5 | Pub/sub, subject wildcards, queue groups, request/reply |
| `SanityStompIT` | 6 | Send/subscribe, receipts, ACK modes (client/auto), transaction commit/abort |
| `SanityRedisStreamsIT` | 4 | XADD/XREAD, XREADGROUP, XACK/XCLAIM, XPENDING |
| `SanityPgWireIT` | 4 | Simple query, prepared statements, transactions, COPY protocol |
| `SanityMySqlIT` | 3 | COM_QUERY result set, prepared statements, COM_PING |
| `SanityOpenWireIT` | 3 | Persistent/non-persistent, message selectors, exclusive consumer |
| `SanityPulsarIT` | 4 | Exclusive/shared subscription, failover, dead letter topic |
| `SanityRmqStreamsIT` | 3 | Publish/confirm, offset tracking, credit-based flow |
| `SanityZeroMqIT` | 4 | PUB/SUB, PUSH/PULL, REQ/REP, multipart |
| `SanityKinesisIT` | 4 | Put/Get 1K records (HTTP API) |
| `SanityPubSubIT` | 5 | Publish/pull (HTTP API) |
| `SanityS3IT` | 4 | Put/get, range read, listObjectsV2, multipart upload |

### Cross-Protocol Tests

| Class | Tests | Key Scenarios |
|-------|-------|---------------|
| `SanityCrossProtocolProduceConsumeIT` | 5 | Kafka→MQTT, MQTT→NATS, STOMP→AMQP, NATS→Kafka, AMQP10→MQTT+NATS (fan-out) |
| `SanityCrossProtocolTxnIT` | 3 | Kafka txn commit→MQTT visible, abort→AMQP empty, interleaved txn+non-txn |
| `SanityCrossProtocolHttpIT` | 3 | HTTP→Kafka, Kafka→HTTP, HTTP→MQTT |
| `SanityCrossProtocolTenantIT` | 3 | Tenant isolation (Kafka→MQTT), same topic different data, MQTT wildcard scoped |

### Feature-Specific Tests

| Class | Tests | Key Scenarios |
|-------|-------|---------------|
| `SanityTransactionIT` | 8 | Single txn commit/abort, multi-partition, epoch bump, zombie fencing, READ_COMMITTED |
| `SanityPartitionMappingIT` | 14 | Per-protocol partition consistency, key-based routing, round-robin, edge cases |
| `SanityLingerTimerIT` | 2 | Single message linger flush (5ms), batch size flush (1K msgs) |
| `SanitySchemaRegistryIT` | 3 | Register/lookup, backward compatibility enforcement, compatible evolution |
| `SanitySecurityIT` | 4 | Bad credentials rejected, garbage bytes close, oversized frame rejected, SCRAM auth |
| `SanityAclIT` | 2 | Deny overrides allow, unauthorized produce blocked |
| `SanityHttpMessagingIT` | 3 | HTTP pipeline produce, 404 not found, PubSub path separation |
| `SanityHttpRestIT` | 3 | Health/topic list, readiness/metrics, tenant CRUD |
| `SanityKafkaStreamsMultiGenIT` | 1 | KTable join across processor generations |

---

## Assertion Patterns

### Message Count

```java
assertThat(received).hasSize(MESSAGE_COUNT);            // exact 1000
assertThat(received.size()).isGreaterThanOrEqualTo(90);  // QoS 0: ≥90%
```

### Partition Ordering

```java
for (var r : received) {
    byPartition.computeIfAbsent(r.partition(), _ -> new ArrayList<>()).add(r.offset());
}
// Per-partition offsets must be strictly increasing
byPartition.values().forEach(offsets ->
    assertThat(offsets).isSorted());
```

### Cross-Protocol Delivery

```java
// MQTT subscribe first (push model needs subscription before produce)
mqttClient.subscribe(topic);
// Kafka produce
kafkaProducer.send(new ProducerRecord<>(topic, "key", "value")).get();
// Assert MQTT receives within timeout
assertThat(latch.await(10, SECONDS)).isTrue();
```

---

## Timing Baselines

| Scenario | Deadline | Notes |
|----------|----------|-------|
| Produce 1K + consume all | 15s | Includes linger (5ms) + consumer poll lag |
| Single message linger flush | 5s | Linger timer: 5ms; overhead < 500ms |
| Consumer group rebalance | 10-15s | Join + partition assignment |
| Schema registration | 5s | HTTP to Schema Registry |
| Cross-protocol delivery | 10s | Routing + push delivery |

---

*Last updated: 2026-03-25*
