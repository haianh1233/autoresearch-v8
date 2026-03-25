# AMQP 1.0 Wire Protocol Specification

> **Related:** [../PROTOCOLS.md](../PROTOCOLS.md) §AMQP 1.0

---

## Frame Format

```
Size    : uint32  (total frame size including this field)
Doff    : uint8   (data offset in 4-byte words; minimum 2 = 8 bytes header)
Type    : uint8   (0x00=AMQP, 0x01=SASL)
Channel : uint16  (type-specific; session channel for AMQP, 0 for SASL)
Extended: [Doff*4 - 8 bytes]  (usually 0; extended header)
Body    : [remaining bytes]   (performative + payload)
```

---

## Frame Types

| Type | Value | Description |
|------|-------|-------------|
| AMQP | 0x00 | AMQP protocol frames (Open, Begin, Attach, Transfer, etc.) |
| SASL | 0x01 | SASL authentication frames (before AMQP Open) |

---

## SASL Performatives (type=0x01)

| Descriptor | Code | Name | Direction | Purpose |
|-----------|------|------|-----------|---------|
| `0x00000000:0x00000040` | 0x40 | sasl-mechanisms | S→C | Advertise supported SASL mechanisms |
| `0x00000000:0x00000041` | 0x41 | sasl-init | C→S | Client selects mechanism + initial response |
| `0x00000000:0x00000042` | 0x42 | sasl-challenge | S→C | Server challenge (multi-step, e.g., SCRAM) |
| `0x00000000:0x00000043` | 0x43 | sasl-response | C→S | Client response to challenge |
| `0x00000000:0x00000044` | 0x44 | sasl-outcome | S→C | Auth result (code: 0=ok, 1=auth, 2=sys, 3=sys-perm, 4=sys-temp) |

### SASL Outcome Codes

| Code | Name | Description |
|------|------|-------------|
| 0 | ok | Authentication succeeded |
| 1 | auth | Authentication failed (bad credentials) |
| 2 | sys | System error (temporary) |
| 3 | sys-perm | System error (permanent) |
| 4 | sys-temp | System error (temporary, retry later) |

---

## AMQP Performatives (type=0x00)

| Descriptor | Code | Name | Direction | Purpose |
|-----------|------|------|-----------|---------|
| `0x00000000:0x00000010` | 0x10 | open | C↔S | Connection negotiation (max-frame-size, channel-max, idle-timeout) |
| `0x00000000:0x00000011` | 0x11 | begin | C↔S | Open a session (next-outgoing-id, incoming-window, outgoing-window) |
| `0x00000000:0x00000012` | 0x12 | attach | C↔S | Attach a link (name, role, source, target, snd-settle-mode, rcv-settle-mode) |
| `0x00000000:0x00000013` | 0x13 | flow | C↔S | Flow control (link-credit, delivery-count, drain) |
| `0x00000000:0x00000014` | 0x14 | transfer | C→S | Deliver message (delivery-id, delivery-tag, message-format, settled, more) |
| `0x00000000:0x00000015` | 0x15 | disposition | C↔S | Settle deliveries (role, first, last, settled, state) |
| `0x00000000:0x00000016` | 0x16 | detach | C↔S | Detach link (handle, closed, error) |
| `0x00000000:0x00000017` | 0x17 | end | C↔S | End session (error) |
| `0x00000000:0x00000018` | 0x18 | close | C↔S | Close connection (error) |

---

## AMQP Type System

### Primitive Types

| Type | Code | Size |
|------|------|------|
| null | 0x40 | 0 |
| boolean (true) | 0x41 | 0 |
| boolean (false) | 0x42 | 0 |
| boolean | 0x56 | 1 |
| ubyte | 0x50 | 1 |
| ushort | 0x60 | 2 |
| uint | 0x70 | 4 |
| uint (small) | 0x52 | 1 |
| uint (zero) | 0x43 | 0 |
| ulong | 0x80 | 8 |
| ulong (small) | 0x53 | 1 |
| ulong (zero) | 0x44 | 0 |
| byte | 0x51 | 1 |
| short | 0x61 | 2 |
| int | 0x71 | 4 |
| int (small) | 0x54 | 1 |
| long | 0x81 | 8 |
| long (small) | 0x55 | 1 |
| float | 0x72 | 4 (IEEE 754) |
| double | 0x82 | 8 (IEEE 754) |
| char | 0x73 | 4 (UTF-32BE) |
| timestamp | 0x83 | 8 (ms since epoch) |
| uuid | 0x98 | 16 |

### Variable-Width Types

| Type | Code | Size Prefix |
|------|------|-------------|
| binary (vbin8) | 0xa0 | 1 byte |
| binary (vbin32) | 0xb0 | 4 bytes |
| string (str8-utf8) | 0xa1 | 1 byte |
| string (str32-utf8) | 0xb1 | 4 bytes |
| symbol (sym8) | 0xa3 | 1 byte |
| symbol (sym32) | 0xb3 | 4 bytes |

### Compound Types

| Type | Code | Description |
|------|------|-------------|
| list (list0) | 0x45 | empty list |
| list (list8) | 0xc0 | size:1 + count:1 + elements |
| list (list32) | 0xd0 | size:4 + count:4 + elements |
| map (map8) | 0xc1 | size:1 + count:1 + key-value pairs |
| map (map32) | 0xd1 | size:4 + count:4 + key-value pairs |
| array (array8) | 0xe0 | size:1 + count:1 + constructor + elements |
| array (array32) | 0xf0 | size:4 + count:4 + constructor + elements |

---

## Message Format

```
┌─ header              (delivery-count, durable, priority, ttl, first-acquirer)
├─ delivery-annotations (per-hop metadata, map)
├─ message-annotations  (per-message metadata, map)
├─ properties          (message-id, user-id, to, subject, reply-to, correlation-id,
│                        content-type, content-encoding, absolute-expiry-time,
│                        creation-time, group-id, group-sequence, reply-to-group-id)
├─ application-properties (user key-value map → maps to Ivy headers)
├─ body                (one of: amqp-value | amqp-sequence | data section)
└─ footer              (post-body annotations, map)
```

---

## Disposition Outcomes

| Outcome | Descriptor | Description |
|---------|-----------|-------------|
| accepted | `0x00000000:0x00000024` | Message processed successfully |
| rejected | `0x00000000:0x00000025` | Message rejected (error condition) |
| released | `0x00000000:0x00000026` | Message released (requeue) |
| modified | `0x00000000:0x00000027` | Modified (delivery-failed, undeliverable-here flags) |

---

## Settlement Modes

| Sender Mode | Receiver Mode | Semantics |
|------------|--------------|-----------|
| settled (0) | — | At-most-once (fire and forget) |
| unsettled (1) | first (0) | At-least-once (receiver settles on accept) |
| unsettled (1) | second (1) | Exactly-once (two-phase settlement) |

---

## Error Conditions

| Condition | Description |
|-----------|-------------|
| `amqp:internal-error` | Internal server error |
| `amqp:not-found` | Resource not found |
| `amqp:unauthorized-access` | Authorization denied |
| `amqp:decode-error` | Frame decode failure |
| `amqp:resource-limit-exceeded` | Resource limit exceeded |
| `amqp:not-allowed` | Operation not allowed |
| `amqp:invalid-field` | Invalid field value |
| `amqp:not-implemented` | Feature not implemented |
| `amqp:resource-locked` | Resource locked by another session |
| `amqp:precondition-failed` | Precondition check failed |
| `amqp:resource-deleted` | Resource deleted |
| `amqp:illegal-state` | Illegal state for operation |
| `amqp:frame-size-too-small` | Frame size below minimum |

---

*Last updated: 2026-03-25*
