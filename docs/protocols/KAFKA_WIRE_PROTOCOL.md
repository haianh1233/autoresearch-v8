# Kafka Wire Protocol Specification

> **Related:** [../PROTOCOLS.md](../PROTOCOLS.md) §Kafka, [../IDEMPOTENT_PRODUCER.md](../IDEMPOTENT_PRODUCER.md),
> [../TRANSACTIONS.md](../TRANSACTIONS.md), [../CONSUMER_GROUPS.md](../CONSUMER_GROUPS.md)

---

## Wire Frame Format

```
Request/Response => Size (int32, big-endian) + Payload
  Size: 4 bytes, max 16,777,216 (16 MB)
  No handshake required; TCP connection with ordered request/response
```

### Request Header (v1, non-flexible)

```
RequestApiKey      : int16     (API key identifier)
RequestApiVersion  : int16     (protocol version)
CorrelationId      : int32     (client-assigned, echoed in response)
ClientId           : string    (int16 length prefix, nullable: -1 = null)
```

### Request Header (v2, flexible — KIP-482)

Same fields + tagged fields section (varint-encoded).

### Response Header

```
CorrelationId : int32
```

---

## Primitive Types

| Type | Size | Encoding |
|------|------|----------|
| int8 | 1 | signed big-endian |
| int16 | 2 | signed big-endian |
| int32 | 4 | signed big-endian |
| int64 | 8 | signed big-endian |
| uint16 | 2 | unsigned big-endian |
| uint32 | 4 | unsigned big-endian |
| bool | 1 | 0=false, 1=true |
| float64 | 8 | IEEE 754 double |
| string | 2+N | int16 length + UTF-8 (-1 = null) |
| bytes | 4+N | int32 length + raw (-1 = null) |
| uuid | 16 | raw 128-bit |
| varint | 1-5 | zigzag-encoded variable-length |
| varlong | 1-10 | zigzag-encoded variable-length |

### Flexible Encoding (KIP-482)

Flexible versions use compact encoding:
- **Compact string:** varint length + UTF-8 (0 = null, 1 = empty)
- **Compact array:** varint count + elements (0 = null)
- **Tagged fields:** varint tag + varint size + data (omitted if default)

---

## Record Batch Format (Magic V2)

```
BaseOffset             : int64     [0]   first offset in batch
Length                 : int32     [8]   batch size excluding BaseOffset+Length
PartitionLeaderEpoch   : int32     [12]  epoch of partition leader
Magic                  : int8      [16]  always 2 (v0/v1 removed in Kafka 4.0)
CRC                    : uint32    [17]  CRC-32C from Attributes through Records
Attributes             : int16     [21]  see bit layout below
LastOffsetDelta        : int32     [23]  last offset relative to BaseOffset
BaseTimestamp          : int64     [27]  first record timestamp
MaxTimestamp           : int64     [35]  max timestamp in batch
ProducerId             : int64     [43]  -1 if non-idempotent
ProducerEpoch          : int16     [51]  -1 if non-idempotent
BaseSequence           : int32     [53]  -1 if non-idempotent
RecordsCount           : int32     [57]  number of records
Records                : [Record]  [61+] variable-length records
```

### Attributes Bit Layout (int16)

```
Bit 15-7: Reserved (must be 0)
Bit 6:    Delete Horizon Flag
Bit 5:    Control Flag (1=control batch, 0=data batch)
Bit 4:    Transactional Flag (1=part of transaction)
Bit 3:    Timestamp Type (0=CREATE_TIME, 1=LOG_APPEND_TIME)
Bit 2-0:  Compression (0=NONE, 1=GZIP, 2=SNAPPY, 3=LZ4, 4=ZSTD)
```

### Individual Record Format (within batch)

```
Length        : varint   record size
Attributes   : int8     unused (reserved)
TimestampDelta: varlong  offset from BaseTimestamp
OffsetDelta   : varint   offset from BaseOffset
KeyLength     : varint   -1 = null
Key           : bytes
ValueLength   : varint   -1 = null
Value         : bytes
HeadersCount  : varint
Headers       : [{KeyLength: varint, Key: bytes, ValueLength: varint, Value: bytes}]
```

---

## Compression Codecs

| Value | Codec | Notes |
|-------|-------|-------|
| 0 | NONE | No compression |
| 1 | GZIP | JDK Deflater, levels 1-9 (default 6) |
| 2 | SNAPPY | Google Snappy, third-party |
| 3 | LZ4 | LZ4 frame format, levels 1-17 (default 9) |
| 4 | ZSTD | Zstandard (KIP-110), levels -131072 to 22 (default 3) |

---

## API Keys (Ivy-Supported Subset)

| Key | Name | Versions | Category |
|-----|------|----------|----------|
| 0 | Produce | 3-13 | Core |
| 1 | Fetch | 4-18 | Core |
| 2 | ListOffsets | 1-11 | Core |
| 3 | Metadata | 0-13 | Core |
| 8 | OffsetCommit | 2-10 | Groups |
| 9 | OffsetFetch | 1-10 | Groups |
| 10 | FindCoordinator | 0-6 | Groups |
| 11 | JoinGroup | 0-9 | Groups |
| 12 | Heartbeat | 0-4 | Groups |
| 13 | LeaveGroup | 0-5 | Groups |
| 14 | SyncGroup | 0-5 | Groups |
| 15 | DescribeGroups | 0-5 | Groups |
| 16 | ListGroups | 0-4 | Groups |
| 17 | SaslHandshake | 0-1 | Auth |
| 18 | ApiVersions | 0-4 | Core |
| 19 | CreateTopics | 2-7 | Admin |
| 20 | DeleteTopics | 1-6 | Admin |
| 21 | DeleteRecords | 0-2 | Admin |
| 22 | InitProducerId | 0-6 | Transactions |
| 24 | AddPartitionsToTxn | 0-5 | Transactions |
| 25 | AddOffsetsToTxn | 0-3 | Transactions |
| 26 | EndTxn | 0-5 | Transactions |
| 28 | TxnOffsetCommit | 0-3 | Transactions |
| 29 | DescribeAcls | 0-3 | ACL |
| 30 | CreateAcls | 0-3 | ACL |
| 31 | DeleteAcls | 0-3 | ACL |
| 32 | DescribeConfigs | 0-4 | Admin |
| 35 | DescribeLogDirs | 0-4 | Admin |
| 36 | SaslAuthenticate | 0-2 | Auth |
| 37 | CreatePartitions | 0-3 | Admin |
| 38 | CreateDelegationToken | 0-3 | Auth |
| 39 | RenewDelegationToken | 0-2 | Auth |
| 40 | ExpireDelegationToken | 0-2 | Auth |
| 41 | DescribeDelegationToken | 0-3 | Auth |
| 42 | DeleteGroups | 0-2 | Groups |
| 44 | IncrementalAlterConfigs | 0-1 | Admin |
| 47 | OffsetDelete | 0-0 | Groups |
| 50 | DescribeUserScramCredentials | 0-0 | Auth |
| 51 | AlterUserScramCredentials | 0-0 | Auth |
| 60 | DescribeCluster | 0-1 | Admin |
| 61 | DescribeProducers | 0-0 | Transactions |
| 65 | DescribeTransactions | 0-0 | Transactions |
| 66 | ListTransactions | 0-0 | Transactions |
| 68 | ConsumerGroupHeartbeat | 0-0 | Groups (KIP-848) |
| 75 | DescribeTopicPartitions | 0-0 | Admin |
| 88 | StreamsGroupHeartbeat | 0-0 | Streams (KIP-848) |

---

## Key Request/Response Formats

### Produce (API Key 0)

**Request (v9+, flexible):**
```
TransactionalId  : compact_nullable_string
Acks             : int16  (-1=ALL, 0=NONE, 1=LEADER)
TimeoutMs        : int32
TopicData        : compact_array [{
    Name         : compact_string
    PartitionData: compact_array [{
        Index    : int32
        Records  : compact_bytes (RecordBatch)
    }]
}]
```

**Response:**
```
Responses        : compact_array [{
    Name         : compact_string
    PartitionResponses: compact_array [{
        Index         : int32
        ErrorCode     : int16
        BaseOffset    : int64
        LogAppendTimeMs: int64
        LogStartOffset: int64
    }]
}]
ThrottleTimeMs   : int32
```

### Fetch (API Key 1)

**Request (v12+, flexible):**
```
MaxWaitMs        : int32
MinBytes         : int32
MaxBytes         : int32
IsolationLevel   : int8  (0=READ_UNCOMMITTED, 1=READ_COMMITTED)
SessionId        : int32
SessionEpoch     : int32
Topics           : compact_array [{
    TopicId      : uuid
    Partitions   : compact_array [{
        Partition : int32
        CurrentLeaderEpoch: int32
        FetchOffset: int64
        PartitionMaxBytes: int32
    }]
}]
```

**Response:**
```
ThrottleTimeMs   : int32
SessionId        : int32
Responses        : compact_array [{
    TopicId      : uuid
    Partitions   : compact_array [{
        PartitionIndex   : int32
        ErrorCode        : int16
        HighWatermark    : int64
        LastStableOffset : int64
        AbortedTransactions: compact_nullable_array [{
            ProducerId   : int64
            FirstOffset  : int64
        }]
        Records          : compact_nullable_bytes
    }]
}]
```

---

## Error Codes (Key Subset)

| Code | Name | Retriable |
|------|------|-----------|
| 0 | NONE | — |
| 1 | OFFSET_OUT_OF_RANGE | Yes |
| 3 | UNKNOWN_TOPIC_OR_PARTITION | Yes |
| 5 | LEADER_NOT_AVAILABLE | Yes |
| 6 | NOT_LEADER_OR_FOLLOWER | Yes |
| 7 | REQUEST_TIMED_OUT | Yes |
| 14 | COORDINATOR_LOAD_IN_PROGRESS | Yes |
| 16 | NOT_COORDINATOR | Yes |
| 22 | ILLEGAL_GENERATION | Yes |
| 25 | UNKNOWN_MEMBER_ID | Yes |
| 27 | REBALANCE_IN_PROGRESS | Yes |
| 29 | TOPIC_AUTHORIZATION_FAILED | No |
| 33 | UNSUPPORTED_SASL_MECHANISM | No |
| 34 | ILLEGAL_SASL_STATE | No |
| 36 | TOPIC_ALREADY_EXISTS | No |
| 42 | INVALID_REQUEST | No |
| 45 | OUT_OF_ORDER_SEQUENCE_NUMBER | No |
| 46 | DUPLICATE_SEQUENCE_NUMBER | No |
| 47 | INVALID_PRODUCER_EPOCH | No |
| 48 | INVALID_TXN_STATE | No |
| 58 | SASL_AUTHENTICATION_FAILED | No |
| 59 | UNKNOWN_PRODUCER_ID | No |
| 87 | INVALID_RECORD | No |
| 89 | THROTTLING_QUOTA_EXCEEDED | Yes |
| 90 | PRODUCER_FENCED | No |
| 100 | UNKNOWN_TOPIC_ID | Yes |
| 110 | FENCED_MEMBER_EPOCH | Yes |

Full error code table: 134 codes (0-133). See Kafka `Errors.java` for complete reference.

---

*Last updated: 2026-03-25*
