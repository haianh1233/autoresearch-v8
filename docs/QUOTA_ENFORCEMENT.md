# Quota Enforcement

> **Related:** [SECURITY.md](SECURITY.md) §Layer 7, [MULTI_TENANT.md](MULTI_TENANT.md),
> [CONFIG_SYSTEM.md](CONFIG_SYSTEM.md)

---

## Overview

Ivy enforces per-tenant, per-principal rate limits via a **token bucket algorithm** with
nanosecond precision. Quotas prevent any single tenant or user from monopolizing broker resources.

---

## Token Bucket Algorithm

```java
class TokenBucket {
    private double tokens;                 // current available tokens
    private long lastRefillNanos;          // System.nanoTime() at last refill
    private final double capacity;         // max tokens (= ratePerSecond, 1s burst)
    private final double refillRate;       // tokens per millisecond
    private final ReentrantLock lock;      // thread-safe refill + consume

    QuotaResult tryConsume(double amount) {
        lock.lock();
        try {
            refill();
            if (tokens >= amount) {
                tokens -= amount;
                return QuotaResult.allowed();
            }
            double deficit = amount - tokens;
            long throttleMs = Math.min((long)(deficit / refillRate), MAX_THROTTLE_MS);
            return QuotaResult.throttled(throttleMs);
        } finally {
            lock.unlock();
        }
    }

    private void refill() {
        long now = System.nanoTime();
        double elapsedMs = (now - lastRefillNanos) / 1_000_000.0;
        tokens = Math.min(capacity, tokens + elapsedMs * refillRate);
        lastRefillNanos = now;
    }
}
```

**Key properties:**
- **Nanosecond precision:** `System.nanoTime()` (NOT `currentTimeMillis()`) for sub-ms accuracy
- **Floating-point accumulation:** `double` tokens prevent integer truncation on high-rate quotas
- **Burst capacity:** equals 1 second of tokens (`capacity = ratePerSecond`)
- **Thread-safe:** `ReentrantLock` (not `synchronized`) — supports `tryLock` for non-blocking fast path
- **Max throttle:** 30 seconds cap to prevent client-side timeout storms

---

## Quota Types

| Type | Unit | Description |
|------|------|-------------|
| `PRODUCER_BYTE_RATE` | bytes/sec | Total payload bytes for Produce operations |
| `CONSUMER_BYTE_RATE` | bytes/sec | Total payload bytes for Fetch operations |
| `REQUEST_RATE` | requests/sec | All API requests (Produce, Fetch, Metadata, etc.) |
| `CONNECTION_CREATION_RATE` | connections/sec | New TCP connections per second |

---

## 4-Level Resolution Hierarchy

Most specific match wins. Empty string (`""`) = wildcard:

```
Level 1: quota:{tenant}:{user}:{clientId}:{type}   ← per-user, per-client
Level 2: quota:{tenant}:{user}::{type}              ← per-user, any client
Level 3: quota:{tenant}:::{type}                     ← per-tenant default
Level 4: (built-in)                                   ← Long.MAX_VALUE (unlimited)
```

```java
class HierarchicalQuotaResolver {
    ConcurrentHashMap<String, Long> quotaConfigs;   // key → ratePerSecond

    long resolveRate(TenantId tenant, String user, String clientId, QuotaType type) {
        // Level 1: most specific
        Long rate = quotaConfigs.get(key(tenant, user, clientId, type));
        if (rate != null) return rate;

        // Level 2: per-user, any client
        rate = quotaConfigs.get(key(tenant, user, "", type));
        if (rate != null) return rate;

        // Level 3: per-tenant default
        rate = quotaConfigs.get(key(tenant, "", "", type));
        if (rate != null) return rate;

        // Level 4: unlimited
        return Long.MAX_VALUE;
    }

    static String key(TenantId t, String user, String clientId, QuotaType type) {
        return "quota:" + t.id() + ":" + user + ":" + clientId + ":" + type.name();
    }
}
```

---

## Multi-Broker Coordination

In cluster mode, quotas are **statically split** across active brokers:

```
effectiveQuota = configuredQuota / activeBrokerCount
```

**Periodic sync** every 10 seconds via PG reconciles actual usage:

```sql
-- Each broker reports its usage
INSERT INTO metadata_kv (key, value, version)
VALUES ('quota_usage:' || :brokerId || ':' || :quotaKey, :usageBytes, :version)
ON CONFLICT (key) DO UPDATE SET value = :usageBytes, version = :version;

-- Aggregation query (run by each broker)
SELECT SUM(value::bigint) FROM metadata_kv WHERE key LIKE 'quota_usage:%:' || :quotaKey;
```

**Trade-off:** Static split is imprecise (one broker may be busier) but avoids distributed
coordination overhead. The 10s sync provides eventual consistency — total cluster throughput
converges to the configured quota within one sync interval.

---

## Throttle Response

When `TokenBucket.tryConsume()` returns `QuotaResult.throttled(ms)`:

| Protocol | Throttle Signal | Client Behaviour |
|----------|----------------|-----------------|
| Kafka | `throttle_time_ms` in ProduceResponse/FetchResponse | Client sleeps before next request |
| MQTT | Delayed PUBACK by `throttleMs` | Implicit backpressure |
| AMQP 0-9-1 | `Channel.Flow(active=false)` | Producer pauses |
| AMQP 1.0 | Reduce `link-credit` in Flow frame | Sender slows |
| HTTP | `429 Too Many Requests` + `Retry-After` header | Client retries after delay |

**Max throttle:** 30 seconds. Higher deficits are capped to prevent client timeout cascades.

---

## Quota Storage

**PostgreSQL:** `metadata_kv` table with key format `quota:{tenant}:{user}:{clientId}:{type}`

**Internal topic:** `__quotas` (log-compacted) for cluster-wide state propagation

**Hot-reload:** quota config changes detected via PG `LISTEN/NOTIFY` on `quota_change` channel,
applied within 100ms. Fallback: polling every 10s.

---

## Configuration

```yaml
security:
  quotas:
    enabled: true
    max-throttle-ms: 30000          # cap on throttle response
    sync-interval-ms: 10000         # multi-broker sync frequency
    defaults:
      producer-byte-rate: -1        # -1 = unlimited
      consumer-byte-rate: -1
      request-rate: -1
      connection-creation-rate: 100
```

Per-tenant override via `tenants.config.quotas` JSONB:

```json
{
  "quotas": {
    "produceByteRate": 104857600,
    "consumeByteRate": 209715200,
    "requestRate": 10000,
    "connectionCreationRate": 50
  }
}
```

---

*Last updated: 2026-03-25*
