# Ivy Broker — System Overview

## Vision

A focused, production-quality multi-protocol message broker with eight protocol adapters
(Kafka, AMQP 0-9-1, AMQP 1.0, MQTT 3.1.1, MQTT 5.0, MySQL-wire, PgWire, HTTP REST),
PostgreSQL as the single source of truth, HRW-based partition leadership for clustering,
and first-class dead letter queues.

Everything is a log. Every protocol writes to the same partitions. Every consumer reads
from the same offsets. PostgreSQL decides durability; the broker decides routing.

---

## Architecture Layers

```
┌──────────────────────────────────────────────────────────────────┐
│                         ivy-server                               │
│                                                                  │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────┐ ┌──────┐ ┌──────┐ │
│  │ Kafka  │ │AMQP091 │ │AMQP 10 │ │MQTT311 │ │ MQTT5  │ │MySQL │ │ PG   │ │HTTP  │ │
│  │Handler │ │Handler │ │Handler │ │Handler │ │Handler │ │Handlr│ │Handlr│ │Handlr│ │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └──────┘ └──────┘ └──────┘ │
│  NettyPipelineFactory │ ProtocolDetector │ BrokerMain            │
├──────────────────────────────────────────────────────────────────┤
│                         ivy-codec                                │
│   ivy-protocol-kafka  │  ivy-protocol-amqp   │  ivy-protocol-mqtt   │  ivy-protocol-pg/mysql  │  ivy-protocol-http  │
├──────────────────────────────────────────────────────────────────┤
│                         ivy-broker                               │
│  BrokerEngine │ WriteWorker │ ReadAccumulator │ DlqRouter        │
│  ConsumerGroupCoordinator │ TransactionCoordinator               │
│  HRWRouter │ ClusterManager │ HeartbeatWriter │ ForwardWriteMgr  │
│  InterBrokerRpcServer │ InterBrokerRpcClient │ MetadataImage     │
├──────────────────────────────────────────────────────────────────┤
│                         ivy-storage                              │
│  LogSegment (read cache) │ OffsetIndex (mmap) │ StorageFlusher   │
│  PostgresStorageEngine (source of truth)                         │
├──────────────────────────────────────────────────────────────────┤
│                         ivy-common                               │
│  Branded Types │ Value Records │ Sealed Interfaces               │
│  BrokerEngine SPI │ StorageEngine SPI │ AuthEngine SPI           │
└──────────────────────────────────────────────────────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │         PostgreSQL (HA)           │
              │  messages │ partitions │ topics   │
              │  broker_registry │ consumer_*     │
              │  transactions │ dlq_entries       │
              └──────────────────────────────────┘
```

---

## Module Dependency Graph

```
ivy-server
  ├── ivy-protocol-kafka
  ├── ivy-protocol-amqp       (AMQP 0-9-1 + 1.0)
  ├── ivy-protocol-mqtt       (MQTT 3.1.1 + 5.0)
  ├── ivy-protocol-postgresql
  ├── ivy-protocol-mysql
  ├── ivy-protocol-http       (REST produce/consume, port 8081)
  ├── ivy-broker
  │     └── ivy-storage
  │           └── ivy-common
  └── ivy-common
```

Dependency rules (enforced at build time):
- `ivy-common` has **zero** external runtime dependencies
- `ivy-storage` depends only on `ivy-common` + JDBC/HikariCP
- `ivy-broker` depends on `ivy-common` + `ivy-storage`
- Each `ivy-protocol-*` depends on `ivy-common` + `ivy-broker` + Netty only
- No protocol module may depend on another protocol module
- `ivy-server` depends on all modules; it is the only assembly point
- Codec and handler are **co-located per protocol** — there is no separate `ivy-codec` module

---

## Data Flow Summary

### Produce (any protocol → stored in PG)
```
Client (Kafka/AMQP/MQTT) → Handler → BrokerEngine.write()
  → HRWRouter: am I the partition owner?
    YES → WriteAccumulator → WriteWorker → PG COMMIT → ACK → LogSegment (async)
    NO  → ForwardWriteManager → InterBrokerRpc → owner broker → same path
```

### Consume (any protocol reading from PG/LogSegment)
```
Client (Kafka/AMQP/MQTT) → Handler → BrokerEngine.fetch()
  → ReadAccumulator:
      L1: LogSegment (owner's in-memory cache, <1ms)
      L2: Owner cache via InterBrokerRpc fetch (0.5ms)
      L3: PostgreSQL fetchRange (2ms fallback)
  → Return records → Handler encodes in protocol wire format → Client
```

### SQL Query (MySQL/PgWire — read-only view)
```
Client (JDBC/psql) → Handler → SQL parser → View resolver
  → SELECT from topic    → BrokerEngine.fetch()
  → SELECT from metadata → direct PG query (topics, partitions, consumer_groups, broker_registry)
  → SHOW TABLES          → list topics for tenant
  → Return ResultSet in MySQL/PgWire wire format

Note: SQL clients see messages from ALL producing protocols (Kafka, AMQP, MQTT, HTTP).
The protocol_id column identifies the producer: 1=Kafka, 2=AMQP 0-9-1, 3=AMQP 1.0,
4=MQTT 3.1.1, 5=MQTT 5.0, 8=HTTP.
```

### HTTP Produce/Consume (REST API — port 8081)
```
Producer:  POST /topics/{topic}/messages         → BrokerEngine.write()
Consumer:  GET  /topics/{topic}/messages?offset= → BrokerEngine.fetch()
Long-poll: GET  /topics/{topic}/messages?waitMs= → BrokerEngine.fetch() + FlushEvent
Metadata:  GET  /topics                          → BrokerEngine.listTopics()

Auth: Bearer token (JWT) or X-API-Key header — per-request, stateless.
No re-auth: tokens expire at JWT exp; client obtains a new token and retries.
```

### Dead Letter Queue
```
Consumer nacks / TTL expires / max-retries exceeded
  → DlqRouter.route(originalMessage, reason)
    → inject x-dlq-* headers
    → BrokerEngine.write() to __dlq.<original-topic>
    → stored as normal partition → consumable by any protocol
```

---

## Key Design Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| Storage source of truth | PostgreSQL (PG-first) | ACID durability; ACK only after PG COMMIT |
| LogSegment role | Async read cache | Performance optimization; not required for correctness |
| Leadership election | HRW (Rendezvous Hash) | No Raft complexity; deterministic; O(1) lookup |
| Protocol count | 8 (focused) | Kafka + AMQP 0-9-1 + AMQP 1.0 + MQTT 3.1.1 + MQTT 5.0 + MySQL + PgWire + HTTP |
| Module count | 5 (clean) | Each module has a single clear responsibility |
| DLQ | First-class, all messaging protocols | Missing in ivy-v9; essential for AMQP/MQTT semantics |
| Multi-tenancy | Yes, SNI-based | Essential for SaaS; TenantId scopes all operations |
| Java version | 26 + `--enable-preview` | Valhalla value classes; matches mature reference projects |
| SQL protocols | Read-only query view | Observability/debugging; not full messaging endpoints |
| Transactions | Kafka-style, routed via BrokerEngine | Universal across all protocols |
| Consensus | PG CAS (epoch fencing) | Replaces Raft; PG is already the source of truth |

---

## Ports

| Protocol | Default Port | TLS Port |
|----------|-------------|----------|
| Kafka | 9092 | 9093 |
| AMQP 0-9-1 | 5672 | 5671 |
| AMQP 1.0 | 5673 | 5674 |
| MQTT 3.1.1 | 1883 | 8883 |
| MQTT 5.0 | 1884 | 8884 (or shared 1883 via version negotiation) |
| MySQL wire | 3306 | 3307 |
| PgWire | 5432 | 5433 |
| Inter-broker RPC | 9094 | — |
| HTTP REST API (produce/consume) | 8081 | 8443 |
| HTTP health/metrics | 8080 | — |

---

## Technology Stack

| Component | Library | Version |
|-----------|---------|---------|
| Language | Java | 26 + `--enable-preview` |
| Network I/O | Netty | 4.2.x |
| Kafka client (testing only) | kafka-clients | **4.2.0** |
| Database | PostgreSQL JDBC | 42.7.x |
| Connection pool | HikariCP | 6.x |
| Metrics | Micrometer + Prometheus | 1.14.x |
| Compression | Zstd, Snappy, LZ4 | latest |
| Serialization | Jackson | 2.18.x |
| Build | Maven | 3.9.x |
| Testing | JUnit 5 + Testcontainers | 5.x / 1.20.x |
| Concurrency | JCTools | 4.x |
| Caching | Caffeine | 3.x |

---

## Design Documentation Index

| Topic | Document |
|-------|---------|
| Write path (produce → PG) | [WRITE_PATH.md](WRITE_PATH.md) |
| Read path (fetch, push) | [READ_PATH.md](READ_PATH.md) |
| Storage (segments, flusher, cleaner) | [STORAGE.md](STORAGE.md) |
| Clustering (HRW, heartbeat, fencing) | [CLUSTERING.md](CLUSTERING.md) |
| Multi-tenancy (SNI, TenantId, ACLs) | [MULTI_TENANT.md](MULTI_TENANT.md) |
| Transactions (exactly-once, 2PC) | [TRANSACTIONS.md](TRANSACTIONS.md) |
| **Shutdown & crash recovery** | **[SHUTDOWN_AND_RECOVERY.md](SHUTDOWN_AND_RECOVERY.md)** |
| Re-authentication | [RE_AUTH.md](RE_AUTH.md) |
| PostgreSQL schema | [POSTGRES_SCHEMA.md](POSTGRES_SCHEMA.md) |
| Protocol details | [PROTOCOLS.md](PROTOCOLS.md) |
| Dead letter queues | [DEAD_LETTER_QUEUE.md](DEAD_LETTER_QUEUE.md) |
| Internal load balancer | [INTERNAL_LOAD_BALANCER.md](INTERNAL_LOAD_BALANCER.md) |
| Module structure | [MODULE_DESIGN.md](MODULE_DESIGN.md) |
| Implementation rules | [RULES.md](RULES.md) |
| Implementation phases | [IMPLEMENTATION_PHASES.md](IMPLEMENTATION_PHASES.md) |

---

## Non-Goals

- Raft / Paxos consensus (HRW + PG CAS is sufficient)
- ISR (In-Sync Replicas) — PG replication handles durability
- More than 8 protocols in initial version
- Schema Registry (can be added later)
- Multi-cluster federation (single PG cluster scope for now)
- Built-in Kafka Connect / Streams compatibility beyond wire protocol
