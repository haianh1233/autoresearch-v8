# Certificate Management

> **Related:** [SECURITY.md](SECURITY.md) §TLS, [RE_AUTH.md](RE_AUTH.md) §Category E,
> [MULTI_TENANT.md](MULTI_TENANT.md) §TlsTenantConfig

---

## Overview

Ivy uses TLS for all external connections (Rule R26). Certificate management covers per-tenant
TLS contexts, hot-reload, mTLS client authentication, certificate validation, CRL/OCSP
revocation checking, and inter-broker mutual TLS.

---

## Per-Tenant SslContext

Each tenant has its own `SslContext` loaded from tenant-specific certificates:

```java
class SslContextPool {
    // tenantId → current SslContext (swapped atomically on reload)
    private final ConcurrentHashMap<TenantId, AtomicReference<SslContext>> pool;
}
```

### SNI Resolution (Chicken-and-Egg Problem)

The broker needs the `TenantId` to select the correct `SslContext`, but `TenantId` is derived
from the SNI hostname in the TLS `ClientHello` — which arrives **before** the handshake completes.

**Solution:** `IvySniHandler` extends Netty's `SniHandler` with a `Mapping<String, SslContext>`:

```java
class IvySniHandler extends SniHandler {
    IvySniHandler(SslContextPool pool, TenantRegistry registry) {
        super((hostname) -> {
            TenantId tenantId = registry.resolveByHostname(hostname);
            return pool.getOrDefault(tenantId);  // returns tenant-specific or default SslContext
        });
    }
}
```

The SNI hostname is captured during the handshake and stored in the channel attribute for
subsequent tenant resolution in Layer 2.

### Hot-Reload

```java
// Background thread checks for certificate file changes
class CertificateWatcher {
    private final Duration checkInterval = Duration.ofSeconds(300); // 5 min default

    void onCertificateChanged(TenantId tenantId, Path certPath, Path keyPath) {
        SslContext newCtx = buildSslContext(certPath, keyPath);
        AtomicReference<SslContext> ref = pool.get(tenantId);
        SslContext old = ref.getAndSet(newCtx);          // atomic swap
        ReferenceCountUtil.safeRelease(old);              // OpenSSL reference counting
    }
}
```

**Key:** `ReferenceCountUtil.safeRelease()` is called on the old `SslContext` to prevent
native memory leaks when using Netty's OpenSSL provider (`SslProvider.OPENSSL`).

---

## mTLS Client Authentication

### Modes

| Mode | `clientAuth` | Behaviour |
|------|-------------|-----------|
| `NONE` | `ClientAuth.NONE` | No client cert requested (default) |
| `OPTIONAL` | `ClientAuth.OPTIONAL` | Client cert requested but not required; if presented, validated |
| `REQUIRE` | `ClientAuth.REQUIRE` | Client cert mandatory; handshake fails without one |

Configured per-tenant in `TlsTenantConfig.mtlsMode`.

### Certificate Validation (6 Checks)

When a client presents a certificate, the broker validates:

1. **Chain trust:** Certificate chain must terminate at a trusted CA in the tenant's trust store.
2. **Expiry:** `notBefore ≤ now ≤ notAfter`. Expired certificates are rejected.
3. **Basic constraints:** If the certificate has `BasicConstraints` extension, `cA` must be `false`
   (leaf certificates only; intermediate CAs are not accepted as client certs).
4. **Key usage:** If `KeyUsage` extension is present, `digitalSignature` bit must be set.
5. **Extended Key Usage (EKU):** If `ExtendedKeyUsage` extension is present, `id-kp-clientAuth`
   (OID 1.3.6.1.5.5.7.3.2) must be included.
6. **CRL/OCSP revocation check** (if enabled — see below).

### Principal Extraction

The authenticated principal is extracted from the certificate (configurable):

```java
enum CertificatePrincipalSource {
    CN,           // Common Name from Subject DN (default)
    SAN_DNS,      // First DNS-type Subject Alternative Name
    SAN_EMAIL,    // First email-type SAN
    SAN_URI,      // First URI-type SAN
    FULL_DN       // Full Subject Distinguished Name
}
```

---

## CRL Checking (Certificate Revocation List)

### Configuration

```yaml
security:
  tls:
    crl:
      enabled: false                   # default: disabled
      refresh-interval-ms: 3600000     # 1 hour
      cache-max-age-ms: 86400000       # 24 hours
      failure-policy: SOFT_FAIL        # SOFT_FAIL or HARD_FAIL
      distribution-points: []          # override CRL URLs (empty = use cert's CDP extension)
```

### Behaviour

| Policy | CRL Fetch Fails | Certificate in CRL | Certificate Not in CRL |
|--------|-----------------|--------------------|-----------------------|
| `SOFT_FAIL` | Accept cert (log warning) | Reject | Accept |
| `HARD_FAIL` | Reject cert | Reject | Accept |

### Background Refresh

A background thread periodically fetches CRLs from distribution points:

```
CrlRefreshThread (1 per broker):
  Every refreshInterval:
    1. For each tenant with CRL enabled:
       a. Fetch CRL from distribution point (HTTP GET, 10s timeout)
       b. Parse X509CRL, verify signature against issuer CA
       c. Store in ConcurrentHashMap<TenantId, X509CRL>
    2. Check all connected mTLS clients against updated CRL:
       a. For each connection with client cert:
          if cert.serialNumber IN crl.revokedCertificates:
            trigger graceful close (Category C signals)
```

---

## OCSP Checking (Online Certificate Status Protocol)

### Configuration

```yaml
security:
  tls:
    ocsp:
      enabled: false
      stapling-required: false         # if true, reject TLS without OCSP staple
      response-max-age-ms: 86400000    # 24 hours
      failure-policy: SOFT_FAIL
```

### OCSP Stapling

When `stapling-required = true`:
1. The TLS handshake must include an OCSP response (stapled by the client or upstream LB).
2. If no OCSP response is stapled → handshake fails.
3. The stapled response is validated: signature, issuer, freshness (`thisUpdate` + `maxAge`).

When `stapling-required = false` (default):
1. OCSP response is checked if present; absence is acceptable.
2. Broker does NOT contact OCSP responder directly (avoids latency and privacy concerns).

---

## Inter-Broker mTLS

All inter-broker communication (port 9094) uses mandatory mutual TLS:

```yaml
security:
  tls:
    inter-broker:
      require-mtls: true
      cluster-ca-path: /etc/ivy/certs/cluster-ca.pem     # CA that signs all broker certs
      cert-path: /etc/ivy/certs/broker.pem
      key-path: /etc/ivy/certs/broker-key.pem
      verify-hostname: true                                # CN must match brokerId
```

### Identity Verification

- Broker certificate CN must match the `brokerId` UUID.
- All broker certificates are signed by the same cluster CA.
- The cluster CA is NOT the same as tenant-facing CAs (separate trust domains).

### Certificate Rotation

Inter-broker certs can be rotated without downtime:
1. Deploy new cert + key files to all brokers.
2. `CertificateWatcher` detects file change, atomically swaps `SslContext`.
3. Existing connections continue with old cert; new connections use new cert.
4. After all connections have recycled (max-lifetime: 30 min default), old cert is no longer in use.

---

## Per-Tenant TLS Configuration

Stored in `tenants.config` JSONB column:

```json
{
  "tls": {
    "certPath": "/etc/ivy/tenants/acme/cert.pem",
    "keyPath": "/etc/ivy/tenants/acme/key.pem",
    "caCertPath": "/etc/ivy/tenants/acme/ca.pem",
    "mtlsMode": "OPTIONAL",
    "cipherSuites": ["TLS_AES_256_GCM_SHA384", "TLS_AES_128_GCM_SHA256"],
    "minProtocol": "TLSv1.2",
    "crl": {
      "enabled": true,
      "failurePolicy": "SOFT_FAIL"
    },
    "ocsp": {
      "enabled": false
    },
    "principalSource": "CN"
  }
}
```

---

*Last updated: 2026-03-25*
