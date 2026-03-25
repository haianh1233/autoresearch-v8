# Security Architecture

> **Related:** [RE_AUTH.md](RE_AUTH.md), [MULTI_TENANT.md](MULTI_TENANT.md), [ACL_DESIGN.md](ACL_DESIGN.md),
> [CERT_MANAGEMENT.md](CERT_MANAGEMENT.md), [AUDIT_LOGGING.md](AUDIT_LOGGING.md), [RULES.md](RULES.md) §R24–R36

---

## 8-Layer Security Pipeline

Every client connection passes through these layers in order:

```
Layer 0 ─ Connection Metadata     ivy-server    Passive logging: source IP, port, timestamp.
                                                 NEVER rejects.

Layer 1 ─ TLS Termination         ivy-server    Mandatory TLS 1.2+ for external ports.
                                                 Per-tenant SslContext via SNI (hot-reload).
                                                 Cipher enforcement (see §Cipher Suites).

Layer 2 ─ Tenant Resolution       ivy-server    SNI hostname → TenantId (deterministic UUID).
                                                 Connection rejected if tenant SUSPENDED/DELETED.

Layer 3 ─ Connection Quota        ivy-auth      Reject if tenant exceeds max connections.
                                                 Per-IP rate limiting (see RE_AUTH.md §IP-Level).

Layer 4 ─ Protocol Authentication ivy-codec     SASL handshake (SCRAM/OAUTHBEARER/PLAIN/DELEGATION_TOKEN).
                                                 mTLS client cert → CN extraction.
                                                 Protocol-native framing (Kafka, MQTT, AMQP, etc.).

Layer 5 ─ Identity Mapping        ivy-auth      PrincipalResolverChain: raw auth → ResolvedIdentity.
                                                 Group enrichment, user mapping, passthrough.

Layer 6 ─ ACL Authorization       ivy-auth      DENY-first evaluation per request.
                                                 Protocol-scoped (Kafka ACLs ≠ MQTT ACLs).
                                                 Caffeine-cached, invalidated on re-auth.
                                                 See ACL_DESIGN.md.

Layer 7 ─ Request Quota           ivy-auth      Token-bucket rate limiting per (tenant, principal).
                                                 Produce/consume byte rate, request rate,
                                                 connection creation rate.

Layer 8 ─ Audit Logging           ivy-auth      Structured JSON events to PG audit_log table.
                                                 See AUDIT_LOGGING.md.
```

---

## Authentication Mechanisms

### SCRAM-SHA-256 / SCRAM-SHA-512 (Primary)

Implementation follows RFC 5802 (SCRAM) and RFC 7677 (SCRAM-SHA-256).

**Key Derivation:**
```
SaltedPassword  = PBKDF2(password, salt, iterations, keyLength)
ClientKey       = HMAC(SaltedPassword, "Client Key")
StoredKey       = SHA-256(ClientKey)     ← stored in credentials table
ServerKey       = HMAC(SaltedPassword, "Server Key")  ← stored in credentials table
```

**Storage:** Only `StoredKey` and `ServerKey` are stored (plus `salt` and `iterations`).
The plaintext password and `ClientKey` are **never** stored or logged (Rule R25).

**Minimum iterations:** 4096 (configurable per-tenant; NIST SP 800-132 minimum).

**4-Message Exchange:**
```
C→S: client-first-message  (username, nonce)
S→C: server-first-message  (salt, iterations, combined-nonce)
C→S: client-final-message  (channel-binding, proof = ClientKey XOR HMAC(StoredKey, AuthMessage))
S→C: server-final-message  (server-signature = HMAC(ServerKey, AuthMessage))
```

**Constant-Time Comparison:** All credential comparisons use `MessageDigest.isEqual()` (Rule R31).
Never `Arrays.equals()`, `String.equals()`, or `==` on credential bytes.

**Timing Side-Channel Prevention:** When the username does not exist, the broker still runs
PBKDF2 with a deterministic dummy salt to prevent timing oracle attacks (Rule R32).

### SASL/OAUTHBEARER (JWT)

Single-round authentication. Client sends a JWT; broker validates.

**8-Step Validation Pipeline:**

1. **Signature verification** — RS256/RS384/RS512 (RSA) or ES256/ES384 (ECDSA) only.
   `alg=none` and all symmetric HMAC algorithms (HS256/HS384/HS512) are **rejected**
   (CVE-2015-9235 mitigation). Algorithm must match key type.
2. **Issuer (`iss`)** — must match tenant's configured allowed issuers list.
3. **Audience (`aud`)** — broker's audience string must appear in the `aud` claim.
4. **Expiration (`exp`)** — `exp` must be > `now - clockSkewTolerance` (default 5s).
5. **Not-before (`nbf`)** — `nbf` (optional) must be ≤ `now + clockSkewTolerance`.
6. **Scope extraction** — `scope` or `scp` claim mapped to permission set.
7. **Principal extraction** — from configurable claim (default `sub`).
8. **Tenant validation** — tenant from configurable claim must match connection's `TenantId`.

**JWKS caching:** Public keys cached 1 hour with background refresh. Synchronous refresh
on cache miss (first request for a new key ID).

### SASL/PLAIN

Single-round: client sends `\0username\0password` in `SaslAuthenticate`.

**Constraints:**
- **TLS mandatory** (Rule R26). `PLAIN` is removed from advertised mechanisms on non-TLS connections.
- Delegates to `ScramAuthenticationProvider` — password verified against SCRAM stored keys,
  not a separate plaintext store.

### Delegation Tokens (HMAC-SHA-256)

See [RE_AUTH.md](RE_AUTH.md) §Delegation Token Re-Auth Lifecycle.

- Token ID + HMAC proof sent in `SaslAuthenticate`.
- Principal is set to the token **owner**.
- Token lifecycle: Create → Renew → Expire → Describe.

### mTLS / EXTERNAL

Client presents X.509 certificate during TLS handshake.

- `clientAuth=REQUIRE` on the TLS listener.
- Principal extracted from certificate CN (or configurable SAN field).
- No SASL exchange; identity derived from the certificate chain.
- See [CERT_MANAGEMENT.md](CERT_MANAGEMENT.md) for validation, CRL/OCSP, and hot-reload.

### ANONYMOUS

Used for pre-auth operations only (SASL handshake, API version negotiation).
`SecurityContext.ANONYMOUS` with `TenantId.SYSTEM`. Cannot access any user data.

---

## Auth Failure Tracking

### Per-Connection

- `MAX_REAUTH_ATTEMPTS = 3` per connection (configurable).
- After max attempts → `RE_AUTH_FAILED` → connection closed.

### Per-IP Rate Limiting

```
Failures 1–2:  no delay (free attempts)
Failures 3–4:  progressive delay: (N-2) * 1000ms + jitter [0, 500ms), capped 10s
Failure 5+:    300-second lockout; all auth attempts from IP return error immediately
```

Lockout is in-memory (lost on restart). See [RE_AUTH.md](RE_AUTH.md) §IP-Level Auth Failure Rate Limiting.

---

## Security Epochs (Instant Revocation)

Versioned epoch counters enable sub-microsecond revocation checks on every operation.
See [RE_AUTH.md](RE_AUTH.md) §Security Epoch Registry for full design.

**Summary:**
- `authEpoch = max(tenantEpoch, principalEpoch)` captured at auth time.
- Checked before every `BrokerEngine` operation (~2–4 ns).
- Credential/ACL/quota change → epoch increment → immediate rejection.
- Propagated via PG LISTEN/NOTIFY + delta sync + polling fallback.

---

## Credential Storage

| Auth Method | Stored Form | Never Stored |
|-------------|-------------|--------------|
| SCRAM-SHA-256 | salt, StoredKey, ServerKey, iterations | password, ClientKey |
| SCRAM-SHA-512 | salt, StoredKey, ServerKey, iterations | password, ClientKey |
| OAUTHBEARER | issuer, audience, JWKS URI (per-tenant config) | token value |
| mTLS | trusted CA cert per tenant | private key |
| Delegation Token | tokenId, owner, hmacKey, expiry | — |
| PLAIN | delegates to SCRAM stored keys | password |

All credential types are stored in the `credentials` table, scoped by `(tenant_id, username)`.
Rotation grace period (dual-credential window) described in [RE_AUTH.md](RE_AUTH.md) §SCRAM Credential Rotation.

---

## Cipher Suite Enforcement

### TLS 1.3 (Preferred)

```
TLS_AES_256_GCM_SHA384
TLS_AES_128_GCM_SHA256
TLS_CHACHA20_POLY1305_SHA256
```

### TLS 1.2 (Minimum)

Only ECDHE key exchange + AES-GCM cipher:
```
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
```

**Rejected:** TLS 1.0, TLS 1.1, all CBC ciphers, all RC4, all DES, all NULL, all EXPORT,
all static RSA key exchange (no forward secrecy).

### Inter-Broker TLS

- Mandatory mTLS on inter-broker port (default 9094).
- Cluster CA signing: all broker certificates signed by a shared CA.
- CN must match `brokerId` for identity verification.
- Same cipher suite restrictions as client-facing TLS.

---

## PrincipalResolverChain (Identity Mapping)

After raw authentication, the principal is mapped through a resolver chain:

```java
sealed interface PrincipalResolver {
    Optional<ResolvedIdentity> resolve(AuthenticatedPrincipal raw, TenantContext ctx);
}
```

**Built-in Resolvers (evaluated in order):**

| Resolver | Description |
|----------|-------------|
| `UserMappingResolver` | Configurable IDP-to-local-name mapping (e.g., `CN=alice@corp.com` → `alice`) |
| `GroupEnrichmentResolver` | Add/remove groups based on rules (e.g., `*@admin.corp.com` → group `admins`) |
| `JwtClaimResolver` | Extract principal and groups from JWT claims (configurable claim names) |
| `MtlsSanResolver` | Extract principal from certificate SAN fields |
| `PassthroughResolver` | Use auth principal as-is (default fallback) |

**Output:**
```java
record ResolvedIdentity(
    String username,
    Set<String> groups,
    Map<String, String> attributes,
    TenantId tenantId
)
```

---

## Token Bucket Quota Enforcement

### Quota Types

| Type | Unit | Default |
|------|------|---------|
| `PRODUCER_BYTE_RATE` | bytes/sec | Long.MAX_VALUE (unlimited) |
| `CONSUMER_BYTE_RATE` | bytes/sec | Long.MAX_VALUE |
| `REQUEST_RATE` | requests/sec | Long.MAX_VALUE |
| `CONNECTION_CREATION_RATE` | connections/sec | Long.MAX_VALUE |

### Resolution Hierarchy (4 levels, most specific wins)

1. `(tenant, user, clientId, type)` — per-user per-client
2. `(tenant, user, <default>, type)` — per-user
3. `(tenant, <default>, <default>, type)` — per-tenant
4. Built-in default → `Long.MAX_VALUE` (unlimited)

### Multi-Broker Coordination

In cluster mode, quotas are statically split: `effectiveQuota = configuredQuota / activeBrokerCount`.
Periodic sync every 10s via PG reconciles actual usage across brokers.

### Enforcement

- Produce/Fetch responses include `throttle_time_ms` header when quota is exceeded.
- Kafka clients respect this natively (sleep before next request).
- Non-Kafka protocols: broker delays response by `throttle_time_ms`.

---

## Module Placement

| Component | Module | Responsibility |
|-----------|--------|---------------|
| `DefaultAuthEngine` | `ivy-auth` | Authentication orchestration |
| `ScramAuthenticator` | `ivy-auth` | SCRAM-SHA-256/512 challenge/response |
| `JwtValidator` | `ivy-auth` | OAUTHBEARER 8-step pipeline |
| `DelegationTokenManager` | `ivy-auth` | Token lifecycle (create/renew/expire) |
| `AclAuthorizer` | `ivy-auth` | DENY-first ACL evaluation (see ACL_DESIGN.md) |
| `TokenBucketQuotaManager` | `ivy-auth` | Per-tenant/principal rate limiting |
| `PrincipalResolverChain` | `ivy-auth` | Identity mapping |
| `SecurityEpochRegistry` | `ivy-auth` | Epoch-based instant revocation |
| `IpAuthRateLimiter` | `ivy-auth` | Per-IP failure tracking and lockout |
| `AuditWriter` | `ivy-auth` | Structured audit event emission |
| `ReAuthManager` | `ivy-auth` | Per-connection re-auth state machine |
| `ReAuthScheduler` | `ivy-auth` | Timer management for session lifetime |
| `CredentialRevocationHandler` | `ivy-auth` | Registry + reactive revocation push |
| `SecurityContext` | `ivy-common` | Value record carried on every operation |
| `SslContextPool` | `ivy-server` | Per-tenant TLS context with hot-reload |
| `TenantResolverHandler` | `ivy-server` | SNI → TenantId in Netty pipeline |

**Key principle:** `ivy-broker` has ZERO security awareness. All auth, ACL, quota, and audit
decisions are made in `ivy-auth` / `ivy-server` before requests reach the broker engine.

---

## Configuration

```yaml
security:
  # SASL mechanisms (advertised in strength order; strongest first)
  sasl:
    mechanisms: [SCRAM-SHA-512, SCRAM-SHA-256, OAUTHBEARER, PLAIN]
    scram:
      min-iterations: 4096            # PBKDF2 minimum (NIST SP 800-132)
      default-iterations: 8192
    oauthbearer:
      clock-skew-tolerance-ms: 5000   # JWT clock skew tolerance
      jwks-cache-duration-ms: 3600000 # 1 hour JWKS cache
    plain:
      require-tls: true               # PLAIN disabled on non-TLS (always true in production)

  # IP-level rate limiting
  ip-rate-limit:
    free-attempts: 2
    lockout-threshold: 5
    lockout-duration-ms: 300000       # 5 minutes
    max-delay-ms: 10000

  # Security epochs
  epochs:
    poll-interval-ms: 60000           # full epoch poll fallback
    listen-notify-channel: security_epoch

  # TLS (see CERT_MANAGEMENT.md for full config)
  tls:
    min-protocol: TLSv1.2
    inter-broker:
      require-mtls: true
      cluster-ca-path: /etc/ivy/certs/cluster-ca.pem
```

Per-tenant overrides via `tenants.config.auth` JSONB column. See [MULTI_TENANT.md](MULTI_TENANT.md) §AuthTenantConfig.

---

*Last updated: 2026-03-25*
*See also: [RE_AUTH.md](RE_AUTH.md), [ACL_DESIGN.md](ACL_DESIGN.md), [CERT_MANAGEMENT.md](CERT_MANAGEMENT.md),
[AUDIT_LOGGING.md](AUDIT_LOGGING.md), [THREAT_MODEL.md](THREAT_MODEL.md)*
