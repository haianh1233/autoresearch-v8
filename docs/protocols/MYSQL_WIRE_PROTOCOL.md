# MySQL Wire Protocol Specification

> **Related:** [../PROTOCOLS.md](../PROTOCOLS.md) §MySQL Wire Protocol

---

## Packet Format

```
Length    : uint24 (3 bytes, little-endian)
SequenceId: uint8  (1 byte, wraps at 255)
Payload   : [Length bytes]
```

Max payload: 16,777,215 bytes (2^24 - 1). Larger payloads split across multiple packets.

---

## Handshake

### Server → Client: HandshakeV10

```
ProtocolVersion   : uint8 = 10
ServerVersion     : null-terminated string (e.g., "9.0.0-ivy")
ConnectionId      : uint32
AuthPluginData1   : 8 bytes (first part of scramble)
Filler            : uint8 = 0x00
CapabilityFlags1  : uint16 (lower 2 bytes)
CharacterSet      : uint8 (e.g., 0x21 = utf8_general_ci)
StatusFlags       : uint16
CapabilityFlags2  : uint16 (upper 2 bytes)
AuthPluginDataLen : uint8 (total length of auth data)
Reserved          : 10 bytes (all 0x00)
AuthPluginData2   : [max(13, authPluginDataLen - 8)] bytes
AuthPluginName    : null-terminated string (e.g., "mysql_native_password")
```

### Client → Server: HandshakeResponse41

```
CapabilityFlags   : uint32
MaxPacketSize     : uint32
CharacterSet      : uint8
Reserved          : 23 bytes (all 0x00)
Username          : null-terminated string
AuthResponse      : length-encoded string (scrambled password)
Database          : null-terminated string (if CLIENT_CONNECT_WITH_DB)
AuthPluginName    : null-terminated string (if CLIENT_PLUGIN_AUTH)
```

---

## COM_* Command Bytes

| Byte | Command | Description |
|------|---------|-------------|
| 0x00 | COM_SLEEP | Internal (not used by clients) |
| 0x01 | COM_QUIT | Close connection |
| 0x02 | COM_INIT_DB | Change default schema |
| 0x03 | COM_QUERY | Text protocol query |
| 0x04 | COM_FIELD_LIST | List columns in table |
| 0x05 | COM_CREATE_DB | Create database (deprecated) |
| 0x06 | COM_DROP_DB | Drop database (deprecated) |
| 0x07 | COM_REFRESH | Flush tables/logs |
| 0x08 | COM_SHUTDOWN | Shutdown server (deprecated) |
| 0x09 | COM_STATISTICS | Server statistics string |
| 0x0A | COM_PROCESS_INFO | Thread list (deprecated) |
| 0x0B | COM_CONNECT | Internal |
| 0x0C | COM_PROCESS_KILL | Kill connection |
| 0x0D | COM_DEBUG | Dump debug info |
| 0x0E | COM_PING | Keepalive ping |
| 0x0F | COM_TIME | Internal |
| 0x10 | COM_DELAYED_INSERT | Internal (deprecated) |
| 0x11 | COM_CHANGE_USER | Change user on connection |
| 0x16 | COM_STMT_PREPARE | Prepare statement (binary protocol) |
| 0x17 | COM_STMT_EXECUTE | Execute prepared statement |
| 0x18 | COM_STMT_SEND_LONG_DATA | Send BLOB data |
| 0x19 | COM_STMT_CLOSE | Close prepared statement |
| 0x1A | COM_STMT_RESET | Reset prepared statement |
| 0x1B | COM_SET_OPTION | Set connection options |
| 0x1C | COM_STMT_FETCH | Fetch rows from cursor |
| 0x1D | COM_DAEMON | Internal |
| 0x1E | COM_BINLOG_DUMP | Start binlog replication |
| 0x1F | COM_TABLE_DUMP | Dump table |

---

## Response Packets

### OK_Packet (header byte 0x00)

```
Header          : uint8 = 0x00
AffectedRows    : length-encoded integer
LastInsertId    : length-encoded integer
StatusFlags     : uint16 (if CLIENT_PROTOCOL_41)
Warnings        : uint16 (if CLIENT_PROTOCOL_41)
Info            : string (remaining bytes)
```

### ERR_Packet (header byte 0xFF)

```
Header          : uint8 = 0xFF
ErrorCode       : uint16
SqlStateMarker  : uint8 = '#' (if CLIENT_PROTOCOL_41)
SqlState        : 5 bytes (SQLSTATE code)
ErrorMessage    : string (remaining bytes)
```

### EOF_Packet (header byte 0xFE)

```
Header          : uint8 = 0xFE
Warnings        : uint16
StatusFlags     : uint16
```

**Note:** EOF_Packet deprecated in MySQL 5.7.5+ when `CLIENT_DEPRECATE_EOF` capability set.

---

## Result Set Format (COM_QUERY response)

```
S→C: ColumnCount      (length-encoded integer)
S→C: ColumnDefinition (per column, repeated)
  Catalog   : length-encoded string ("def")
  Schema    : length-encoded string
  Table     : length-encoded string
  OrgTable  : length-encoded string
  Name      : length-encoded string
  OrgName   : length-encoded string
  FixedLen  : uint8 = 0x0C
  CharSet   : uint16
  ColumnLen : uint32
  ColumnType: uint8 (see Column Types)
  Flags     : uint16
  Decimals  : uint8
  Filler    : uint16 = 0x0000
S→C: EOF_Packet (or OK if CLIENT_DEPRECATE_EOF)
S→C: Row Data (per row, repeated)
  Values: [length-encoded string per column, 0xFB = NULL]
S→C: EOF_Packet (or OK)
```

---

## Column Types

| Value | Type | Description |
|-------|------|-------------|
| 0x00 | MYSQL_TYPE_DECIMAL | Exact decimal |
| 0x01 | MYSQL_TYPE_TINY | int8 |
| 0x02 | MYSQL_TYPE_SHORT | int16 |
| 0x03 | MYSQL_TYPE_LONG | int32 |
| 0x04 | MYSQL_TYPE_FLOAT | float32 |
| 0x05 | MYSQL_TYPE_DOUBLE | float64 |
| 0x06 | MYSQL_TYPE_NULL | NULL |
| 0x07 | MYSQL_TYPE_TIMESTAMP | Timestamp |
| 0x08 | MYSQL_TYPE_LONGLONG | int64 |
| 0x09 | MYSQL_TYPE_INT24 | int24 |
| 0x0A | MYSQL_TYPE_DATE | Date |
| 0x0B | MYSQL_TYPE_TIME | Time |
| 0x0C | MYSQL_TYPE_DATETIME | DateTime |
| 0x0D | MYSQL_TYPE_YEAR | Year |
| 0x0F | MYSQL_TYPE_VARCHAR | Variable string |
| 0x10 | MYSQL_TYPE_BIT | Bit field |
| 0xF5 | MYSQL_TYPE_JSON | JSON document |
| 0xF6 | MYSQL_TYPE_NEWDECIMAL | Precise decimal |
| 0xF7 | MYSQL_TYPE_ENUM | Enumeration |
| 0xF8 | MYSQL_TYPE_SET | Set |
| 0xF9 | MYSQL_TYPE_TINY_BLOB | Tiny BLOB |
| 0xFA | MYSQL_TYPE_MEDIUM_BLOB | Medium BLOB |
| 0xFB | MYSQL_TYPE_LONG_BLOB | Long BLOB |
| 0xFC | MYSQL_TYPE_BLOB | BLOB |
| 0xFD | MYSQL_TYPE_VAR_STRING | Variable string |
| 0xFE | MYSQL_TYPE_STRING | Fixed string |
| 0xFF | MYSQL_TYPE_GEOMETRY | Geometry |

---

## Capability Flags (Key Subset)

| Flag | Bit | Description |
|------|-----|-------------|
| CLIENT_LONG_PASSWORD | 0 | Longer passwords |
| CLIENT_PROTOCOL_41 | 9 | New 4.1 protocol |
| CLIENT_SECURE_CONNECTION | 13 | Secure password hash |
| CLIENT_PLUGIN_AUTH | 19 | Authentication plugin |
| CLIENT_CONNECT_ATTRS | 20 | Connection attributes |
| CLIENT_PLUGIN_AUTH_LENENC_CLIENT_DATA | 21 | Long auth data |
| CLIENT_DEPRECATE_EOF | 24 | EOF replaced by OK |
| CLIENT_SSL | 11 | SSL support |
| CLIENT_CONNECT_WITH_DB | 3 | Can set database in handshake |

---

## Auth Plugins

| Plugin | Algorithm |
|--------|-----------|
| `mysql_native_password` | SHA1(password) XOR SHA1(scramble + SHA1(SHA1(password))) |
| `caching_sha2_password` | SHA256-based (MySQL 8.0+ default) |
| `sha256_password` | RSA + SHA256 |

---

*Last updated: 2026-03-25*
