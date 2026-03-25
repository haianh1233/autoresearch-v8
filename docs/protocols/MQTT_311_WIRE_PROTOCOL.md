# MQTT 3.1.1 Wire Protocol Specification

> **Related:** [../PROTOCOLS.md](../PROTOCOLS.md) §MQTT 3.1.1, [../MQTT_SESSIONS.md](../MQTT_SESSIONS.md)

---

## Fixed Header (All Packets)

```
Byte 1: [Packet Type : 4 bits] [Flags : 4 bits]
Bytes 2+: Remaining Length (Variable Byte Integer, 1-4 bytes)
```

### Variable Byte Integer (VBI) Encoding

```
do {
    encodedByte = X MOD 128
    X = X DIV 128
    if (X > 0) encodedByte = encodedByte OR 128
    output(encodedByte)
} while (X > 0)
```

Max value: 268,435,455 (0x0FFFFFFF) = 256 MB.

---

## Packet Types

| Type | Value | Direction | Flags | Description |
|------|-------|-----------|-------|-------------|
| CONNECT | 1 | C→S | 0000 | Client connection request |
| CONNACK | 2 | S→C | 0000 | Connection acknowledgement |
| PUBLISH | 3 | C↔S | DUP/QoS/RETAIN | Publish message |
| PUBACK | 4 | C↔S | 0000 | QoS 1 acknowledgement |
| PUBREC | 5 | C↔S | 0000 | QoS 2 received (phase 1) |
| PUBREL | 6 | C↔S | 0010 | QoS 2 release (phase 2) |
| PUBCOMP | 7 | C↔S | 0000 | QoS 2 complete (phase 3) |
| SUBSCRIBE | 8 | C→S | 0010 | Subscribe to topics |
| SUBACK | 9 | S→C | 0000 | Subscribe acknowledgement |
| UNSUBSCRIBE | 10 | C→S | 0010 | Unsubscribe from topics |
| UNSUBACK | 11 | S→C | 0000 | Unsubscribe acknowledgement |
| PINGREQ | 12 | C→S | 0000 | Keepalive ping |
| PINGRESP | 13 | S→C | 0000 | Keepalive pong |
| DISCONNECT | 14 | C→S | 0000 | Client disconnect |

---

## PUBLISH Flags (Byte 1, bits 3-0)

```
Bit 3: DUP     (0=first delivery, 1=redelivery)
Bit 2-1: QoS   (00=0, 01=1, 10=2, 11=reserved)
Bit 0: RETAIN  (0=normal, 1=retained message)
```

---

## CONNECT Packet

```
Variable Header:
  Protocol Name   : uint16 length + "MQTT" (4 bytes)
  Protocol Level  : uint8 = 0x04 (MQTT 3.1.1)
  Connect Flags   : uint8
    Bit 7: Username Flag
    Bit 6: Password Flag
    Bit 5: Will Retain
    Bit 4-3: Will QoS (0-2)
    Bit 2: Will Flag
    Bit 1: Clean Session
    Bit 0: Reserved (must be 0)
  Keep Alive      : uint16 (seconds)

Payload (in order):
  Client Identifier : UTF-8 string (uint16 length + bytes)
  Will Topic        : UTF-8 string (if Will Flag set)
  Will Message      : bytes (uint16 length + bytes, if Will Flag set)
  Username          : UTF-8 string (if Username Flag set)
  Password          : bytes (uint16 length + bytes, if Password Flag set)
```

---

## CONNACK Packet

```
Variable Header:
  Session Present : uint8 (bit 0: 0=new session, 1=existing session resumed)
  Return Code     : uint8
```

### CONNACK Return Codes

| Code | Name | Description |
|------|------|-------------|
| 0 | Connection Accepted | Success |
| 1 | Unacceptable Protocol Version | Server does not support this MQTT level |
| 2 | Identifier Rejected | Client ID not allowed |
| 3 | Server Unavailable | Network OK but MQTT service unavailable |
| 4 | Bad User Name or Password | Invalid credentials |
| 5 | Not Authorized | Client not authorized |

---

## QoS Message Flows

### QoS 0 (At Most Once)

```
C→S: PUBLISH (QoS=0)     [fire and forget, no ack]
```

### QoS 1 (At Least Once)

```
C→S: PUBLISH (QoS=1, packetId=N)
S→C: PUBACK (packetId=N)
```

### QoS 2 (Exactly Once)

```
C→S: PUBLISH (QoS=2, packetId=N)
S→C: PUBREC (packetId=N)         [phase 1: received]
C→S: PUBREL (packetId=N)         [phase 2: release]
S→C: PUBCOMP (packetId=N)        [phase 3: complete]
```

---

## SUBSCRIBE / SUBACK

```
SUBSCRIBE Variable Header:
  Packet Identifier : uint16

SUBSCRIBE Payload:
  Topic Filter 1 : UTF-8 string
  Requested QoS 1: uint8 (0, 1, or 2)
  Topic Filter 2 : UTF-8 string
  Requested QoS 2: uint8
  ...

SUBACK Payload:
  Return Code 1  : uint8 (0x00=QoS 0, 0x01=QoS 1, 0x02=QoS 2, 0x80=Failure)
  Return Code 2  : uint8
  ...
```

---

## Topic Wildcards

| Wildcard | Matches | Example |
|----------|---------|---------|
| `+` | Exactly one level | `sensors/+/temp` matches `sensors/room1/temp` |
| `#` | Zero or more levels (must be last) | `sensors/#` matches `sensors/room1/temp` |

Topics starting with `$` are reserved (e.g., `$SYS/`) and NOT matched by `#` at root level.

---

*Last updated: 2026-03-25*
