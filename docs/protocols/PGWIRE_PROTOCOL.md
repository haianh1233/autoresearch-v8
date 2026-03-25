# PostgreSQL Wire Protocol (PgWire) Specification

> **Related:** [../PROTOCOLS.md](../PROTOCOLS.md) §PostgreSQL Wire Protocol

---

## Startup Sequence

```
C→S: SSLRequest (optional)
  Length: int32 = 8
  Code:   int32 = 80877103
S→C: 'S' (SSL ok) or 'N' (SSL not supported)

C→S: StartupMessage
  Length:  int32 (total message size including this field)
  Protocol: int32 = 196608 (version 3.0 = 3<<16 | 0)
  Parameters: key-value pairs (null-terminated strings)
    "user"     → username
    "database" → database name
    "\0"       → terminator

S→C: Authentication message(s)
S→C: ParameterStatus (multiple)
S→C: BackendKeyData
S→C: ReadyForQuery ('I')
```

---

## Message Types (Frontend → Backend)

| Byte | Name | Description |
|------|------|-------------|
| `Q` | Query | Simple query (SQL string, null-terminated) |
| `P` | Parse | Extended query: prepare statement |
| `B` | Bind | Extended query: bind parameters to portal |
| `D` | Describe | Describe prepared statement or portal |
| `E` | Execute | Execute portal |
| `C` | Close | Close prepared statement or portal |
| `S` | Sync | Sync (end of extended query pipeline) |
| `H` | Flush | Flush output buffer |
| `X` | Terminate | Close connection |
| `p` | PasswordMessage | Password response (MD5/SCRAM) |
| `F` | FunctionCall | Call a stored function |
| `d` | CopyData | COPY data row |
| `c` | CopyDone | COPY complete |
| `f` | CopyFail | COPY failed |

---

## Message Types (Backend → Frontend)

| Byte | Name | Description |
|------|------|-------------|
| `R` | Authentication | Auth request/response (subtypes below) |
| `K` | BackendKeyData | Process ID + secret key (for cancel) |
| `S` | ParameterStatus | Runtime parameter (e.g., server_version) |
| `Z` | ReadyForQuery | Transaction status indicator |
| `T` | RowDescription | Column metadata for result set |
| `D` | DataRow | Single result row |
| `C` | CommandComplete | SQL command completed (e.g., "SELECT 5") |
| `E` | ErrorResponse | Error with structured fields |
| `N` | NoticeResponse | Non-fatal notice |
| `I` | EmptyQueryResponse | Empty query string |
| `1` | ParseComplete | Statement parsed successfully |
| `2` | BindComplete | Parameters bound successfully |
| `3` | CloseComplete | Statement/portal closed |
| `n` | NoData | Describe returned no rows |
| `t` | ParameterDescription | Parameter types for prepared statement |
| `G` | CopyInResponse | Server ready for COPY IN data |
| `H` | CopyOutResponse | Server sending COPY OUT data |
| `d` | CopyData | COPY data row |
| `c` | CopyDone | COPY complete |

---

## Authentication Messages (type `R`)

| Subtype | Value | Description |
|---------|-------|-------------|
| AuthenticationOk | 0 | Authentication successful |
| AuthenticationKerberosV5 | 2 | Kerberos V5 required |
| AuthenticationCleartextPassword | 3 | Cleartext password required |
| AuthenticationMD5Password | 5 | MD5 password required (+ 4-byte salt) |
| AuthenticationSASL | 10 | SASL mechanism list follows |
| AuthenticationSASLContinue | 11 | SASL challenge data |
| AuthenticationSASLFinal | 12 | SASL final response |

---

## Simple Query Protocol

```
C→S: Query('Q')
  Length: int32
  SQL:    null-terminated string

S→C: RowDescription('T')   ← column metadata
  NumFields: int16
  Fields: [{
    Name:       null-terminated string
    TableOID:   int32
    ColumnAttr: int16
    TypeOID:    int32
    TypeSize:   int16
    TypeMod:    int32
    Format:     int16 (0=text, 1=binary)
  }]

S→C: DataRow('D')          ← per row
  NumColumns: int16
  Columns: [{
    Length: int32 (-1 = NULL)
    Data:   [Length bytes]
  }]

S→C: CommandComplete('C')
  Tag: null-terminated string (e.g., "SELECT 5", "INSERT 0 1")

S→C: ReadyForQuery('Z')
  Status: byte ('I'=idle, 'T'=in transaction, 'E'=error)
```

---

## Extended Query Protocol

```
C→S: Parse('P')     → name, query, param_types[]
S→C: ParseComplete('1')

C→S: Bind('B')      → portal, statement, param_formats[], param_values[], result_formats[]
S→C: BindComplete('2')

C→S: Describe('D')  → type ('S'=statement, 'P'=portal), name
S→C: RowDescription('T') or NoData('n')

C→S: Execute('E')   → portal, max_rows (0=unlimited)
S→C: DataRow('D')...
S→C: CommandComplete('C')

C→S: Sync('S')
S→C: ReadyForQuery('Z')
```

---

## ErrorResponse Fields

| Code | Field | Description |
|------|-------|-------------|
| `S` | Severity | ERROR, FATAL, PANIC, WARNING, NOTICE, DEBUG, INFO, LOG |
| `V` | Severity (non-localized) | Always English |
| `C` | SQLSTATE Code | 5-character error code |
| `M` | Message | Human-readable error message |
| `D` | Detail | Optional detailed error message |
| `H` | Hint | Suggestion for fixing the problem |
| `P` | Position | Cursor position in query string |
| `W` | Where | Call stack context |
| `F` | File | Source file name |
| `L` | Line | Source line number |
| `R` | Routine | Source function name |

### Common SQLSTATE Codes

| Code | Name |
|------|------|
| 00000 | Successful Completion |
| 08000 | Connection Exception |
| 28000 | Invalid Authorization Specification |
| 28P01 | Invalid Password |
| 42501 | Insufficient Privilege |
| 42601 | Syntax Error |
| 42P01 | Undefined Table |
| 42703 | Undefined Column |

---

## Type OIDs

| OID | Type Name | Size |
|-----|-----------|------|
| 16 | bool | 1 |
| 17 | bytea | variable |
| 20 | int8 (bigint) | 8 |
| 21 | int2 (smallint) | 2 |
| 23 | int4 (integer) | 4 |
| 25 | text | variable |
| 700 | float4 (real) | 4 |
| 701 | float8 (double) | 8 |
| 1043 | varchar | variable |
| 1114 | timestamp | 8 |
| 1184 | timestamptz | 8 |
| 2950 | uuid | 16 |
| 3802 | jsonb | variable |

---

*Last updated: 2026-03-25*
