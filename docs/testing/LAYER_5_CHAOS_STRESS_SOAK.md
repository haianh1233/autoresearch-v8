# Layer 5: Chaos, Stress & Soak Tests

> **Related:** [../TESTING_STRATEGY.md](../TESTING_STRATEGY.md), [../STORAGE.md](../STORAGE.md) §Crash Recovery

---

## Overview

Complex E2E tests validate broker resilience (chaos), scalability (stress), and long-term
stability (soak). They use failure injection, high concurrency, and sustained load.

**Naming:** `*E2ETest.java` in `complex/` package
**Tags:** `@Tag("failure")`, `@Tag("stress")`, `@Tag("soak")`
**Execution:** `mvn verify -Pcomplex-e2e -pl e2e` (~30-45 min)

---

## Chaos Tests (@Tag("failure"))

### BrokerCrashRecoveryE2ETest

| Test | Injection | Assertion |
|------|-----------|-----------|
| `crashMidWrite_noDataLoss` | `broker.close()` + `startWithDataDir()` | All ACK'd messages durable after restart |
| `offsetContinuity_afterRestart` | Crash + restart | New message offset > last pre-crash offset |
| `allProtocols_reconnect` | Crash + restart | Kafka, MQTT, NATS clients reconnect successfully |

### NetworkPartitionE2ETest

| Test | Injection | Assertion |
|------|-----------|-----------|
| `noDuplicateWrites` | `dockerClient.pauseContainerCmd(pg)` | 10 messages pre+during partition, no duplicates after heal |
| `consumerOffset_consistent` | PG pause + unpause | Consumer resumes from committed offset, reads only new batch |

### DiskFullE2ETest

| Test | Injection | Assertion |
|------|-----------|-----------|
| `backpressure_propagated` | `broker.setSimulateDiskFull(true)` | Broker still responsive (admin API works) |
| `noCorruptSegments` | Disk full during writes | Only pre-diskfull messages readable, CRC valid |
| `readsStillWork` | Disk full active | Read path unaffected (5 pre-populated messages readable) |

### CorruptSegmentE2ETest

| Test | Injection | Assertion |
|------|-----------|-----------|
| `bitFlip_detected` | XOR flip byte at offset 100 in `.log` file | Healthy messages before corrupt point still readable |
| `corruptMessage_skipped` | 16-byte XOR flip at segment midpoint | Broker skips corrupt, returns remaining healthy records |
| `recovery_truncates` | Truncate segment at 80%, fill with 0xFF | Broker recovers, produces post-recovery message successfully |

### StorageOutageE2ETest

| Test | Injection | Assertion |
|------|-----------|-----------|
| `pgDown_hotTierContinues` | PG pause | 5 messages buffered in hot-tier, all 6 consumed after heal |
| `circuitBreaker_activates` | PG pause + 2s wait | Circuit breaker activates, reads from hot-tier still work |
| `engineReturns_flushResumes` | PG pause + unpause | MQTT subscriber receives all 5 buffered messages after recovery |

### GracefulShutdownUnderLoadE2ETest

| Test | Injection | Assertion |
|------|-----------|-----------|
| `inFlightRequests_complete` | `broker.close()` during continuous produce | All ACK'd messages survive shutdown |
| `newConnections_rejected` | Shutdown initiated, attempt 3 new connections | Some/all rejected, shutdown completes cleanly |
| `cleanRestart` | Produce 5 → shutdown → restart | Post-restart produce succeeds, MQTT connects cleanly |

---

## Stress Tests (@Tag("stress"))

### ConnectionScalabilityE2ETest

| Test | Configuration | Assertion |
|------|--------------|-----------|
| `rampUp_1000` | 1K NATS connections, 100/batch ramp | All 1K reach CONNECTED status |
| `sustainedPlateau_noLeak` | Hold 100 connections for 10s | Heap growth < 50 MB |
| `noFileDescriptorExhaustion` | 200 connections | FD increase < connections × 5 |
| `eventLoop_notBlocked` | 50 background connections | Kafka produce responds in < 5s |

### PartitionScalabilityE2ETest

| Test | Configuration | Assertion |
|------|--------------|-----------|
| `tenThousandPartitions` | Topic with 10K partitions | All 10K messages produced and consumed |
| `perPartitionOrdering_atScale` | 100 partitions × 10 msgs | Per-partition FIFO order preserved |
| `writeAccumulator_noContention` | 500 partitions × 200 samples | p99 latency < 500ms |

### LargeMessageE2ETest

| Test | Payload | Assertion |
|------|---------|-----------|
| `kafka_1MB` | 1,048,576 bytes | Produce + consume round-trip intact |
| `s3_multipart_30MB` | 5 parts × 6 MB | Content-length = 30 MB after complete |
| `mqtt_64KB` | 65,536 bytes | Subscribe + receive intact |
| `noOOM` | 10 × 1 MB sequential | Heap growth < 100 MB |

### ThroughputBenchmarkE2ETest

| Test | Workload | Target |
|------|----------|--------|
| `kafka_sustained` | 10K messages | > 100 msg/s |
| `mqtt_sustained` | 1K messages | > 10 msg/s |
| `multiProtocol_concurrent` | 500 × 3 protocols | All 1.5K delivered |
| `latency_p99` | 500 messages | < 200 ms p99 |

---

## Soak Tests (@Tag("soak"))

Duration configurable: `-Dsoak.duration.ms=86400000` (24h production, 60s CI default)

### SoakMultiProtocolLoadTest

| Test | Duration | Assertion |
|------|----------|-----------|
| `noMemoryLeak` | 24h | Heap growth < 50% of initial + 100 MB |
| `noFdLeak` | 24h | FD count growth < 50% of initial + 50 |
| `latencyStable` | 24h | p99 < 3× initial window or 1s cap |
| `gcPausesSubMs` | 24h | Average GC pause < 50 ms |

### SoakSegmentLifecycleTest

| Test | Duration | Assertion |
|------|----------|-----------|
| `segmentRotation_noDataLoss` | 24h | 10K messages produced = 10K consumed |
| `segmentCleaning_oldsRemoved` | 24h | Segment dir accessible, count bounded |
| `tierMigration_checkpointsAdvance` | 24h | Lake directory exists |
| `diskUsageMonitor_triggersCleanup` | 24h | Data dir exists, usage bounded |

### SoakConsumerLagCatchupTest

| Test | Duration | Assertion |
|------|----------|-----------|
| `lag_accumulates` | 1h | End offset ≥ produced count |
| `catchup_withBackfill` | 1h | All 1K messages readable from offset 0 |
| `noOOM_duringCatchup` | 1h | Heap growth < 100 MB during catchup |
| `offsetIndex_backfilled` | 1h | 1K messages via OffsetIndex lookup |
| `fullCatchup_achieved` | 1h | Consumer catches up to producer HEAD |

---

## Injection Mechanisms

| Mechanism | Method | Used By |
|-----------|--------|---------|
| Broker crash | `broker.close()` | BrokerCrashRecovery |
| Data preservation | `broker.setPreserveDataDir(true)` | BrokerCrashRecovery |
| PG network partition | `dockerClient.pauseContainerCmd(pg)` | NetworkPartition, StorageOutage |
| Disk full simulation | `broker.setSimulateDiskFull(true)` | DiskFull |
| Segment corruption | Direct byte XOR in `.log` files | CorruptSegment |
| Segment truncation | `RandomAccessFile.setLength()` + 0xFF fill | CorruptSegment |

---

## Key Thresholds

| Metric | Threshold | Test |
|--------|-----------|------|
| Memory leak | < 50% growth | SoakMultiProtocol |
| FD leak | < 50% growth | SoakMultiProtocol |
| Latency degradation | < 3× initial p99 | SoakMultiProtocol |
| GC pause average | < 50 ms | SoakMultiProtocol |
| Write p99 | < 200 ms | ThroughputBenchmark |
| Partition p99 | < 500 ms | PartitionScalability |
| Connections | 1000+ sustained | ConnectionScalability |
| FD per connection | ~3 | ConnectionScalability |
| Partitions | 10K+ | PartitionScalability |
| Catchup heap | < 100 MB | ConsumerLagCatchup |

---

*Last updated: 2026-03-25*
