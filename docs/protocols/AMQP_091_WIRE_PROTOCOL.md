# AMQP 0-9-1 Wire Protocol Specification

> **Related:** [../PROTOCOLS.md](../PROTOCOLS.md) §AMQP 0-9-1, [../AMQP_EXCHANGES.md](../AMQP_EXCHANGES.md)

---

## Frame Format

```
Type    : uint8   (1=METHOD, 2=HEADER, 3=BODY, 8=HEARTBEAT)
Channel : uint16  (0 for connection-level, 1+ for channel-level)
Size    : uint32  (payload size, excluding type+channel+size+frame-end)
Payload : [Size bytes]
FrameEnd: uint8   (0xCE — frame boundary marker)
```

Total overhead: 8 bytes per frame (1+2+4+payload+1).

---

## Frame Types

| Type | Value | Description |
|------|-------|-------------|
| METHOD | 1 | RPC method frame (class-id + method-id + arguments) |
| HEADER | 2 | Content header (class-id + weight + body-size + properties) |
| BODY | 3 | Content body (raw message bytes, max frame-max-size) |
| HEARTBEAT | 8 | Keepalive (channel=0, size=0, empty payload) |

---

## Class IDs and Method IDs

### Connection (class-id=10)

| Method | ID | Direction | Key Fields |
|--------|-----|-----------|------------|
| Start | 10 | S→C | version-major, version-minor, mechanisms, locales, server-properties |
| StartOk | 11 | C→S | mechanism, response (SASL), locale, client-properties |
| Tune | 30 | S→C | channel-max, frame-max, heartbeat |
| TuneOk | 31 | C→S | channel-max, frame-max, heartbeat |
| Open | 40 | C→S | virtual-host |
| OpenOk | 41 | S→C | — |
| Close | 50 | C↔S | reply-code, reply-text, class-id, method-id |
| CloseOk | 51 | C↔S | — |

### Channel (class-id=20)

| Method | ID | Direction | Key Fields |
|--------|-----|-----------|------------|
| Open | 10 | C→S | — |
| OpenOk | 11 | S→C | — |
| Close | 40 | C↔S | reply-code, reply-text, class-id, method-id |
| CloseOk | 41 | C↔S | — |
| Flow | 20 | C↔S | active (bool) |
| FlowOk | 21 | C↔S | active (bool) |

### Exchange (class-id=40)

| Method | ID | Direction | Key Fields |
|--------|-----|-----------|------------|
| Declare | 10 | C→S | exchange, type, passive, durable, auto-delete, internal, arguments |
| DeclareOk | 11 | S→C | — |
| Delete | 20 | C→S | exchange, if-unused |
| DeleteOk | 21 | S→C | — |
| Bind | 30 | C→S | destination, source, routing-key, arguments |
| BindOk | 31 | S→C | — |
| Unbind | 40 | C→S | destination, source, routing-key, arguments |
| UnbindOk | 51 | S→C | — |

### Queue (class-id=50)

| Method | ID | Direction | Key Fields |
|--------|-----|-----------|------------|
| Declare | 10 | C→S | queue, passive, durable, exclusive, auto-delete, arguments |
| DeclareOk | 11 | S→C | queue, message-count, consumer-count |
| Bind | 20 | C→S | queue, exchange, routing-key, arguments |
| BindOk | 21 | S→C | — |
| Unbind | 50 | C→S | queue, exchange, routing-key, arguments |
| UnbindOk | 51 | S→C | — |
| Purge | 30 | C→S | queue |
| PurgeOk | 31 | S→C | message-count |
| Delete | 40 | C→S | queue, if-unused, if-empty |
| DeleteOk | 41 | S→C | message-count |

### Basic (class-id=60)

| Method | ID | Direction | Key Fields |
|--------|-----|-----------|------------|
| Qos | 10 | C→S | prefetch-size, prefetch-count, global |
| QosOk | 11 | S→C | — |
| Consume | 20 | C→S | queue, consumer-tag, no-local, no-ack, exclusive, arguments |
| ConsumeOk | 21 | S→C | consumer-tag |
| Cancel | 30 | C→S | consumer-tag, no-wait |
| CancelOk | 31 | S→C | consumer-tag |
| Publish | 40 | C→S | exchange, routing-key, mandatory, immediate |
| Return | 50 | S→C | reply-code, reply-text, exchange, routing-key |
| Deliver | 60 | S→C | consumer-tag, delivery-tag, redelivered, exchange, routing-key |
| Get | 70 | C→S | queue, no-ack |
| GetOk | 71 | S→C | delivery-tag, redelivered, exchange, routing-key, message-count |
| GetEmpty | 72 | S→C | — |
| Ack | 80 | C→S | delivery-tag, multiple |
| Nack | 120 | C→S | delivery-tag, multiple, requeue |
| Reject | 90 | C→S | delivery-tag, requeue |
| Recover | 110 | C→S | requeue |
| RecoverOk | 111 | S→C | — |

### Confirm (class-id=85)

| Method | ID | Direction | Key Fields |
|--------|-----|-----------|------------|
| Select | 10 | C→S | — |
| SelectOk | 11 | S→C | — |

### Tx (class-id=90)

| Method | ID | Direction | Key Fields |
|--------|-----|-----------|------------|
| Select | 10 | C→S | — |
| SelectOk | 11 | S→C | — |
| Commit | 20 | C→S | — |
| CommitOk | 21 | S→C | — |
| Rollback | 30 | C→S | — |
| RollbackOk | 31 | S→C | — |

---

## Data Types

| Type | Encoding |
|------|----------|
| octet | uint8 (1 byte) |
| short | uint16 (2 bytes, big-endian) |
| long | uint32 (4 bytes, big-endian) |
| longlong | uint64 (8 bytes, big-endian) |
| shortstr | uint8 length + UTF-8 (max 255 bytes) |
| longstr | uint32 length + raw bytes |
| field-table | uint32 length + key-value pairs (shortstr key + type-tag + value) |
| timestamp | uint64 (seconds since epoch) |

---

## Content Header (HEADER frame, type=2)

```
ClassId      : uint16  (60 for Basic)
Weight       : uint16  (always 0)
BodySize     : uint64  (total body size across BODY frames)
PropertyFlags: uint16  (bit flags for which properties are present)
Properties   : [variable, depending on flags]
```

### Basic Properties (class-id=60)

| Bit | Property | Type |
|-----|----------|------|
| 15 | content-type | shortstr |
| 14 | content-encoding | shortstr |
| 13 | headers | field-table |
| 12 | delivery-mode | octet (1=non-persistent, 2=persistent) |
| 11 | priority | octet (0-9) |
| 10 | correlation-id | shortstr |
| 9 | reply-to | shortstr |
| 8 | expiration | shortstr (TTL in milliseconds as string) |
| 7 | message-id | shortstr |
| 6 | timestamp | timestamp |
| 5 | type | shortstr |
| 4 | user-id | shortstr |
| 3 | app-id | shortstr |

---

## Reply Codes

| Code | Name | Description |
|------|------|-------------|
| 200 | REPLY_SUCCESS | Success |
| 311 | CONTENT_TOO_LARGE | Content too large for broker |
| 312 | NO_ROUTE | No route for mandatory message |
| 313 | NO_CONSUMERS | No consumers for immediate message |
| 320 | CONNECTION_FORCED | Connection forced closed by operator |
| 402 | INVALID_PATH | Invalid virtual host |
| 403 | ACCESS_REFUSED | Access refused (ACL deny) |
| 404 | NOT_FOUND | Queue/exchange not found |
| 405 | RESOURCE_LOCKED | Resource locked by another connection |
| 406 | PRECONDITION_FAILED | Declare with different parameters |
| 501 | FRAME_ERROR | Frame decoding error |
| 502 | SYNTAX_ERROR | Malformed frame |
| 503 | COMMAND_INVALID | Command invalid in current state |
| 504 | CHANNEL_ERROR | Channel error (unexpected class/method) |
| 505 | UNEXPECTED_FRAME | Unexpected frame type |
| 506 | RESOURCE_ERROR | Out of resources |
| 530 | NOT_ALLOWED | Operation not allowed (e.g., delete default exchange) |
| 540 | NOT_IMPLEMENTED | Method not implemented |
| 541 | INTERNAL_ERROR | Internal server error |

---

## Connection Handshake

```
S→C: 'AMQP' 0x00 0x00 0x09 0x01    (protocol header, 8 bytes)
S→C: Connection.Start(mechanisms="PLAIN AMQPLAIN EXTERNAL")
C→S: Connection.StartOk(mechanism="PLAIN", response="\0user\0pass")
S→C: Connection.Tune(channel-max=2047, frame-max=131072, heartbeat=60)
C→S: Connection.TuneOk(channel-max=2047, frame-max=131072, heartbeat=60)
C→S: Connection.Open(virtual-host="/")
S→C: Connection.OpenOk()
```

---

*Last updated: 2026-03-25*
