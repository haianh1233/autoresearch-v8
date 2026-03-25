# Configuration System

> **Related:** [MULTI_TENANT.md](MULTI_TENANT.md) §TenantConfig, [SECURITY.md](SECURITY.md) §Configuration

---

## Overview

Ivy uses a **6-level cascading configuration** system. Higher levels override lower levels.
Per-tenant and per-protocol overrides are hot-reloadable from PostgreSQL without broker restart.

---

## Resolution Hierarchy (Highest Wins)

```
Level 5 ─ Tenant override        Per-tenant config in PG (tenants.config JSONB)
                                  Example: tenant "fintech" → max_message_bytes=10MB

Level 4 ─ Protocol override      Per-protocol config in broker.yaml
                                  Example: protocols.kafka.linger-ms=5

Level 3 ─ Server override        Per-broker config in broker.yaml
                                  Example: broker-2 max-connections=5000

Level 2 ─ YAML file              broker.yaml (cluster defaults)
                                  Example: storage.retention-ms=604800000

Level 1 ─ Environment variable   IVY_ prefix, dot→underscore
                                  Example: IVY_STORAGE_RETENTION_MS=604800000

Level 0 ─ Hardcoded default      Built into ConfigKey definitions
                                  Example: retention-ms default = 604800000
```

---

## ConfigKey Definition

```java
record ConfigKey<T>(
    String name,              // dot-separated: "storage.retention-ms"
    Class<T> type,            // Integer.class, Long.class, String.class, etc.
    T defaultValue,           // Level 0 fallback
    boolean hotReloadable,    // true = can change without restart
    Predicate<T> validator    // validation (e.g., value > 0)
) {}
```

**Example keys:**

| Key | Type | Default | Hot-Reload | Validation |
|-----|------|---------|------------|------------|
| `storage.retention-ms` | Long | 604800000 | Yes | > 0 |
| `storage.segment-max-bytes` | Int | 536870912 | Yes | > 0 |
| `storage.flush-interval-ms` | Int | 200 | Yes | > 0 |
| `broker.max-connections` | Int | 10000 | Yes | > 0 |
| `broker.max-message-bytes` | Int | 1048576 | Yes | > 0, ≤ 16MB |
| `security.sasl.scram.min-iterations` | Int | 4096 | No | ≥ 4096 |
| `security.reauth.max-reauth-ms` | Long | 0 | Yes | ≥ 0 |
| `security.acl.cache-ttl-minutes` | Int | 5 | Yes | > 0 |

---

## Resolution Logic

```java
class ConfigResolver {
    <T> T resolve(ConfigKey<T> key, TenantId tenantId, ProtocolId protocol) {
        // Level 5: tenant override (from PG tenants.config JSONB)
        T tenantValue = tenantOverrides.get(tenantId, key);
        if (tenantValue != null) return tenantValue;

        // Level 4: protocol override (from broker.yaml protocols section)
        T protocolValue = protocolOverrides.get(protocol, key);
        if (protocolValue != null) return protocolValue;

        // Level 3: server override (from broker.yaml server section)
        T serverValue = serverOverrides.get(key);
        if (serverValue != null) return serverValue;

        // Level 2: YAML file default
        T yamlValue = yamlConfig.get(key);
        if (yamlValue != null) return yamlValue;

        // Level 1: environment variable (IVY_STORAGE_RETENTION_MS)
        String envKey = "IVY_" + key.name().replace('.', '_').replace('-', '_').toUpperCase();
        String envValue = System.getenv(envKey);
        if (envValue != null) return key.type().cast(parse(envValue, key.type()));

        // Level 0: hardcoded default
        return key.defaultValue();
    }
}
```

---

## Hot-Reload

### Tenant Overrides (PG-Backed)

Tenant config changes are detected via PG `LISTEN/NOTIFY` on `tenant_config_change` channel:

```
Admin updates tenants.config JSONB for tenant "acme"
  → PG trigger fires NOTIFY tenant_config_change, 'acme-tenant-uuid'
  → All brokers receive notification
  → ConfigResolver reloads tenant config from PG
  → Affected connections pick up new config on next operation
```

**Reload latency:** <100ms (LISTEN/NOTIFY) with 60s polling fallback.

### YAML File Reload

`BrokerConfigWatcher` uses `WatchService` (NIO) to detect `broker.yaml` file changes:

```
File modified → re-parse YAML → validate all changed keys → atomic swap of config map
```

Only `hotReloadable = true` keys take effect. Non-hot-reloadable keys log a warning and
require broker restart.

---

## Per-Tenant Config Examples

```json
// tenants.config JSONB for tenant "fintech"
{
  "auth": {
    "saslMechanisms": ["SCRAM-SHA-512", "OAUTHBEARER"],
    "maxReauthMs": 1800000,
    "oauthbearer": {
      "allowedIssuers": ["https://auth.fintech.com"],
      "audience": "ivy-broker"
    }
  },
  "tls": {
    "certPath": "/etc/ivy/tenants/fintech/cert.pem",
    "keyPath": "/etc/ivy/tenants/fintech/key.pem",
    "mtlsMode": "REQUIRE"
  },
  "quotas": {
    "produceByteRate": 104857600,
    "consumeByteRate": 209715200,
    "requestRate": 10000,
    "maxConnections": 500,
    "maxTopics": 100,
    "maxPartitionsPerTopic": 32
  },
  "storage": {
    "retentionMs": 2592000000,
    "maxMessageBytes": 10485760
  }
}
```

---

## Ceiling Constraints

Per-tenant overrides cannot **exceed** broker-level limits:

```java
<T extends Comparable<T>> T resolveWithCeiling(ConfigKey<T> key, TenantId tenantId) {
    T tenantValue = tenantOverrides.get(tenantId, key);
    T brokerMax = resolve(key, null, null);  // Level 0-4 resolution (no tenant)
    if (tenantValue != null && tenantValue.compareTo(brokerMax) > 0) {
        return brokerMax;  // ceiling: tenant cannot exceed broker max
    }
    return tenantValue != null ? tenantValue : brokerMax;
}
```

**Example:** Broker sets `max-message-bytes = 16MB`. Tenant "fintech" sets `max-message-bytes = 10MB`
→ effective = 10MB. If tenant tries 20MB → capped at 16MB.

---

## Environment Variable Mapping

Environment variables use `IVY_` prefix with dots and dashes converted to underscores:

| Config Key | Environment Variable |
|-----------|---------------------|
| `storage.retention-ms` | `IVY_STORAGE_RETENTION_MS` |
| `security.sasl.scram.min-iterations` | `IVY_SECURITY_SASL_SCRAM_MIN_ITERATIONS` |
| `broker.max-connections` | `IVY_BROKER_MAX_CONNECTIONS` |

---

## broker.yaml Structure

```yaml
broker:
  id: ${BROKER_ID}                    # UUID, from env or auto-generated
  host: 0.0.0.0
  max-connections: 10000
  max-message-bytes: 1048576

protocols:
  kafka:
    port: 9092
    tls-port: 9093
    linger-ms: 5
  amqp091:
    port: 5672
    tls-port: 5671
  amqp10:
    port: 5673
    tls-port: 5674
  mqtt311:
    port: 1883
    tls-port: 8883
  mqtt5:
    port: 1884
    tls-port: 8884
  mysql:
    port: 3306
    tls-port: 3307
  postgresql:
    port: 5432
    tls-port: 5433
  http:
    port: 8081
    tls-port: 8443

storage:
  postgresql:
    jdbc-url: jdbc:postgresql://localhost:5432/ivy
    username: ivy
    password: ${IVY_PG_PASSWORD}
  log-segments:
    data-dir: /var/lib/ivy/data
    retention-ms: 604800000

security:
  sasl:
    mechanisms: [SCRAM-SHA-512, SCRAM-SHA-256, OAUTHBEARER, PLAIN]
  tls:
    min-protocol: TLSv1.2
  acl:
    default-policy: DENY

cluster:
  secret: ${IVY_CLUSTER_SECRET}
  inter-broker-port: 9094

tenants:
  mode: sni                           # "single" or "sni"
  auto-provision: false
```

---

*Last updated: 2026-03-25*
