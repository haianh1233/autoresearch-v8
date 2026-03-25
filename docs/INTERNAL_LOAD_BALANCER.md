# Internal Load Balancer & Partition Leader Resolver

> **Related:** [WRITE_PATH.md](WRITE_PATH.md), [READ_PATH.md](READ_PATH.md),
> [CLUSTERING.md](CLUSTERING.md), [SECURITY.md](SECURITY.md), [MULTI_TENANT.md](MULTI_TENANT.md),
> [STORAGE.md](STORAGE.md), [CERT_MANAGEMENT.md](CERT_MANAGEMENT.md)

## Overview

The internal load balancer handles the complete request lifecycle from TCP connection to
write/read completion across **five phases**, executed in strict order:

```
TCP Connection
  ↓
Phase A — Netty Pipeline Assembly (TLS, SNI tenant resolution, protocol detection)
  ↓
Phase B — Security Pipeline (8-layer: auth, identity mapping, ACL, quota)
  ↓
Phase C — Transformation Pipeline (masking, schema, encryption, JSONata — may change key)
  ↓
Phase 1 — Partition Selection (DestinationParser → DestinationResolver → PartitionRouter)
  ↓
Phase 2 — Ownership Routing (HRW → Local/Forward/NoLeader)
  ↓
Phase 3 — Write Forwarding or Local Write (WriteAccumulator → PG COMMIT → ACK)
```

**Critical ordering invariant: Transform (Phase C) BEFORE Resolve (Phase 1).** Transforms can
change the message key, and the key determines partition assignment. Resolving first then
transforming routes messages to the wrong partition.

```
TCP Connection Established
  ↓
[proxy-protocol] → [tls + SNI] → [tenant-resolver] → [tenant-context] → [auth]
  → [protocol-detector] → [negotiation] → [flow-controller] → [exception-handler]
  ↓
Protocol Handler (decode wire → RawMessage)
  ↓
TransformationPipeline.execute(ctx, List<RawMessage>)
  │  schema validation, masking, encryption, JSONata, header injection
  │  RawMessage → TransformedMessage (key may change!)
  ↓
RouteTable.resolve(tenant, protocol, transformedMessage)
  │  DestinationParser → DestinationResolver → PartitionRouter
  │  TransformedMessage → PartitionId (Murmur2 / round-robin / explicit)
  ↓
HRWRouter.ownerOf(partitionId)
  ├─ [OWNER == self]  → WriteAccumulator → WriteWorker → PG COMMIT → ACK
  └─ [OWNER != self]  → Forwarder → InterBrokerRpcClient → owner → ACK
```

---

## Phase A: Netty Pipeline Assembly

### Handler Chain (8 Layers)

Every new TCP connection gets the following Netty pipeline, assembled by `ChannelPipelineFactory`:

```
Layer 0: [proxy-protocol]     (optional)  HAProxy PROXY v1/v2 — must come before TLS
Layer 1: [tls]                (conditional) SSL/TLS termination — AtomicReference<SslContext> for hot-reload
Layer 2: [tenant-resolver]    (conditional) SNI hostname → TenantId — fires on SslHandshakeCompletionEvent
Layer 3: [tenant-context]     (conditional) TenantRegistry lookup + status validation (ACTIVE/SUSPENDED/DELETED)
Layer 4: [auth]               (conditional) SASL handshake — stores AuthenticatedPrincipal on channel
Layer 5: [protocol-detector]  (always)     Magic bytes → DetectionResult (timeout: 5s)
Layer 6: [negotiation]        (always)     Wires protocol-specific codec + handler into pipeline
Layer 7: [flow-controller]    (always)     Channel-level backpressure management
Layer 8: [exception-handler]  (always)     Catch-all ServerExceptionHandler
```

**Key design patterns:**
- **Conditional insertion:** handlers only added if feature enabled (TLS, multi-tenant, auth)
- **Self-removing handlers:** `TenantResolverHandler` and `ProtocolNegotiationHandler` remove
  themselves after first use — minimizes pipeline depth on the hot path
- **AtomicReference hot-reload:** `SslContext` swapped atomically; in-flight connections keep
  old context, new connections get new context — zero downtime certificate rotation

### SNI Routing (TLS ClientHello → TenantId)

**Chicken-and-egg problem:** the broker needs `TenantId` to select the per-tenant `SslContext`,
but `TenantId` is derived from the SNI hostname in the TLS `ClientHello` — which arrives
**before** the handshake completes.

**Solution:** Java's `ExtendedSSLSession.getRequestedServerNames()` provides SNI extraction
after the handshake event, without requiring a custom `SniHandler`:

```java
class TenantResolverHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof SslHandshakeCompletionEvent) {
            // Extract SNI from TLS session
            SSLSession session = ctx.pipeline().get(SslHandler.class).engine().getSession();
            String sniHostname = ((ExtendedSSLSession) session)
                .getRequestedServerNames().stream()
                .filter(n -> n instanceof SNIHostName)
                .map(n -> ((SNIHostName) n).getAsciiName())
                .findFirst().orElse(null);

            // Deterministic UUID derivation (RFC 4122 v3, MD5 name-based)
            // "acme.broker.example.com" → subdomain "acme" → UUID
            String subdomain = sniHostname.split("\\.")[0];
            TenantId tenantId = new TenantId(UUID.nameUUIDFromBytes(subdomain.getBytes(UTF_8)));

            ctx.channel().attr(TENANT_KEY).set(tenantId);
            ctx.pipeline().remove(this);  // self-remove after resolution
        }
    }
}
```

**Per-tenant SslContext pool:**
```java
class SslContextPool {
    // tenantId → current SslContext (swapped atomically on cert reload)
    ConcurrentHashMap<TenantId, AtomicReference<SslContext>> pool;

    void updateSslContext(TenantId tenantId, SslContext newCtx) {
        AtomicReference<SslContext> ref = pool.get(tenantId);
        SslContext old = ref.getAndSet(newCtx);        // atomic swap
        ReferenceCountUtil.safeRelease(old);            // OpenSSL ref counting
    }
}
```

### Protocol Detection (Magic Bytes)

`ProtocolDetector` extends `ByteToMessageDecoder` and matches incoming bytes against ordered rules:

| Confidence | Protocol | Magic Bytes / Pattern |
|-----------|----------|----------------------|
| EXACT | AMQP 0-9-1 | `AMQP\x00\x00\x09\x01` (8 bytes) |
| EXACT | AMQP 1.0 | `AMQP\x00\x01\x00\x00` (8 bytes) |
| EXACT | AMQP 1.0 SASL | `AMQP\x03\x01\x00\x00` (8 bytes) |
| HIGH | NATS | `"CONNECT "` or `"INFO "` prefix |
| HIGH | STOMP | `"STOMP\n"` or `"CONNECT\n"` |
| HIGH | HTTP | `"GET "`, `"POST "`, `"PUT "`, etc. |
| HIGH | Kafka | 4-byte len + API key ∈ [0-74] + version ∈ [0-16] |
| HIGH | MQTT | Fixed header: type ∈ [1-14] + flag validation |
| HIGH | PgWire | 4-byte len + version 196608 (3.0) |
| MEDIUM | Redis | Starts with `+`, `-`, `:`, `$`, `*` |
| LOW | MySQL | Greeting byte 0x0a at offset 4 |

**Key:** First match wins; no backtracking. RMQ Streams checked before Kafka (both have 4-byte
frame length; RMQ Streams' `0x000F` command key distinguishes).

After detection, `ProtocolNegotiationHandler` looks up the `ProtocolBundle` via `ServiceLoader`
and wires the protocol-specific codec + handler into the pipeline:

```java
void onProtocolDetected(DetectionResult result) {
    ProtocolBundle bundle = ProtocolBundleRegistry.lookup(result.protocolId());
    pipeline.addBefore("flow-controller", "codec", bundle.codec().decoder(codecContext));
    pipeline.addBefore("flow-controller", "encoder", bundle.codec().encoder(codecContext));
    pipeline.addBefore("flow-controller", "handler", bundle.handler());
    pipeline.remove("protocol-detector");
    pipeline.remove("negotiation");
}
```

---

## Phase B: Security Pipeline (8 Layers)

> Full details in [SECURITY.md](SECURITY.md). Summary here for pipeline context.

After Netty pipeline assembly, every operation passes through the security layers:

| Layer | Module | What It Does |
|-------|--------|-------------|
| 0 | ivy-server | Connection metadata logging (IP, port, timestamp) — never rejects |
| 1 | ivy-server | TLS 1.2+ mandatory; cipher suite enforcement (AES-GCM only) |
| 2 | ivy-server | SNI → TenantId; reject if SUSPENDED/DELETED |
| 3 | ivy-auth | Connection quota (max connections per tenant); per-IP rate limiting |
| 4 | ivy-codec | SASL: SCRAM-SHA-256/512, OAUTHBEARER (JWT), PLAIN, DELEGATION_TOKEN, mTLS |
| 5 | ivy-auth | PrincipalResolverChain: raw auth → ResolvedIdentity (user mapping, groups) |
| 6 | ivy-auth | ACL authorization: DENY-first, protocol-scoped, Caffeine-cached |
| 7 | ivy-auth | Token-bucket quotas: produce/consume byte rate, request rate |
| 8 | ivy-auth | Audit: structured JSON events → PG `audit_log` table |

**Security Epoch check** (Rule R33): before every `BrokerEngine` operation, `SecurityEpochRegistry.checkValid(ctx)` verifies `authEpoch >= max(tenantEpoch, principalEpoch)`. Cost: ~2-4ns.

---

## Phase C: Transformation Pipeline

### Payload Abstraction (Dual-View)

Messages arrive as binary (Kafka, MQTT) or JSON (HTTP, PgWire). The transformation pipeline
needs JSON; storage needs bytes. `Payload` provides both with lazy conversion:

```java
sealed interface Payload permits BinaryPayload, JsonPayload {
    byte[] asBytes();       // zero-cost for BinaryPayload; serializes for JsonPayload
    JsonNode asJson();      // zero-cost for JsonPayload; parses for BinaryPayload
    int sizeBytes();        // for quota accounting
}

record BinaryPayload(byte[] data) implements Payload {
    // asBytes() → data (zero-cost); asJson() → ObjectMapper.readTree(data) (on demand)
}
record JsonPayload(JsonNode node) implements Payload {
    // asJson() → node (zero-cost); asBytes() → ObjectMapper.writeValueAsBytes(node) (on demand)
}
```

### Message Pipeline Types (Compiler-Enforced Ordering)

```java
record RawMessage(TopicName topic, PartitionIndex requestedPartition, Key key,
    Payload value, Map<String,String> headers, Timestamp timestamp,
    ProducerIdentity producerId, SequenceNumber seq, TransactionId txnId)

record TransformedMessage(/* same fields — key/value/headers may differ */)

record LocalResolvedTransformedMessage(/* + PartitionId pid, PartitionIndex idx */)
record RemoteResolvedTransformedMessage(/* + BrokerId target, LeaderEpoch epoch */)

record PartitionedResolvedBatch(
    LocalResolvedTransformedBatch local,
    RemoteResolvedTransformedBatch remote)
```

`Resolver.resolve()` accepts ONLY `TransformedMessage` — compiler prevents resolving before
transforming.

### TransformStep Interface

```java
interface TransformStep {
    TransformResult apply(SecurityContext ctx, TransformedMessage msg);
    boolean modifiesKey();   // true → skipped for transactional writes
}

sealed interface TransformResult {
    record Transformed(TransformedMessage msg) implements TransformResult {}
    record RouteToDeadLetter(TransformedMessage msg, String reason) implements TransformResult {}
    record Rejected(String reason) implements TransformResult {}
}
```

### Concrete Steps

| Step | `modifiesKey()` | Description |
|------|----------------|-------------|
| `SchemaValidationStep` | `false` | Validate against Avro/JSON/Protobuf schema (tenant-scoped) |
| `DataMaskingStep` | `false` | Mask PII fields (e.g., `email` → `***@***.com`) |
| `FieldEncryptionStep` | `false` | Encrypt fields with tenant KMS key |
| `JsonataTransformStep` | **`true`** | JSONata expression restructure; may produce new key |
| `HeaderEnrichmentStep` | `false` | Inject metadata headers (source protocol, tracing, timestamp) |

### Pipeline Execution

```java
class TransformationPipeline {
    List<TransformedMessage> execute(SecurityContext ctx, List<RawMessage> batch) {
        List<TransformedMessage> result = new ArrayList<>(batch.size());
        for (RawMessage raw : batch) {
            TransformedMessage msg = TransformedMessage.from(raw);
            boolean accepted = true;
            for (TransformStep step : steps) {
                // Transactional key-skip: key was part of AddPartitionsToTxn
                if (msg.txnId() != null && step.modifiesKey()) continue;
                switch (step.apply(ctx, msg)) {
                    case Transformed t        -> msg = t.msg();
                    case RouteToDeadLetter d  -> { dlqRouter.route(d); accepted = false; }
                    case Rejected r           -> { log.warn(r.reason()); accepted = false; }
                }
                if (!accepted) break;
            }
            if (accepted) result.add(msg);
        }
        return result;
    }
}
```

**NoOpPipeline:** singleton zero-cost default when no transforms configured. Checked via identity
equality before invocation: `if (pipeline != NoOpPipeline.INSTANCE) pipeline.execute(...)`.

**SecurityContext propagation:** every step receives `SecurityContext` — tenant-scoped schema
lookup, authorized routing, trusted header enrichment (reads tenantId/principal from ctx, NOT
from message — prevents spoofing).

---

## Phase 1: Client-Side Partition Selection

### RouteTable

Orchestrates the three-step resolution for every incoming message:

```java
// ivy-server/src/main/java/com/ivy/server/routing/RouteTable.java
public final class RouteTable {
    private final Map<ProtocolId, DestinationParser> parsers;      // one per protocol
    private final DestinationResolver resolver;                     // metadata-backed
    private final PartitionRouter router;                           // Murmur2 / round-robin

    public Optional<RoutedPartition> resolve(
            TenantId tenantId, ProtocolId protocol,
            String address, byte[] key, Integer explicitPartition) {

        // Step 1: protocol-specific address → baseName + optional partition hint
        DestinationParser parser = parsers.get(protocol);
        ParsedDestination parsed = parser.parse(address);

        // Step 2: resolve topic name → DestinationId + partitionCount
        Optional<ResolvedDestination> dest = resolver.resolve(tenantId, parsed.baseName());
        if (dest.isEmpty()) return Optional.empty();

        // Step 3: select partition
        Integer effective = parsed.hasExplicitPartition() ? parsed.partition() : explicitPartition;
        return Optional.of(router.route(dest.get().destinationId(),
                                        dest.get().partitionCount(),
                                        key, effective));
    }
}
```

**Critical ordering: Transform BEFORE Resolve.**
Message transformation (header injection, schema validation, key rewrite) MUST happen
before `RouteTable.resolve()` is called. Transformation can change the message key,
and the key determines which partition is selected.

---

### DestinationParser (per-protocol address parsing)

Each protocol has a different address format. `DestinationParser` extracts the
logical topic name and an optional explicit partition number.

| Protocol | Example address | baseName | partition |
|----------|----------------|----------|-----------|
| Kafka | `orders` (partition in frame) | `orders` | from Produce frame |
| AMQP 0-9-1 | queue name on Basic.Publish | queue name | from routing key suffix |
| AMQP 1.0 | `orders` (target address on Attach) | `orders` | null |
| MQTT 3.1.1/5.0 | `sensors/temp/2` | `sensors.temp` | `2` |
| MySQL | `orders` (from SQL) | `orders` | null |
| PgWire | `orders` (from SQL) | `orders` | null |

**Universal suffix rule (non-Kafka):**
If the address ends with a separator + non-negative integer, that integer is the
explicit partition. Everything before it is the base name.

```java
// Example: "sensors/temp/2" → baseName="sensors.temp", partition=2
// Example: "sensors/temp"   → baseName="sensors.temp", partition=null
```

Kafka is special: partition comes from the wire frame (ProduceRequest), not the topic name.

---

### DestinationResolver

Resolves a `(TenantId, topicName)` pair to a `DestinationId` (UUID) and `partitionCount`.

```java
// ivy-broker/src/main/java/com/ivy/broker/destination/DestinationResolver.java
public interface DestinationResolver {
    Optional<ResolvedDestination> resolve(TenantId tenantId, String topicName);
}

// ivy-broker/src/main/java/com/ivy/broker/destination/MetadataBackedDestinationResolver.java
public final class MetadataBackedDestinationResolver implements DestinationResolver {

    public Optional<ResolvedDestination> resolve(TenantId tenantId, String topicName) {
        return metadataManager.getTopic(tenantId, topicName)
            .map(topic -> {
                // DestinationId is deterministic: same tenant+topic → same UUID always
                String seed = tenantId.id().toString() + ":" + topicName;
                UUID destUuid = UUID.nameUUIDFromBytes(seed.getBytes(UTF_8));
                return new ResolvedDestination(new DestinationId(destUuid), topic.partitionCount());
            });
    }
}
```

An `AutoCreateDestinationResolver` wraps `MetadataBackedDestinationResolver` and
auto-creates the topic if it does not exist (configurable per-tenant).

---

### PartitionRouter

Selects a partition number given a destination and an optional key.

```java
// ivy-common/src/main/java/com/ivy/common/routing/PartitionRouter.java
public interface PartitionRouter {
    RoutedPartition route(DestinationId destinationId, int partitionCount,
                          byte[] key, Integer explicitPartition);
}

// ivy-common/src/main/java/com/ivy/common/routing/DefaultPartitionRouter.java
public final class DefaultPartitionRouter implements PartitionRouter {

    // MAX_CACHE_SIZE=65,536 — cleared wholesale on overflow (simple LRU approximation)
    private final ConcurrentHashMap<String, PartitionId> partitionIdCache =
            new ConcurrentHashMap<>(256, 0.75f, 4);
    private final AtomicInteger roundRobinCounter = new AtomicInteger();

    public RoutedPartition route(DestinationId destinationId, int partitionCount,
                                  byte[] key, Integer explicitPartition) {
        // Fast path: single-partition topic
        if (partitionCount == 1) {
            return new RoutedPartition(getPartitionId(destinationId, 0), 0);
        }

        int partitionNum = selectPartition(partitionCount, key, explicitPartition);
        return new RoutedPartition(getPartitionId(destinationId, partitionNum), partitionNum);
    }

    private int selectPartition(int partitionCount, byte[] key, Integer explicit) {
        if (explicit != null) {                               // Priority 1: explicit
            if (explicit < 0 || explicit >= partitionCount)
                throw new InvalidPartitionException(explicit, partitionCount);
            return explicit;
        }
        if (key != null && key.length > 0) {                 // Priority 2: Murmur2(key)
            return Murmur2.partition(key, partitionCount);
        }
        return Murmur2.toPositive(                            // Priority 3: round-robin
            roundRobinCounter.getAndIncrement()) % partitionCount;
    }

    // PartitionId is deterministic: same (destinationId, partitionNum) → same UUID
    private PartitionId getPartitionId(DestinationId destinationId, int partitionNum) {
        String cacheKey = destinationId.id() + ":" + partitionNum;
        return partitionIdCache.computeIfAbsent(cacheKey, k -> {
            if (partitionIdCache.size() >= MAX_CACHE_SIZE) {
                partitionIdCache.clear();  // simple eviction
            }
            return new PartitionId(UUID.nameUUIDFromBytes(k.getBytes(UTF_8)));
        });
    }
}
```

### PartitionId & DestinationId UUID Derivation

All UUIDs are **RFC 4122 version 3 (MD5 name-based)** via `UUID.nameUUIDFromBytes()`.
Same inputs always produce the same UUID — across restarts, across brokers, across clusters.

```
DestinationId = UUID.nameUUIDFromBytes( tenantId + ":" + topicName )
PartitionId   = UUID.nameUUIDFromBytes( destinationId + ":" + partitionNum )

Equivalently (fully expanded):
PartitionId   = UUID.nameUUIDFromBytes( tenantId + ":" + topicName + ":" + partitionNum )
```

This means two tenants with the same topic name "orders" get different `PartitionId`s
at every layer — storage, routing, HRW ownership — with no explicit tenant scoping needed.

### Murmur2 Hash

Kafka-compatible Murmur2 — identical output to `org.apache.kafka.common.utils.Utils.murmur2()`.
Used for both key-based partition selection and write-worker affinity.

```java
// ivy-common/src/main/java/com/ivy/common/util/Murmur2.java
public final class Murmur2 {
    private static final int SEED = 0x9747b28c;
    private static final int M    = 0x5bd1e995;
    private static final int R    = 24;

    public static int hash(byte[] data) { ... }          // core hash
    public static int toPositive(int n) { return n & 0x7FFFFFFF; }  // NOT Math.abs
    public static int partition(byte[] key, int count) { // key → partition num
        return toPositive(hash(key)) % count;
    }
}
```

**`toPositive` vs `Math.abs`:** `Math.abs(Integer.MIN_VALUE) == Integer.MIN_VALUE` (negative!).
`n & 0x7FFFFFFF` is always non-negative. This is Kafka's exact implementation.

---

## Phase 2: Broker-Side Ownership Routing (HRW)

After `PartitionId` is known, `DefaultBrokerEngine` determines which broker owns it.

### HRW (Highest Random Weight) Partition Leadership

Rendezvous hashing assigns each partition to exactly one broker deterministically.
No coordinator, no Raft, no election rounds — every broker independently computes
the same result.

**Score function:**
```
score(brokerId, partitionId) = HMAC-SHA-256(clusterSecret, partitionId + "|" + brokerId)
leader(partitionId)          = broker with HIGHEST score among ACTIVE brokers
```

```java
// ivy-broker/src/main/java/com/ivy/broker/cluster/HRWRouter.java
public final class HRWRouter {

    // ArrayBlockingQueue<Mac> — pool of 4, no ThreadLocal (VT-safe, see RULES.md R19)
    private static final int MAC_POOL_SIZE = 4;
    private final ArrayBlockingQueue<Mac> macPool = new ArrayBlockingQueue<>(MAC_POOL_SIZE);

    public Optional<BrokerId> ownerOf(PartitionId partitionId) {
        Set<BrokerId> activeBrokers = metadataImage.get().activeBrokers().keySet();
        if (activeBrokers.isEmpty()) return Optional.empty();

        BrokerId winner = null;
        BigInteger highestScore = BigInteger.ZERO;

        // Sort brokers for determinism (tie-break: lower BrokerId wins)
        for (BrokerId brokerId : new TreeSet<>(activeBrokers)) {
            BigInteger score = hmacScore(partitionId, brokerId);
            if (score.compareTo(highestScore) > 0
                    || (score.compareTo(highestScore) == 0
                        && (winner == null || brokerId.compareTo(winner) < 0))) {
                highestScore = score;
                winner = brokerId;
            }
        }
        return Optional.ofNullable(winner);
    }

    // Bulk: O(N*P) but called only on topology change, not on hot path
    public Map<PartitionId, BrokerId> computeOwnership(Set<PartitionId> partitions) {
        Map<PartitionId, BrokerId> result = new HashMap<>(partitions.size());
        for (PartitionId pid : partitions) {
            ownerOf(pid).ifPresent(owner -> result.put(pid, owner));
        }
        return result;
    }

    // Returns Set<PartitionId> that would change owner if topology changes old→new
    public Set<PartitionId> affectedPartitions(
            Set<BrokerId> oldBrokers, Set<BrokerId> newBrokers, Set<PartitionId> all) {
        Set<PartitionId> affected = new HashSet<>();
        for (PartitionId pid : all) {
            BrokerId oldOwner = ownerOf(pid, oldBrokers).orElse(null);
            BrokerId newOwner = ownerOf(pid, newBrokers).orElse(null);
            if (!Objects.equals(oldOwner, newOwner)) affected.add(pid);
        }
        return affected;  // ~1/(N+1) of all partitions on broker join/leave
    }

    private BigInteger hmacScore(PartitionId partitionId, BrokerId brokerId) {
        String input = partitionId.id().toString() + "|" + brokerId.id().toString();
        Mac mac = acquireMac();
        try {
            mac.reset();
            mac.init(new SecretKeySpec(clusterSecret, "HmacSHA256"));
            byte[] hash = mac.doFinal(input.getBytes(UTF_8));
            return new BigInteger(1, hash);   // unsigned: 1 = positive signum
        } finally {
            releaseMac(mac);                  // return to pool even on exception
        }
    }

    private Mac acquireMac() {
        Mac mac = macPool.poll();
        return mac != null ? mac : Mac.getInstance("HmacSHA256");
    }

    private void releaseMac(Mac mac) {
        macPool.offer(mac);  // silently drop if pool is full
    }
}
```

**Why HMAC-SHA-256 (not Murmur2)?**
- HMAC is keyed: `clusterSecret` prevents external parties from predicting ownership
- BigInteger unsigned comparison: platform-independent, no floating point
- Crypto-quality distribution: no clustering of partitions on any single broker

**Why Mac pool (not ThreadLocal)?**
Virtual threads can be parked and resumed on different platform threads.
`ThreadLocal<Mac>` would leak HMAC state across unrelated virtual threads.
`ArrayBlockingQueue<Mac>` size 4 matches the 4 WriteWorker threads without contention.

---

### RoutingDecision

```java
// ivy-broker/src/main/java/com/ivy/broker/cluster/RoutingDecision.java
sealed interface RoutingDecision {
    record Local()                          implements RoutingDecision {}
    record Forward(BrokerId targetBroker)   implements RoutingDecision {}
    record NoLeader()                       implements RoutingDecision {}
}

// Inside DefaultBrokerEngine:
RoutingDecision decide(PartitionId partitionId) {
    return HRWRouter.ownerOf(partitionId)
        .map(owner -> owner.equals(selfBrokerId)
            ? (RoutingDecision) new RoutingDecision.Local()
            : new RoutingDecision.Forward(owner))
        .orElse(new RoutingDecision.NoLeader());
}
```

---

### Write Worker Affinity

Within a single broker, writes for the same partition always go to the same `WriteWorker`
thread. This gives single-writer ordering without locks.

```java
// ivy-broker/src/main/java/com/ivy/broker/write/WriteDispatcher.java
int workerIndex(PartitionId partitionId) {
    byte[] idBytes = ByteBuffer.allocate(16)
        .putLong(partitionId.id().getMostSignificantBits())
        .putLong(partitionId.id().getLeastSignificantBits())
        .array();
    return Murmur2.toPositive(Murmur2.hash(idBytes)) % workerCount;  // default: 4 workers
}
```

This is separate from HRW ownership. HRW decides which broker owns a partition.
Murmur2(partitionId) decides which thread within that broker processes the write.

---

## Phase 3: Write Forwarding (Non-Owner → Owner)

When `RoutingDecision.Forward(targetBroker)` is returned:

```
Non-owner broker:
  1. ForwardWriteManager.forward(targetBroker, partitionId, writes, leaderEpoch)
  2. Check hop count: reject if hopCount >= MAX_HOPS (1)
  3. InterBrokerRpcClient.send(ForwardWriteRequest)

Owner broker:
  4. Validate epoch: MetadataImage.epochFor(partitionId) == request.leaderEpoch
     → WrongEpochException → non-owner refreshes MetadataImage, retries once
  5. WriteAccumulator.accumulate(writes)
  6. WriteWorker: PG COMMIT → ForwardWriteResponse(baseOffset, recordCount)
  7. Async: LogSegment cache + FlushEventDispatcher

Non-owner broker:
  8. Return WriteResult to protocol handler → ACK to client
```

### Forwarder (Batch-by-Broker RPC)

The `Forwarder` groups messages by target broker to minimize RPC round-trips:

```java
class Forwarder {
    List<OperationResult<WriteResult>> forward(SecurityContext ctx,
            RemoteResolvedTransformedBatch batch) {
        // Group by targetBroker: 100 msgs to 3 brokers = 3 RPCs (not 100)
        Map<BrokerId, List<RemoteResolvedTransformedMessage>> byBroker =
            batch.messages().stream()
                .collect(Collectors.groupingBy(m -> m.targetBroker()));

        List<OperationResult<WriteResult>> results = new ArrayList<>();
        for (var entry : byBroker.entrySet()) {
            try {
                ForwardWriteResponse resp = rpcClient.send(
                    entry.getKey(), ForwardWriteRequest.from(ctx, entry.getValue()));
                results.addAll(resp.toOperationResults());
            } catch (RpcException e) {
                // RPC failure → StorageError for each message in this broker's batch
                entry.getValue().forEach(msg ->
                    results.add(OperationResult.failure(new StorageError(e))));
            }
        }
        return results;  // same order as input messages
    }
}
```

**Key properties:**
- One RPC per broker, not per message (amortizes network overhead)
- No retry in Forwarder — protocol handler decides retry policy
- RPC failure → `StorageError` for every message destined for that broker

### ForwardWriteRequest / ForwardWriteResponse

```java
// NOT a PendingWrite — wire-safe (no CompletableFuture, no callbacks)
record ForwardWriteRequest(
    PartitionId        partitionId,
    LeaderEpoch        leaderEpoch,     // epoch fence: owner rejects if stale
    HopCount           hopCount,        // max 1; prevents routing loops
    SecurityContext    securityContext, // tenant + principal (owner re-validates)
    List<WriteEntry>   entries          // key, value, headers, timestamp, producerId, epoch, seq
) implements InterBrokerMessage {}

record ForwardWriteResponse(
    long       baseOffset,
    int        recordCount,
    ErrorCode  errorCode           // NONE | WRONG_EPOCH | NOT_LEADER | INTERNAL_ERROR
) implements InterBrokerMessage {}
```

### ForwardFetchRequest / ForwardFetchResponse (Read Tier 2)

```java
record ForwardFetchRequest(
    PartitionId     partitionId,
    long            fetchOffset,
    int             maxBytes,
    HopCount        hopCount,
    SecurityContext securityContext
) implements InterBrokerMessage {}

record ForwardFetchResponse(
    List<Record> records,
    long         highWatermark,
    long         lastStableOffset,
    ErrorCode    errorCode
) implements InterBrokerMessage {}
```

### HopCount (Loop Prevention)

```java
// ivy-common/src/main/java/com/ivy/common/cluster/HopCount.java
record HopCount(int value) {
    static final int MAX_HOPS = 1;                   // A→B allowed, A→B→C rejected
    static HopCount origin()  { return new HopCount(0); }
    HopCount increment()      { return new HopCount(value + 1); }
    boolean exceedsLimit()    { return value > MAX_HOPS; }
}
```

**Why max 1 hop?**
If A forwards to B, and B's MetadataImage is also stale (thinks C owns it), a second
forward A→B→C would be allowed. At max 1 hop, B must reject and let A refresh metadata.
This prevents routing chains that could mask split-brain scenarios.

---

## Epoch Fencing (3-Layer Defense)

### Layer 1: MetadataImage check (fast, in-memory)

```java
// In DefaultBrokerEngine.write():
LeaderEpoch currentEpoch = metadataImage.get().epochFor(partitionId);
if (!currentEpoch.equals(requestEpoch)) {
    throw new WrongEpochException(partitionId, requestEpoch, currentEpoch);
}
```

### Layer 2: PG CAS (safety net)

```sql
-- In WriteWorker.processBatch():
UPDATE partition_offsets
SET next_offset    = next_offset + :batchSize
WHERE partition_id = :pid
  AND leader_epoch = :expectedEpoch    ← atomic compare-and-swap
RETURNING next_offset - :batchSize AS base_offset;

-- 0 rows → WrongEpochException (someone else claimed between Layer 1 and now)
```

### Layer 3: Incarnation ID (zombie prevention)

Each broker restart generates a new `incarnationId` (UUIDv7).
Stored in `broker_registry`. Inter-broker RPC messages carry `incarnationId`.
If the receiver detects a stale incarnation, it rejects and triggers peer reconnection.

### WrongEpoch Recovery Flow

```
non-owner receives WrongEpochException or ForwardWriteResponse(WRONG_EPOCH):
  1. Reload MetadataImage from PG (authoritative refresh)
  2. Recompute owner via HRWRouter.ownerOf(partitionId)
  3. If new owner == self:     process locally (recovered ownership)
  4. If new owner != self:     retry forward to new owner once (hopCount=0 on retry)
  5. If still WRONG_EPOCH:     return NOT_LEADER_OR_AVAILABLE to client
```

---

## MetadataImage (Ownership Snapshot)

An immutable snapshot of cluster topology, published atomically via `VarHandle`.

```java
// ivy-broker/src/main/java/com/ivy/broker/cluster/MetadataImage.java
record MetadataImage(
    Map<BrokerId, BrokerInfo>  activeBrokers,   // brokerId → {host, port, interBrokerPort}
    Map<PartitionId, BrokerId> ownership,        // partitionId → current owner
    Map<PartitionId, Integer>  epochs,           // partitionId → leader_epoch
    long                       version           // monotonically increasing
) {
    // Immutable: all maps are Map.copyOf() — defensive copies at construction time
    // VarHandle.setRelease() for publish, VarHandle.getAcquire() for consume
}

// MetadataImageHolder — atomic reference wrapper
final class MetadataImageHolder {
    private static final VarHandle IMAGE_HANDLE = /* VarHandle for 'image' field */;
    private volatile MetadataImage image;

    MetadataImage get() { return (MetadataImage) IMAGE_HANDLE.getAcquire(this); }

    void update(MetadataImage incoming) {
        MetadataImage current;
        do {
            current = get();
            if (incoming.version() <= current.version()) return;  // stale, discard
        } while (!IMAGE_HANDLE.compareAndSet(this, current, incoming));
    }
}
```

### MetadataImage Update Triggers (3-Layer Propagation)

```
Layer 1: PG LISTEN/NOTIFY (fastest, <100ms, best-effort)
  → PG fires NOTIFY on any INSERT/UPDATE to broker_registry or partition_offsets
  → MetadataPoller receives notification → reload changed rows → update MetadataImageHolder

Layer 2: MetadataUpdateBroadcast (inter-broker push, ~1ms)
  → After BrokerFencingPipeline runs or partition is claimed
  → InterBrokerRpcClient.broadcast(MetadataUpdateBroadcast) to all peers
  → Peers call MetadataImageHolder.update()

Layer 3: PG polling fallback (reliable, every 30s)
  → MetadataPoller periodic SELECT from broker_registry + partition_offsets
  → Catches any updates missed by Layer 1 or Layer 2
  → Authoritative: PG is always the ground truth
```

Layer 1 is best-effort (can drop under load). Layer 3 is always authoritative.
In the worst case (both Layer 1 and 2 missed), metadata converges within 30 seconds.

---

## Inter-Broker RPC Protocol

### Message Types (sealed interface)

```java
// ivy-broker/src/main/java/com/ivy/broker/cluster/InterBrokerMessage.java
sealed interface InterBrokerMessage {
    // Write routing
    record ForwardWriteRequest(...)        implements InterBrokerMessage {}
    record ForwardWriteResponse(...)       implements InterBrokerMessage {}

    // Read routing (Tier 2 fallback)
    record ForwardFetchRequest(...)        implements InterBrokerMessage {}
    record ForwardFetchResponse(...)       implements InterBrokerMessage {}

    // Metadata sync
    record MetadataUpdateBroadcast(...)    implements InterBrokerMessage {}
    record MetadataSyncRequest(long fromVersion) implements InterBrokerMessage {}
    record MetadataSyncResponse(MetadataImage delta) implements InterBrokerMessage {}

    // Liveness
    record HeartbeatPing(BrokerId src, UUID incarnationId) implements InterBrokerMessage {}
    record HeartbeatPong(BrokerId src)                     implements InterBrokerMessage {}

    // Push delivery
    record FlushNotification(PartitionId, long highWatermark) implements InterBrokerMessage {}
    record InterestAdd(PartitionId, BrokerId subscriber)      implements InterBrokerMessage {}
    record InterestRemove(PartitionId, BrokerId subscriber)   implements InterBrokerMessage {}
}
```

### Wire Frame

```
[size:4]          total frame length (excluding this field)
[version:1]       protocol version (1)
[type:1]          message type discriminator
[correlationId:4] request/response matching (0 for fire-and-forget)
[hopCount:1]      loop prevention (max 1)
[payload:N]       message-specific binary encoding
```

### InterBrokerRpcServer (Inbound)

```java
// ivy-broker/src/main/java/com/ivy/broker/cluster/InterBrokerRpcServer.java
// Netty ChannelInboundHandlerAdapter on inter_broker_port (main_port + 2)
void channelRead(ChannelHandlerContext ctx, InterBrokerMessage msg) {
    switch (msg) {
        case ForwardWriteRequest  r -> handleForwardWrite(ctx, r);
        case ForwardFetchRequest  r -> handleForwardFetch(ctx, r);
        case MetadataUpdateBroadcast b -> metadataImageHolder.update(b.image());
        case MetadataSyncRequest  r -> handleMetadataSync(ctx, r);
        case HeartbeatPing        p -> ctx.writeAndFlush(new HeartbeatPong(selfBrokerId));
        case FlushNotification    n -> flushEventDispatcher.dispatch(n.partitionId(), n.hwm());
        case InterestAdd          a -> subscriptionRegistry.add(a.partitionId(), a.subscriber());
        case InterestRemove       r -> subscriptionRegistry.remove(r.partitionId(), r.subscriber());
        default -> {} // response messages: handled by correlation tracker, ignored here
    }
}
```

### InterBrokerRpcClient (Outbound)

```java
// ivy-broker/src/main/java/com/ivy/broker/cluster/InterBrokerRpcClient.java
// One persistent Netty connection per peer broker, reconnects with backoff

CompletableFuture<ForwardWriteResponse> forwardWrite(
        BrokerId target, ForwardWriteRequest request)

CompletableFuture<ForwardFetchResponse> forwardFetch(
        BrokerId target, ForwardFetchRequest request)

void broadcast(InterBrokerMessage msg)   // fire-and-forget to all active peers
```

**Connection lifecycle:**
- One Netty channel per peer, persistent (no per-request connection overhead)
- Reconnect with exponential backoff: 100ms → 200ms → 400ms → ... → 30s max
- Jitter: ±25% on each interval to prevent thundering herd
- Circuit breaker: 5 consecutive failures → OPEN → 30s cooldown → HALF_OPEN

**Circuit breaker states:**
```
CLOSED ──(5 failures)──→ OPEN ──(30s)──→ HALF_OPEN
  ↑                                          │
  └──────────(1 success)────────────────────┘
```

On OPEN: `ForwardWriteRequest` falls through to `RoutingDecision.NoLeader` if owner
unreachable. Clients get `LEADER_NOT_AVAILABLE` and retry with backoff.

---

## Read Path (3-Tier)

Reads follow the same ownership logic but with a tiered fallback:

```
ReadAccumulator.fetch(partitionId, fetchOffset, maxBytes):

Tier 1 — Local LogSegment (owner or cached non-owner)
  LogSegments.floorSegment(fetchOffset) → segment file
  segmentReader.read(segment, offset, maxBytes)
  → OS page cache: ~0.1ms

  [miss] ↓

Tier 2 — Owner's LogSegment (inter-broker fetch)
  RoutingDecision for this partitionId
  If owner != self:
    ForwardFetchRequest → owner → ForwardFetchResponse(records, hwm)
    ~ 0.5ms (RPC + owner L1 hit)

  [miss or circuit open] ↓

Tier 3 — PostgreSQL (authoritative)
  SELECT offset_num, key, value, headers, timestamp_ms
  FROM messages
  WHERE partition_id = :pid AND tenant_id = :tenantId
    AND offset_num >= :from
  ORDER BY offset_num LIMIT :maxRows
  ~ 2-5ms
```

**Why non-owners may have Tier 1 hits:**
After ownership transfer, the previous owner's LogSegment stays warm (no eviction).
Consumers that were on the old owner continue getting Tier 1 hits until segments rotate.
This is desirable: it reduces Tier 2/3 pressure during rebalancing.

**Transaction isolation in reads:**
- `READ_UNCOMMITTED`: return all records including uncommitted (offset < HWM)
- `READ_COMMITTED` (default): cap at LSO (last stable offset); filter aborted transaction records
  - Fast abort scan: `idx_messages_control` index (only control records, `WHERE is_control=true`)

---

## Write Worker Dispatch Pipeline

```
Protocol Handler
  PendingWrite → DefaultBrokerEngine.write()
    ↓ (RoutingDecision.Local)
  WriteAccumulator.accumulate(write)       ← per-partition buffer
    ↓ (batch full: 1K msgs OR 1MB OR 5ms linger)
  WriteDispatcher.dispatch(batch)
    ↓ (Murmur2(partitionId) % 4 → worker index)
  WriteWorker[i].processBatch(partitionId, batch)
    ↓
  ProducerStateManager.checkSequence()     ← idempotency + epoch fencing (in-memory)
    ↓ (epoch FENCED → reject; DUPLICATE → return cached offset; OK → proceed)
  OffsetAllocator.allocate(partitionId, count)   ← AtomicLong CAS
    ↓
  PG transaction:
    UPDATE partition_offsets WHERE leader_epoch = :epoch   ← CAS epoch fence
    COPY messages FROM STDIN BINARY                        ← binary bulk insert
    UPSERT producer_state                                  ← idempotency record
    COMMIT                                                 ← ACK fires here
    ↓ (async post-ACK)
  StorageFlusher → LogSegment.append()           ← read cache (200ms batch)
  FlushEventDispatcher.dispatch(FlushEvent)      ← notify push consumers
```

---

## Namespace Strategy per Protocol

How protocol-native addresses map to `(baseName, partitionNum)`:

```java
sealed interface ParsedDestination {
    String  baseName();
    boolean hasExplicitPartition();
    int     partition();   // valid only if hasExplicitPartition() == true
}
```

| Protocol | DestinationParser | Example input | baseName | partition |
|----------|------------------|---------------|----------|-----------|
| Kafka | `KafkaDestinationParser` | topic from frame, partition from frame | topic | from Produce frame |
| AMQP 0-9-1 | `Amqp091DestinationParser` | queue = `"orders"`, routing-key = `"created"` | `orders.created` | null |
| AMQP 1.0 | `Amqp10DestinationParser` | target address = `"orders"` | `orders` | null |
| MQTT | `MqttDestinationParser` | `"sensors/temp/2"` | `sensors.temp` | `2` |
| MySQL | `SqlDestinationParser` | `"orders"` (from SQL) | `orders` | null |
| PgWire | `SqlDestinationParser` | `"orders"` (from SQL) | `orders` | null |

**MQTT shared subscriptions (MQTT 5.0):**
`$share/my-group/sensors/temp` → strip `$share/<group>/` prefix → `sensors.temp`.
The `my-group` maps to a Kafka-style consumer group via `ConsumerGroupCoordinator`.

---

## Configuration

```yaml
broker:
  cluster:
    cluster-secret: ${CLUSTER_SECRET}          # shared HMAC key for HRW scoring
    hrw-mac-pool-size: 4                        # Mac pool — matches write worker count

  write:
    worker-threads: 4                           # WriteWorker count (partition affinity)
    batch-size: 1000                            # max messages per batch
    batch-bytes: 1048576                        # max bytes per batch (1MB)
    linger-ms: 5                                # max wait before forced flush

  routing:
    partition-cache-size: 65536                 # DefaultPartitionRouter UUID cache
    auto-create-topics: true                    # AutoCreateDestinationResolver
    default-partition-count: 1                  # for auto-created topics

  rpc:
    max-in-flight: 1000                         # max concurrent forwarded requests
    request-timeout-ms: 5000
    reconnect-initial-ms: 100
    reconnect-max-ms: 30000
    circuit-breaker-failure-threshold: 5
    circuit-breaker-cooldown-ms: 30000

  metadata:
    poll-interval-ms: 30000                     # PG polling fallback (Layer 3)
    pg-notify-enabled: true                     # PG LISTEN/NOTIFY (Layer 1)
```

---

## Key Classes Summary

| Class | Module | Responsibility |
|-------|--------|---------------|
| `RouteTable` | ivy-server | Orchestrates parser + resolver + router per request |
| `DestinationParser` | ivy-protocol-* | Protocol-specific address → baseName + partition |
| `DestinationResolver` | ivy-broker | topicName → DestinationId + partitionCount |
| `MetadataBackedDestinationResolver` | ivy-broker | Implementation using MetadataManager |
| `AutoCreateDestinationResolver` | ivy-broker | Auto-create topics on first reference |
| `PartitionRouter` | ivy-common | Interface: destinationId + key → RoutedPartition |
| `DefaultPartitionRouter` | ivy-common | explicit → Murmur2(key) → round-robin |
| `Murmur2` | ivy-common | Kafka-compatible hash (partition selection + worker affinity) |
| `RoutedPartition` | ivy-common | Result: PartitionId + partitionNum |
| `HRWRouter` | ivy-broker | HMAC-SHA-256 rendezvous hash → ownerOf(partitionId) |
| `RoutingDecision` | ivy-broker | Sealed: Local / Forward(BrokerId) / NoLeader |
| `MetadataImage` | ivy-broker | Immutable ownership snapshot, VarHandle publish |
| `MetadataImageHolder` | ivy-broker | CAS update, version-ordered |
| `MetadataPoller` | ivy-broker | PG polling (Layer 3) + LISTEN/NOTIFY (Layer 1) |
| `WriteDispatcher` | ivy-broker | Murmur2(partitionId) → worker index |
| `WriteAccumulator` | ivy-broker | Per-partition batch: 1K/1MB/5ms triggers |
| `WriteWorker` | ivy-broker | PG-first write, epoch fencing, offset allocation |
| `ForwardWriteManager` | ivy-broker | Non-owner → ForwardWriteRequest → owner |
| `InterBrokerRpcServer` | ivy-broker | Inbound Netty dispatcher on inter_broker_port |
| `InterBrokerRpcClient` | ivy-broker | Outbound per-peer channels, circuit breaker |
| `InterBrokerMessage` | ivy-broker | Sealed interface: 13 message types |
| `HopCount` | ivy-common | Loop prevention (MAX_HOPS=1) |
| `ReadAccumulator` | ivy-broker | L1 LogSegment → L2 owner → L3 PG |
