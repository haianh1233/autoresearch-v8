# Layer 6: Security & Cross-Protocol Tests

> **Related:** [../TESTING_STRATEGY.md](../TESTING_STRATEGY.md), [../SECURITY.md](../SECURITY.md),
> [../ACL_DESIGN.md](../ACL_DESIGN.md), [../RE_AUTH.md](../RE_AUTH.md)

---

## Overview

Security tests validate authentication, authorization, TLS, quotas, and re-authentication under
load. Cross-protocol tests validate message routing, transaction visibility, and consumer group
coordination across all protocol boundaries.

**Tags:** `@Tag("security")`, `@Tag("cross-protocol")`, `@Tag("multi-tenant")`

---

## Security Tests (@Tag("security"))

### SaslAllMechanismsE2ETest

| Test | Mechanism | Assertion |
|------|-----------|-----------|
| `saslPlain_allProtocols` | PLAIN (Kafka + AMQP) | Anonymous mode connections succeed |
| `scramSha512_kafka` | SCRAM-SHA-512 | Kafka producer connects |
| `scramSha256_amqp` | SCRAM-SHA-256 | AMQP 0-9-1 connection open |
| `jwt_kafka_oauthbearer` | OAUTHBEARER (JWT) | Kafka producer sends message |
| `authFailure_wrongPassword` | Wrong credential | Rejected (enforced when ivy-auth configured) |
| `authFailure_expiredToken` | Expired JWT | Rejected (enforced when ivy-auth configured) |

**Note:** Tests run in permissive mode (Red phase). Full enforcement requires ivy-auth module
with configured credential store.

### TlsPerProtocolE2ETest

| Test | Protocol | Assertion |
|------|----------|-----------|
| `kafka_tls_mutualAuth` | Kafka | mTLS produces message |
| `mqtt_tls_mutualAuth` | MQTT 5.0 | mTLS connects with SSLContext |
| `amqp_tls_mutualAuth` | AMQP 0-9-1 | Declares queue, publishes, consumes |
| `certRotation_noDisconnect` | AMQP 0-9-1 | Pre/post-rotation messages both received (no session drop) |
| `tls10_rejected` | Any | TLS 1.0 handshake → `SSLHandshakeException` (only TLS 1.2+ allowed) |

### AclDenyPathsE2ETest

Per-protocol error codes when ACL denies access:

| Test | Protocol | Error Code | Description |
|------|----------|-----------|-------------|
| `kafka_topicDeny_errorCode29` | Kafka | 29 (`TOPIC_AUTHORIZATION_FAILED`) | |
| `mqtt_topicDeny_reasonCode0x87` | MQTT 5.0 | 0x87 (`Not Authorized`) | |
| `amqp_exchangeDeny_close403` | AMQP 0-9-1 | 403 (`ACCESS_REFUSED`) | |
| `nats_subjectDeny_errPermissions` | NATS | `-ERR Permissions violation` | |
| `aclHotReload_noRestart` | All | — | ACL rule change takes effect without restart |

### QuotaThrottleE2ETest

| Test | Protocol | Assertion |
|------|----------|-----------|
| `kafka_throttled_throttleTimeMs` | Kafka | `produce-throttle-time-avg` ≥ 0 after 1K message burst |
| `mqtt_throttled_delayedPuback` | MQTT | PUBACK latency measurable after 100 QoS 1 burst |
| `perTenantQuota_independent` | Kafka × 2 tenants | Tenant B latency < 5s while Tenant A bursting 500 msgs |
| `burstAllowance_oneSec` | Kafka | 10 message burst completes < 10s (fits in 1s bucket) |

### ReAuthUnderLoadE2ETest

| Test | Category | Assertion |
|------|----------|-----------|
| `tokenRefresh_noMessageLoss` | A (KIP-368) | All ACK'd messages present after JWT refresh |
| `credentialRotation_sessionContinues` | B (MQTT AUTH) | MQTT session receives 5 msgs after password change |
| `certRenewal_newConnectionsNewCert` | E (mTLS) | Pre/post-rotation messages received, new connection succeeds |
| `multiProtocol_simultaneousReAuth` | All | 4 protocol threads complete re-auth within 10s (no deadlock) |
| `categoryC_gracefulReconnect` | C (NATS/STOMP) | Reconnect with fresh credentials, messages before/after received |

### MultiTenancyIsolationE2ETest

| Test | Assertion |
|------|-----------|
| `noCrossTenantLeakage` | Tenant B receives 0 messages from Tenant A |
| `perTenantAcl` | ACL prevents cross-tenant read |
| `perTenantQuota` | 100 message burst within quota |
| `sniRouting_correctTenant` | Standard connection routes to correct tenant |

---

## Cross-Protocol Tests (@Tag("cross-protocol"))

### CrossProtocolE2ETest

| Test | From | To | Assertion |
|------|------|----|-----------|
| `kafkaToMqtt_messageDelivered` | Kafka | MQTT | MQTT latch counts down; content matches |
| `mqttToKafka_messageDelivered` | MQTT | Kafka | Kafka consumer polls message within 10s |
| `amqpToStomp_messageDelivered` | AMQP 0-9-1 | STOMP | STOMP receives MESSAGE frame with matching body |
| `allProtocols_simultaneousActive` | — | — | 10+ protocols connected concurrently without port/resource conflict |

### CrossProtocolTransactionE2ETest

| Test | Scenario | Assertion |
|------|----------|-----------|
| `committedTxn_visibleOnAllProtocols` | Kafka txn commits 3 msgs | MQTT receives exactly 3 with "committed-" prefix |
| `abortedTxn_invisibleOnAllProtocols` | Kafka txn aborts 3 msgs | MQTT + NATS receive 0 messages |
| `readCommitted_isolation` | Kafka txn in-flight | STOMP times out (no message); after commit → receives |

**Isolation model:**
- READ_UNCOMMITTED (MQTT, NATS, STOMP default): see open transactions immediately
- READ_COMMITTED (Kafka with `isolation.level=read_committed`): blocked by LSO until commit

### CrossProtocolConsumerGroupE2ETest

| Test | Scenario | Assertion |
|------|----------|-----------|
| `mixedProtocolGroup_partitionAssignment` | Kafka + AMQP in same group | Total received (Kafka + AMQP) = 10 (partitioned, no duplicates) |
| `rebalance_onConsumerJoinLeave` | Consumer 2 joins/leaves | Partitions redistributed; total 10 messages consumed |
| `exactlyOncePerGroup` | Kafka + STOMP in group | 10 unique messages total (no duplicates across protocols) |

**Group naming conventions:**
- AMQP: queue name `{groupId}.{topic}`
- STOMP: destination `/queue/{groupId}.{topic}`

---

## Test Infrastructure

### ProtocolClientFactory (Aggressive Timeouts)

| Setting | Value | Rationale |
|---------|-------|-----------|
| `REQUEST_TIMEOUT_MS` | 5,000 | Fail fast on connectivity issues |
| `DELIVERY_TIMEOUT_MS` | 10,000 | Covers linger + batch + PG round-trip |
| `MAX_BLOCK_MS` | 5,000 | Producer metadata fetch timeout |
| `RECONNECT_BACKOFF_MS` | 200 | Quick reconnect on transient failure |
| `RETRIES` | 1-3 | Limited retries to detect real failures |

### StompTestClient

```java
class StompTestClient {
    void connect();
    void subscribe(destination, subscriptionId, ackMode);
    void send(destination, body);
    void sendWithReceipt(destination, body, receiptId);
    void beginTransaction(txnId);
    void commitTransaction(txnId);
    void abortTransaction(txnId);
    StompFrame readFrame();  // blocking, 5s socket timeout
}
```

### MessageFidelityChecker

```java
class MessageFidelityChecker {
    void assertMessagePreserved(ProducedMessage original, ConsumedMessage consumed);
    boolean supportsKeys(Protocol protocol);   // Kafka/AMQP: yes, MQTT: no
}
```

---

## Execution

```bash
# Security tests only
mvn verify -Pcomplex-e2e -pl e2e -Dgroups=security

# Cross-protocol tests only
mvn verify -Pcomplex-e2e -pl e2e -Dgroups=cross-protocol

# Multi-tenant tests only
mvn verify -Pcomplex-e2e -pl e2e -Dgroups=multi-tenant

# All complex tests
mvn verify -Pcomplex-e2e -pl e2e
```

---

*Last updated: 2026-03-25*
