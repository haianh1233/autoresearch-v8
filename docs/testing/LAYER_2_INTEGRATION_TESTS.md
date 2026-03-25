# Layer 2: Integration Tests

> **Related:** [../TESTING_STRATEGY.md](../TESTING_STRATEGY.md), [LAYER_1_UNIT_TESTS.md](LAYER_1_UNIT_TESTS.md)

---

## Overview

Integration tests validate cross-module interactions with **real dependencies** (PostgreSQL via
Testcontainers, real storage engines) but without a full broker start. They run in < 5 seconds
per test.

**Naming:** `*IT.java` (Failsafe plugin, `mvn verify -pl ivy-testing`)

---

## What Constitutes an Integration Test

| Unit Test | Integration Test |
|-----------|-----------------|
| Single class, all deps mocked | Cross-module, real PG or storage engine |
| No I/O, no network | PG via Testcontainers, file I/O |
| `*Test.java` (Surefire) | `*IT.java` (Failsafe) |
| < 100ms per test | < 5s per test |

---

## TestBrokerLauncher Variants

```java
// In-memory only (no PG, fastest)
TestBrokerLauncher broker = TestBrokerLauncher.startInMemory();

// PG-backed (real PostgreSQL via Testcontainers)
TestBrokerLauncher broker = TestBrokerLauncher.start(pg.getJdbcUrl());

// Crash recovery (preserves data directory between restarts)
TestBrokerLauncher broker = TestBrokerLauncher.startWithDataDir(existingDataDir);
```

---

## PostgreSQL Integration Setup

```java
@BeforeAll
static void startPg() {
    pg = new PostgreSQLContainer<>("postgres:17-alpine")
        .withReuse(true);   // warm start on subsequent runs
    pg.start();
}

@Test
void shouldFlushSegmentToPg() {
    var engine = new PostgresStorageEngine(pg.getJdbcUrl());
    var segment = createTestSegment(tempDir);
    segment.append(testEntry);
    segment.seal();

    var flusher = new StorageFlusher(engine, segmentStore);
    flusher.flushAll();

    var records = engine.fetchRange(partitionId, 0, 100);
    assertThat(records).hasSize(1);
}
```

---

## Key Integration Test Classes

| Class | Dependencies | Focus |
|-------|-------------|-------|
| `StorageFlusherIntegrationTest` | InMemoryStorageEngine + LogSegment | Segment write → flush → engine storage lifecycle |
| `PgConsumerOffsetStoreTest` | PostgreSQL | Offset commit/fetch persistence across restarts |
| `PgStorageEngineTest` | PostgreSQL | Full read/write with binary COPY, fetchRange, HWM |
| `FlushEventDispatcherTest` | In-memory | Dispatcher thread coordination, event ordering, FIFO guarantee |
| `FetchHandlerRealTest` | ReadAccumulator + LogSegment | Real multi-partition fetch, record batching, HWM invariant |
| `ProduceHandlerRealTest` | WriteAccumulator + WriteWorker | Real write path with in-memory engine |
| `AdminHandlersRealTest` | MetadataManager | Topic create/delete with real storage, partition creation |

---

## Test Data Builders

```java
// Builder pattern for test criteria
var criteria = PassFailCriteria.builder()
    .maxP99LatencyMs(100.0)
    .minThroughput(100_000L)
    .maxHeapGrowthPercent(5.0)
    .build();

// Helper for group members
private GroupMember member(String id) {
    return GroupMember.of(MemberId.of(id), "client-" + id, "host-" + id,
        30_000, 60_000, "consumer", List.of("range", "roundrobin"));
}
```

---

## Test Isolation

### Unique Names

```java
String uniqueTopic() { return "integ-" + UUID.randomUUID().toString().substring(0, 8); }
```

### Temporary Directories

```java
@TempDir Path tempDir;

@AfterEach
void cleanup() throws IOException {
    Files.walkFileTree(tempDir, new SimpleFileVisitor<>() {
        @Override public FileVisitResult visitFile(Path f, BasicFileAttributes a) throws IOException {
            Files.deleteIfExists(f);
            return FileVisitResult.CONTINUE;
        }
    });
}
```

### Executor Cleanup

```java
@AfterEach
void tearDown() {
    scheduler.shutdownNow();   // prevent thread leaks between tests
}
```

---

## Execution

```bash
mvn verify -pl ivy-testing                          # all integration tests
mvn verify -pl ivy-testing -Dit.test=PgStorageEngineTest   # single class
```

---

*Last updated: 2026-03-25*
