# Dead Letter Queue (DLQ)

## Overview

DLQ is a first-class feature available across all 5 protocols. Messages are routed to a
dead letter topic when they cannot be successfully delivered after exhausting retry logic.

**DLQ topic naming:** `__dlq.<original-topic-name>`

DLQ topics are auto-created on first write. They use the same partition count as the
original topic and can be consumed by any of the 5 protocols.

---

## DLQ Trigger Conditions

| Condition | Protocols | Description |
|-----------|-----------|-------------|
| `NACK` | AMQP | `Basic.Nack(requeue=false)` or `Basic.Reject(requeue=false)` |
| `TTL_EXPIRED` | AMQP, MQTT | Message or queue TTL exceeded before delivery |
| `MAX_RETRIES` | All | Delivery attempt count exceeds `max.delivery.attempts` |
| `SCHEMA_INVALID` | All | Message fails schema validation (if schema registry configured) |
| `CONSUMER_FAILURE` | Kafka | Consumer group `max.poll.failures` exceeded |

---

## DLQ Headers

When a message is routed to the DLQ, the following headers are injected:

| Header Key | Type | Description |
|------------|------|-------------|
| `x-dlq-reason` | string | `NACK`, `TTL_EXPIRED`, `MAX_RETRIES`, `SCHEMA_INVALID`, `CONSUMER_FAILURE` |
| `x-dlq-original-topic` | string | Original topic name |
| `x-dlq-original-partition` | string | Original partition UUID |
| `x-dlq-original-offset` | long | Original message offset |
| `x-dlq-retry-count` | int | Number of delivery attempts made |
| `x-dlq-failed-at` | ISO-8601 | Timestamp of the last failure |
| `x-dlq-consumer-group` | string | Consumer group that rejected (if applicable) |
| `x-dlq-error-message` | string | Human-readable failure reason (optional) |

For protocols that don't support arbitrary headers (MQTT 3.1.1):
DLQ metadata is prepended to the payload as a compact binary prefix.

---

## DlqRouter

```
DlqRouter.shouldRoute(PendingWrite write, int deliveryAttempts, DlqConfig config):
  → true if any trigger condition is met

DlqRouter.route(PendingWrite original, DlqReason reason, int retryCount, String consumerGroup):
  → resolve DLQ topic: "__dlq." + original.topicName
  → auto-create DLQ partition if not exists
  → build new PendingWrite:
      key     = original.key
      value   = original.value
      headers = original.headers + x-dlq-* headers (appended)
      partitionId = DLQ partition (same partition_num as original)
      is_dlq  = true
  → BrokerEngine.write(dlqWrite)    ← normal write path
  → INSERT INTO dlq_entries (...)   ← metadata for observability
```

---

## Per-Protocol DLQ Behavior

### Kafka

**Consumer-side DLQ (application-controlled):**
```
Consumer polls messages → processes → on failure:
  producer.send(DLQ_TOPIC, record)   ← application explicitly sends to DLQ
  consumer.commitSync()              ← advance offset
```

**Broker-side DLQ (retry exhaustion):**
- Configure `max.delivery.attempts` per consumer group
- After N failures (nack without requeue), broker routes to DLQ
- Tracked via `consumer_group_delivery_attempts` (in-memory per partition)

```yaml
# Consumer group DLQ config
consumer-groups:
  my-group:
    max-delivery-attempts: 5
    dlq-enabled: true
```

**Wire protocol:** DLQ headers carried as Kafka record headers (key-value pairs in record batch).

---

### AMQP 0-9-1

AMQP has the richest native DLQ semantics via **Dead Letter Exchange (DLX)**.

**Per-queue DLX configuration (on Queue.Declare):**
```
Queue.Declare(
  queue     = "my-queue",
  arguments = {
    "x-dead-letter-exchange"     : "dlx.exchange",     ← DLX exchange name
    "x-dead-letter-routing-key"  : "my-queue.dlq",     ← routing key for DLQ
    "x-message-ttl"              : 300000,              ← 5min TTL → DLQ on expiry
    "x-max-length"               : 10000,               ← queue overflow → DLQ
    "x-delivery-limit"           : 5                    ← max-retries → DLQ
  }
)
```

**DLQ triggers in AMQP handler:**
1. `Basic.Nack(requeue=false)` → DlqRouter.route(msg, NACK)
2. `Basic.Reject(requeue=false)` → DlqRouter.route(msg, NACK)
3. Message TTL exceeded (`x-message-ttl`) → DlqRouter.route(msg, TTL_EXPIRED)
4. Delivery count exceeds `x-delivery-limit` → DlqRouter.route(msg, MAX_RETRIES)

**DLQ headers in AMQP:** Carried as AMQP message headers (application-level properties).

**Ivy mapping:**
- DLX exchange name maps to `__dlq.<original-topic>` partition
- `x-death` header array added per AMQP spec (for compatibility with RabbitMQ clients)

---

### MQTT 3.1.1

MQTT has no native DLQ concept. Ivy implements it at the broker level.

**DLQ triggers in MQTT handler:**
1. Message TTL exceeded (via `x-message-expiry` extension property in MQTT 3.1.1)
2. Will message delivery failure → DlqRouter.route(willMessage, CONSUMER_FAILURE)
3. Max delivery attempts exceeded (configurable per topic subscription)

**DLQ for QoS levels:**
- QoS 0 (fire and forget): No DLQ (messages are not tracked)
- QoS 1 (at least once): On PUBACK failure after N retries → DLQ
- QoS 2 (exactly once): On PUBCOMP failure after N retries → DLQ

**DLQ topic in MQTT:** `$dlq/<original-topic>` (using MQTT topic hierarchy)

**DLQ metadata in MQTT 3.1.1:**
Since MQTT 3.1.1 has no user properties, DLQ metadata is encoded as a binary prefix:
```
[magic: 2 bytes = 0xDEAD]
[reason: 1 byte]           -- 1=NACK, 2=TTL, 3=MAX_RETRIES, 4=SCHEMA, 5=CONSUMER
[retry_count: 4 bytes]
[failed_at_ms: 8 bytes]
[original_topic_len: 2 bytes]
[original_topic: N bytes]
[original_payload follows]
```

---

### MySQL Wire / PgWire

Read-only protocols — they do not produce messages, so they have no DLQ interaction.
However, they **can read from DLQ topics**:

```sql
-- Read dead letter messages for a topic
SELECT key, value, offset_num, timestamp_ms
FROM __dlq_my_topic     -- maps to topic "__dlq.my-topic"
WHERE offset_num > 0
LIMIT 100;

-- Query DLQ metadata
SELECT *
FROM dlq_entries
WHERE tenant_id = current_tenant()
  AND original_partition_id = ?
ORDER BY failed_at DESC
LIMIT 50;
```

---

## DLQ Configuration

```yaml
broker:
  dlq:
    enabled: true
    default-max-delivery-attempts: 5    # global default; override per topic
    retention-ms: 604800000             # 7 days for DLQ messages
    auto-create: true                   # auto-create DLQ topic on first write
    header-prefix: "x-dlq-"

topics:
  my-critical-topic:
    dlq:
      max-delivery-attempts: 3
      enabled: true
  my-lossy-topic:
    dlq:
      enabled: false                    # discard instead of DLQ
```

---

## DLQ Observability

**Metrics exposed via Prometheus:**
```
ivy_dlq_messages_total{topic, reason, tenant}     # total messages routed to DLQ
ivy_dlq_retry_count_p99{topic, tenant}            # p99 retry count at DLQ time
ivy_dlq_queue_size{topic, tenant}                 # current DLQ backlog
```

**SQL query view (MySQL/PgWire):**
```sql
-- DLQ summary by reason
SELECT reason, COUNT(*), AVG(retry_count)
FROM dlq_entries
WHERE tenant_id = ?
  AND failed_at > now() - interval '1 hour'
GROUP BY reason;

-- Recent DLQ entries for a topic
SELECT * FROM dlq_entries
WHERE original_partition_id IN (
    SELECT partition_id FROM partitions
    WHERE topic_id = (SELECT topic_id FROM topics WHERE name = ?)
)
ORDER BY failed_at DESC
LIMIT 20;
```

---

## DLQ Consumer Pattern

A DLQ consumer should:
1. Subscribe to `__dlq.<topic-name>`
2. Inspect `x-dlq-reason` header
3. Either:
   - **Reprocess**: fix the issue, re-produce to original topic
   - **Alert**: send notification for manual intervention
   - **Discard**: acknowledge and drop (for poison pills)

```java
// Example: Kafka DLQ consumer
consumer.subscribe(List.of("__dlq.orders"));
while (true) {
    records = consumer.poll(Duration.ofSeconds(1));
    for (record : records) {
        String reason = getHeader(record, "x-dlq-reason");
        if ("SCHEMA_INVALID".equals(reason)) {
            alertOpsTeam(record);
        } else if ("MAX_RETRIES".equals(reason)) {
            int retryCount = Integer.parseInt(getHeader(record, "x-dlq-retry-count"));
            if (retryCount < 10) {
                reproductToOriginalTopic(record);
            } else {
                logPoisonPill(record);
            }
        }
    }
    consumer.commitSync();
}
```
