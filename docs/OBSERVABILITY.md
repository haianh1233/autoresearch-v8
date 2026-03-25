# Observability & Metrics

> **Related:** [IMPLEMENTATION_PHASES.md](IMPLEMENTATION_PHASES.md) §Phase 5,
> [WRITE_PATH.md](WRITE_PATH.md), [READ_PATH.md](READ_PATH.md), [SECURITY.md](SECURITY.md)

---

## Overview

Ivy uses **Micrometer** for metrics collection with **Prometheus** as the primary scrape target.
Health/readiness endpoints support Kubernetes liveness and readiness probes. All metrics are
tenant-tagged for per-tenant observability in multi-tenant deployments.

---

## Endpoints

| Endpoint | Port | Purpose |
|----------|------|---------|
| `GET /health` | 8080 | Liveness probe — returns 200 if JVM is running |
| `GET /ready` | 8080 | Readiness probe — returns 200 if broker is ACTIVE and PG connected |
| `GET /metrics` | 8080 | Prometheus scrape endpoint (text/plain; OpenMetrics format) |

### Readiness Checks

`/ready` returns 503 until ALL of these pass:
1. Broker state == `ACTIVE` (not `STARTING`, `DRAINING`, `SHUTDOWN`, `FENCED`)
2. At least one PG connection available in HikariCP pool
3. MetadataImage loaded (at least one successful PG poll)
4. All WriteWorker threads alive
5. FlushEventDispatcher thread alive

---

## Key Metrics

### Write Path

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `ivy_write_latency_ms` | Histogram | tenant, protocol | Submit to PG COMMIT (p50, p95, p99) |
| `ivy_write_records_total` | Counter | tenant, protocol | Messages written |
| `ivy_write_bytes_total` | Counter | tenant, protocol | Payload bytes written |
| `ivy_write_batch_size` | Histogram | — | Messages per WriteWorker batch |
| `ivy_write_pg_commit_ms` | Histogram | — | PG transaction latency (BEGIN→COMMIT) |
| `ivy_write_accumulator_linger_ms` | Histogram | — | Time in WriteAccumulator before flush |
| `ivy_write_dispatcher_queue_depth` | Gauge | — | WriteDispatcher MpscArrayQueue size |
| `ivy_write_dispatcher_fill_ratio` | Gauge | — | Queue fill ratio (0.0–1.0) |
| `ivy_write_backpressure_total` | Counter | — | Times backpressure throttle activated |
| `ivy_write_epoch_fence_total` | Counter | — | WrongEpochException count |
| `ivy_write_duplicate_total` | Counter | — | Idempotent duplicate detections |
| `ivy_write_forward_total` | Counter | target_broker | Forwarded writes to non-local owner |
| `ivy_write_forward_latency_ms` | Histogram | target_broker | Inter-broker forward round-trip |

### Read Path

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `ivy_read_latency_ns` | Histogram | tenant, protocol, tier | Per-tier read latency |
| `ivy_read_records_total` | Counter | tenant, protocol, tier | Records read per tier |
| `ivy_read_bytes_total` | Counter | tenant, protocol | Payload bytes read |
| `ivy_cache_hit_ratio` | Gauge | — | Tier 1 (LogSegment) hit rate (0.0–1.0) |
| `ivy_longpoll_wait_ms` | Histogram | outcome | Long-poll duration (data_arrived / timeout) |
| `ivy_longpoll_active` | Gauge | — | Currently registered long-poll waiters |
| `ivy_consumer_lag` | Gauge | tenant, group, partition | HWM − committed offset |
| `ivy_backfill_requests_total` | Counter | — | BackfillScheduler requests from PG |
| `ivy_read_through_fills_total` | Counter | — | Tier 3 → Tier 1 cache population events |

### Storage

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `ivy_segment_count` | Gauge | state | Segments per lifecycle state (ACTIVE/SEALED/FLUSHED) |
| `ivy_segment_bytes` | Gauge | state | Total bytes per lifecycle state |
| `ivy_flusher_latency_ms` | Histogram | — | StorageFlusher cycle latency |
| `ivy_flusher_circuit_breaker` | Gauge | — | 0=CLOSED, 1=OPEN, 2=HALF_OPEN |
| `ivy_cleaner_deleted_total` | Counter | — | Segments deleted by SegmentCleaner |
| `ivy_disk_usage_ratio` | Gauge | — | Disk usage ratio (0.0–1.0) |
| `ivy_pg_pool_active` | Gauge | — | HikariCP active connections |
| `ivy_pg_pool_idle` | Gauge | — | HikariCP idle connections |
| `ivy_pg_pool_pending` | Gauge | — | HikariCP pending connection requests |

### Security

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `ivy_auth_success_total` | Counter | tenant, mechanism, protocol | Successful authentications |
| `ivy_auth_failure_total` | Counter | tenant, mechanism, protocol | Failed authentications |
| `ivy_reauth_triggered_total` | Counter | tenant, protocol | Re-auth state entered |
| `ivy_reauth_success_total` | Counter | tenant, protocol | Successful re-authentications |
| `ivy_reauth_failure_total` | Counter | tenant, protocol, reason | Failed re-auths |
| `ivy_acl_denied_total` | Counter | tenant, protocol, resource | ACL denials |
| `ivy_quota_throttled_total` | Counter | tenant, quota_type | Quota throttle events |
| `ivy_epoch_revocation_total` | Counter | tenant | Security epoch revocations |
| `ivy_ip_lockout_total` | Counter | — | IP-level auth lockouts |

### Connections

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `ivy_connections_active` | Gauge | tenant, protocol | Currently connected clients |
| `ivy_connections_created_total` | Counter | tenant, protocol | Total connections accepted |
| `ivy_connections_closed_total` | Counter | tenant, protocol, reason | Connections closed (reason: normal/auth_failed/timeout/reauth/shutdown) |

### Cluster

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `ivy_broker_status` | Gauge | — | 0=STARTING, 1=ACTIVE, 2=DRAINING, 3=SHUTDOWN, 4=FENCED |
| `ivy_partitions_owned` | Gauge | — | Partitions owned by this broker |
| `ivy_metadata_version` | Gauge | — | Current MetadataImage version |
| `ivy_metadata_propagation_ms` | Histogram | layer | Metadata update latency per layer (notify/broadcast/poll) |
| `ivy_heartbeat_latency_ms` | Histogram | — | Heartbeat write latency |
| `ivy_inter_broker_rpc_latency_ms` | Histogram | message_type | Inter-broker RPC round-trip |
| `ivy_inter_broker_circuit_breaker` | Gauge | peer | Circuit breaker state per peer |

### Push Delivery

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `ivy_push_subscribers_active` | Gauge | tenant, protocol | Active push subscribers |
| `ivy_push_delivery_latency_ms` | Histogram | protocol | PG COMMIT to subscriber delivery |
| `ivy_push_delivery_total` | Counter | protocol | Messages pushed to subscribers |
| `ivy_push_delivery_failures_total` | Counter | protocol | Failed push deliveries |

### JVM (Auto-Registered by Micrometer)

| Metric | Description |
|--------|-------------|
| `jvm_memory_used_bytes` | Heap and non-heap usage |
| `jvm_gc_pause_seconds` | GC pause durations (Shenandoah: sub-ms target) |
| `jvm_threads_live` | Thread count (platform + virtual) |
| `jvm_buffer_memory_used_bytes` | Direct buffer usage (Netty) |
| `process_cpu_usage` | Process CPU utilization |

---

## Per-Tenant Metric Tagging

All business metrics include a `tenant` label for multi-tenant observability:

```java
meterRegistry.counter("ivy_write_records_total",
    "tenant", tenantId.id().toString(),
    "protocol", protocolId.name())
    .increment(batchSize);
```

**Cardinality control:** tenant label is bounded by the number of tenants (typically <1000).
Protocol label is bounded by 8. Combined cardinality stays manageable for Prometheus.

---

## Prometheus Scrape Configuration

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'ivy-broker'
    scrape_interval: 15s
    metrics_path: '/metrics'
    static_configs:
      - targets: ['ivy-broker-0:8080', 'ivy-broker-1:8080', 'ivy-broker-2:8080']
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'jvm_.*'
        action: keep     # keep JVM metrics
```

---

## Alerting Rules (Examples)

```yaml
groups:
  - name: ivy-broker
    rules:
      - alert: IvyHighWriteLatency
        expr: histogram_quantile(0.99, ivy_write_latency_ms) > 30
        for: 5m
        labels: { severity: warning }
        annotations: { summary: "Write p99 latency > 30ms for 5 minutes" }

      - alert: IvyConsumerLagHigh
        expr: ivy_consumer_lag > 100000
        for: 10m
        labels: { severity: warning }
        annotations: { summary: "Consumer lag > 100K for {{ $labels.group }}" }

      - alert: IvyDiskUsageHigh
        expr: ivy_disk_usage_ratio > 0.85
        for: 5m
        labels: { severity: critical }
        annotations: { summary: "Disk usage > 85% — SegmentCleaner aggressive eviction" }

      - alert: IvyPgPoolExhausted
        expr: ivy_pg_pool_pending > 0
        for: 1m
        labels: { severity: critical }
        annotations: { summary: "PG connection pool exhausted — requests queuing" }

      - alert: IvyBrokerFenced
        expr: ivy_broker_status == 4
        labels: { severity: critical }
        annotations: { summary: "Broker fenced — partitions being reassigned" }
```

---

## Micrometer Configuration

```yaml
metrics:
  enabled: true
  export:
    prometheus:
      enabled: true
      step: 15s                # histogram bucket rotation interval
  tags:
    broker: ${BROKER_ID}       # global tag: broker identity
    cluster: ${CLUSTER_NAME}   # global tag: cluster name
  histogram:
    percentiles: [0.5, 0.95, 0.99]
    sla: [1, 5, 10, 25, 50, 100, 250, 500, 1000]   # ms buckets for latency histograms
```

---

## PostgreSQL Readiness Checker (Startup)

Before accepting connections, the broker runs a 7-check PG readiness suite. All checks must
pass (or warn) before the broker transitions to `ACTIVE`.

```java
class PgReadinessChecker {
    List<ReadinessCheck> checks = List.of(
        new SynchronousCommitCheck(),     // synchronous_commit must be "on"
        new StandbyDetectionCheck(),      // must NOT be a standby replica
        new UnloggedTableCheck(),         // unlogged tables allowed (config-controlled)
        new WalLevelCheck(),              // wal_level >= "logical"
        new XidAgeCheck(),                // XID age < 2 billion (prevent wraparound)
        new AutovacuumCheck(),            // autovacuum must be enabled
        new ConnectionLimitCheck()        // max_connections > pool size
    );

    sealed interface CheckResult permits Pass, Warn, Fail {}
    // Warn: logged, startup continues
    // Fail: throws PgReadinessException → broker stays in STARTING
}
```

| Check | Query | Pass Condition | Severity |
|-------|-------|----------------|----------|
| SynchronousCommit | `SHOW synchronous_commit` | `= 'on'` | FAIL |
| StandbyDetection | `SELECT pg_is_in_recovery()` | `= false` | FAIL |
| UnloggedTable | `SHOW fsync` | `= 'on'` (or unlogged explicitly allowed) | WARN |
| WalLevel | `SHOW wal_level` | `∈ {'logical', 'replica'}` | WARN |
| XidAge | `SELECT age(datfrozenxid) FROM pg_database` | `< 2,000,000,000` | WARN |
| Autovacuum | `SHOW autovacuum` | `= 'on'` | WARN |
| ConnectionLimit | `SHOW max_connections` | `> pool.maximum-pool-size` | FAIL |

---

## Storage Health Monitor

Monitors disk usage and PG connectivity on a 5-second interval. Distinct from `SegmentCleaner`
(which manages segment lifecycle) — this monitors overall storage health.

```java
class StorageHealthMonitor {
    enum DiskZone { NORMAL, THROTTLE, REJECT }

    DiskZone currentZone() {
        double usage = diskUsageRatio();
        if (usage >= 0.95) return REJECT;      // refuse all writes
        if (usage >= 0.85) return THROTTLE;    // apply write throttle
        return NORMAL;                          // full speed
    }
}
```

| Zone | Disk Usage | Write Behaviour | Metric |
|------|-----------|-----------------|--------|
| `NORMAL` | < 85% | Full speed | `ivy_disk_zone{zone="NORMAL"} = 1` |
| `THROTTLE` | 85–95% | Writes delayed via `THROTTLE_TIME_MS` | `ivy_disk_zone{zone="THROTTLE"} = 1` |
| `REJECT` | > 95% | All writes rejected with `KAFKA_STORAGE_ERROR` | `ivy_disk_zone{zone="REJECT"} = 1` |

**Recovery callbacks:** when zone transitions from THROTTLE/REJECT back to NORMAL, the monitor
invokes `onRecovery()` to resume accepting writes and clear backpressure signals.

**PG Connectivity Probe:**
```sql
SELECT 1;  -- executed every 5s on dedicated health-check connection
```
If probe fails 3 consecutive times → readiness endpoint returns 503.

---

## Prometheus Scrape Implementation

Metrics scrape is offloaded to a **virtual thread** to prevent blocking the Netty event loop:

```java
class PrometheusMetricsHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest req) {
        // Offload to virtual thread — prometheusTextOutput() serializes all meters
        // which can take 10-50ms with high cardinality (Rule R-no-block-eventloop)
        Thread.ofVirtual().start(() -> {
            String body = prometheusRegistry.scrape();   // text exposition format
            ByteBuf content = Unpooled.copiedBuffer(body, UTF_8);
            ctx.writeAndFlush(new DefaultFullHttpResponse(
                HTTP_1_1, OK, content,
                headers().set(CONTENT_TYPE, "text/plain; version=0.0.4; charset=utf-8"),
                EmptyHttpHeaders.INSTANCE));
        });
    }
}
```

**Why virtual thread?** With 1000+ tenants × 8 protocols × 60+ metric types, serializing all
meters can take 10-50ms. Running on the event loop would stall all connections on that thread.

---

## Structured Logging

All log output uses structured JSON format with consistent fields:

```json
{
  "timestamp": "2026-03-25T14:32:01.123Z",
  "level": "INFO",
  "logger": "com.ivy.broker.write.WriteWorker",
  "message": "Batch committed",
  "tenantId": "550e8400-e29b-41d4-a716-446655440000",
  "connectionId": "ch-1234",
  "protocol": "KAFKA",
  "partitionId": "f3e9d4b7-...",
  "batchSize": 1000,
  "latencyMs": 3.2
}
```

### Sensitive Field Masking

Fields matching these patterns are automatically masked to `"***"`:
`password`, `token`, `secret`, `key` (credential context), `credential`, `hmac`

```java
class SensitiveFieldMasker {
    static final Set<String> MASKED = Set.of(
        "password", "token", "secret", "key", "credential", "hmac",
        "saslAuthBytes", "authData", "initialResponse");

    String mask(String fieldName, Object value) {
        return MASKED.contains(fieldName.toLowerCase()) ? "***" : String.valueOf(value);
    }
}
```

### Correlation ID Propagation

**NOT using MDC** (ThreadLocal — incompatible with virtual threads). Instead, `ScopedValue`
carries tenant context and connection ID through the entire request lifecycle:

```java
ScopedValue<TenantContext> TENANT_CTX = ScopedValue.newInstance();
ScopedValue<ConnectionId> CONN_ID = ScopedValue.newInstance();

// In protocol handler:
ScopedValue.where(TENANT_CTX, ctx).where(CONN_ID, connId).run(() -> {
    // All code in this scope has access to tenant + connection for logging
    engine.write(tenantId, writes, secCtx);
});

// In logger:
String tenantId = TENANT_CTX.orElse(TenantContext.SYSTEM).tenantId().toString();
String connId = CONN_ID.orElse(ConnectionId.UNKNOWN).toString();
```

### Log Levels

| Level | Usage |
|-------|-------|
| ERROR | Startup failures, unrecoverable PG errors, data corruption detected |
| WARN | PG readiness warnings, circuit breaker opens, auth failures, segment CRC mismatch |
| INFO | Startup progress, leadership changes, tenant lifecycle events, config reload |
| DEBUG | Per-operation detail (segment lookup, partition assignment, ACL evaluation) |
| TRACE | Wire-level frame decode/encode (disabled in production) |

---

## JVM Flags & GC Tuning

### Recommended Production Configuration

```bash
# GC: Shenandoah (sub-ms pauses, preferred)
-XX:+UseShenandoahGC
-XX:ShenandoahGCHeuristics=adaptive

# Fallback: ZGC (if Shenandoah unavailable)
# -XX:+UseZGC -XX:+ZGenerational

# Memory
-Xms4g -Xmx4g                              # fixed heap (no resize overhead)
-XX:MaxDirectMemorySize=512m                # Netty direct buffer limit
-XX:+UseCompactObjectHeaders                # 12→8 byte object headers (JEP 450)

# Diagnostics
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/ivy/
-Xlog:gc*:file=/var/log/ivy/gc.log:time,uptime,level,tags:filecount=5,filesize=20m

# Java 26 preview features
--enable-preview                            # Valhalla value classes, ScopedValue, StructuredConcurrency

# Virtual threads
-Djdk.virtualThreadScheduler.parallelism=0  # 0 = CPU count (default)
```

### Key Metrics to Monitor

| Metric | Alert Threshold | Rationale |
|--------|----------------|-----------|
| `jvm_gc_pause_seconds_max` | > 10ms | Shenandoah target: sub-ms; 10ms indicates GC pressure |
| `jvm_buffer_direct_used_bytes / capacity` | > 80% | Netty buffer near exhaustion → OOM risk |
| `jvm_threads_live` | > 10,000 | Virtual thread leak (should be bounded by connections) |
| `process_cpu_usage` | > 80% sustained | CPU saturation → latency increase |
| `jvm_memory_used_bytes{area="heap"}` | > 85% of max | Heap pressure → more frequent GC |

---

## Grafana Dashboard Panels

### Recommended Dashboard Layout (6 panels)

**Panel 1: Message Throughput by Protocol** (timeseries)
```
rate(ivy_write_records_total{protocol=~"$protocol"}[1m])
```
Unit: msgs/sec. Variables: protocol (multi-select).

**Panel 2: Write Latency Heatmap** (heatmap)
```
rate(ivy_write_latency_ms_bucket[5m])
```
Shows P50/P95/P99 distribution over time. `le` (bucket upper bound) on Y-axis.

**Panel 3: Active Connections** (gauge, by protocol)
```
ivy_connections_active{protocol=~"$protocol"}
```
Thresholds: green < 70%, yellow < 90%, red ≥ 90%.

**Panel 4: Storage I/O** (bytes/sec)
```
rate(ivy_write_bytes_total[1m])   # write throughput
rate(ivy_read_bytes_total[1m])    # read throughput
```

**Panel 5: Consumer Lag** (timeseries, per group)
```
ivy_consumer_lag{tenant="$tenant", group=~"$group"}
```

**Panel 6: Error Rate** (timeseries)
```
rate(ivy_write_epoch_fence_total[1m])         # epoch fencing errors
rate(ivy_auth_failure_total[1m])              # auth failures
rate(ivy_acl_denied_total[1m])                # ACL denials
rate(ivy_push_delivery_failures_total[1m])    # push delivery failures
```

---

## Distributed Tracing (Planned)

> **Status:** Not yet implemented. Documented here for future reference.

### Design Direction

- **OpenTelemetry SDK** for span creation and context propagation
- **ScopedValue** for trace context (NOT ThreadLocal — virtual thread safe)
- **Inter-broker propagation:** trace context carried in `ForwardWriteRequest`/`ForwardFetchRequest`
  headers (same mechanism as `SecurityContext` propagation)
- **Protocol-specific propagation:**
  - Kafka: `traceparent` header in message headers (W3C Trace Context)
  - MQTT 5.0: user property `traceparent`
  - AMQP: application-property `traceparent`
  - HTTP: `traceparent` HTTP header
- **Sampling:** configurable rate (default 1%) to limit overhead
- **Exporter:** OTLP gRPC to Jaeger/Tempo collector

### Planned Span Structure

```
ivy-broker:produce
  ├─ ivy-broker:transform (transformation pipeline)
  ├─ ivy-broker:resolve (partition routing)
  ├─ ivy-broker:forward (inter-broker RPC, if non-owner)
  ├─ ivy-broker:accumulate (WriteAccumulator batching)
  ├─ ivy-broker:pg-commit (PG transaction)
  └─ ivy-broker:notify (FlushEventDispatcher)
```

---

## WAL Lag Monitoring (Planned)

> **Status:** Not yet implemented. PG replication lag is invisible without this.

```sql
-- Query for monitoring PG replication lag (run every 30s)
SELECT
    client_addr,
    state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS send_lag_bytes,
    pg_wal_lsn_diff(sent_lsn, flush_lsn) AS flush_lag_bytes,
    pg_wal_lsn_diff(flush_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

Planned metrics:
- `ivy_pg_wal_send_lag_bytes` — bytes not yet sent to replica
- `ivy_pg_wal_flush_lag_bytes` — bytes sent but not flushed on replica
- `ivy_pg_wal_replay_lag_bytes` — bytes flushed but not replayed on replica

Alert: `ivy_pg_wal_replay_lag_bytes > 100MB for 5m` → WARNING (replica falling behind)

---

*Last updated: 2026-03-25*
