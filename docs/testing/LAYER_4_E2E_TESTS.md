# Layer 4: Full E2E Tests (Docker)

> **Related:** [../TESTING_STRATEGY.md](../TESTING_STRATEGY.md)

---

## Overview

Full E2E tests run the broker inside a **Docker container** with PostgreSQL on a shared Docker
network. They validate production-like deployment, multi-tenant SNI routing, schema validation,
and storage backends.

**Naming:** `Docker*IT.java` with `@Tag("e2e")`
**Execution:** `mvn verify -Pe2e -pl e2e` (~15-20 min)

---

## ContainerTestBase Architecture

```java
class ContainerTestBase {
    static Network sharedNetwork = Network.newNetwork();
    static PostgreSQLContainer<?> pg;
    static IvyBrokerContainer broker;

    @BeforeAll
    static void startContainers() {
        synchronized (BROKER_LOCK) {
            if (broker != null && broker.isRunning()) return;

            pg = new PostgreSQLContainer<>("postgres:16-alpine")
                .withNetwork(sharedNetwork)
                .withNetworkAliases("postgres");
            pg.start();

            broker = new IvyBrokerContainer("ivy-e2e:latest")
                .withNetwork(sharedNetwork)
                .withJdbcUrl("jdbc:postgresql://postgres:5432/ivy?user=ivy&password=ivy")
                .withKafkaPort(findFreePort())
                .withEnv("KAFKA_ADVERTISED_HOST", "localhost")
                .withEnv("BROKER_KAFKA_NO_SASL", "true")
                .waitingFor(Wait.forLogMessage(".*Broker started.*", 1)
                    .withStartupTimeout(Duration.ofSeconds(120)));
            broker.start();
        }
    }
}
```

---

## IvyBrokerContainer Configuration

### Protocol Ports (Container-Internal)

| Protocol | Port | Protocol | Port |
|----------|------|----------|------|
| Kafka | 9092 | Pulsar | 6650 |
| MQTT | 1883 | RMQ Streams | 5552 |
| AMQP 0-9-1 | 5672 | ZeroMQ | 5555 |
| AMQP 1.0 | 5673 | S3 | 9000 |
| NATS | 4222 | HTTP | 8080 |
| STOMP | 61613 | Admin/SR | 8081 |
| OpenWire | 61616 | PgWire | 5432 |
| Redis | 6379 | MySQL | 3306 |

### Builder Methods

```java
broker = new IvyBrokerContainer()
    .withMultiTenant("tenant1.localhost", "tenant2.localhost")  // SNI + self-signed SAN cert
    .withSchemaValidation()                                     // Confluent wire format gate
    .withPostgresBackend(jdbcUrl)                               // PG storage engine
    .withDataVolume(hostPath)                                   // persistent storage for recovery
    .withKafkaPort(9093);                                       // custom Kafka port
```

---

## Complete Test Inventory (~200 tests)

### Core Protocol E2E

| Class | Tests | Focus |
|-------|-------|-------|
| `DockerKafkaIT` | 29 | Full Kafka: produce/consume, transactions, idempotent producer, HWM accuracy, consumer groups, offset commit |
| `DockerPushProtocolIT` | 18 | MQTT QoS, NATS, STOMP, AMQP push delivery, retained messages, will messages, AMQP transactions |
| `DockerDataProtocolIT` | 9 | OpenWire, Redis Streams, PgWire, MySQL, Pulsar, ZeroMQ |
| `DockerHttpProtocolIT` | 5 | Kinesis, Google Pub/Sub, HTTP cross-protocol, health, metrics |
| `DockerHttpProduceConsumeIT` | 4 | HTTP→Kafka, Kafka→HTTP, batch produce |

### Cross-Protocol

| Class | Tests | Focus |
|-------|-------|-------|
| `DockerCrossProtocolIT` | 5 | Kafka↔MQTT, NATS↔Kafka, Kafka→Redis |
| `DockerSharedTopicIT` | 5 | All 15 protocols writing to single topic, ordering guarantees |
| `DockerSqlProtocolIT` | 7 | PgWire/MySQL as message sinks/sources, Kafka→SQL, SQL→Kafka |

### Multi-Tenant

| Class | Tests | Focus |
|-------|-------|-------|
| `DockerMultiTenantIT` | 14 | SNI routing, data isolation, offset independence, consumer group isolation |
| `DockerSniMultiTenantIT` | 7 | SNI via Kafka SSL (port 9093), per-tenant certificate |
| `DockerSingleTenantIT` | 5 | Default mode, no namespace prefix |

### Transactions & Schema

| Class | Tests | Focus |
|-------|-------|-------|
| `DockerTransactionIsolationIT` | 17 | READ_UNCOMMITTED vs READ_COMMITTED, LSO pinning, epoch fencing, exactly-once |
| `DockerSchemaRegistryIT` | 24 | Avro/Protobuf/JSON Schema, compatibility evolution, multi-tenant isolation |
| `DockerSchemaValidationGateIT` | 7 | Magic byte validation, unknown schema ID rejection, bypass for non-Confluent |

### Infrastructure

| Class | Tests | Focus |
|-------|-------|-------|
| `DockerPostgresBackendIT` | 4 | PG storage path, health, topic creation |
| `DockerStandaloneIT` | 3 | Single container, no dependencies |
| `DockerTieredStorageS3IT` | 3 | Tier-1 hot, S3 migration, cold storage read |
| `DockerSecurityIT` | 4 | PROXY protocol, brute-force lockout, delegation tokens |
| `DockerKafkaStreamsMultiGenIT` | 1 | Topology upgrades, state store migration |
| `DockerPartitionMappingIT` | 13 | Partition routing consistency across all protocols |

---

## Multi-Tenant E2E Detail

### Certificate Generation

```java
// Self-signed SAN cert for SNI routing
broker.withMultiTenant("tenant1.localhost", "tenant2.localhost");
// Generates: DNS.1=localhost, DNS.2=tenant1.localhost, DNS.3=tenant2.localhost
// Mounts to /certs in container
```

### Key Isolation Assertions

```java
@Test void sameTopicName_dataIsolated() {
    // Tenant A writes 2 msgs → Tenant B reads 0 msgs on same topic name
}

@Test void offsetsIndependent() {
    // Tenant A: offsets 0-4, Tenant B: offsets 0-2 (independent counters)
}

@Test void consumerGroups_isolated() {
    // Same group ID, different tenants → independent offset tracking
}
```

---

## Transaction Isolation E2E Detail

### Cross-Protocol Visibility

```
1. MQTT subscriber subscribed (push model)
2. Kafka txn producer opens transaction, sends 5 msgs
3. MQTT receives all 5 immediately (READ_UNCOMMITTED default)
4. Kafka READ_COMMITTED consumer blocks (LSO pins at transaction start)
5. Kafka producer commits → LSO advances → RC consumer unblocks
```

### Key Assertions

```java
@Test void mqtt_defaultRU_seesOpenTransaction() { /* MQTT sees uncommitted */ }
@Test void kafka_readCommitted_blockedByLso() { /* Kafka RC blocks until commit */ }
@Test void abort_rcNeverSeesMessages() { /* Aborted txn invisible to RC */ }
@Test void disconnect_autoAbort_lsoUnpins() { /* TCP drop → auto-abort → LSO advances */ }
@Test void exactlyOnce_noDuplicates() { /* Epoch fencing + idempotent producer */ }
```

---

## Timing

| Phase | Duration |
|-------|----------|
| PG container startup | 3-5s |
| Broker container startup | 10-15s |
| Individual test | 1-3s (Kafka) / 5-10s (transactions, schema) |
| Full suite (~200 tests) | ~15-20 min |

---

*Last updated: 2026-03-25*
