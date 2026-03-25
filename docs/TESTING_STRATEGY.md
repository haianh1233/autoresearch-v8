# Testing Strategy

> **Related:** [IMPLEMENTATION_PHASES.md](IMPLEMENTATION_PHASES.md), [OBSERVABILITY.md](OBSERVABILITY.md)

---

## Test Pyramid (6 Tiers)

```
              ┌──────────────┐
              │  Benchmark   │  JMH microbenchmarks, stress
              ├──────────────┤
           │  Complex E2E    │  Chaos, durability, soak (>30min)
           ├─────────────────┤
         │   Sanity Tests     │  Fast integration, in-process (~60s)
         ├────────────────────┤
       │    Full E2E Tests     │  All protocols, Docker, PG (~15min)
       ├───────────────────────┤
     │  Integration Tests       │  Cross-module, real PG, in-memory broker
     ├──────────────────────────┤
   │  Unit / Fast Integration    │  Pure logic, mocks, no I/O (<5s/class)
   └─────────────────────────────┘
```

| Tier | Command | Duration | When |
|------|---------|----------|------|
| Unit + Fast Integration | `mvn clean install` | ~2 min | Every build |
| Integration | `mvn verify -pl ivy-testing` | ~5-10 min | PR merge |
| Sanity | `mvn test -Pe2e,sanity -pl e2e` | ~60s cold, ~20s warm | Every commit |
| Full E2E | `mvn verify -Pe2e -pl e2e` | ~15-20 min | PR merge |
| Complex E2E | `mvn verify -Pcomplex-e2e -pl e2e` | ~30-45 min | Nightly |
| Benchmark | `mvn verify -Pbenchmark -pl benchmark` | ~10-15 min | Weekly |

---

## Test Base Classes

### SanityTestBase (In-Process Broker)

Singleton PostgreSQL container + shared `TestBrokerLauncher` per JVM:

```java
class SanityTestBase {
    static PostgreSQLContainer<?> pg;          // reused via testcontainers.reuse.enable=true
    static TestBrokerLauncher broker;           // shared singleton per JVM
    static final Object BROKER_LOCK = new Object();

    @BeforeAll
    static void startBroker() {
        synchronized (BROKER_LOCK) {
            if (broker == null) {
                pg = new PostgreSQLContainer<>("postgres:17-alpine").withReuse(true);
                pg.start();
                broker = TestBrokerLauncher.start(pg.getJdbcUrl());
            }
        }
    }

    String uniqueTopic() { return "test-" + UUID.randomUUID().toString().substring(0, 8); }
    static final int MESSAGE_COUNT = 1000;   // baseline for all sanity tests
}
```

### ContainerTestBase (Docker-Based)

Singleton `IvyBrokerContainer` + PostgreSQL on shared Docker network:

```java
class ContainerTestBase {
    static Network sharedNetwork = Network.newNetwork();
    static PostgreSQLContainer<?> pg;
    static IvyBrokerContainer broker;

    @BeforeAll
    static void startContainers() {
        pg = new PostgreSQLContainer<>("postgres:17-alpine")
            .withNetwork(sharedNetwork).withNetworkAliases("postgres");
        pg.start();
        broker = new IvyBrokerContainer("ivy-e2e:latest")
            .withNetwork(sharedNetwork)
            .withJdbcUrl("jdbc:postgresql://postgres:5432/ivy")
            .waitingFor(Wait.forLogMessage(".*Broker started.*", 1)
                .withStartupTimeout(Duration.ofSeconds(120)));
        broker.start();
    }
}
```

### TestBrokerLauncher (In-Process Factory)

```java
class TestBrokerLauncher {
    static TestBrokerLauncher start(String jdbcUrl);          // PG backend
    static TestBrokerLauncher startInMemory();                 // no PG
    static TestBrokerLauncher startWithDataDir(Path dataDir);  // crash recovery testing

    int port(String protocol);         // random OS-assigned port
    String kafkaBootstrap();           // "localhost:{kafkaPort}"
    String mqttUri();                  // "tcp://localhost:{mqttPort}"

    void setSimulateDiskFull(boolean); // inject disk-full errors
    void setPreserveDataDir(boolean);  // keep data for recovery tests
}
```

---

## Test Tags

| Tag | Purpose | Example |
|-----|---------|---------|
| `@Tag("kafka")` | Kafka protocol tests | `SanityKafkaIT` |
| `@Tag("mqtt")` | MQTT protocol tests | `SanityMqttIT` |
| `@Tag("amqp091")` | AMQP 0-9-1 tests | `SanityAmqp091IT` |
| `@Tag("amqp10")` | AMQP 1.0 tests | `SanityAmqp10IT` |
| `@Tag("cross-protocol")` | Cross-protocol routing | `CrossProtocolE2ETest` |
| `@Tag("multi-tenant")` | Multi-tenancy isolation | `MultiTenancyIsolationE2ETest` |
| `@Tag("security")` | Auth, ACL, TLS | `SaslAllMechanismsE2ETest` |
| `@Tag("failure")` | Chaos/failure injection | `BrokerCrashRecoveryE2ETest` |
| `@Tag("stress")` | Load/scalability | `ConnectionScalabilityE2ETest` |
| `@Tag("soak")` | Long-running (hours) | `SoakMultiProtocolLoadTest` |
| `@Tag("sanity")` | Fast integration | All `Sanity*IT` classes |
| `@Tag("e2e")` | Docker-based suite | All `Docker*IT` classes |

Run by tag: `mvn test -Pe2e -pl e2e -Dgroups=kafka`

---

## E2E Test Inventory

### Sanity Tests (~44 tests, ~60s total)

| Test Class | Tests | Protocol |
|-----------|-------|----------|
| `SanityKafkaIT` | 8 | Kafka produce/consume, compression, admin |
| `SanityMqttIT` | 4 | QoS 0/1, wildcards |
| `SanityAmqp091IT` | 6 | Direct/fanout/topic exchange, confirms, DLX |
| `SanityAmqp10IT` | 4 | Link credit, sessions, settlement |
| `SanityRedisStreamsIT` | 4 | XADD/XREAD, groups, XCLAIM |
| `SanityPgWireIT` | 4 | Simple/prepared query, transactions |
| `SanityMySqlIT` | 3 | COM_QUERY, prepared statements |
| `SanityCrossProtocolIT` | 5 | Kafka→MQTT, MQTT→Kafka, NATS→AMQP |
| `SanityTransactionIT` | 8 | Commit/abort, multi-partition, isolation |
| `SanityPartitionMappingIT` | 14 | Per-protocol partition consistency |

### Docker E2E Tests (~50 tests)

| Test Class | Tests | Focus |
|-----------|-------|-------|
| `DockerKafkaIT` | 4 | Produce/consume, admin, groups, offsets |
| `DockerPushProtocolIT` | 5 | MQTT/NATS/STOMP/AMQP push delivery |
| `DockerCrossProtocolIT` | 4 | Kafka↔MQTT, MQTT↔NATS, STOMP↔AMQP |
| `DockerSecurityIT` | 3 | PROXY protocol, brute-force, delegation tokens |
| `DockerMultiTenantIT` | 4 | SNI routing, tenant isolation |
| `DockerTransactionIsolationIT` | 4 | ACID isolation levels |
| `DockerHttpProduceConsumeIT` | 3 | HTTP→Kafka, Kafka→HTTP, batch |
| `DockerPostgresBackendIT` | 3 | PG storage path, health, metadata |

### Cross-Protocol Tests

| Test Class | Tests | Scenario |
|-----------|-------|---------|
| `CrossProtocolE2ETest` | 4 | Kafka↔MQTT, STOMP→AMQP, all-simultaneous |
| `CrossProtocolTransactionE2ETest` | 3 | Commit visible, abort invisible, READ_COMMITTED |
| `CrossProtocolConsumerGroupE2ETest` | 3 | Mixed protocol group, rebalance, exactly-once |

---

## Chaos Testing (Failure Injection)

| Failure | Test Class | Injection Method | Assertions |
|---------|-----------|-----------------|------------|
| **Broker crash** | `BrokerCrashRecoveryE2ETest` | `broker.close()` + `startWithDataDir()` | ACK'd writes durable, offsets continuous, clients reconnect |
| **Network partition** | `NetworkPartitionE2ETest` | `pg.getDockerClient().pauseContainerCmd()` | No duplicate writes, offsets consistent after heal |
| **Disk full** | `DiskFullE2ETest` | `broker.setSimulateDiskFull(true)` | Backpressure propagated, no corruption, reads work |
| **Segment corruption** | `CorruptSegmentE2ETest` | Bit-flip injection in segment reader | CRC detected, corrupt skipped, recovery truncates |
| **PG outage** | `StorageOutageE2ETest` | `pg.pause()` / `pg.unpause()` | Hot tier continues, circuit breaker activates, flush resumes |
| **Graceful shutdown** | `GracefulShutdownUnderLoadE2ETest` | `broker.shutdown()` with backlog | In-flight complete, new rejected, clean restart |

---

## Performance Benchmarks

### JMH Microbenchmarks

| Benchmark | Hot Path | Configuration |
|-----------|----------|--------------|
| `CodecBenchmark` | Kafka encode/decode | `@Fork(1) @Warmup(3×10s) @Measurement(3×10s)` |
| `WriteAccumulatorBenchmark` | Batch accumulation | Memory layout impact |

### E2E Throughput Tests

| Test | Workload | Target | Assertion |
|------|----------|--------|-----------|
| `kafka_sustained_throughput` | 10K msgs | >100 msg/s | `assertTrue(throughput > 100.0)` |
| `mqtt_sustained_throughput` | 1K msgs | >10 msg/s | `assertTrue(throughput > 10.0)` |
| `multiProtocol_concurrent` | 500×3 protocols | 1.5K total | All delivered |
| `latency_p99` | Mixed | <200ms | `assertTrue(p99 < 200ms)` |

### Performance Targets (from IMPLEMENTATION_PHASES.md)

| Configuration | Target |
|---------------|--------|
| Single broker, 1 partition, unbatched | ~20K msg/s |
| Single broker, 4 partitions, batched | ~80K msg/s |
| 3 brokers, 12 partitions | ~200K msg/s |
| Write latency p99 (local PG) | <15 ms |
| Read latency p99 (L1 hit) | <2 ms |
| Cluster failover | <15 s |

---

## Soak Tests (Long-Running)

| Test | Duration | Assertions |
|------|----------|------------|
| `SoakMultiProtocolLoadTest` | 24h | No memory leak, no FD leak, latency stable, GC pauses sub-ms |
| `SoakSegmentLifecycleTest` | 24h | Segment rotation safe, retention enforced, disk quota maintained |
| `SoakConsumerLagCatchupTest` | 1h | Lag accumulates, backfill catches up, no OOM |

---

## Testcontainers Setup

```java
// PostgreSQL (shared, reusable across test runs)
new PostgreSQLContainer<>("postgres:17-alpine")
    .withReuse(true)                          // ~/.testcontainers.properties: reuse.enable=true
    .withNetwork(sharedNetwork)
    .withNetworkAliases("postgres")
    .withDatabaseName("ivy").withUsername("ivy").withPassword("ivy");

// Broker container (Docker-based E2E)
new IvyBrokerContainer("ivy-e2e:latest")
    .withNetwork(sharedNetwork)
    .withEnv("KAFKA_ADVERTISED_HOST", "localhost")
    .withEnv("BROKER_KAFKA_NO_SASL", "true")  // test mode
    .waitingFor(Wait.forLogMessage(".*Broker started.*", 1));
```

**No ToxiProxy** — network partition simulated via Docker pause/unpause.

---

## JUnit Configuration

```properties
# junit-platform.properties
junit.jupiter.execution.timeout.mode=enabled
junit.jupiter.execution.timeout.thread.mode.default=SEPARATE_THREAD
junit.jupiter.execution.timeout.testable.method.default=20s
junit.jupiter.execution.timeout.beforeall.method.default=180s
junit.jupiter.execution.timeout.beforeeach.method.default=15s
junit.jupiter.execution.timeout.afterall.method.default=120s
```

---

## TDD Workflow (Ducktape Iteration Protocol)

1. **Red:** Write failing `ivy-testing` unit test that reproduces the issue
2. **Green:** Fix broker code until unit test passes (deterministic, fast feedback)
3. **Refactor:** Run expensive E2E/Ducktape to verify end-to-end (non-deterministic)
4. **Stabilize:** Multi-threaded tests must pass **3 consecutive runs** before commit

```bash
# Quick feedback loop (unit + sanity: ~60s)
mvn test -Pe2e,sanity -pl e2e -Dgroups=kafka

# Full validation (E2E: ~15min)
mvn verify -Pe2e -pl e2e

# Chaos validation (complex: ~30min)
mvn verify -Pcomplex-e2e -pl e2e -Dgroups=failure
```

---

*Last updated: 2026-03-25*
