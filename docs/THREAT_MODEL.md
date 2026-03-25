# Threat Model

> **Related:** [SECURITY.md](SECURITY.md), [ACL_DESIGN.md](ACL_DESIGN.md), [CERT_MANAGEMENT.md](CERT_MANAGEMENT.md),
> [MULTI_TENANT.md](MULTI_TENANT.md)

---

## Attacker Personas

### P1: External Attacker (Unauthenticated)
- **Access:** Network-level access to broker ports (e.g., internet-facing load balancer).
- **Goal:** Gain unauthorized access, exfiltrate data, disrupt service.
- **Capabilities:** Port scanning, TLS probing, brute-force credentials, exploit protocol bugs.

### P2: Compromised Client (Authenticated, Tenant-Scoped)
- **Access:** Valid credentials for one tenant.
- **Goal:** Escalate privileges, access other tenants' data, exceed quotas.
- **Capabilities:** Craft malicious wire frames, abuse protocol features, attempt cross-tenant access.

### P3: Malicious Insider (Authenticated, Admin-Level)
- **Access:** Admin credentials, potentially to multiple tenants.
- **Goal:** Exfiltrate data, sabotage operations, cover tracks.
- **Capabilities:** Create/delete tenants, modify ACLs, access audit logs, credential manipulation.

### P4: Compromised Broker (Internal)
- **Access:** Broker process or host-level access to one node in the cluster.
- **Goal:** Read/write any tenant's data, impersonate other brokers, poison metadata.
- **Capabilities:** Direct PG access, forge inter-broker RPC, manipulate MetadataImage.

---

## Threat Matrix

### T1: Credential Theft
| Aspect | Detail |
|--------|--------|
| **Threat** | Attacker obtains SCRAM credentials or JWT tokens |
| **Vector** | Network sniffing, credential store breach, log exfiltration |
| **Controls** | TLS mandatory (R26), SCRAM stored keys only (never plaintext), credentials never logged (R25), JWTs validated per-request (exp/iss/aud) |
| **Residual** | Compromised JWT valid until expiry; mitigated by short TTL + re-auth |

### T2: Cross-Tenant Data Access
| Aspect | Detail |
|--------|--------|
| **Threat** | Tenant A reads/writes Tenant B's messages |
| **Vector** | Forged TenantId, SQL injection, partition UUID collision |
| **Controls** | Deterministic partition UUIDs (tenantId+topic+partNum), defense-in-depth SQL (R6), ACL per-tenant, SecurityContext.tenantId immutable on re-auth (R27) |
| **Residual** | Negligible — UUID collision probability is 2^-122 |

### T3: Brute-Force Authentication
| Aspect | Detail |
|--------|--------|
| **Threat** | Attacker guesses credentials via repeated auth attempts |
| **Vector** | Rapid connection cycling, re-auth abuse |
| **Controls** | Per-connection max 3 attempts, per-IP progressive backoff + 300s lockout, constant-time credential comparison, timing side-channel prevention (dummy SCRAM on unknown user) |
| **Residual** | Lockout may block legitimate users sharing IP (NAT); configurable threshold |

### T4: Privilege Escalation via Re-Auth
| Aspect | Detail |
|--------|--------|
| **Threat** | Client re-authenticates with higher-privilege credentials or different tenant |
| **Vector** | SASL mechanism switch, tenant mismatch on re-auth |
| **Controls** | Tenant immutability (R27), mechanism anti-downgrade, ACL cache invalidation (R28), security epoch check |
| **Residual** | Principal CAN change within same tenant (by design — different service account) |

### T5: Stale Credential Abuse
| Aspect | Detail |
|--------|--------|
| **Threat** | Revoked user continues to operate on existing connection |
| **Vector** | Long-lived connections that outlive credential revocation |
| **Controls** | Security epoch registry (~2-4ns per-operation check), CredentialRevocationHandler (connection registry push), ReAuthScheduler (proactive timer) |
| **Residual** | Propagation delay: <100ms (LISTEN/NOTIFY) to 60s (polling worst-case) |

### T6: Denial of Service
| Aspect | Detail |
|--------|--------|
| **Threat** | Attacker exhausts broker resources (connections, CPU, disk, PG connections) |
| **Vector** | Connection flooding, message flooding, large message abuse |
| **Controls** | Per-tenant connection limits, token-bucket quota (produce/consume/request rate), max message size (1MB default), disk budget enforcement (85% reject threshold), connection creation rate limit |
| **Residual** | Shared PG pool — a misbehaving tenant can impact PG latency for others |

### T7: Man-in-the-Middle (Inter-Broker)
| Aspect | Detail |
|--------|--------|
| **Threat** | Attacker intercepts or forges inter-broker RPC messages |
| **Vector** | Network-level access between broker nodes |
| **Controls** | Mandatory mTLS on inter-broker port, CN verification (must match brokerId), incarnation ID validation, 30s clock skew limit, hop count limit (R13) |
| **Residual** | Compromised cluster CA allows impersonation; mitigate with HSM-backed CA |

### T8: Audit Log Tampering
| Aspect | Detail |
|--------|--------|
| **Threat** | Insider deletes or modifies audit records to cover tracks |
| **Vector** | Direct PG access, admin API abuse |
| **Controls** | Audit table is append-only (no UPDATE/DELETE permission for broker user), partitioned by time (old partitions moved to read-only tablespace), optional S3 export (immutable) |
| **Residual** | PG superuser can bypass; mitigate with PG audit extension (pgAudit) |

### T9: Zombie Broker
| Aspect | Detail |
|--------|--------|
| **Threat** | Restarted broker with stale state writes to partitions it no longer owns |
| **Vector** | Network partition, slow restart, clock skew |
| **Controls** | Incarnation ID (new UUIDv7 per restart), epoch fencing (3-layer: MetadataImage → PG CAS → incarnation), heartbeat monitor (10s stale threshold) |
| **Residual** | Extremely unlikely — all three layers must fail simultaneously |

### T10: Certificate Revocation Bypass
| Aspect | Detail |
|--------|--------|
| **Threat** | mTLS client uses a revoked certificate |
| **Vector** | CRL distribution point unreachable, stale OCSP response |
| **Controls** | CRL background refresh (1h), SOFT_FAIL/HARD_FAIL policy, OCSP stapling, cert expiry timer in ReAuthScheduler |
| **Residual** | SOFT_FAIL accepts certs when CRL unavailable — trade-off: availability vs security |

---

## Defense-in-Depth Layers

```
Network        → TLS 1.2+ mandatory, cipher suite enforcement
Transport      → SNI tenant isolation, per-tenant SslContext
Authentication → SCRAM/JWT/mTLS, mechanism ordering, IP rate limiting
Identity       → PrincipalResolverChain (mapping + enrichment)
Authorization  → DENY-first ACLs, protocol-scoped, cached + invalidated
Rate Limiting  → Token bucket per (tenant, principal, quotaType)
Data Isolation → Deterministic partition UUIDs, SQL defense-in-depth (R6)
Epoch Fencing  → Security epochs (credentials), leader epochs (partitions)
Audit          → Append-only structured log, partitioned, exported
```

---

*Last updated: 2026-03-25*
