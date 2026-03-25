# Re-Authentication Design

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
