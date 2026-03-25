# Ivy Broker

A focused, production-quality multi-protocol message broker.

## What it is

- **7 protocols**: Kafka, AMQP 0-9-1, AMQP 1.0, MQTT 3.1.1, MQTT 5.0, MySQL wire (read-only), PgWire (read-only)
- **PostgreSQL** as single source of truth (PG-first writes, ACK after COMMIT)
- **Clustering** via HRW partition leadership — no Raft, no Zookeeper
- **Dead letter queues** — first-class, all protocols, configurable triggers
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
| [RULES.md](docs/RULES.md) | Invariants enforced across the codebase |

## Module Structure

```
ivy-common/    — branded types, sealed interfaces (zero external deps)
ivy-storage/   — LogSegment cache + PostgresStorageEngine
ivy-broker/    — engine, write/read path, clustering, DLQ
ivy-codec/     — all 5 protocol wire codecs
ivy-server/    — Netty pipeline, handlers, BrokerMain
```

## Quick Architecture

```
Client → Protocol Handler → BrokerEngine
  → HRWRouter: am I the owner?
    YES → WriteAccumulator → WriteWorker → PG COMMIT → ACK
    NO  → ForwardWriteManager → owner broker → same path
```

## Ports

| Protocol | Port |
|----------|------|
| Kafka | 9092 |
| AMQP 0-9-1 | 5672 |
| AMQP 1.0 | 5673 |
| MQTT 3.1.1 | 1883 |
| MQTT 5.0 | 1884 |
| MySQL | 3306 |
| PgWire | 5432 |
| Inter-broker RPC | 9094 |
| HTTP metrics | 8080 |
