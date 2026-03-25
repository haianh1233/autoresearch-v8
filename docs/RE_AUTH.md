# Re-Authentication Design

> **Related:** [MULTI_TENANT.md](MULTI_TENANT.md), [RULES.md](RULES.md) §R27–R30,
> [SECURITY.md](SECURITY.md), [TRANSACTIONS.md](TRANSACTIONS.md), [CLUSTERING.md](CLUSTERING.md)

---

## Problem Statement

A persistent messaging connection has two distinct lifecycles that diverge over time:

```
Connection lifetime:  ─────────────────────────────────────────────────────────► (hours/days)
Credential lifetime:  ──────────┤  ──────────┤  ──────────┤  (hours; expires/rotates)
                              exp         exp         exp
```

Without re-auth, the only options are:
1. **Never expire credentials** — unacceptable for security (compromised cred = permanent access)
2. **Close + reconnect on expiry** — loses producer state, consumer group offsets, in-flight messages

Re-auth solves this: the client refreshes credentials on the existing TCP connection without losing
messaging session state. This is especially critical for:
- **Long-lived producers** that use OAuth JWT tokens (typically 1-hour TTL)
- **Consumer groups** that would lose partition assignments on reconnect
- **Exactly-once producers** that would lose `producerId`/`producerEpoch` on reconnect
- **Admin operations** holding locks or in-progress transactions

**Requirements:**
1. No disconnect required for credential refresh on protocols that support it (Kafka, MQTT 5.0, Pulsar)
2. For protocols without in-band re-auth (AMQP, STOMP, NATS), graceful close must allow the client to reconnect cleanly
3. Tenant MUST NOT change across re-auth — enforced at the state machine level
4. ACL cache invalidated on every successful re-auth to pick up permission changes
5. Idle connections NOT forcibly closed — only enforced on next request (KIP-368 semantics)
6. Max 3 re-auth attempts per connection to prevent brute-force via re-auth

---

## Architecture: 3-Tier Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Tier 1: Per-Connection State Machine (ReAuthManager)                  │
│  ─ AtomicReference<ReAuthState> with CAS transitions (lock-free)       │
│  ─ Tenant immutability enforcement                                      │
│  ─ MAX_REAUTH_ATTEMPTS = 3 per connection                               │
│  ─ Stores in Netty channel attribute; one instance per live connection  │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │ calls
┌──────────────────────────────────▼──────────────────────────────────────┐
│  Tier 2: Protocol-Specific Handlers (ivy-codec)                        │
│  ─ Category A: Kafka SaslHandshake+SaslAuthenticate (KIP-368)          │
│  ─ Category A: MQTT 5.0 AUTH packet                                     │
│  ─ Category A: Pulsar CommandAuthChallenge/CommandAuthResponse          │
│  ─ Category A: RMQ Streams SASL re-send                                 │
│  ─ Category B: MySQL COM_CHANGE_USER                                    │
│  ─ Category B: Redis AUTH command                                       │
│  ─ Category C: Graceful error frame + close (AMQP, NATS, STOMP, etc.)  │
│  ─ Category D: Per-request (stateless, no session to re-auth)          │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │ scheduled by / triggers
┌──────────────────────────────────▼──────────────────────────────────────┐
│  Tier 3: Scheduling + Reactive Revocation (ivy-auth)                   │
│  ─ ReAuthScheduler: proactive timer with jitter [0, 5000ms)            │
│  ─ CredentialRevocationHandler: registry + reactive push               │
│  ─ ReAuthMetrics: LongAdder counters, timer gauge                       │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Module Placement

| Component | Module | Responsibility |
|-----------|--------|---------------|
| `ReAuthState` | `ivy-common` | Enum (5 states) |
| `ReAuthConfig` | `ivy-common` | Config record |
| `CredentialRevokedEvent` | `ivy-common` | Event value record |
| `ReAuthManager` | `ivy-auth` | Per-connection state machine |
| `ReAuthScheduler` | `ivy-auth` | Timer management |
| `CredentialRevocationHandler` | `ivy-auth` | Registry + reactive revocation |
| `ReAuthMetrics` | `ivy-auth` | Counters and gauges |
| `KafkaSaslAuthenticateHandler` | `ivy-codec` (kafka) | KIP-368 mid-session re-auth |
| `MqttReAuthHandler` | `ivy-codec` (mqtt) | AUTH packet handling |
| `PulsarReAuthHandler` | `ivy-codec` (pulsar) | Challenge/response protocol |
| `RmqStreamsReAuthHandler` | `ivy-codec` (rmq) | SASL re-auth |
| `MysqlChangeUserHandler` | `ivy-codec` (mysql) | COM_CHANGE_USER |
| `RedisAuthHandler` | `ivy-codec` (redis) | AUTH command |
| `ProtocolReAuthHandler` | `ivy-codec` (shared) | Category C graceful close |

**Key rule:** `ivy-broker` has ZERO re-auth awareness. All state machine transitions happen in
`ivy-auth`/`ivy-codec` before any request reaches the broker engine.

---

Re-authentication (re-auth) allows a client on a persistent connection to refresh its
credentials without disconnecting. This is required when:

- A session token or OAuth JWT is about to expire (`exp` claim)
- Credentials are administratively rotated or revoked
- The broker enforces a maximum session lifetime (`maxReauthMs`)
- The client's ACL set must be refreshed after a permission change
- A TLS client certificate is approaching its `notAfter` expiry

---

## Protocol Classification

Protocols fall into five categories based on how (or whether) they support re-auth:

| Category | Protocols | Mechanism |
|----------|-----------|-----------|
| **A — Native in-band** | Kafka (KIP-368), MQTT 5.0 (AUTH packet) | Re-auth frame on existing TCP without disconnect |
| **B — Protocol command** | MySQL (COM_CHANGE_USER) | A dedicated command resets session credentials in-band |
| **C — Reconnect-based** | AMQP 0-9-1, AMQP 1.0, MQTT 3.1.1, PgWire | Broker closes gracefully; client reconnects with new credentials |
| **D — Stateless per-request** | HTTP | Every request carries credentials; no session to re-auth |
| **E — Certificate-bound (mTLS)** | Any TLS listener with `clientAuth=REQUIRE` | No in-band re-auth path; broker closes gracefully before cert expiry |

---

## Category A: Kafka KIP-368 (Native In-Band Re-Auth)

Kafka introduced re-authentication in KIP-368. The client re-authenticates on the existing
TCP connection using the same API keys as the initial handshake (API 17 `SaslHandshake` +
API 36 `SaslAuthenticate`), but after the connection is already established.

### Supported Mechanisms

Both `SCRAM-SHA-256` and `OAUTHBEARER` support in-band re-auth on Kafka:

- **SCRAM-SHA-256**: full 4-message challenge/response exchange (RFC 5802)
- **OAUTHBEARER**: single-round — client sends a new JWT in the `SaslAuthenticate`
  payload; broker validates signature, `iss`, `aud`, `exp`, `nbf`, `sub`

### Wire Flow (SCRAM-SHA-256)

```
─── existing authenticated connection ───────────────────────────────────────────────

C→S: SaslHandshakeRequest(mechanism="SCRAM-SHA-256")
     [API key 17, version ≥ 1]
     Note: sent on existing connection, not during new TLS handshake

S→C: SaslHandshakeResponse(errorCode=0, mechanisms=[SCRAM-SHA-256, OAUTHBEARER])
     Note: broker sets sessionLifetimeMs in the response to signal re-auth window

C→S: SaslAuthenticateRequest(saslAuthBytes=<client-first-message>)
     [API key 36]

S→C: SaslAuthenticateResponse(saslAuthBytes=<server-first-message>)

C→S: SaslAuthenticateRequest(saslAuthBytes=<client-final-message>)

S→C: SaslAuthenticateResponse(
       errorCode=0,
       saslAuthBytes=<server-final>,
       sessionLifetimeMs=3600000  ← new session lifetime (0 = no limit)
     )

─── connection is now re-authenticated ───────────────────────────────────────────────
```

### Wire Flow (OAUTHBEARER)

```
─── existing authenticated connection ───────────────────────────────────────────────

C→S: SaslHandshakeRequest(mechanism="OAUTHBEARER")

S→C: SaslHandshakeResponse(errorCode=0)

C→S: SaslAuthenticateRequest(saslAuthBytes=<OAUTHBEARER token value>)

S→C: SaslAuthenticateResponse(
       errorCode=0,
       saslAuthBytes=b"",
       sessionLifetimeMs=<exp_ms - now_ms>  ← derived from JWT exp claim
     )

─── connection is now re-authenticated ───────────────────────────────────────────────
```

### JWT Validation Pipeline (OAUTHBEARER)

When the credential is an OAUTHBEARER token, the broker runs an 8-step pipeline:

1. **Signature verification** — RS256/RS384/RS512 (RSA) or ES256/ES384 (ECDSA) only.
   `alg=none` and all HMAC symmetric algorithms (HS256/HS384/HS512) are **rejected**
   (CVE-2015-9235 mitigation). Algorithm must match the key type (RSA key → RS*, EC key → ES*).
2. **Issuer (`iss`) validation** — must match the tenant's configured allowed issuers list.
3. **Audience (`aud`) validation** — broker's audience string must appear in the `aud` claim.
4. **Expiration (`exp`) check** — `exp` must be > `now - clockSkewTolerance` (default 5s).
5. **Not-before (`nbf`) check** — `nbf` (optional) must be ≤ `now + clockSkewTolerance`.
6. **Scope extraction** — `scope` or `scp` claim mapped to permission set.
7. **Principal extraction** — principal extracted from configurable claim (default `sub`).
8. **Tenant validation** — tenant extracted from configurable claim; must match the
   connection's existing `TenantId` (tenant immutability invariant).

JWKS public keys are cached for 1 hour with a background refresh; synchronous refresh
occurs on cache miss (first request for a new key ID).

### Key Rules

1. **Tenant immutability**: re-auth MUST NOT change the tenant. If the new credentials
   resolve to a different `TenantId`, the broker closes the connection immediately with
   `ILLEGAL_SASL_STATE` error.

2. **Principal may change**: the user identity may change (e.g., different service account)
   as long as the tenant is the same.

3. **ACL cache invalidated**: on every successful re-auth, all cached ACL decisions for
   this connection are flushed. The next authorization check re-evaluates from `acl_entries`.

4. **Idle connection behaviour**: per Kafka KIP-368, idle connections are NOT forcibly closed
   at session expiry. The session lifetime is only enforced when the client next sends a request.
   A request arriving on an expired session triggers an immediate re-auth challenge
   (or `RE_AUTHENTICATION_REQUIRED` error).

5. **In-flight requests**: the broker must not begin processing a new Produce or Fetch request
   on a connection that has entered `RE_AUTH_REQUIRED` state. Outstanding in-flight requests
   that started before the expiry window are completed normally.

### `SaslHandshakeResponse.sessionLifetimeMs`

The broker sets this to signal the remaining lifetime of the new session:

```
sessionLifetimeMs = 0          → no broker-imposed expiry (credential has no TTL)
sessionLifetimeMs = N > 0      → client should re-auth before N milliseconds elapse
```

`ReAuthScheduler` reads this value and schedules the next re-auth attempt at
`sessionLifetimeMs - reAuthBufferMs` (default buffer: 30s) with jitter `[0, 5000ms)`.

For OAUTHBEARER, `sessionLifetimeMs` is derived from the JWT `exp` claim:
`sessionLifetimeMs = (exp * 1000) - System.currentTimeMillis()`.

---

## Category A: MQTT 5.0 AUTH Packet

MQTT 5.0 supports re-auth via the `AUTH` packet (type 15), which can be sent by either
party at any time after the initial `CONNACK`.

```
S→C: AUTH(reasonCode=0x19 [Re-authenticate], authMethod="SCRAM-SHA-256", authData=<server-challenge>)
C→S: AUTH(reasonCode=0x18 [Continue], authData=<client-response>)
S→C: AUTH(reasonCode=0x00 [Success], authData=<server-final>)
```

The broker initiates by sending `AUTH(0x19)`. The client continues with `AUTH(0x18)` rounds
until the broker sends `AUTH(0x00)`.

On failure: broker sends `DISCONNECT(0x87 Not Authorized)` and closes the connection.

The `authMethod` in the `AUTH` packet MUST match the `authMethod` in the original `CONNECT`.
Changing the mechanism mid-session is not allowed.

---

## Category B: MySQL COM_CHANGE_USER

MySQL clients send `COM_CHANGE_USER` (0x11) to reset session credentials in-band.

```
C→S: COM_CHANGE_USER(user, auth_response, database, charset, auth_plugin_name)
S→C: AuthSwitchRequest(plugin_name, auth_plugin_data)  [if plugin switch needed]
C→S: AuthSwitchResponse(auth_data)
S→C: OK_Packet  or  ERR_Packet
```

On success, the session is re-authenticated with the new principal. All session state
(prepared statements, in-progress result sets) is reset.

---

## Category C: Reconnect-Based Re-Auth

AMQP 0-9-1, AMQP 1.0, MQTT 3.1.1, and PgWire have no wire-level re-auth frame.
Re-auth is accomplished via a graceful close + reconnect.

### Server-Initiated Graceful Close

When `ReAuthScheduler` determines that a connection's session lifetime has expired:

1. The broker stops delivering new messages on the connection.
2. The broker sends a protocol-specific "please reconnect" signal:
   - **AMQP 0-9-1**: `Connection.Close(reply-code=320, reply-text="re-authenticate")`
   - **AMQP 1.0**: `close(error={condition="amqp:unauthorized-access", description="session-expired"})`
   - **MQTT 3.1.1**: `DISCONNECT` (no reason code in 3.1.1; broker simply closes)
   - **PgWire**: `ErrorResponse(severity=FATAL, code=28P01, message="re-authentication required")` then closes
3. The TCP connection is closed after the protocol close handshake (or immediately on
   MQTT 3.1.1 where no server-side DISCONNECT exists in the standard).
4. The client reconnects with fresh credentials.

### Credential Revocation (Reactive)

If credentials are revoked while a connection is active (e.g., user deleted):

1. `CredentialRevocationHandler` is notified via the internal credential-change event.
2. It looks up all active connections using those credentials (via `ConnectionRegistry`).
3. Each matching connection receives a graceful close (same protocol signals as above).
4. The client must reconnect with new valid credentials.

---

## Category D: Stateless Per-Request Auth (HTTP)

HTTP connections carry credentials on every request (`Authorization: Bearer <token>` or
`X-API-Key: <key>`). There is no concept of re-auth because there is no session to refresh.

Token expiry is handled at the request level:
- If the token is expired, the broker returns `401 Unauthorized`.
- The client obtains a new token and retries the request.

---

## Category E: Certificate-Bound (mTLS)

When a TLS listener is configured with `clientAuth=REQUIRE`, the client presents an X.509
certificate during the TLS handshake. The certificate's `notAfter` field plays the role that
`exp` plays for JWT tokens.

TLS 1.3 does not support post-handshake renegotiation, so there is no in-band re-auth path
for mTLS connections. The only option is close + reconnect with a new certificate.

### Broker Behaviour

`ReAuthScheduler` treats the certificate expiry as the session lifetime:

```
sessionLifetimeMs = (cert.notAfter.getTime()) - System.currentTimeMillis()
```

When the timer fires (at `certExpiresAt - reAuthBufferMs`):

1. The broker sends a protocol-appropriate graceful close (same signals as Category C).
2. The client reconnects with a renewed certificate.
3. A new TLS handshake establishes the new certificate identity.

### Key Differences from JWT

- No separate re-auth round trip; the new certificate replaces the old one at connection time.
- `TenantId` is derived from SNI, not the certificate subject, so tenant immutability is
  enforced by the SNI even if the certificate CN changes.
- Certificate rotation is managed by an external PKI (cert-manager, Vault, ACME). The broker
  does not generate or renew certificates.

---

## SCRAM Credential Rotation Grace Period

When a broker admin rotates a SCRAM password, connections that authenticated with the old
credential and are mid-re-auth (or about to re-auth) would fail immediately if the old
credential is invalidated at the moment of rotation.

### Solution: Dual-Credential Grace Window

The `credentials` table stores a previous SCRAM credential alongside the current one:

```sql
ALTER TABLE credentials ADD COLUMN prev_scram_256  JSONB;   -- nullable
ALTER TABLE credentials ADD COLUMN prev_valid_until TIMESTAMPTZ;  -- nullable
```

During SCRAM re-auth, the broker tries credentials in order:

1. Try `scram_256` (current credential) → use if valid.
2. If step 1 fails AND `prev_valid_until > now()` → try `prev_scram_256`.
3. If both fail → re-auth fails.

Default grace window: **5 minutes** (configurable via `reauth.credentialGracePeriodMs`).

When a new credential is stored:

```
prev_scram_256   = current scram_256   (old credential)
prev_valid_until = now() + credentialGracePeriodMs
scram_256        = new credential
```

After `prev_valid_until` has passed, `prev_scram_256` is cleared on the next write. Old
credentials are never accepted outside the grace window.

---

## ReAuthManager (Per-Connection State Machine)

Each active connection has a `ReAuthManager` instance stored in the Netty channel attribute.

### States

```
AUTHENTICATED
  │
  ├── [session lifetime expires or credential revoked]
  ↓
RE_AUTH_REQUIRED
  │
  ├── [client sends SaslHandshake / AUTH / COM_CHANGE_USER]
  ↓
IN_PROGRESS                    ← challenge/response rounds in flight
  │
  ├── [success]      ├── [failure or max attempts]   ├── [grace period elapsed]
  ↓                  ↓                                ↓
AUTHENTICATED    RE_AUTH_FAILED → conn closed      TIMED_OUT → conn closed
```

### CAS Transitions

```java
// AtomicReference<ReAuthState> on connection
//
// AUTHENTICATED → RE_AUTH_REQUIRED  (scheduled timer or revocation push)
// RE_AUTH_REQUIRED → IN_PROGRESS    (first re-auth frame received from client)
// IN_PROGRESS → AUTHENTICATED       (challenge completed successfully)
// IN_PROGRESS → RE_AUTH_FAILED      (max attempts = 3 exceeded, or tenant mismatch)
// IN_PROGRESS → TIMED_OUT           (gracePeriodMs elapsed without client response)
```

### Terminal States

`RE_AUTH_FAILED` and `TIMED_OUT` are both terminal: the connection is closed immediately
and no further requests are processed.

### Invariants

- `MAX_REAUTH_ATTEMPTS = 3` per connection lifetime
- Tenant MUST NOT change across re-auth (checked in `IN_PROGRESS → AUTHENTICATED` transition)
- ACL cache invalidated atomically with the state transition to `AUTHENTICATED`
- A connection in `RE_AUTH_FAILED` or `TIMED_OUT` state is immediately closed
- Old `SecurityContext` fully replaced atomically on success; no stale reference held

---

## ReAuthScheduler

`ReAuthScheduler` manages session expiry timers across all active connections.

**Capacity**: maximum 100,000 concurrent timers (bounded to limit heap usage). Connections
beyond this limit are assigned no proactive timer; re-auth is triggered reactively when the
client next sends a request.

### Session Lifetime Calculation

```
if (credential has no TTL and maxReauthMs == 0):
    sessionLifetimeMs = 0          → no scheduled re-auth

if (credential is a JWT):
    remaining = (exp * 1000) - now
    sessionLifetimeMs = (maxReauthMs == 0) ? remaining : min(maxReauthMs, remaining)

if (credential is SCRAM/PLAIN):
    sessionLifetimeMs = maxReauthMs   (0 = no broker-imposed limit)

if (credential is an X.509 certificate):
    remaining = cert.notAfter.getTime() - now
    sessionLifetimeMs = (maxReauthMs == 0) ? remaining : min(maxReauthMs, remaining)

if (credential is already expired):
    sessionLifetimeMs = 1          → immediate re-auth on next request
```

### Timer Scheduling

```
scheduleReAuth(connectionId, sessionLifetimeMs):
    if (sessionLifetimeMs <= 0): return        // no timer
    delay = sessionLifetimeMs - reAuthBufferMs
    jitter = random(0, 5000)                   // ms; prevents thundering-herd
    if (delay - jitter <= 0): triggerImmediately()
    else: scheduler.schedule(connectionId, delay - jitter, this::triggerReAuth)
```

Default `reAuthBufferMs = 30_000` (30 seconds before expiry).

Timers are cancelled on connection close via `ReAuthScheduler.cancel(connectionId)`.

---

## CredentialRevocationHandler

Bridges credential lifecycle events to active connections.

```java
class CredentialRevocationHandler {
    // Maps (tenantId, username) → Set<ConnectionId>
    private final ConcurrentHashMap<TenantUsername, Set<ConnectionId>> registry;

    void onConnect(TenantId, String username, ConnectionId)    // register
    void onDisconnect(ConnectionId)                            // unregister

    // Specific principal revocation
    void onCredentialRevoked(TenantId, String username)        // close all matching connections

    // Wildcard: revoke ALL principals for a tenant (e.g., tenant suspended)
    void onTenantRevoked(TenantId tenantId)                    // close all tenant connections
}
```

Credential revocation events are pushed via the internal `__credentials` compacted topic.
Any broker that holds connections for the revoked principal reacts within one poll interval.

`TenantUsername` is a composite key `(tenantId, username)` — bare `username` alone is
insufficient because the same username may exist in multiple tenants.

---

## ReAuthMetrics

`ReAuthMetrics` exposes the following counters and gauges:

| Metric | Type | Description |
|--------|------|-------------|
| `reauth.triggered.total` | Counter | Times `RE_AUTH_REQUIRED` state was entered |
| `reauth.success.total` | Counter | Successful re-auth completions |
| `reauth.failure.total` | Counter | Re-auth failures (wrong credential, tenant mismatch) |
| `reauth.timeout.total` | Counter | Connections timed out waiting for re-auth |
| `reauth.timer.active` | Gauge | Currently scheduled re-auth timers |
| `reauth.timer.scheduled.total` | Counter | Timers created since startup |
| `reauth.timer.cancelled.total` | Counter | Timers cancelled (connection closed before expiry) |
| `reauth.revocation.connections_affected` | Counter | Connections closed by credential revocation |

---

## Configuration

```yaml
broker:
  reauth:
    maxReauthMs: 3600000          # 0 = no broker-imposed limit (default: 0)
    reAuthBufferMs: 30000         # schedule re-auth this many ms before expiry (default: 30000)
    maxReauthAttempts: 3          # per connection; FAILED after this many failures (default: 3)
    gracePeriodMs: 60000          # ms after RE_AUTH_REQUIRED before TIMED_OUT (default: 60000)
    credentialGracePeriodMs: 300000  # SCRAM rotation grace window (default: 300000 = 5 min)
    revocationPollMs: 5000        # how often to poll __credentials for revocations (default: 5000)
    failureAction: CLOSE_CONNECTION  # CLOSE_CONNECTION | CONTINUE_READ_ONLY (default: CLOSE_CONNECTION)
    timerCapacity: 100000         # max concurrent re-auth timers (default: 100000)
```

`failureAction: CONTINUE_READ_ONLY` allows a connection with a failed re-auth to continue
consuming messages (but not producing) until the TCP connection is explicitly closed. This is
useful in analytics scenarios where losing a consumer mid-batch is more disruptive than
serving slightly-stale authorization decisions.

Per-tenant overrides:

```yaml
tenants:
  acme:
    auth:
      maxReauthMs: 7200000        # tenant-level override: 2-hour sessions
```

---

## Kafka Produce → PostgreSQL/MySQL Consume: Re-Auth Bridge Pattern

A common operational pattern is:

1. **Kafka producer** uses re-auth to maintain long-lived connections (hours/days) while
   rotating credentials on a schedule.
2. The messages it produces are stored in the shared partition log (PG `messages` table).
3. An operator or monitoring tool connects via **MySQL wire** or **PgWire** to query the
   same messages with a read-only SQL view.

The SQL consumer does not "consume" in the messaging sense (no offset tracking, no group
coordination). It performs ad-hoc queries against the broker's PostgreSQL-backed message store.

### Re-Auth in Kafka Producer (OAUTHBEARER, token rotation every hour)

```
Initial auth:
  SaslHandshake(OAUTHBEARER) → SaslAuthenticate(<JWT, exp=+1h>) → AUTHENTICATED
  SaslAuthenticateResponse.sessionLifetimeMs = 3600000

After ~59 minutes (ReAuthScheduler fires):
  SaslHandshake(OAUTHBEARER)                                  → RE_AUTH_REQUIRED
  SaslAuthenticate(<new JWT, exp=+1h>)                        → IN_PROGRESS
  [broker validates signature, iss, aud, exp, sub]            → AUTHENTICATED
  SaslAuthenticateResponse.sessionLifetimeMs = 3600000         (fresh 1 hour)

Producer continues producing without any disconnect.
```

### Re-Auth in Kafka Producer (SCRAM-SHA-256, 1-hour sessions)

```
Initial auth:
  SaslHandshake(SCRAM-SHA-256) → SaslAuthenticate → AUTHENTICATED
  SaslAuthenticateResponse.sessionLifetimeMs = 3600000

After ~59 minutes (ReAuthScheduler fires):
  SaslHandshake(SCRAM-SHA-256)                          → RE_AUTH_REQUIRED
  SaslAuthenticate(<new-challenge-round>)               → IN_PROGRESS
  SaslAuthenticate(<final>)                             → AUTHENTICATED (new session)
  SaslAuthenticateResponse.sessionLifetimeMs = 3600000  (fresh 1 hour)

Producer continues producing without any disconnect.
```

### MySQL/PgWire Consumer Query

After the Kafka producer has written messages (with or without re-auth in progress):

```sql
-- Via MySQL wire (port 3306):
SELECT key, value, offset_num, timestamp_ms, protocol_id
FROM my_topic
WHERE offset_num > 1000
ORDER BY offset_num
LIMIT 100;

-- Via PgWire (port 5432):
SELECT offset_num, key, value, headers, timestamp_ms
FROM my_topic
WHERE timestamp_ms > 1700000000000
ORDER BY offset_num
LIMIT 100;
```

The `protocol_id` column shows which protocol produced each message:

| protocol_id | Producing protocol |
|-------------|--------------------|
| 1 | Kafka |
| 2 | AMQP 0-9-1 |
| 3 | AMQP 1.0 |
| 4 | MQTT 3.1.1 |
| 5 | MQTT 5.0 |
| 8 | HTTP |

SQL consumers authenticate once at connection time (MySQL handshake or PgWire startup).
Re-auth for SQL consumers follows Category B (MySQL: `COM_CHANGE_USER`) or Category C
(PgWire: reconnect). Both are handled via `CredentialRevocationHandler` for reactive
push-based revocation.

---

## Category A: Pulsar CommandAuthChallenge

Pulsar uses a broker-initiated challenge/response model. The broker sends a random challenge;
the client responds with proof of knowledge.

```
S→C: CommandAuthChallenge(challenge_data=<32 random bytes>, protocol_version, sequence_id=N)
C→S: CommandAuthResponse(response_data=<HMAC proof>, sequence_id=N)
S→C: CommandAuthChallenge(challenge_data=b"", sequence_id=N)  ← empty = success

On failure:
S→C: CommandError(error=AuthenticationError, message="Re-authentication failed")
     [connection closed]
```

The `sequence_id` prevents replay — a stale response with an old sequence ID is rejected.
Clients advertise support via the `supportsAuthRefresh` capability flag in `CommandConnect`.

---

## Category A: RabbitMQ Streams SASL Re-Authentication

RMQ Streams uses a proprietary SASL framing. The client re-sends `SaslAuthenticate` (API key
`0x0013`) on an already-authenticated connection.

```
C→S: SaslAuthenticate(mechanism="SCRAM-SHA-256", sasl_data=<client-first-message>)
S→C: SaslAuthenticate(sasl_data=<server-first-message>)
C→S: SaslAuthenticate(sasl_data=<client-final-message>)
S→C: SaslAuthenticate(sasl_data=b"")  ← empty sasl_data = success
```

**Invariants:**
- Mechanism MUST match the mechanism used during initial auth (error code 20 on mismatch)
- Username MUST match the original authenticated user (error code 21 on mismatch)
- On success, the broker re-evaluates stream-level permissions and may send
  `MetadataUpdate(STREAM_NOT_AVAILABLE)` for streams the re-authed user no longer has access to

---

## Category B: Redis AUTH Command

Redis uses the `AUTH` command for re-authentication. Behaviour differs between RESP2 and RESP3:

```
RESP2 (Redis < 6):
  C→S: *2\r\n$4\r\nAUTH\r\n$<len>\r\n<password>\r\n
  S→C: +OK\r\n   or   -ERR invalid password\r\n

RESP3 (Redis ≥ 6, supports username):
  C→S: *3\r\n$4\r\nAUTH\r\n$<ulen>\r\n<username>\r\n$<plen>\r\n<password>\r\n
  S→C: +OK\r\n   or   -WRONGPASS invalid username-password pair\r\n
```

**On success:** Session preserved; subscriptions and transactions remain active. Current principal
updated to the new user.

**On failure:** Connection NOT closed (unlike Kafka/MQTT). Client may retry. After `MAX_REAUTH_ATTEMPTS`
(3) failures, the connection is closed.

**RESP2 pub/sub note:** AUTH is blocked during an active SUBSCRIBE session in RESP2. The client
must unsubscribe first, then AUTH, then re-subscribe. RESP3 clients may AUTH during pub/sub.

---

## Category C: Extended Protocol Coverage

### NATS

```
S→C: -ERR 'User Authentication Expired'\r\n
     [server closes TCP connection]
```

NATS has no graceful disconnect frame. The broker sends the error string and immediately closes
the TCP connection. Well-behaved NATS clients reconnect automatically.

### STOMP

```
S→C: ERROR\r\n
     message:Authentication credentials expired\r\n
     \r\n
     \0
     [server closes TCP connection]
```

The `message` header provides a human-readable reason. Per the STOMP spec, the server closes
after sending the ERROR frame.

### OpenWire (ActiveMQ Classic protocol)

```
S→C: ExceptionResponse(correlationId=0, exception=JMSSecurityException("credentials expired"))
     [server closes TCP connection]
```

OpenWire `ExceptionResponse` with a `JMSSecurityException` signals the client that re-auth is
required. JMS clients should catch this and reconnect via the ConnectionFactory.

---

## Category D: Extended Stateless Coverage

### AWS Kinesis / SQS / SNS / EventBridge (SigV4)

Every request is independently authenticated via AWS Signature Version 4. There is no session
concept; credential expiry (IAM role token) is transparent to the broker — the next request with
an expired SigV4 credential returns `403 Forbidden`. The client rotates credentials via the AWS
SDK credential provider chain.

```
GET /streams/my-stream/records?ShardIterator=...
Authorization: AWS4-HMAC-SHA256 Credential=AKIAIOSFODNN7EXAMPLE/20240101/us-east-1/kinesis/aws4_request,
               SignedHeaders=host;x-amz-date,
               Signature=<computed>
X-Amz-Date: 20240101T000000Z
X-Amz-Security-Token: <session-token>   ← present for IAM role credentials

On expired token: HTTP 403 {"__type":"InvalidSignatureException",...}
```

### Google Pub/Sub (gRPC + OAuth2)

Every gRPC call carries an OAuth2 Bearer token in the `Authorization` metadata header. There is
no session to re-auth; the gRPC channel is long-lived but each RPC is independently authorized.
Token expiry returns `UNAUTHENTICATED` status; the client obtains a new token and retries.

### S3-Compatible API (Pre-signed URLs)

Pre-signed URLs embed credentials in the URL with an expiry (`X-Amz-Expires`). After expiry,
the broker returns `403 AccessDenied`. There is no re-auth path; the client must generate a new
pre-signed URL. This is handled entirely at the HTTP layer with no session state in the broker.

---

## Shutdown Integration

On broker shutdown, all active re-auth timers must be cancelled before the connection draining
phase to prevent spurious re-auth triggers during shutdown:

```
BrokerMain.shutdown():
  1. Stop accepting new connections
  2. reAuthScheduler.cancelAll()          ← cancel all pending timers
  3. Drain existing connections (wait for in-flight requests)
  4. Close all channels
  5. Shut down event loop groups

channelInactive() hook (per-connection cleanup):
  reAuthScheduler.cancel(connectionId)              ← idempotent
  credentialRevocationHandler.unregister(connId)    ← remove from registry
```

**Race condition on shutdown:** If a timer fires after `cancelAll()` but before the channel is
closed, the `requireReAuth()` CAS will succeed but no response can be sent (channel closing).
The terminal state machine handles this: `TIMED_OUT` → close channel, which is a no-op if
the channel is already closing.

---

## Failure Modes

| Failure | Cause | Broker Behaviour | Client Expectation |
|---------|-------|------------------|--------------------|
| **CAS race: timeout beats success** | Server timer fires while `IN_PROGRESS` success response is in flight | Timer transitions to `TIMED_OUT` first; `onReAuthSuccess()` CAS fails | Client gets `TIMED_OUT` close even though credential was valid; retry connects fresh |
| **ReAuthScheduler at MAX_TIMERS** | 100,000+ concurrent connections | Timer not scheduled; re-auth triggered reactively on next request | Client's next request triggers re-auth challenge; slight extra latency |
| **AuthEngine unavailable during re-auth** | PG connection down; credential store unreachable | `onReAuthFailure()` called; connection closed | Client reconnects; may succeed once PG recovers |
| **SASL mechanism mismatch** | Client switches from SCRAM to OAUTHBEARER on re-auth | `RE_AUTH_FAILED` immediately; connection closed | Client should use same mechanism as initial auth |
| **Tenant mismatch** | New credential resolves to different tenant | `RE_AUTH_FAILED`; `ILLEGAL_SASL_STATE` error to client | Client must use credential for the SNI-resolved tenant |
| **Max attempts (3) exceeded** | Client sends 3 wrong credentials | `RE_AUTH_FAILED`; connection closed with auth error | Client must reconnect; exponential backoff recommended |
| **Grace period elapsed** | Client does not respond to re-auth challenge within `gracePeriodMs` | `TIMED_OUT`; connection closed | Client must reconnect |
| **Event loop thread blocked** | Re-auth CAS in progress when blocking I/O called | Netty pipeline stalls; watchdog timer eventually closes channel | Transparent to client (TCP timeout) |
| **CredentialRevocationHandler registry leak** | `unregister()` not called on disconnect | Stale `connectionId`s accumulate; revocation triggers orphaned IDs | Harmless — `reAuthTrigger.accept()` for a closed channel is a no-op |

---

## Security Epoch Registry (Instant Revocation)

> **Related:** [SECURITY.md](SECURITY.md) §Security Epochs

The `CredentialRevocationHandler` (polling `__credentials`) handles reactive revocation but has
inherent latency (one poll interval). The **Security Epoch Registry** provides a complementary
sub-microsecond revocation check on every operation.

### Epoch Model

```java
record SecurityContext(
    TenantId tenantId,
    AuthenticatedPrincipal principal,
    long authEpoch,              // captured at auth time: max(tenantEpoch, principalEpoch)
    UUID sessionId,              // UUIDv7 per connection
    String clientIp,
    ProtocolId protocol
)
```

- On authentication (initial or re-auth): `authEpoch = max(tenantEpoch, principalEpoch)`.
- On credential change: `principalEpoch++` for that `(tenantId, username)`.
- On ACL change: `principalEpoch++` for the affected principal.
- On quota change: `principalEpoch++` for the affected principal.
- On tenant suspend/delete: `tenantEpoch++` for that `TenantId`.

### Per-Operation Check

Before every `BrokerEngine` operation (produce, fetch, subscribe, group join, etc.):

```java
void checkValid(SecurityContext ctx) {
    long currentTenantEpoch  = tenantEpochs.get(ctx.tenantId());       // ConcurrentHashMap.get
    long currentPrincipalEpoch = principalEpochs.get(
        new TenantUsername(ctx.tenantId(), ctx.principal().name()));    // ConcurrentHashMap.get
    long currentEpoch = Math.max(currentTenantEpoch, currentPrincipalEpoch);
    if (ctx.authEpoch() < currentEpoch) {
        throw new SessionRevokedException(ctx.sessionId());
    }
}
```

**Cost:** ~2–4 ns (two `ConcurrentHashMap.get()` + two `AtomicLong.get()` + one comparison).

### Epoch Propagation (3-Layer Resilience)

| Layer | Mechanism | Latency | Reliability |
|-------|-----------|---------|-------------|
| 1 | PG `LISTEN/NOTIFY` on `security_epoch` channel | <100 ms | Best-effort (may miss on PG reconnect) |
| 2 | Delta sync on PG reconnect | 1–5 ms | Reliable (replays missed epochs) |
| 3 | Full poll every 60s | 60 s worst-case | Authoritative (catches all misses) |

### PostgreSQL Tables

```sql
CREATE TABLE tenant_epochs (
    tenant_id  UUID PRIMARY KEY,
    epoch      BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE principal_epochs (
    tenant_id  UUID NOT NULL,
    username   TEXT NOT NULL,
    epoch      BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, username)
);
```

### Connection Lifecycle After Revocation

1. **Handler level:** catch `SessionRevokedException`, send protocol error, trigger re-auth (Category A)
   or graceful close (Category C).
2. **WriteWorker level:** complete all in-flight `CompletableFuture`s with exception (no PG write
   attempted).
3. **ReadAccumulator level:** return empty `ReadResult` with error flag.
4. **Optional force-close:** `ConnectionRegistry.closeByPrincipal(tenantId, username)` for
   immediate disconnection (used when `failureAction = CLOSE_CONNECTION`).

### Interaction with ReAuthManager

When `checkValid()` throws `SessionRevokedException`:

- **Category A protocols:** transition to `RE_AUTH_REQUIRED` (same as timer-based trigger).
  The client responds with in-band re-auth. On success, `authEpoch` is refreshed to current.
- **Category C protocols:** graceful close + reconnect. New connection captures fresh epoch.
- **Category D protocols:** next request fails with 401/403. Client retries with new credentials.

---

## IP-Level Auth Failure Rate Limiting

Per-connection `MAX_REAUTH_ATTEMPTS` prevents brute-force on a single connection, but an attacker
can open many connections from the same IP. **IP-level rate limiting** closes this gap.

### Algorithm

```java
class IpAuthRateLimiter {
    // Maps client IP → failure state
    private final ConcurrentHashMap<InetAddress, IpFailureState> perIpState;

    record IpFailureState(
        AtomicInteger failureCount,
        AtomicLong lockoutUntil      // epoch millis; 0 = not locked out
    ) {}

    // Constants
    static final int FREE_ATTEMPTS = 2;            // no delay for first 2 failures
    static final int LOCKOUT_THRESHOLD = 5;         // lock out after 5 failures
    static final long LOCKOUT_DURATION_MS = 300_000; // 5 minutes
    static final long MAX_DELAY_MS = 10_000;         // cap per-attempt delay
}
```

### Progressive Backoff

| Failure # | Delay | Action |
|-----------|-------|--------|
| 1–2 | 0 ms | Immediate response (free attempts) |
| 3 | 1000 ms + jitter [0, 500ms) | Delayed response |
| 4 | 2000 ms + jitter [0, 500ms) | Delayed response |
| 5+ | Rejected immediately | 300s lockout; all auth attempts from this IP return error |

**Formula:** `delay = max(0, (failures - FREE_ATTEMPTS) * 1000) + jitter`, capped at `MAX_DELAY_MS`.

### Lockout Behaviour

- Lockout applies to **all** connections from the IP (both initial auth and re-auth).
- During lockout, auth attempts return `SASL_AUTHENTICATION_FAILED` immediately (no credential lookup).
- After `LOCKOUT_DURATION_MS` elapses, failure counter resets to 0.
- Lockout state is in-memory only (lost on restart — acceptable, prevents persistent DoS).

### Timing Side-Channel Prevention

When authentication fails because the **user does not exist**, the broker must still run a
dummy SCRAM computation (or Argon2id hash) to prevent timing oracle attacks:

```java
// WRONG — timing reveals user existence
if (user == null) return AUTH_FAILED;              // fast path: ~1µs
storedKey = computeScram(password, user.salt());    // slow path: ~5ms

// RIGHT — constant-time regardless of user existence
byte[] salt = (user != null) ? user.salt() : DUMMY_SALT;
int iterations = (user != null) ? user.iterations() : DEFAULT_ITERATIONS;
byte[] computed = computeScram(password, salt, iterations);  // always runs: ~5ms
if (user == null) return AUTH_FAILED;
return MessageDigest.isEqual(computed, user.storedKey()) ? AUTH_OK : AUTH_FAILED;
```

**Rule:** Always use `MessageDigest.isEqual()` (constant-time) for credential comparison.
Never use `Arrays.equals()` or `==` on credential bytes.

---

## Inter-Broker Re-Auth Context Propagation

> **Related:** [CLUSTERING.md](CLUSTERING.md) §Inter-Broker RPC

When a non-owner broker forwards a write to the partition owner, the security context must travel
with the request. Without this, the owner broker cannot enforce epochs or revocation.

### InterBrokerSecurityContext

```java
record InterBrokerSecurityContext(
    TenantId tenantId,
    BrokerId sourceBrokerId,
    UUID incarnationId,           // source broker's incarnation (zombie detection)
    long leaderEpoch,             // partition epoch at time of forwarding
    AuthenticatedPrincipal principal,
    long authEpoch                // client's authEpoch (for epoch revocation check)
)
```

### Validation on Receiving Broker

The owner broker validates the forwarded context before processing:

1. **mTLS CN check:** the source broker's TLS client certificate CN must match `sourceBrokerId`.
2. **Incarnation check:** `incarnationId` must match the source broker's current incarnation in
   `MetadataImage`. Mismatch → reject (zombie broker).
3. **Epoch check:** `leaderEpoch` must match or exceed the current epoch for the partition.
   Stale epoch → `WrongEpochException`.
4. **Clock skew:** request timestamp must be within 30s of local clock (prevents replay).
5. **Auth epoch check:** `authEpoch` is checked against the **owner's** epoch registry.
   If revoked, the forwarded write is rejected and the source broker receives a
   `SessionRevokedException` which it propagates to the client.

### Wire Format Extension

The `ForwardWriteRequest` RPC message (see [INTERNAL_LOAD_BALANCER.md](INTERNAL_LOAD_BALANCER.md))
is extended with security context fields:

```
ForwardWriteRequest:
  [existing fields: partitionId, batchSize, hopCount, ...]
  + tenantId:       16 bytes (UUID)
  + sourceBrokerId: 16 bytes (UUID)
  + incarnationId:  16 bytes (UUID)
  + principalName:  2-byte length + UTF-8 bytes
  + authEpoch:      8 bytes (long)
  + requestTimestamp: 8 bytes (epoch millis)
```

---

## Delegation Token Re-Auth Lifecycle

> **Related:** [SECURITY.md](SECURITY.md) §Delegation Tokens

Delegation tokens allow a principal to create a short-lived token that another client can use
for authentication, without sharing the original credentials.

### Token Structure

```java
record DelegationToken(
    TenantId tenantId,
    String tokenId,              // unique identifier (UUIDv7)
    String owner,                // principal who created the token
    String requester,            // principal who requested (may differ from owner)
    Set<String> renewers,        // principals allowed to renew
    long issueTimestamp,
    long maxTimestamp,            // absolute max lifetime (default 7 days)
    long expiryTimestamp,         // current expiry (renewable up to maxTimestamp)
    byte[] hmacKey               // HMAC-SHA-256 key (32 bytes, SecureRandom)
)
```

### SASL/DELEGATION_TOKEN Authentication

```
C→S: SaslHandshake(mechanism="DELEGATION_TOKEN")
S→C: SaslHandshakeResponse(mechanisms=[...])
C→S: SaslAuthenticate(tokenId + ":" + HMAC-SHA-256(hmacKey, tokenId))
S→C: SaslAuthenticate(errorCode=0, sessionLifetimeMs=<remaining>)
```

### Re-Auth with Delegation Tokens

When a delegation token is used for KIP-368 re-auth:

1. Client sends `SaslHandshake(DELEGATION_TOKEN)` on existing connection.
2. Broker validates: token exists, not expired, HMAC matches, tenant matches.
3. On success: `sessionLifetimeMs = min(expiryTimestamp - now, maxReauthMs)`.
4. Principal is set to the token **owner** (not the requester).

### Lifecycle APIs (Kafka Wire)

| API | Key | Description |
|-----|-----|-------------|
| `CreateDelegationToken` (38) | — | Create token; returns tokenId + hmacKey |
| `RenewDelegationToken` (39) | tokenId | Extend expiry (cannot exceed maxTimestamp) |
| `ExpireDelegationToken` (40) | tokenId | Set expiry to now (immediate invalidation) |
| `DescribeDelegationToken` (41) | owner filter | List tokens for principal(s) |

### Token Revocation

Expiring/deleting a token increments `principalEpoch` for the token owner, which triggers
`SessionRevokedException` on all connections authenticated with that token (via Security Epoch
Registry). The affected connections enter `RE_AUTH_REQUIRED` state.

---

## Audit Events for Re-Auth

> **Related:** [AUDIT_LOGGING.md](AUDIT_LOGGING.md)

Every re-auth attempt (success or failure) emits a structured audit event.

### Event Structure

```json
{
  "eventType": "RE_AUTHENTICATE",
  "timestamp": "2026-03-25T14:32:01.123Z",
  "tenantId": "550e8400-e29b-41d4-a716-446655440000",
  "principal": "alice",
  "newPrincipal": "alice",
  "clientIp": "10.0.1.42",
  "protocol": "KAFKA",
  "mechanism": "OAUTHBEARER",
  "sessionId": "01913e4b-7c8a-7000-8000-000000000001",
  "connectionId": "ch-1234",
  "result": "SUCCESS",
  "previousAuthEpoch": 41,
  "newAuthEpoch": 42,
  "sessionLifetimeMs": 3600000,
  "reAuthReason": "TOKEN_EXPIRY",
  "durationMs": 12,
  "errorCode": null,
  "errorMessage": null
}
```

### Event Fields

| Field | Type | Description |
|-------|------|-------------|
| `reAuthReason` | enum | `TOKEN_EXPIRY`, `CREDENTIAL_REVOKED`, `SESSION_LIFETIME`, `TENANT_REVOKED`, `MANUAL` |
| `result` | enum | `SUCCESS`, `FAILED_CREDENTIAL`, `FAILED_TENANT_MISMATCH`, `FAILED_MECHANISM_MISMATCH`, `FAILED_MAX_ATTEMPTS`, `TIMED_OUT` |
| `mechanism` | text | `SCRAM-SHA-256`, `SCRAM-SHA-512`, `OAUTHBEARER`, `DELEGATION_TOKEN`, `PLAIN`, `mTLS` |
| `durationMs` | long | Time from `RE_AUTH_REQUIRED` → terminal state (latency of re-auth handshake) |

### Masking

Credential fields are **never** included in audit events. The `mechanism` identifies how auth
was performed; the actual token/password/certificate is not logged (Rule R25).

---

## SASL Mechanism Ordering and Anti-Downgrade

### Mechanism Registry

The broker advertises mechanisms in strength order and enforces anti-downgrade rules:

```java
enum SaslMechanismStrength {
    SCRAM_SHA_512(4),    // strongest
    SCRAM_SHA_256(3),
    OAUTHBEARER(2),
    DELEGATION_TOKEN(2),
    PLAIN(1);            // weakest (TLS required)

    final int strength;
}
```

### Ordering Rules

1. `SaslHandshakeResponse.mechanisms` lists mechanisms in descending strength order.
2. **No downgrade on re-auth:** the mechanism used for re-auth must have strength ≥ the initial
   auth mechanism. If the client attempts `PLAIN` after initially authenticating with `SCRAM-SHA-512`,
   the broker rejects with `ILLEGAL_SASL_STATE`.
3. **`PLAIN` requires TLS:** if the connection is not TLS-encrypted, `PLAIN` is removed from the
   advertised mechanisms list. Attempting `PLAIN` without TLS → `UNSUPPORTED_SASL_MECHANISM`.
4. **Mechanism immutability for MQTT 5.0:** the `authMethod` in `AUTH` packets must match
   the `authMethod` in the original `CONNECT`. Changing mechanism is not allowed.

---

## Quota Re-Evaluation on Re-Auth

When re-auth succeeds and the principal changes (e.g., different service account within same tenant),
quota buckets must be re-evaluated.

### Quota Transition Flow

```
Re-auth success (principal A → principal B, same tenant):
  1. ACL cache invalidated (existing Rule R28)
  2. Quota bucket for principal A: decrement connection count
  3. Quota bucket for principal B: increment connection count
  4. Rate limiter re-bound:
     - old: TokenBucket keyed by (tenantId, "A", clientId, quotaType)
     - new: TokenBucket keyed by (tenantId, "B", clientId, quotaType)
  5. Connection quota checked: if principal B exceeds max connections → RE_AUTH_FAILED
```

### Same-Principal Re-Auth

When the principal does NOT change (most common case — same user refreshing JWT):

1. ACL cache invalidated (R28).
2. Quota config re-read (in case admin changed quota limits).
3. Rate limiter tokens NOT reset (prevents abuse: re-auth to refill quota).
4. Connection count unchanged.

---

## Cross-Protocol ACL Invalidation on Re-Auth

> **Related:** [ACL_DESIGN.md](ACL_DESIGN.md) §Protocol-Scoped ACLs

When a Kafka producer re-auths and its ACLs change, other protocols may be affected if the
same principal has cross-protocol sessions.

### Invalidation Scope

ACL cache invalidation on re-auth is scoped to the **principal across all protocols**, not just
the re-authenticating connection's protocol:

```java
// On successful re-auth:
aclAuthorizer.invalidateByPrincipal(tenantId, principal);
// This clears Caffeine cache entries for ALL protocol combinations:
//   (tenantId, principal, KAFKA, *)
//   (tenantId, principal, MQTT, *)
//   (tenantId, principal, AMQP, *)
//   etc.
```

### Why Cross-Protocol Invalidation

An admin may revoke a principal's MQTT SUBSCRIBE permission while the principal is re-authing
on a Kafka connection. Without cross-protocol invalidation, the stale MQTT ACL cache entry
would persist until its TTL expires (up to 5 minutes).

### Push Notification to Other Connections

Beyond cache invalidation, the `CredentialRevocationHandler` can push re-auth triggers to
**all connections** for the affected principal (not just the one that re-authenticated):

```java
// After successful re-auth that changed ACLs:
if (aclsChanged) {
    credentialRevocationHandler.triggerReAuthForPrincipal(tenantId, principal);
    // Sends RE_AUTH_REQUIRED to all other connections for this principal
}
```

This ensures MQTT subscribers and AMQP consumers pick up permission changes promptly.

---

## CRL/OCSP Certificate Revocation Checking (Category E Extension)

> **Related:** [CERT_MANAGEMENT.md](CERT_MANAGEMENT.md)

Category E covers certificate expiry, but a certificate may be **revoked** before expiry
(compromised private key, employee departure, etc.).

### CRL (Certificate Revocation List) Checking

```java
record CrlConfig(
    boolean enabled,                   // default: false
    Duration refreshInterval,          // default: 1 hour
    Duration cacheMaxAge,              // default: 24 hours
    CrlFailurePolicy failurePolicy     // SOFT_FAIL or HARD_FAIL
)
```

**SOFT_FAIL (default):** If the CRL cannot be fetched (network error, timeout), the certificate
is **accepted**. This prevents a CRL distribution point outage from causing a total auth failure.

**HARD_FAIL:** If the CRL cannot be fetched, the certificate is **rejected**. Use when
security requirements outweigh availability (e.g., financial services).

### OCSP Stapling

```java
record OcspConfig(
    boolean enabled,                   // default: false
    boolean staplingRequired,          // default: false (if true, reject if no staple)
    Duration responseMaxAge,           // default: 24 hours
    OcspFailurePolicy failurePolicy    // SOFT_FAIL or HARD_FAIL
)
```

OCSP stapling allows the TLS server to include a pre-fetched OCSP response in the TLS handshake,
eliminating the client's need to contact the OCSP responder directly. Configured per-tenant
in `TlsTenantConfig`.

### Integration with Re-Auth

For mTLS Category E connections, CRL/OCSP checks occur:
1. At initial TLS handshake (connection establishment).
2. On CRL refresh (background thread checks if any connected cert's serial appears in new CRL).
3. If a connected certificate is found in a CRL update → immediate graceful close (same as
   Category C signals).

---

## `DRAIN` Failure Action

In addition to `CLOSE_CONNECTION` and `CONTINUE_READ_ONLY`, a third failure action is available:

### Configuration

```yaml
broker:
  reauth:
    failureAction: DRAIN    # CLOSE_CONNECTION | CONTINUE_READ_ONLY | DRAIN
    drainTimeoutMs: 30000   # max time for DRAIN before forced close (default: 30s)
```

### Behaviour

When `failureAction = DRAIN` and re-auth fails:

1. The connection enters `DRAINING` state (new sub-state of `RE_AUTH_FAILED`).
2. **No new requests accepted** — any new Produce/Fetch/Subscribe returns `RE_AUTHENTICATION_REQUIRED`.
3. **In-flight requests complete normally** — outstanding PG transactions, pending fetches, and
   consumer group commits finish.
4. **Push deliveries stop** — no new messages pushed to MQTT/AMQP subscribers.
5. After all in-flight work completes (or `drainTimeoutMs` elapses), the connection is closed.

### Use Case

`DRAIN` is useful for exactly-once producers where an abrupt close could leave a transaction
in `ONGOING` state (requiring timeout-based abort). The drain period allows the producer to
complete its current transaction before the connection is severed.

---

## Known Limitations

| # | Limitation | Impact | Status |
|---|-----------|--------|--------|
| 1 | **`CONTINUE_READ_ONLY` failure action not implemented** — all failures close immediately | In-flight requests on failed connections are dropped | OPEN |
| 2 | **Grace period not enforced at timer level** — `gracePeriodMs` config exists but no timer fires after `IN_PROGRESS` timeout | Client could delay re-auth indefinitely while in `IN_PROGRESS` | OPEN — needs a second timer: `gracePeriodMs` from `startReAuth()` |
| 3 | **MQTT 5.0 multi-step SCRAM not wired** — only single-round mechanisms work for MQTT re-auth | SCRAM-SHA-256 AUTH packet exchange not handled | OPEN |
| 4 | **ReAuthScheduler not wired in server bootstrap** — each protocol handler schedules its own timer | No centralized timer; MAX_TIMERS cap not enforced globally | OPEN |
| 5 | **CredentialRevocationHandler not wired to admin API** — reactive push disabled | Credential revocation does not close active connections | OPEN — only timer-based re-auth works |
| 6 | **Kafka session lifetime computation** — `sessionLifetimeMs` must be computed from `min(maxReauthMs, tokenExp - now - buffer)`; currently correct in design but ensure no hardcoding in implementation | Incorrect lifetime breaks re-auth scheduling | VERIFY in implementation |
| 7 | **AMQP 0-9-1 `Connection.UpdateSecret` not implemented** | OAuth2 RabbitMQ clients cannot refresh in-band; must reconnect | WONTFIX — not part of AMQP 0-9-1 standard |
| 8 | **AMQP 1.0 SASL renegotiation not implemented** | AMQP 1.0 clients must reconnect | WONTFIX — spec does not define mid-session SASL |
| 9 | **ACL cache invalidation callback** — must be wired to actual `AclAuthorizer.invalidateByPrincipal()`, not a no-op lambda | New ACL rules not visible after re-auth until cache TTL expires | OPEN — wire real invalidator when AclAuthorizer is available |
| 10 | **Redis handler must use `ReAuthManager`** — bare failure counter is inconsistent with architecture | Redis max-attempts not CAS-safe; tenant immutability not checked | OPEN — refactor to use `ReAuthManager` |
| 11 | **No re-auth metrics export** — counters exist but not connected to metrics endpoint | No visibility into re-auth activity | OPEN |
| 12 | **Delegation token re-auth** — delegation tokens can be used for Kafka KIP-368 re-auth but the renewal flow (create new token, use it for re-auth) is not explicitly tested | Token-based re-auth may have edge cases | OPEN |

---

## Key Design Decisions

### D1: Per-Connection State Machine vs Centralized

**Chosen:** Per-connection `ReAuthManager` with `AtomicReference<ReAuthState>`.

**Rejected:** Centralized session store (e.g., `ConcurrentHashMap<ConnectionId, ReAuthState>`).

**Rationale:** At 100K concurrent connections, a centralized store becomes a contention point.
Per-connection state is naturally partitioned — no connection shares re-auth state with another.
The only cross-connection operation is revocation lookup in `CredentialRevocationHandler`,
which uses a read-heavy `CopyOnWriteArraySet` per principal (small sets, few writes).

### D2: CAS-Based Transitions (No Locks)

**Chosen:** `AtomicReference.compareAndSet()` for all state transitions.

**Rejected:** `ReentrantLock` or `synchronized`.

**Rationale:** Re-auth handlers run on Netty's event loop threads, which must never block.
CAS is non-blocking and correct for single-connection transitions; only one thread at a time
can win a transition (the loser gets an immediate failure response).

### D3: Idle Connection Preservation (KIP-368 Semantics)

**Chosen:** Only enforce session expiry when the client next sends a request.

**Rejected:** Proactive forced close at session expiry.

**Rationale:** Proactive close breaks Kafka producers that are legitimately idle (e.g., batching
builds up). KIP-368 specifies that idle connections are NOT forcibly closed. The latency cost of
re-auth on the first request after expiry is acceptable (~50ms for SCRAM).

### D4: `TenantUsername` Composite Key in Revocation Registry

**Chosen:** Registry key = `(TenantId, username)` composite.

**Rejected:** `username` alone as key.

**Rationale:** Same username (`alice`) can exist in multiple tenants. A bare username key would
cause revocation of `acme/alice` to also close `beta/alice`'s connections — a cross-tenant
isolation breach. The composite key prevents this.

### D5: Jitter on Re-Auth Timers

**Chosen:** `[0, 5000ms)` random jitter subtracted from the fire time.

**Rejected:** No jitter; exact `sessionLifetimeMs - buffer` delay.

**Rationale:** Without jitter, all connections authenticated within the same second (e.g., after
a broker restart that re-onboards 10K clients) would fire re-auth at the exact same instant,
creating a thundering herd against the credential store. 5s jitter spreads the load naturally.

---

## Test Strategy

### Unit Tests

| Class | Test Focus | Key Cases |
|-------|-----------|-----------|
| `ReAuthManagerTest` | State machine transitions | CAS transitions, tenant mismatch → FAILED, max attempts, terminal state immutability, concurrent CAS race (only 1 winner) |
| `ReAuthSchedulerTest` | Timer scheduling | Jitter distribution, MAX_TIMERS capacity, cancel-on-disconnect, cert expiry scheduling, disabled config |
| `CredentialRevocationHandlerTest` | Revocation registry | Tenant isolation (same username, different tenants), wildcard revocation, concurrent register/revoke, empty-set cleanup |
| `KafkaReAuthHandlerTest` | KIP-368 wire format | Mechanism mismatch, tenant mismatch, SCRAM 4-message exchange, OAUTHBEARER single-round, `sessionLifetimeMs` computation |
| `MqttReAuthHandlerTest` | AUTH packet | MQTT 5.0 only (3.1.1 rejected), authMethod immutability, reason codes |
| `CategoryBCReAuthTest` | MySQL/Redis/graceful-close | COM_CHANGE_USER session reset, Redis pub/sub AUTH blocked in RESP2, NATS error string, STOMP ERROR frame |
| `SecurityEpochTest` | Epoch revocation | Credential change → epoch increment → next operation rejected, tenant suspend → all operations rejected, epoch propagation via LISTEN/NOTIFY |
| `IpRateLimiterTest` | IP-level auth rate limiting | Progressive backoff timing, lockout at 5 failures, lockout expiry after 300s, timing side-channel (constant-time on unknown user) |
| `MechanismOrderingTest` | Anti-downgrade | Downgrade attempt rejected, PLAIN without TLS rejected, MQTT authMethod immutability |
| `DelegationTokenReAuthTest` | Token re-auth | Token used for KIP-368 re-auth, expired token rejected, revoked token triggers epoch |
| `QuotaReEvaluationTest` | Quota on principal change | Principal change → new quota bucket, same principal → config re-read, connection quota checked |

### Integration Tests

| Test | Scenario |
|------|---------|
| `KafkaReAuthIT` | Real Kafka producer holds connection through 3 OAUTHBEARER renewals; verifies no message loss |
| `KafkaScramReAuthIT` | SCRAM re-auth mid-batch produce; verifies `producerId` preserved |
| `MqttReAuthIT` | MQTT 5.0 client re-auths via AUTH packet; subscription preserved |
| `CredentialRevokeIT` | Admin revokes credential; all matching connections closed within 5s |
| `TenantMismatchIT` | Re-auth with credential for wrong tenant → connection closed, error logged |
| `MaxAttemptsIT` | 4 consecutive wrong credentials → `RE_AUTH_FAILED` after 3rd, connection closed |
| `SecurityEpochIT` | Credential revocation via epoch → active producer gets SessionRevokedException within 100ms |
| `InterBrokerReAuthIT` | Forwarded write with revoked authEpoch rejected by owner broker |
| `DelegationTokenLifecycleIT` | Create → authenticate → renew → expire → re-auth fails |
| `CrossProtocolAclIT` | Kafka re-auth with ACL change → MQTT subscriber ACL cache invalidated |
| `IpLockoutIT` | 5 failed auths from same IP → 300s lockout → all new connections from IP rejected |

### E2E Tests

| Test | Scenario |
|------|---------|
| `LongRunningProducerE2E` | Kafka producer runs for 3+ hours with hourly JWT rotation; zero dropped messages |
| `CredentialRotationE2E` | SCRAM password rotated during active produce; grace period allows seamless transition |
| `CategoryCReconnectE2E` | AMQP client reconnects after graceful close; consumer group rejoins cleanly |
| `EpochRevocationE2E` | Admin deletes user → epoch increments → all connections for that user close within 200ms across cluster |
| `DrainFailureActionE2E` | Re-auth fails with `DRAIN` action → in-flight transaction completes → connection closes cleanly |

---

*Last updated: 2026-03-25*
*See also: [MULTI_TENANT.md](MULTI_TENANT.md) §TenantContext, [RULES.md](RULES.md) §R27–R36,
[SECURITY.md](SECURITY.md), [ACL_DESIGN.md](ACL_DESIGN.md), [AUDIT_LOGGING.md](AUDIT_LOGGING.md),
[CERT_MANAGEMENT.md](CERT_MANAGEMENT.md)*
