# Layer 1: Unit Tests

> **Related:** [../TESTING_STRATEGY.md](../TESTING_STRATEGY.md), [../RULES.md](../RULES.md) §R19–R20

---

## Overview

Unit tests validate individual classes in isolation with no external dependencies (no PG, no
Docker, no network I/O). They run on every build in < 5 seconds per class.

**Naming:** `*Test.java` (Surefire plugin, `mvn test`)

---

## Mocking Strategy (No DI Framework)

Ivy uses **manual constructor injection** — no Spring, no Guice, no Mockito. Test doubles are
hand-written stubs implementing the real interfaces:

### Stub Pattern (Most Common)

```java
private static class StubStorageEngine implements StorageEngine {
    private final Map<String, Boolean> existsMap = new HashMap<>();
    private final Map<String, List<PartitionId>> partitionsMap = new HashMap<>();

    void register(String topic, boolean exists, List<PartitionId> partitions) {
        existsMap.put(topic, exists);
        partitionsMap.put(topic, partitions);
    }

    @Override public boolean topicExists(TenantId t, TopicName n) {
        return existsMap.getOrDefault(n.value(), false);
    }

    @Override public List<PartitionId> partitionsFor(TenantId t, TopicName n) {
        return partitionsMap.getOrDefault(n.value(), List.of());
    }

    // All other methods: no-op or null
}
```

### Recording Stub Pattern

Tracks method calls for verification:

```java
private static class RecordingStorageEngine implements StorageEngine {
    final List<TopicName> deletedTopics = new ArrayList<>();

    @Override public void deleteTopic(TenantId t, TopicName n) {
        deletedTopics.add(n);
    }
}

// In test:
assertThat(stub.deletedTopics).containsExactly(TopicName.of("orders"));
```

### Direct Construction

Tests instantiate handlers with explicit null or stub dependencies:

```java
@Test
void shouldHandleProduce() {
    var handler = new ProduceHandler(stubEngine, stubMetrics, null /* no ACL in unit test */);
    var result = handler.handle(request);
    assertThat(result.errorCode()).isEqualTo(0);
}
```

---

## Environment Abstraction (Deterministic Testing)

Per Rule R19, no direct calls to `System.currentTimeMillis()`, `UUID.randomUUID()`, or
`Thread.sleep()` in production code. Tests use:

- **Explicit timestamps:** `Timestamp.of(1700000000000L)` (fixed value)
- **UUID generation:** `UUID.randomUUID()` in tests is fine (tests don't depend on specific values)
- **`@TempDir` for file I/O:** JUnit 5 annotation for auto-cleaned temporary directories

```java
@TempDir
Path tempDir;

@Test
void shouldLookupExactOffset() throws Exception {
    Path indexPath = tempDir.resolve("test.index");
    try (var index = new OffsetIndex(indexPath)) {
        index.append(0, 0);
        index.append(100, 4096);
        assertThat(index.lookup(50).position()).isEqualTo(0);
    }
}
```

---

## Key Unit Test Classes

### Storage & Transactions

| Class | Tests | Focus |
|-------|-------|-------|
| `ProducerStateManagerTest` | 27 | Idempotency sliding window, epoch fencing, sequence validation, duplicate detection, window eviction |
| `OffsetIndexTest` | 5 | Binary search, boundary handling, sparse index lookup |
| `IsolationFilterTest` | 8 | Tenant isolation, control record filtering, READ_COMMITTED vs READ_UNCOMMITTED, LSO boundary |
| `LogSegmentTest` | ~10 | Append, read, CRC32C validation, segment rotation, file format |

### Codec & Protocol Handlers

| Class | Tests | Focus |
|-------|-------|-------|
| `ProduceHandlerTest` | 5 | API key verification, request parsing, response encoding |
| `FetchHandlerRealTest` | 8 | Real ReadAccumulator integration, record batching, HWM invariant |
| `JoinGroupHandlerTest` | 5 | Rebalance barrier, member assignment, generation ID |
| `DescribeTopicsHandlerTest` | 5 | Stub storage engine, metadata response encoding |
| `KafkaFrameDecoderTest` | 6 | Wire-level decode correctness, frame boundaries |

### Clustering & Consumer Groups

| Class | Tests | Focus |
|-------|-------|-------|
| `ClassicRebalanceProtocolTest` | 6 | Protocol selection by intersection, session timeout, leader election |
| `MetadataPollingSyncTest` | 6 | Scheduler-based polling, idempotent start/stop, version non-downgrade |
| `EpochFencingValidatorTest` | 4 | 3-layer epoch validation, WrongEpoch detection |

### Security

| Class | Tests | Focus |
|-------|-------|-------|
| `HierarchicalQuotaResolverTest` | 9 | 4-level resolution, fallback, null handling, tenant isolation |
| `TokenBucketTest` | 7 | Refill, consume, burst, throttle calculation, thread-safety |
| `AclAuthorizerTest` | 10 | DENY-first evaluation, protocol scoping, wildcard, CIDR |

---

## Test Patterns

### Parameterized Tests

```java
static Stream<Arguments> protocolPairs() {
    return Stream.of(
        Arguments.of(Protocol.KAFKA, Protocol.KAFKA),
        Arguments.of(Protocol.KAFKA, Protocol.MQTT_5),
        Arguments.of(Protocol.AMQP_091, Protocol.NATS));
}

@ParameterizedTest
@MethodSource("protocolPairs")
void shouldRouteAcrossProtocols(Protocol producer, Protocol consumer) {
    // test cross-protocol routing logic
}
```

### Edge Case / Boundary Testing

```java
@Test void shouldEvictOldestBeyondWindow() { /* 6 batches, window=5, verify eviction */ }
@Test void shouldRejectNonZeroFirstSequence() { /* first write sequence must be 0 */ }
@Test void shouldDetectDuplicateWithinWindow() { /* replay same sequence → cached offset */ }
@Test void shouldRejectGapInSequence() { /* skip sequence → OUT_OF_ORDER */ }
```

### Concurrency Testing

```java
@Test
void concurrentAuthShouldReturnIndependentContexts() throws Exception {
    int threadCount = 10;
    SecurityContext[] contexts = new SecurityContext[threadCount];
    Thread[] threads = new Thread[threadCount];
    for (int i = 0; i < threadCount; i++) {
        int idx = i;
        threads[idx] = new Thread(() -> {
            contexts[idx] = SecurityTestHelper.testContext(SecurityTestHelper.testTenant());
        });
        threads[idx].start();
    }
    for (Thread t : threads) t.join();
    for (SecurityContext ctx : contexts) assertThat(ctx).isNotNull();
}
```

---

## Assertion Libraries

- **Primary:** AssertJ (`assertThat(...).isTrue()`, `.hasSize()`, `.contains()`, `.allMatch()`)
- **Exceptions:** `assertThatThrownBy(() -> ...).isInstanceOf(...).hasMessageContaining(...)`
- **JUnit 5:** `assertEquals()`, `assertTrue()`, `@Test`, `@ParameterizedTest`, `@TempDir`

---

## Value Record Testing

Records provide implicit `equals`/`hashCode`/`toString`:

```java
assertThat(state.epoch()).isEqualTo(ProducerEpoch.of(1));
assertThat(state.lastSequence()).isEqualTo(0);
assertThat(state.batchWindow()).hasSize(1);
```

---

## Execution

```bash
mvn test                           # all unit tests (~2 min)
mvn test -pl ivy-core              # core module only
mvn test -Dtest=ProducerStateManagerTest   # single class
```

---

*Last updated: 2026-03-25*
