# AMQP Exchange & Binding Engine

> **Related:** [PROTOCOLS.md](PROTOCOLS.md) §AMQP 0-9-1, [ACL_DESIGN.md](ACL_DESIGN.md) §AMQP Wire-Op Mapping,
> [DEAD_LETTER_QUEUE.md](DEAD_LETTER_QUEUE.md) §AMQP DLX

---

## Overview

AMQP 0-9-1 uses exchanges, queues, and bindings to route messages. Ivy implements the full
routing model: direct, fanout, topic, and headers exchange types, with dead letter exchange
(DLX) support.

---

## Exchange Types

### Direct Exchange

Exact match: `routing_key == binding_key`. One message → all queues bound with that exact key.

```
Exchange "orders.direct"
  ├─ Queue "orders.urgent"    ← binding key "urgent"
  ├─ Queue "orders.standard"  ← binding key "standard"

Publish(routing_key="urgent") → Queue "orders.urgent" only
```

### Fanout Exchange

All bound queues receive every message. Routing key is ignored.

```
Exchange "notifications" (fanout)
  ├─ Queue "email_service"
  ├─ Queue "sms_service"
  ├─ Queue "webhook_service"

Publish → all three queues receive the message
```

### Topic Exchange

Wildcard pattern matching on `.`-separated routing keys:
- `*` matches exactly one word
- `#` matches zero or more words

```
Exchange "stock.topic"
  ├─ Queue "nyse_watchers"  ← pattern "stock.*.nyse"
  ├─ Queue "all_alerts"     ← pattern "stock.#"
  ├─ Queue "crypto"         ← pattern "stock.crypto.*"

Publish(routing_key="stock.usd.nyse")
  → "stock.*.nyse" ✓ (nyse_watchers)
  → "stock.#" ✓ (all_alerts)
  → "stock.crypto.*" ✗

Publish(routing_key="stock.crypto.btc")
  → "stock.#" ✓ (all_alerts)
  → "stock.crypto.*" ✓ (crypto)
```

### Headers Exchange

Matches message headers against binding arguments. `x-match=all` (all must match) or
`x-match=any` (at least one). Routing key ignored.

**Status:** Placeholder — not yet implemented in v8.

---

## Topic Exchange Wildcard Matching Algorithm

```java
static boolean topicMatches(String pattern, String routingKey) {
    // Fast path: no wildcards
    if (pattern.indexOf('*') < 0 && pattern.indexOf('#') < 0) {
        return pattern.equals(routingKey);
    }

    String[] patternWords = pattern.split("\\.");
    String[] routingWords = routingKey.split("\\.");

    int pi = 0, ri = 0;
    while (pi < patternWords.length && ri < routingWords.length) {
        String pw = patternWords[pi];
        if ("#".equals(pw)) return true;            // matches rest
        if ("*".equals(pw)) { pi++; ri++; continue; }  // matches one word
        if (!pw.equals(routingWords[ri])) return false;  // literal mismatch
        pi++; ri++;
    }

    if (pi == patternWords.length && ri == routingWords.length) return true;
    // Pattern has trailing # → matches zero remaining
    if (pi == patternWords.length - 1 && "#".equals(patternWords[pi])) return true;
    return false;
}
```

**Key difference from MQTT:** AMQP uses `.` as separator and `*`/`#` as wildcards.
MQTT uses `/` as separator and `+`/`#` as wildcards.

---

## Routing Engine

```java
class Amqp091RoutingEngine {
    ConcurrentHashMap<String, ExchangeRouting> exchanges;

    record ExchangeRouting(String type, CopyOnWriteArrayList<Binding> bindings) {}
    record Binding(String queue, String routingKey) {}

    Set<String> route(String exchange, String routingKey) {
        ExchangeRouting routing = exchanges.get(exchange);
        if (routing == null) return Set.of();

        Set<String> matched = new HashSet<>();
        switch (routing.type()) {
            case "direct"  -> routing.bindings().stream()
                .filter(b -> b.routingKey().equals(routingKey))
                .forEach(b -> matched.add(b.queue()));
            case "fanout"  -> routing.bindings()
                .forEach(b -> matched.add(b.queue()));
            case "topic"   -> routing.bindings().stream()
                .filter(b -> topicMatches(b.routingKey(), routingKey))
                .forEach(b -> matched.add(b.queue()));
        }
        return matched;
    }

    void bind(String exchange, String queue, String routingKey) {
        exchanges.computeIfPresent(exchange, (k, r) -> {
            r.bindings().add(new Binding(queue, routingKey));
            return r;
        });
    }

    void unbind(String exchange, String queue, String routingKey) {
        exchanges.computeIfPresent(exchange, (k, r) -> {
            r.bindings().removeIf(b ->
                b.queue().equals(queue) && b.routingKey().equals(routingKey));
            return r;
        });
    }
}
```

---

## Default Exchanges (Pre-Declared)

These exchanges exist automatically per tenant and **cannot be deleted**:

| Name | Type | Purpose |
|------|------|---------|
| `""` (empty string) | direct | Default exchange: routing key = queue name |
| `amq.direct` | direct | Standard direct exchange |
| `amq.topic` | topic | Standard topic exchange |
| `amq.fanout` | fanout | Standard fanout exchange |
| `amq.headers` | headers | Standard headers exchange |

Attempting to delete or redeclare with a different type returns
`Connection.Close(reply-code=530, reply-text="Cannot delete default exchange")`.

---

## Queue Lifecycle

### Queue.Declare

```java
void handleDeclare(AmqpQueueDeclareData data) {
    String name = data.queue().isEmpty()
        ? "amq.gen-" + UUID.randomUUID().toString().substring(0, 8)   // auto-name
        : data.queue();

    if (data.passive()) {
        // Check only, no create. Return 404 channel error if absent.
        return queues.containsKey(name)
            ? queueDeclareOk(name, 0, 0)
            : channelClose(404, "NOT_FOUND");
    }

    queues.put(name, new QueueEntry(data.durable(), data.exclusive(), data.autoDelete(),
                                     data.dlxExchange(), data.dlxRoutingKey()));
    if (data.exclusive()) exclusiveQueues.add(name);  // connection-scoped
}
```

| Flag | Behaviour |
|------|-----------|
| `durable` | Queue survives broker restart (requires PG persistence) |
| `exclusive` | Visible only to declaring connection; auto-deleted on close |
| `autoDelete` | Deleted when last consumer cancels (NOT when never had consumers) |
| `passive` | Check existence only; do not create |

### Queue.Bind / Unbind

```
Queue.Bind(exchange, queue, routingKey)   → routingEngine.bind(exchange, queue, routingKey)
Queue.Unbind(exchange, queue, routingKey) → routingEngine.unbind(exchange, queue, routingKey)
```

Non-existent binding on unbind is silent success (per AMQP spec).

### Queue.Delete / Purge

```
Queue.Delete(queue)  → queues.remove(queue); returns messageCount
Queue.Purge(queue)   → clears all messages; returns messageCount purged
```

---

## Dead Letter Exchange (DLX)

Queues declared with `x-dead-letter-exchange` route rejected/expired messages to a DLX:

```
Queue.Declare(
  queue = "order-processing",
  arguments = {
    "x-dead-letter-exchange": "dlx",
    "x-dead-letter-routing-key": "failed-orders",
    "x-message-ttl": 300000,
    "x-delivery-limit": 5
  }
)
```

**DLX triggers:**
- `Basic.Nack(requeue=false)` or `Basic.Reject(requeue=false)`
- Message TTL expired (`x-message-ttl`)
- Delivery limit exceeded (`x-delivery-limit`)
- Queue length exceeded (`x-max-length`)

**Routing:** message published to DLX exchange with `x-dead-letter-routing-key` (or original
routing key if not set). `x-death` header added for RabbitMQ client compatibility.

---

## Exchange-to-Partition Mapping

AMQP exchanges map to Ivy's partition model:

```
AMQP Exchange  →  Ivy topic prefix (exchange name)
AMQP Queue     →  Ivy topic (exchange.name + "." + queue.name)
AMQP Binding   →  routing rule (in-memory per connection)
AMQP Message   →  Ivy PendingWrite (headers + body)
AMQP Consumer  →  Ivy subscription on partition(s)
AMQP Ack       →  Ivy consumer offset commit
```

**Fanout routing** → message written to ALL partitions of the topic (one write per bound queue).

**Tenant isolation:** all exchanges, queues, and bindings keyed by `(tenantId, name)`.
Exchange "orders" in tenant A is completely separate from exchange "orders" in tenant B.

---

## Binding Persistence

**Current (v8):** bindings stored in-memory only (`CopyOnWriteArrayList`). Lost on restart.

**Planned PG persistence (v9):**

```sql
CREATE TABLE amqp_exchanges (
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(10) NOT NULL,
    durable BOOLEAN NOT NULL,
    auto_delete BOOLEAN NOT NULL,
    arguments JSONB,
    PRIMARY KEY (tenant_id, name)
);

CREATE TABLE amqp_bindings (
    tenant_id UUID NOT NULL,
    source_exchange VARCHAR(255) NOT NULL,
    destination_queue VARCHAR(255) NOT NULL,
    routing_key VARCHAR(255) NOT NULL,
    arguments JSONB,
    PRIMARY KEY (tenant_id, source_exchange, destination_queue, routing_key)
);
```

On restart: replay from PG → rebuild `Amqp091RoutingEngine` → durable exchanges/queues/bindings
restored before connections accepted.

---

## Configuration

```yaml
protocols:
  amqp091:
    max-exchanges-per-tenant: 1000
    max-queues-per-tenant: 10000
    max-bindings-per-exchange: 10000
    max-frame-size: 131072           # 128 KB
    heartbeat-interval-s: 60
    channel-max: 2047
```

---

*Last updated: 2026-03-25*
