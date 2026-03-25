# MQTT 5.0 Wire Protocol Specification

> **Related:** [../PROTOCOLS.md](../PROTOCOLS.md) §MQTT 5.0, [../MQTT_SESSIONS.md](../MQTT_SESSIONS.md),
> [MQTT_311_WIRE_PROTOCOL.md](MQTT_311_WIRE_PROTOCOL.md)

---

## Differences from MQTT 3.1.1

- Protocol Level byte: **0x05** (vs 0x04 for 3.1.1)
- **Properties section** added to all packet types
- **Reason codes** replace simple return codes (richer error reporting)
- **AUTH packet** (type 15) for enhanced authentication
- **Shared subscriptions** via `$share/group/filter` prefix
- Server-initiated **DISCONNECT** (3.1.1 has no server DISCONNECT)

---

## New Packet Type

| Type | Value | Direction | Description |
|------|-------|-----------|-------------|
| AUTH | 15 | C↔S | Enhanced authentication exchange |

---

## Property IDs

| ID (hex) | Name | Type | Used In |
|----------|------|------|---------|
| 0x01 | Payload Format Indicator | byte | PUBLISH, Will |
| 0x02 | Message Expiry Interval | uint32 (seconds) | PUBLISH, Will |
| 0x03 | Content Type | UTF-8 string | PUBLISH, Will |
| 0x08 | Response Topic | UTF-8 string | PUBLISH, Will |
| 0x09 | Correlation Data | binary | PUBLISH, Will |
| 0x0B | Subscription Identifier | varint | SUBSCRIBE, PUBLISH |
| 0x11 | Session Expiry Interval | uint32 (seconds) | CONNECT, CONNACK, DISCONNECT |
| 0x12 | Assigned Client Identifier | UTF-8 string | CONNACK |
| 0x13 | Server Keep Alive | uint16 (seconds) | CONNACK |
| 0x15 | Authentication Method | UTF-8 string | CONNECT, CONNACK, AUTH |
| 0x16 | Authentication Data | binary | CONNECT, CONNACK, AUTH |
| 0x17 | Request Problem Information | byte | CONNECT |
| 0x19 | Request Response Information | byte | CONNECT |
| 0x1A | Response Information | UTF-8 string | CONNACK |
| 0x1C | Server Reference | UTF-8 string | CONNACK, DISCONNECT |
| 0x1F | Reason String | UTF-8 string | All ACK/DISCONNECT |
| 0x21 | Receive Maximum | uint16 | CONNECT, CONNACK |
| 0x22 | Topic Alias Maximum | uint16 | CONNECT, CONNACK |
| 0x23 | Topic Alias | uint16 | PUBLISH |
| 0x24 | Maximum QoS | byte | CONNACK |
| 0x25 | Retain Available | byte | CONNACK |
| 0x26 | User Property | UTF-8 pair | All packets |
| 0x27 | Maximum Packet Size | uint32 | CONNECT, CONNACK |
| 0x28 | Wildcard Subscription Available | byte | CONNACK |
| 0x29 | Subscription Identifiers Available | byte | CONNACK |
| 0x2A | Shared Subscription Available | byte | CONNACK |

---

## Reason Codes

### CONNACK Reason Codes

| Code | Name |
|------|------|
| 0x00 | Success |
| 0x80 | Unspecified Error |
| 0x81 | Malformed Packet |
| 0x82 | Protocol Error |
| 0x83 | Implementation Specific Error |
| 0x84 | Unsupported Protocol Version |
| 0x85 | Client Identifier Not Valid |
| 0x86 | Bad User Name or Password |
| 0x87 | Not Authorized |
| 0x88 | Server Unavailable |
| 0x89 | Server Busy |
| 0x8A | Banned |
| 0x8C | Bad Authentication Method |
| 0x90 | Topic Name Invalid |
| 0x95 | Packet Too Large |
| 0x97 | Quota Exceeded |
| 0x99 | Payload Format Invalid |
| 0x9A | Retain Not Supported |
| 0x9B | QoS Not Supported |
| 0x9C | Use Another Server |
| 0x9D | Server Moved |
| 0x9F | Connection Rate Exceeded |

### DISCONNECT Reason Codes

| Code | Name |
|------|------|
| 0x00 | Normal Disconnection |
| 0x04 | Disconnect with Will Message |
| 0x80 | Unspecified Error |
| 0x81 | Malformed Packet |
| 0x82 | Protocol Error |
| 0x87 | Not Authorized |
| 0x89 | Server Busy |
| 0x8B | Server Shutting Down |
| 0x8D | Keep Alive Timeout |
| 0x8E | Session Taken Over |
| 0x93 | Receive Maximum Exceeded |
| 0x94 | Topic Alias Invalid |
| 0x95 | Packet Too Large |
| 0x97 | Quota Exceeded |
| 0x98 | Administrative Action |
| 0x9C | Use Another Server |
| 0x9D | Server Moved |
| 0xA1 | Subscription Identifiers Not Supported |
| 0xA2 | Wildcard Subscriptions Not Supported |

### AUTH Reason Codes

| Code | Name |
|------|------|
| 0x00 | Success |
| 0x18 | Continue Authentication |
| 0x19 | Re-authenticate |

### SUBACK / PUBACK / PUBREC / PUBREL / PUBCOMP

| Code | Name |
|------|------|
| 0x00 | Success / Granted QoS 0 |
| 0x01 | Granted QoS 1 |
| 0x02 | Granted QoS 2 |
| 0x80 | Unspecified Error |
| 0x83 | Implementation Specific Error |
| 0x87 | Not Authorized |
| 0x8F | Topic Filter Invalid |
| 0x91 | Packet Identifier In Use |
| 0x97 | Quota Exceeded |
| 0xA1 | Shared Subscriptions Not Supported |

---

## Enhanced AUTH Packet

```
Fixed Header: type=15 (0xF0), flags=0000
Variable Header:
  Reason Code     : uint8 (0x00=Success, 0x18=Continue, 0x19=Re-authenticate)
  Properties:
    0x15: Authentication Method (UTF-8 string, required)
    0x16: Authentication Data (binary, optional)
    0x1F: Reason String (UTF-8 string, optional)
    0x26: User Property (UTF-8 pair, repeatable)
```

### SCRAM-SHA-256 via AUTH

```
C→S: CONNECT(authMethod="SCRAM-SHA-256", authData=<client-first>)
S→C: AUTH(reasonCode=0x18, authData=<server-first: salt, iterations, nonce>)
C→S: AUTH(reasonCode=0x18, authData=<client-final: proof>)
S→C: CONNACK(reasonCode=0x00, authData=<server-final: signature>)
```

### Broker-Initiated Re-Auth

```
S→C: AUTH(reasonCode=0x19, authMethod="SCRAM-SHA-256", authData=<challenge>)
C→S: AUTH(reasonCode=0x18, authData=<response>)
S→C: AUTH(reasonCode=0x00, authData=<server-final>)
```

---

## Shared Subscriptions

```
$share/<ShareName>/<TopicFilter>
```

- `ShareName`: consumer group name (UTF-8, no wildcards, no `/`)
- `TopicFilter`: standard topic filter (may include `+` and `#`)
- Messages distributed round-robin across group members

---

## Topic Aliases

Property `0x23` (uint16): maps a topic name to a numeric alias for subsequent PUBLISH packets.
Reduces bandwidth for repeated publishes to the same topic.

```
PUBLISH(topicName="sensors/room1/temp", topicAlias=1)   ← establishes alias
PUBLISH(topicName="", topicAlias=1)                       ← uses alias (saves bytes)
```

---

*Last updated: 2026-03-25*
