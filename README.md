# Ivy Broker

A focused, production-quality multi-protocol message broker.

## What it is

- **8 protocols**: Kafka, AMQP 0-9-1, AMQP 1.0, MQTT 3.1.1, MQTT 5.0, MySQL wire (read-only), PgWire (read-only), HTTP REST (produce/consume)
- **PostgreSQL** as single source of truth (PG-first writes, ACK after COMMIT)
- **Clustering** via HRW partition leadership — no Raft, no Zookeeper
- **Dead letter queues** — first-class, all protocols, configurable triggers
- **Re-authentication** — Kafka KIP-368 in-band re-auth; graceful reconnect for AMQP/MQTT; stateless for HTTP
- **Cross-protocol bridges** — any protocol can produce; any protocol can consume from the same partition
- **Java 26** with Valhalla value classes

## Design Documents

| Document | Description |
|----------|-------------|
| [SYSTEM_OVERVIEW.md](docs/SYSTEM_OVERVIEW.md) | Architecture overview, module graph, data flow |
| [MODULE_DESIGN.md](docs/MODULE_DESIGN.md) | Each module's responsibilities and key classes |
| [WRITE_PATH.md](docs/WRITE_PATH.md) | PG-first write path, batching, epoch fencing |
| [READ_PATH.md](docs/READ_PATH.md) | Three-tier read (LogSegment → inter-broker → PG) |
| [CLUSTERING.md](docs/CLUSTERING.md) | HRW leadership, heartbeat, fencing, inter-broker RPC |
| [DEAD_LETTER_QUEUE.md](docs/DEAD_LETTER_QUEUE.md) | DLQ triggers, headers, per-protocol behavior |
| [PROTOCOLS.md](docs/PROTOCOLS.md) | Kafka, AMQP, MQTT, MySQL, PgWire wire protocol details |
| [POSTGRES_SCHEMA.md](docs/POSTGRES_SCHEMA.md) | Full PostgreSQL schema with DDL |
| [STORAGE.md](docs/STORAGE.md) | LogSegment format, OffsetIndex, StorageFlusher |
| [IMPLEMENTATION_PHASES.md](docs/IMPLEMENTATION_PHASES.md) | 5-phase build plan with task checklists |
| [MULTI_TENANT.md](docs/MULTI_TENANT.md) | SNI resolution, partition isolation, tenant lifecycle, per-tenant TLS/auth/quotas |
| [TRANSACTIONS.md](docs/TRANSACTIONS.md) | Transaction design — Kafka KIP-98, AMQP Tx, AMQP 1.0 TransactionController, isolation levels, LSO, abort tracking |
| [RE_AUTH.md](docs/RE_AUTH.md) | Re-authentication design — Kafka KIP-368, MQTT AUTH, MySQL COM_CHANGE_USER, reconnect-based, stateless HTTP |
| [INTERNAL_LOAD_BALANCER.md](docs/INTERNAL_LOAD_BALANCER.md) | RouteTable, PartitionRouter (Murmur2), HRWRouter, write forwarding, read tiers, epoch fencing |
| [RULES.md](docs/RULES.md) | Invariants enforced across the codebase (R1–R30) |

## Module Structure

```
ivy-common/               — branded types, ProtocolBundle SPI, sealed interfaces
ivy-storage/              — LogSegment cache + PostgresStorageEngine
ivy-broker/               — engine, write/read path, clustering, DLQ, re-auth
ivy-protocol-kafka/       — Kafka wire: codec + handlers co-located (KIP-368 re-auth)
ivy-protocol-amqp/        — AMQP 0-9-1 (amqp/) + AMQP 1.0 (amqp10/) sub-packages
ivy-protocol-mqtt/        — MQTT 3.1.1 + 5.0, version-negotiated at CONNECT
ivy-protocol-postgresql/  — PgWire read-only SQL view
ivy-protocol-mysql/       — MySQL wire read-only SQL view
ivy-protocol-http/        — HTTP REST produce/consume (port 8081), stateless auth
ivy-server/               — thin assembly: ProtocolBundleRegistry + Netty bootstrap
ivy-testing/              — Testcontainers fixtures, multi-protocol helpers
```

Each protocol module registers via `ServiceLoader` (`ProtocolBundle` SPI) — `ivy-server` has no handler logic.

## Quick Architecture

```
Client → Protocol Handler → BrokerEngine
  → HRWRouter: am I the owner?
    YES → WriteAccumulator → WriteWorker → PG COMMIT → ACK
    NO  → ForwardWriteManager → owner broker → same path
```

## Ports

| Protocol | Port | TLS Port |
|----------|------|----------|
| Kafka | 9092 | 9093 |
| AMQP 0-9-1 | 5672 | 5671 |
| AMQP 1.0 | 5673 | 5674 |
| MQTT 3.1.1 | 1883 | 8883 |
| MQTT 5.0 | 1884 | 8884 |
| MySQL | 3306 | 3307 |
| PgWire | 5432 | 5433 |
| HTTP REST (produce/consume) | 8081 | 8443 |
| Inter-broker RPC | 9094 | — |
| HTTP health/metrics | 8080 | — |
