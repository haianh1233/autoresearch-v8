# PostgreSQL Schema

PostgreSQL is the **single source of truth** in cluster mode.
ACK is sent to the client only after `COMMIT` succeeds.
LogSegment is an async read cache populated after ACK.

---

## Tables

### broker_registry
Tracks every broker in the cluster. Updated via heartbeat.

```sql
CREATE TABLE broker_registry (
  broker_id          UUID        PRIMARY KEY,
  host               TEXT        NOT NULL,
  port               INT         NOT NULL,
  inter_broker_port  INT         NOT NULL,
  status             TEXT        NOT NULL CHECK (status IN (
                       'STARTING','ACTIVE','DRAINING','SHUTDOWN','FENCED')),
  incarnation_id     UUID        NOT NULL,  -- new UUID on each restart
  last_heartbeat     TIMESTAMPTZ NOT NULL,
  metadata           JSONB       NOT NULL DEFAULT '{}'
);

CREATE INDEX idx_broker_registry_status ON broker_registry(status);
CREATE INDEX idx_broker_registry_heartbeat ON broker_registry(last_heartbeat)
  WHERE status = 'ACTIVE';
```

**Notes:**
- `incarnation_id` changes on every restart. Used to detect stale RPC connections.
- `status = 'FENCED'` means the broker has missed heartbeats and its partitions have been re-elected.
- Stale threshold: `last_heartbeat < now() - interval '10 seconds'`.

---

### topics

```sql
CREATE TABLE topics (
  topic_id         UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id        UUID    NOT NULL,
  name             TEXT    NOT NULL,
  partition_count  INT     NOT NULL DEFAULT 1 CHECK (partition_count > 0),
  retention_ms     BIGINT  NOT NULL DEFAULT 604800000,  -- 7 days
  cleanup_policy   TEXT    NOT NULL DEFAULT 'delete' CHECK (cleanup_policy IN ('delete','compact')),
  max_message_bytes INT    NOT NULL DEFAULT 1048576,    -- 1MB
  config           JSONB   NOT NULL DEFAULT '{}',
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (tenant_id, name)
);

CREATE INDEX idx_topics_tenant ON topics(tenant_id);
```

**Notes:**
- `name` starting with `__dlq.` is a dead letter queue topic (auto-created by DlqRouter).
- `cleanup_policy = 'compact'` is used for internal metadata topics (`__consumer_offsets`, etc.).

---

### partitions

```sql
CREATE TABLE partitions (
  partition_id   UUID  PRIMARY KEY DEFAULT gen_random_uuid(),
  topic_id       UUID  NOT NULL REFERENCES topics(topic_id) ON DELETE CASCADE,
  tenant_id      UUID  NOT NULL,
  partition_num  INT   NOT NULL CHECK (partition_num >= 0),
  state          TEXT  NOT NULL DEFAULT 'ACTIVE' CHECK (state IN ('ACTIVE','OFFLINE','DELETED')),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (topic_id, partition_num)
);

CREATE INDEX idx_partitions_topic ON partitions(topic_id);
CREATE INDEX idx_partitions_tenant ON partitions(tenant_id);
```

---

### partition_offsets
The **CAS target** for partition leadership and write serialization.

```sql
CREATE TABLE partition_offsets (
  partition_id   UUID    PRIMARY KEY REFERENCES partitions(partition_id),
  next_offset    BIGINT  NOT NULL DEFAULT 0,
  leader_id      UUID    REFERENCES broker_registry(broker_id),
  leader_epoch   INT     NOT NULL DEFAULT 0,
  lso            BIGINT  NOT NULL DEFAULT 0   -- last stable offset (transactions)
);
```

**Write fencing pattern:**
```sql
UPDATE partition_offsets
SET next_offset = next_offset + :batch_size
WHERE partition_id = :pid
  AND leader_epoch = :expected_epoch    -- EPOCH FENCE: rejects stale leaders
RETURNING next_offset - :batch_size AS base_offset;
-- If 0 rows updated → WrongEpochException → forward to new owner
```

---

### messages
All messages from all protocols. Append-only.

```sql
CREATE TABLE messages (
  partition_id  UUID      NOT NULL REFERENCES partitions(partition_id),
  offset_num    BIGINT    NOT NULL,
  tenant_id     UUID      NOT NULL,
  key           BYTEA,
  value         BYTEA     NOT NULL,
  headers       BYTEA,               -- packed: [key_len:2][key][val_len:4][val] repeated
  timestamp_ms  BIGINT    NOT NULL,
  protocol_id   SMALLINT  NOT NULL,  -- 1=Kafka,2=AMQP,3=MQTT,4=MySQL,5=PgWire
  is_dlq        BOOLEAN   NOT NULL DEFAULT false,
  PRIMARY KEY (partition_id, offset_num)
) PARTITION BY RANGE (offset_num);   -- optional: time-based or offset-based partitioning

CREATE INDEX idx_messages_tenant ON messages(tenant_id, partition_id, offset_num);
CREATE INDEX idx_messages_timestamp ON messages(partition_id, timestamp_ms);
```

**Write path:**
```sql
-- In same transaction as partition_offsets UPDATE:
COPY messages (partition_id, offset_num, tenant_id, key, value, headers, timestamp_ms, protocol_id)
FROM STDIN WITH (FORMAT binary);
```

**Notes:**
- Binary COPY is ~3-5x faster than parameterized INSERT for bulk writes.
- `is_dlq = true` means this message was routed via DlqRouter.

---

### producer_state
Idempotent producer deduplication. Prevents duplicate messages on retry.

```sql
CREATE TABLE producer_state (
  producer_id      BIGINT    NOT NULL,
  partition_id     UUID      NOT NULL REFERENCES partitions(partition_id),
  producer_epoch   SMALLINT  NOT NULL,
  last_sequence    INT       NOT NULL,
  last_offset      BIGINT    NOT NULL,
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (producer_id, partition_id)
);
```

**Upsert in same write transaction:**
```sql
INSERT INTO producer_state (producer_id, partition_id, producer_epoch, last_sequence, last_offset)
VALUES (:pid, :part, :epoch, :seq, :offset)
ON CONFLICT (producer_id, partition_id)
DO UPDATE SET producer_epoch = :epoch, last_sequence = :seq,
              last_offset = :offset, updated_at = now()
WHERE producer_state.producer_epoch <= :epoch;
```

---

### consumer_groups

```sql
CREATE TABLE consumer_groups (
  group_id      TEXT  NOT NULL,
  tenant_id     UUID  NOT NULL,
  protocol      TEXT  NOT NULL DEFAULT 'kafka' CHECK (protocol IN ('kafka','amqp','mqtt')),
  state         TEXT  NOT NULL DEFAULT 'EMPTY'
                  CHECK (state IN ('EMPTY','PREPARING_REBALANCE','COMPLETING_REBALANCE','STABLE','DEAD')),
  generation    INT   NOT NULL DEFAULT 0,
  leader_member TEXT,          -- member_id of the elected group leader
  protocol_name TEXT,          -- assignment strategy (e.g. 'range', 'roundrobin')
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (group_id, tenant_id)
);

CREATE INDEX idx_consumer_groups_tenant ON consumer_groups(tenant_id);
```

---

### consumer_offsets
Committed offsets per (group, partition).

```sql
CREATE TABLE consumer_offsets (
  group_id          TEXT    NOT NULL,
  partition_id      UUID    NOT NULL REFERENCES partitions(partition_id),
  tenant_id         UUID    NOT NULL,
  committed_offset  BIGINT  NOT NULL,
  metadata          TEXT,
  committed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (group_id, partition_id)
);

CREATE INDEX idx_consumer_offsets_group ON consumer_offsets(group_id, tenant_id);
```

---

### transactions

```sql
CREATE TABLE transactions (
  txn_id          TEXT      NOT NULL,
  producer_id     BIGINT    NOT NULL,
  producer_epoch  SMALLINT  NOT NULL,
  tenant_id       UUID      NOT NULL,
  state           TEXT      NOT NULL
                    CHECK (state IN (
                      'ONGOING','PREPARE_COMMIT','PREPARE_ABORT',
                      'COMPLETE_COMMIT','COMPLETE_ABORT')),
  timeout_ms      INT       NOT NULL DEFAULT 60000,
  partitions      UUID[]    NOT NULL DEFAULT '{}',
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (txn_id, tenant_id)
);

CREATE INDEX idx_transactions_producer ON transactions(producer_id, tenant_id);
CREATE INDEX idx_transactions_state ON transactions(state) WHERE state IN ('ONGOING','PREPARE_COMMIT','PREPARE_ABORT');
```

---

### dlq_entries
Metadata about messages routed to dead letter queues.

```sql
CREATE TABLE dlq_entries (
  dlq_partition_id      UUID      NOT NULL,
  dlq_offset            BIGINT    NOT NULL,
  original_partition_id UUID      NOT NULL,
  original_offset       BIGINT    NOT NULL,
  tenant_id             UUID      NOT NULL,
  reason                TEXT      NOT NULL
                          CHECK (reason IN ('NACK','TTL_EXPIRED','MAX_RETRIES','SCHEMA_INVALID','CONSUMER_FAILURE')),
  retry_count           INT       NOT NULL DEFAULT 0,
  consumer_group        TEXT,
  failed_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (dlq_partition_id, dlq_offset)
);

CREATE INDEX idx_dlq_entries_original ON dlq_entries(original_partition_id, original_offset);
CREATE INDEX idx_dlq_entries_tenant ON dlq_entries(tenant_id, failed_at DESC);
CREATE INDEX idx_dlq_entries_reason ON dlq_entries(reason, tenant_id);
```

---

### credentials

```sql
CREATE TABLE credentials (
  tenant_id    UUID  NOT NULL,
  username     TEXT  NOT NULL,
  scram_256    BYTEA,   -- SCRAM-SHA-256 server-key + stored-key
  scram_512    BYTEA,   -- SCRAM-SHA-512 server-key + stored-key
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, username)
);
```

---

### acl_entries

```sql
CREATE TABLE acl_entries (
  id             UUID  PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id      UUID  NOT NULL,
  principal      TEXT  NOT NULL,
  resource_type  TEXT  NOT NULL CHECK (resource_type IN ('TOPIC','GROUP','CLUSTER','TRANSACTIONAL_ID')),
  resource_name  TEXT  NOT NULL,   -- exact name or '*' for wildcard
  operation      TEXT  NOT NULL CHECK (operation IN ('PRODUCE','CONSUME','CREATE','DELETE','DESCRIBE','ALTER','ALL')),
  allow          BOOLEAN NOT NULL DEFAULT true,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_acl_entries_lookup ON acl_entries(tenant_id, principal, resource_type, resource_name);
```

---

## Internal Topics (Compacted Metadata)

These topics use `cleanup_policy = 'compact'` — only the latest value per key is retained.

| Topic Name | Key | Value | Purpose |
|------------|-----|-------|---------|
| `__consumer_offsets` | `group_id:partition_id` | committed offset | Consumer offset storage |
| `__consumer_groups` | `group_id:tenant_id` | group state JSON | Group metadata |
| `__transactions` | `txn_id:tenant_id` | transaction state JSON | Transaction log |
| `__schemas` | `subject:version` | schema JSON | Schema registry |

---

## Schema Migration Strategy

- Migrations are versioned SQL files in `ivy-storage/src/main/resources/db/migration/`
- Applied on broker startup via `StorageSchemaManager`
- Format: `V{n}__{description}.sql` (e.g., `V1__initial_schema.sql`)
- Idempotent: `CREATE TABLE IF NOT EXISTS`, `CREATE INDEX IF NOT EXISTS`
- Never `DROP` or `ALTER` existing columns without a new migration version

---

## Connection Pool Configuration (HikariCP)

```yaml
storage:
  postgresql:
    jdbc-url: jdbc:postgresql://localhost:5432/ivy
    username: ivy
    password: ${IVY_PG_PASSWORD}
    pool:
      maximum-pool-size: 20      # max connections per broker
      minimum-idle: 5
      connection-timeout-ms: 3000
      idle-timeout-ms: 600000
      max-lifetime-ms: 1800000
      keepalive-time-ms: 30000
```
