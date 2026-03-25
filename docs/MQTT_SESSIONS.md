# MQTT Session Persistence

> **Related:** [PROTOCOLS.md](PROTOCOLS.md) §MQTT, [CONSUMER_GROUPS.md](CONSUMER_GROUPS.md),
> [POSTGRES_SCHEMA.md](POSTGRES_SCHEMA.md)

---

## Overview

MQTT sessions manage per-client state: subscriptions, in-flight QoS messages, and offline
message queues. Session persistence determines whether state survives client disconnection.

---

## Clean Session vs Persistent Session

### MQTT 3.1.1

| `cleanSession` | Behaviour |
|----------------|-----------|
| `true` | Ephemeral: all state destroyed on disconnect. CONNACK `sessionPresent=0`. |
| `false` | Persistent: subscriptions + offline queue survive disconnect. CONNACK `sessionPresent=1` on reconnect. |

### MQTT 5.0

| `cleanStart` | `sessionExpiryInterval` | Behaviour |
|--------------|------------------------|-----------|
| `true` | `0` | Clean + ephemeral (same as 3.1.1 `cleanSession=true`) |
| `true` | `> 0` | Discard old state, create NEW persistent session with TTL |
| `false` | any | Resume existing session (or create new if none). |

`sessionExpiryInterval` (seconds, 0–0xFFFFFFFF): lifetime after disconnect before session is deleted.
`0xFFFFFFFF` = session never expires.

---

## MqttSessionManager

Per-tenant session storage with LRU eviction:

```java
class MqttSessionManager {
    // Per-tenant sessions: insertion-ordered LinkedHashMap for LRU eviction
    ConcurrentHashMap<TenantId, LinkedHashMap<String, SessionEntry>> tenantSessions;

    record SessionEntry(
        MqttSession session,
        boolean cleanSession,
        boolean connected,          // true while channel active
        long expiryTimeMs           // -1 = no expiry; set on disconnect
    ) {}

    SessionResult createOrResume(TenantId tenantId, String clientId, boolean cleanSession) {
        if (cleanSession) {
            // Discard prior session, create fresh
            sessions.remove(clientId);
            sessions.put(clientId, new SessionEntry(new MqttSession(), true, true, -1));
            return new SessionResult(session, sessionPresent=false);
        } else {
            SessionEntry existing = sessions.get(clientId);
            if (existing != null) {
                existing.connected = true;
                return new SessionResult(existing.session, sessionPresent=true);  // resume
            }
            sessions.put(clientId, new SessionEntry(new MqttSession(), false, true, -1));
            return new SessionResult(session, sessionPresent=false);  // new persistent
        }
    }
}
```

---

## MqttSession (Per-Client State)

```java
class MqttSession {
    List<MqttSubscription> subscriptions;              // topic filter + QoS
    int lastPacketId;                                   // 1–65535, wraps
    ConcurrentLinkedQueue<byte[]> offlineQueue;        // max 1000 messages
    volatile int offlineQueueSize;                      // O(1) size tracking

    record MqttSubscription(String topicFilter, MqttQoS qos) {}
}
```

### What Survives Disconnect (Persistent Session)

| State | Survives? | Notes |
|-------|-----------|-------|
| Topic subscriptions (filter + QoS) | Yes | Restored on reconnect |
| Offline message queue | Yes | Up to 1000 messages; oldest dropped on overflow |
| In-flight QoS 2 packet IDs | Yes (planned) | Currently in-memory only — lost on restart |
| Last packet ID counter | Yes | Prevents ID reuse |

### What Is Destroyed (Clean Session)

All subscriptions, offline queue, in-flight states, and will message (on normal disconnect).

---

## Offline Message Queue

When a persistent session client disconnects, messages matching its subscriptions are buffered:

```java
void enqueueOffline(byte[] payload) {
    if (offlineQueueSize >= MAX_OFFLINE_QUEUE_SIZE) {
        offlineQueue.poll();             // drop oldest (QoS 0 semantics)
        offlineQueueSize--;
    }
    offlineQueue.offer(payload);
    offlineQueueSize++;
}

List<byte[]> drainOfflineQueue() {
    List<byte[]> messages = new ArrayList<>();
    byte[] msg;
    while ((msg = offlineQueue.poll()) != null) {
        messages.add(msg);
    }
    offlineQueueSize = 0;
    return messages;
}
```

**Bound:** `MAX_OFFLINE_QUEUE_SIZE = 1000`. Drop-oldest policy (best-effort for QoS 0/1).

On reconnect, `drainOfflineQueue()` delivers all buffered messages before new subscriptions fire.

---

## Will Message (Last Will and Testament)

### Stored on CONNECT

```java
record MqttWillData(String topic, byte[] payload, MqttQoS qos, boolean retain) {}

// In MqttConnectionHandler:
this.willData = connectData.willData();   // stored as handler field
```

### Published on Abnormal Disconnect

```java
@Override
public void channelInactive(ChannelHandlerContext ctx) {
    if (!disconnectReceived && willData != null) {
        willPublisher.publishWill(willData);   // fires LWT
        willData = null;                        // double-publish guard
    }
}
```

### NOT Published on Clean Disconnect

```java
void handleDisconnect(ChannelHandlerContext ctx) {
    this.disconnectReceived = true;   // flag set
    this.willData = null;              // explicitly cleared
    ctx.close();
}
```

### Published on Session Takeover

When a new client connects with the same `clientId`, the old connection is closed:

```java
Channel oldChannel = clientRegistry.put(tenantId + "/" + clientId, ctx.channel());
if (oldChannel != null && oldChannel.isActive()) {
    sendTakeoverDisconnect(oldChannel);   // MQTT 5.0: reason 0x8E "Session Taken Over"
    oldChannel.close();                    // triggers channelInactive → publishes will
}
```

### Will Authorization

Will topic is ACL-checked at CONNECT time (not at publish time). If the principal lacks WRITE
permission on the will topic, CONNECT is rejected with return code 5 (Not Authorized).
See [ACL_DESIGN.md](ACL_DESIGN.md) §MQTT Wire-Op Mapping.

---

## ClientId Uniqueness

Enforced per-tenant via `ConcurrentHashMap<String, Channel>`:

```java
String key = tenantId.id() + "/" + clientId;
Channel old = clientRegistry.put(key, newChannel);
if (old != null && old.isActive()) {
    // Session takeover: close old connection
    old.close();
}
```

Different tenants may have the same `clientId` — the registry key includes `tenantId`.

---

## Session Timeout and Cleanup

### Disconnect with Expiry (MQTT 5.0)

```java
void disconnectWithExpiry(TenantId tenantId, String clientId,
                          long disconnectTimeMs, long expiryIntervalMs) {
    entry.expiryTimeMs = disconnectTimeMs + expiryIntervalMs;
    entry.connected = false;
}
```

### Periodic Sweep

Background thread removes expired sessions:

```java
void removeExpiredSessions(long currentTimeMs) {
    for (var sessions : tenantSessions.values()) {
        synchronized (sessions) {
            sessions.entrySet().removeIf(e -> {
                SessionEntry entry = e.getValue();
                return !entry.connected
                    && entry.expiryTimeMs > 0
                    && currentTimeMs >= entry.expiryTimeMs;
            });
        }
    }
}
```

### LRU Eviction

When sessions per tenant exceed `maxSessionsPerTenant`, oldest (by insertion order) is evicted:

```java
void evictIfNeeded(LinkedHashMap<String, SessionEntry> sessions) {
    while (sessions.size() > maxSessionsPerTenant) {
        var it = sessions.entrySet().iterator();
        it.next();
        it.remove();   // removes oldest insertion
    }
}
```

---

## Recovery After Broker Restart

**Current (v8):** sessions are in-memory only — **all state lost on restart**. Persistent
session clients reconnect with `sessionPresent=0` and must re-subscribe.

**Planned PG Persistence (v9):**

```sql
CREATE TABLE mqtt_sessions (
    tenant_id UUID NOT NULL,
    client_id TEXT NOT NULL,
    subscriptions_json JSONB NOT NULL DEFAULT '[]',
    created_at TIMESTAMPTZ NOT NULL,
    disconnected_at TIMESTAMPTZ,
    expiry_interval_s INTEGER,
    PRIMARY KEY (tenant_id, client_id)
);

CREATE TABLE mqtt_inflight (
    tenant_id UUID NOT NULL,
    client_id TEXT NOT NULL,
    packet_id INTEGER NOT NULL,
    direction TEXT NOT NULL,       -- 'inbound' or 'outbound'
    qos INTEGER NOT NULL,
    topic TEXT NOT NULL,
    payload BYTEA,
    state TEXT NOT NULL,           -- 'PUBREC_SENT', 'PUBREL_RECEIVED'
    PRIMARY KEY (tenant_id, client_id, packet_id, direction)
);

CREATE TABLE mqtt_offline_messages (
    tenant_id UUID NOT NULL,
    client_id TEXT NOT NULL,
    sequence BIGSERIAL,
    topic TEXT NOT NULL,
    payload BYTEA NOT NULL,
    qos SMALLINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, client_id, sequence)
);
```

On restart: replay `mqtt_sessions` → rebuild `MqttSessionManager` → persistent clients
reconnect with `sessionPresent=1`.

---

## Configuration

```yaml
protocols:
  mqtt:
    max-sessions-per-tenant: 10000
    max-offline-queue-size: 1000
    session-expiry-sweep-interval-ms: 60000   # periodic cleanup
    will-delay-interval-max-ms: 0              # 0 = immediate will publish (5.0 delay not yet implemented)
```

---

*Last updated: 2026-03-25*
