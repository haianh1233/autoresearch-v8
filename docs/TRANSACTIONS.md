# Transaction Design

Transactions guarantee that a set of writes either all succeed (COMMIT) or are entirely
invisible to consumers (ABORT). Ivy supports transactions from three protocol families:

| Protocol | Transaction Model | Coordinator |
|----------|-------------------|-------------|
| Kafka | KIP-98 producer transactions — multi-partition, multi-broker | `TransactionCoordinator` (broker-side, `__transaction_state` topic) |
| AMQP 0-9-1 | Channel-level `Tx.Select/Commit/Rollback` — single channel, single broker | `Amqp091TxManager` (per-channel) |
| AMQP 1.0 | `TransactionController` link — multi-link, single session | `Amqp10TxCoordinator` (per-session) |
| MQTT | None (QoS 2 is delivery guarantee, not atomicity across messages) | — |
| HTTP | None (each POST is a single atomic write; no multi-request transactions) | — |
| MySQL / PgWire | None (read-only adapters) | — |

---

## 1. Idempotent Producers (Foundation for Kafka Transactions)

Before transactions, all Kafka producers should use idempotent delivery. Idempotency is
the foundation that makes transactions correct — without it, a network retry could produce
a duplicate message inside or outside the transaction window.

### How It Works

Each producer has a `(producerId, producerEpoch)` pair assigned by `InitProducerId`.
Each `Produce` request carries a `sequence` number per partition, starting from 0 and
incrementing by the batch size.

`WriteWorker` checks the incoming sequence against `producer_state` inside the same PG
transaction as the write:

```
Incoming (producerId=42, epoch=3, partition=P, sequence=100):

producer_state row for (42, P):
  producer_epoch = 3
  last_sequence  = 99
  last_offset    = 8005

Check:
  sequence == last_sequence + 1          → accept (normal case)
  sequence == last_sequence + 1 - N      → duplicate (N messages overlap); return cached last_offset
  sequence > last_sequence + 1           → gap → OUT_OF_ORDER_SEQUENCE error
  producer_epoch > stored epoch          → fencing: new epoch, sequence must restart from 0
  producer_epoch < stored epoch          → stale producer → INVALID_PRODUCER_EPOCH error
```

The `producer_state` upsert is inside the same PG COMMIT as the message COPY:

```sql
-- Step 3: write messages via COPY
-- Step 4: upsert producer state
INSERT INTO producer_state
  (producer_id, partition_id, producer_epoch, last_sequence, last_offset)
VALUES (:pid, :partId, :epoch, :seq, :offset)
ON CONFLICT (producer_id, partition_id) DO UPDATE
SET producer_epoch = :epoch,
    last_sequence  = :seq,
    last_offset    = :offset,
    updated_at     = now()
WHERE producer_state.producer_epoch <= :epoch;
-- Step 5: COMMIT
```

`WHERE producer_epoch <= :epoch` prevents a stale Produce request from overwriting a newer
epoch's state (can happen if a network partition delivers old packets after failover).

### ProducerId Allocation

`TransactionCoordinator.initProducerId()`:
1. Load the current max `producer_id` from PG (`SELECT MAX(producer_id) FROM producer_state`).
2. Increment by 1 and return the new ID with epoch 0.
3. If a `txnId` is provided, look up the existing `(producerId, epoch)` for that `txnId`
   and bump the epoch by 1 (producer restart within same transaction ID).

`ProducerId` is a `BIGINT`; the broker does not reuse IDs.

---

## 2. Kafka Transactions (KIP-98)

### State Machine

```
[client calls InitProducerId(txnId)]
         ↓
    EMPTY (no transaction)
         │
         ├── [AddPartitionsToTxn]
         ↓
       ONGOING
         │
         ├── [Produce(transactional=true)]     ← messages visible to READ_UNCOMMITTED only
         │
         ├── [TxnOffsetCommit]                 ← consumer offsets committed inside txn
         │
         ├── [EndTxn(COMMIT)]
         │       ↓
         │   PREPARE_COMMIT → write commit control records → COMPLETE_COMMIT
         │       → LSO advances past this transaction
         │       → messages now visible to READ_COMMITTED consumers
         │
         └── [EndTxn(ABORT)]
                 ↓
             PREPARE_ABORT → write abort control records → COMPLETE_ABORT
                 → messages permanently hidden from all consumers
```

### API Flow (KIP-98 wire protocol)

**Step 1: InitProducerId** (API key 22)
```
C→S: InitProducerIdRequest(transactionalId="order-processor", transactionTimeoutMs=60000)
S→C: InitProducerIdResponse(producerId=42, producerEpoch=0)
```
- If `transactionalId` is new: allocate fresh `(producerId, epoch=0)`
- If `transactionalId` exists: bump `epoch` (fences previous instance of same `txnId`)
- Stored in `transactions` table: `(txnId, producerId=42, epoch=0, state=EMPTY)`

**Step 2: AddPartitionsToTxn** (API key 24)
```
C→S: AddPartitionsToTxnRequest(transactionalId="order-processor",
                                producerId=42, producerEpoch=0,
                                topics=[{name="orders", partitions=[0,2]},
                                        {name="payments", partitions=[1]}])
S→C: AddPartitionsToTxnResponse(results=[{NONE, NONE, NONE}])
```
Updates `transactions.partitions = ARRAY[<orders/0 UUID>, <orders/2 UUID>, <payments/1 UUID>]`
and transitions state `EMPTY → ONGOING`.

**Step 3: Produce (transactional=true)**
```
C→S: ProduceRequest(transactionalId="order-processor", producerId=42, producerEpoch=0,
                    acks=-1,
                    records=[{partition=0, batch=[...]}])
```
`PendingWrite.isTransactional = true`. Message is written normally via `WriteAccumulator`.
The `lso` of the partition does NOT advance past this message — it is invisible to
`READ_COMMITTED` consumers until the transaction commits.

**Step 4: TxnOffsetCommit** (API key 28, optional)
```
C→S: TxnOffsetCommitRequest(transactionalId, producerId, producerEpoch,
                              groupId="order-consumers",
                              offsets=[{topic="orders", partition=0, offset=500}])
```
Saves offsets into `consumer_offsets` inside the same transaction (atomically with the produces).
The consumer offset is only visible after `EndTxn(COMMIT)`.

**Step 5: EndTxn** (API key 26)
```
C→S: EndTxnRequest(transactionalId="order-processor", producerId=42, producerEpoch=0,
                   committed=true)   ← or false for abort
S→C: EndTxnResponse(errorCode=NONE)
```

On COMMIT:
1. `TransactionCoordinator` transitions state to `PREPARE_COMMIT`.
2. For each partition in `transactions.partitions`:
   - Write a **commit control record** (special message with `isControl=true, transactionResult=COMMIT`)
   - These are ordinary rows in `messages` with a flag; consumer applies them as a "fence"
3. Transition to `COMPLETE_COMMIT`.
4. Advance `partition_offsets.lso` past all offsets belonging to this transaction.
5. `FlushEventDispatcher.dispatch()` wakes up long-polling `READ_COMMITTED` consumers.

On ABORT:
1. `TransactionCoordinator` transitions state to `PREPARE_ABORT`.
2. Write an **abort control record** to each partition.
3. Register the aborted range in `AbortedTransactionIndex`.
4. Transition to `COMPLETE_ABORT`.

### Control Records

Control records are marker messages that signal transaction boundaries to consumers.
They are stored as normal rows in `messages` with a reserved protocol_id = 0 and flags:

```
messages row (control record):
  partition_id  = <partition UUID>
  offset_num    = <next offset after the last transactional message>
  protocol_id   = 0           ← reserved for control records
  key           = <producerId:producerEpoch>
  value         = <1 byte: 1=COMMIT, 0=ABORT>
  headers       = null
  is_control    = true        ← new column in V2 schema migration
```

`FetchHandler` strips control records from consumer responses (they are internal markers,
not user messages). They are used only by the abort scan logic in `ReadAccumulator`.

### LSO (Last Stable Offset)

The **last stable offset** (LSO) is the highest offset that a `READ_COMMITTED` consumer
may safely read up to. It is the offset of the first message in the oldest open transaction
on a given partition (or the high watermark if no transactions are open).

```
partition P has messages at offsets: 0 1 2 3 4 5 6 7 8 9 10 11 ...
                                               │       │
                                    txn A starts=3     txn B starts=7
                                    txn A in progress, txn B committed

LSO of P = 3  (offset of the first message in the oldest open txn A)

READ_UNCOMMITTED consumer sees: offsets 0..11 (all messages)
READ_COMMITTED consumer sees:   offsets 0..2  (stops at LSO-1)
```

`partition_offsets.lso` is updated by `TransactionCoordinator` when:
- A transaction COMMITs or ABORTs → re-compute the new LSO as `MIN(ongoing txn first offsets)`
- No ongoing transactions → `lso = next_offset` (LSO == HWM)

### AbortedTransactionIndex

Consumers using `READ_COMMITTED` must know which offset ranges to skip (aborted messages).
`AbortedTransactionIndex` maps `partitionId → SortedList<AbortedTxnRange>` where:

```java
record AbortedTxnRange(long producerId, long firstOffset, long lastOffset) {}
```

Built by scanning the `transactions` table for `state = 'COMPLETE_ABORT'`.

**ReadAccumulator** (for `READ_COMMITTED` fetches) attaches `abortedTransactions` to the
`FetchResult`. The Kafka `FetchResponse` includes this list and the Kafka client library
filters out aborted messages client-side.

For non-Kafka consumers (AMQP, HTTP with `isolation=read_committed`), the broker filters
aborted messages server-side before returning them.

### Transaction Timeout

`transactions.timeout_ms` (default: 60,000 ms, set by `InitProducerId`).

`TransactionExpiryReaper` runs every 10 seconds:
```
For each transaction in state ONGOING or PREPARE_*:
  if (now() - created_at > timeout_ms):
    → force ABORT
    → write abort control records
    → transition to COMPLETE_ABORT
    → update LSO
```

Producer receives `INVALID_PRODUCER_EPOCH` on the next `Produce` or `EndTxn` call after
the timeout has expired.

### ProducerEpoch Fencing

When a producer restarts with the same `transactionalId`, it calls `InitProducerId` again.
The coordinator bumps the epoch:
```
stored (producerId=42, epoch=3) → new call InitProducerId("order-processor")
→ returns (producerId=42, epoch=4)
→ any in-flight Produce or EndTxn from epoch=3 is rejected with INVALID_PRODUCER_EPOCH
→ open transaction for epoch=3 is aborted
```

This prevents **zombie producers** — stale instances of a producer that reconnect after a
network partition and try to complete a transaction that the new instance has already taken over.

---

## 3. AMQP 0-9-1 Transactions (Channel-Level)

AMQP 0-9-1 transactions are simpler than Kafka transactions: they are channel-scoped and
single-broker (no distributed coordination).

### Wire Flow

```
C→S: Tx.Select          ← "I want this channel to use transactions"
S→C: Tx.SelectOk

C→S: Basic.Publish(...)  ← buffered, not yet committed
C→S: Basic.Publish(...)
C→S: Basic.Publish(...)

C→S: Tx.Commit           ← "commit all the above publishes as one atomic batch"
S→C: Tx.CommitOk
     [broker calls BrokerEngine.write() for all buffered messages in one batch]
     [WriteAccumulator batches them → single WriteWorker PG transaction → all offsets allocated atomically]

     OR

C→S: Tx.Rollback         ← discard buffered messages
S→C: Tx.RollbackOk
```

### Ivy Implementation

`Amqp091TxManager` is a per-channel component (`Amqp091ChannelState.txManager`):

```java
class Amqp091TxManager {
    private final List<PendingWrite> buffer = new ArrayList<>();
    private boolean txActive = false;

    void txSelect()                     // enable tx mode; set txActive=true
    void buffer(PendingWrite write)     // collect messages during tx
    WriteResult txCommit(BrokerEngine)  // flush buffer as single batch
    void txRollback()                   // clear buffer
}
```

**Atomicity guarantee:** all buffered messages are passed to `BrokerEngine.write()` in a
single call. `WriteWorker` processes the entire batch in one PG transaction — either all
messages get offsets and are committed, or none do.

**AMQP Tx does not span partitions atomically in the Kafka sense.** If the batch spans
multiple partitions on different brokers, `ForwardWriteManager` sends them in separate
forwarded writes. This means AMQP Tx provides best-effort all-or-nothing within a single
broker; across brokers it is at-least-once on the write path.

**Acks inside a Tx:** consumer acks (`Basic.Ack`) received during an active transaction are
also buffered. `Tx.Commit` flushes both produces and acks atomically.

### Interaction with Publisher Confirms

`Tx.Select` and `Confirm.Select` are **mutually exclusive** on the same channel.
Enabling both returns `CHANNEL_ERROR: cannot use Tx and Confirm.Select on the same channel`.

---

## 4. AMQP 1.0 Transactions (TransactionController Link)

AMQP 1.0 transactions are more powerful than 0-9-1: they can span multiple links within
a session and support both transactional sends and transactional acks.

### TransactionController Link

To use transactions, the client opens a special **coordinator link** with a well-known
target address `"coordinator"`:

```
C→S: Attach(
       name="txn-coordinator",
       role=sender,
       target=Coordinator(capabilities=[OASIS_AMQP_TXN])
     )
S→C: Attach(role=receiver, ...)
```

All transaction operations flow as `Transfer` frames on this link, with `Declare` and
`Discharge` as the frame types.

### Declare (Begin Transaction)

```
C→S: Transfer(handle=<coordinator link>, delivery-tag=X,
              message=Declare{})
S→C: Disposition(role=receiver, settled=true,
                 state=Declared{txn-id=<opaque bytes>})
```

The broker allocates a `txnId` (UUID bytes) and stores it in the session's
`Amqp10TxCoordinator`:

```java
record Amqp10Transaction(
    TxnId     txnId,
    TenantId  tenantId,
    Instant   createdAt,
    List<PendingWrite> pendingWrites,    // from transactional sends
    List<OffsetCommit> pendingAcks       // from transactional dispositions
)
```

### Transactional Send

Sender link attaches a `txn-id` to each `Transfer`:

```
C→S: Transfer(handle=<sender link>,
              state=TransactionalState{txn-id=<txnId>, outcome=Accepted{}},
              payload=<message>)
```

`Amqp10SenderLinkHandler` detects the `TransactionalState` wrapper and calls
`Amqp10TxCoordinator.addPendingWrite(txnId, pendingWrite)` instead of writing immediately.

### Transactional Ack (Disposition)

Consumer links also participate in transactions:

```
C→S: Disposition(role=receiver,
                 state=TransactionalState{txn-id=<txnId>, outcome=Accepted{}},
                 first=D, last=D)
```

`Amqp10ReceiverLinkHandler` adds a deferred `OffsetCommit` to the transaction.

### Discharge (Commit / Abort)

```
-- Commit:
C→S: Transfer(handle=<coordinator link>, delivery-tag=Y,
              message=Discharge{txn-id=<txnId>, fail=false})
S→C: Disposition(settled=true, state=Accepted{})

-- Abort:
C→S: Transfer(handle=<coordinator link>, delivery-tag=Y,
              message=Discharge{txn-id=<txnId>, fail=true})
S→C: Disposition(settled=true, state=Accepted{})
```

On `Discharge(fail=false)` (commit):
1. `Amqp10TxCoordinator` flushes `pendingWrites` to `BrokerEngine.write()` in one batch.
2. Flushes `pendingAcks` as offset commits.
3. Clears the transaction state.

On `Discharge(fail=true)` (abort):
1. `pendingWrites` and `pendingAcks` are discarded.
2. Transaction state cleared.

### Error: Unknown TxnId

If the client sends a `Discharge` or transactional `Transfer` with an unknown `txnId`:

```
S→C: Disposition(settled=true, state=Rejected{
       error={condition="amqp:transaction-timeout",
              description="transaction not found or expired"}})
```

### Transaction Timeout (AMQP 1.0)

`Amqp10TxCoordinator` cancels a transaction if no `Discharge` is received within
`txnTimeoutMs` (default: 60s). On timeout, all pending writes are discarded and the
transactional link's delivery receives an error disposition.

---

## 5. Isolation Levels

All protocols that can consume messages support one of two isolation levels:

| Level | What is visible | Applicable protocols |
|-------|----------------|---------------------|
| `READ_UNCOMMITTED` | All messages, including those in open Kafka transactions | MQTT, AMQP (non-tx), HTTP (default) |
| `READ_COMMITTED` | Only committed messages (LSO-bounded) | Kafka (default), AMQP (configurable), HTTP (`?isolation=read_committed`) |

### How `READ_COMMITTED` Works in ReadAccumulator

```
fetch(partitionId, startOffset, maxRecords, READ_COMMITTED):

1. Load LSO from partition_offsets.lso
2. Cap fetch range: effectiveMaxOffset = min(requestedMaxOffset, LSO)
3. Read messages up to effectiveMaxOffset from LogSegment or PG
4. Load AbortedTransactionIndex for [startOffset, effectiveMaxOffset]
5. Filter out messages whose (producerId, offset) fall within any aborted range
6. Return remaining messages + highWatermark + LSO + abortedTransactions list
```

For Kafka clients, the `FetchResponse` includes the `abortedTransactions` list;
the client library does its own filtering (double-filtering is safe — idempotent).

For non-Kafka consumers (AMQP, HTTP), the broker filters server-side and never returns
aborted messages. The `abortedTransactions` metadata is omitted from the response.

---

## 6. Cross-Protocol Transactions

### Kafka → SQL Consumer

A Kafka producer runs a transaction across `orders` and `payments` partitions.
An analyst queries via MySQL:

```sql
SELECT * FROM orders WHERE offset_num BETWEEN 100 AND 200;
```

The MySQL handler calls `BrokerEngine.fetch()` with `isolation=READ_COMMITTED` by default.
The query returns only committed messages — if the Kafka transaction is still `ONGOING`,
those offsets are beyond the LSO and are excluded.

### AMQP Produce → Kafka Consume

AMQP 0-9-1 `Tx.Commit` flushes all messages in one `WriteWorker` batch. From a Kafka
consumer's perspective, those messages arrive as a contiguous block at committed offsets
with `isTransactional=false` (AMQP Tx is not a Kafka transaction; the messages are
immediately visible to `READ_COMMITTED` consumers once committed).

### HTTP Produce (no transactions)

HTTP `POST /topics/{topic}/messages` writes a single message per request. It is equivalent
to a non-transactional Kafka produce with `acks=-1`. No cross-request atomicity is provided.

For atomic multi-message HTTP writes, use **batch produce**
(`POST /topics/{topic}/messages/batch`). The entire batch is processed by a single
`WriteWorker` invocation inside one PG COMMIT — all messages get contiguous offsets or
none are written.

---

## 7. TransactionCoordinator

`TransactionCoordinator` lives in `ivy-broker` and is shared across all protocols.

```java
interface TransactionCoordinator {
    // Kafka API
    InitProducerIdResult initProducerId(TenantId, String txnId, int timeoutMs);
    void addPartitionsToTxn(TenantId, String txnId, long producerId, short epoch,
                            List<PartitionId> partitions);
    void endTxn(TenantId, String txnId, long producerId, short epoch, boolean commit);
    void txnOffsetCommit(TenantId, String txnId, long producerId, short epoch,
                         GroupId groupId, Map<PartitionId, OffsetAndMetadata> offsets);

    // AMQP 1.0 API (called by Amqp10TxCoordinator)
    TxnId declareTxn(TenantId tenantId);
    void dischargeTxn(TenantId tenantId, TxnId txnId, boolean fail);

    // Internal
    long lsoFor(TenantId tenantId, PartitionId partitionId);
    List<AbortedTxnRange> abortedRanges(TenantId, PartitionId, long fromOffset, long toOffset);
}
```

### `FindCoordinator`

For Kafka clients, the coordinator for a given `txnId` is the broker with:
```
hash(txnId) % numBrokers → broker index → brokerId from MetadataImage
```
This is deterministic given a stable `MetadataImage`. The coordinator for
`txnId = "order-processor"` does not change unless brokers are added or removed.

If the broker receiving the `InitProducerId` call is NOT the coordinator for that `txnId`,
it forwards the request to the coordinator via `InterBrokerRpcClient`.

---

## 8. PostgreSQL Schema for Transactions

```sql
-- transactions table (see POSTGRES_SCHEMA.md for full DDL)
CREATE TABLE transactions (
  txn_id          TEXT      NOT NULL,
  producer_id     BIGINT    NOT NULL,
  producer_epoch  SMALLINT  NOT NULL,
  tenant_id       UUID      NOT NULL,
  state           TEXT      NOT NULL
                    CHECK (state IN (
                      'EMPTY',
                      'ONGOING',
                      'PREPARE_COMMIT',
                      'PREPARE_ABORT',
                      'COMPLETE_COMMIT',
                      'COMPLETE_ABORT')),
  timeout_ms      INT       NOT NULL DEFAULT 60000,
  partitions      UUID[]    NOT NULL DEFAULT '{}',  -- PartitionIds involved
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (txn_id, tenant_id)
);

-- Sparse index for fast LSO computation:
-- Only rows in active states need to be scanned for LSO
CREATE INDEX idx_transactions_active
  ON transactions(tenant_id, producer_id)
  WHERE state IN ('ONGOING', 'PREPARE_COMMIT', 'PREPARE_ABORT');

-- messages: control records use is_control=true (V2 migration)
ALTER TABLE messages ADD COLUMN is_control BOOLEAN NOT NULL DEFAULT false;

-- partition_offsets: LSO tracked here for fast per-partition reads
-- (lso already in schema; refreshed by TransactionCoordinator on every state change)
```

### State Transitions in SQL

```sql
-- AddPartitionsToTxn: EMPTY → ONGOING
UPDATE transactions
SET state = 'ONGOING',
    partitions = partitions || :newPartitionIds,
    updated_at = now()
WHERE txn_id = :txnId AND tenant_id = :tenantId
  AND producer_epoch = :epoch
  AND state IN ('EMPTY', 'ONGOING');

-- EndTxn(commit): ONGOING → PREPARE_COMMIT
UPDATE transactions
SET state = 'PREPARE_COMMIT', updated_at = now()
WHERE txn_id = :txnId AND tenant_id = :tenantId
  AND producer_epoch = :epoch AND state = 'ONGOING';

-- After writing commit control records: PREPARE_COMMIT → COMPLETE_COMMIT
UPDATE transactions
SET state = 'COMPLETE_COMMIT', updated_at = now()
WHERE txn_id = :txnId AND tenant_id = :tenantId;

-- LSO refresh after transaction completes:
UPDATE partition_offsets
SET lso = (
  SELECT COALESCE(MIN(first_offset_in_txn), next_offset)
  FROM (
    -- find the first offset of each ongoing transaction on this partition
    SELECT MIN(m.offset_num) AS first_offset_in_txn
    FROM messages m
    JOIN transactions t ON m.producer_id = t.producer_id
                        AND m.partition_id = :partitionId
    WHERE t.state IN ('ONGOING','PREPARE_COMMIT','PREPARE_ABORT')
      AND t.tenant_id = :tenantId
      AND :partitionId = ANY(t.partitions)
    GROUP BY t.txn_id
  ) sub
)
WHERE partition_id = :partitionId;
```

---

## 9. Rules and Invariants

**T1. ACK fires only after PREPARE_COMMIT/COMPLETE_COMMIT, never during ONGOING.**
Kafka `Produce(transactional=true)` with `acks=-1` receives its ACK immediately after the
PG COPY commit (the message is durable), but the message is invisible to `READ_COMMITTED`
consumers until `EndTxn(COMMIT)`. Do not confuse durability ACK with transaction COMMIT.

**T2. LSO must be recomputed on every transaction state change.**
Any transition out of `ONGOING`, `PREPARE_COMMIT`, or `PREPARE_ABORT` must trigger an LSO
refresh for all affected partitions. Stale LSO blocks consumers unnecessarily.

**T3. Control records are not delivered to consumers.**
`FetchHandler` and `ReadAccumulator` must filter `is_control=true` rows before encoding the
response. Control records are broker-internal markers only.

**T4. Epoch fencing prevents zombie producers from committing aborted transactions.**
`TransactionCoordinator.endTxn()` validates `(producerId, producerEpoch)` against the stored
epoch. A stale epoch returns `INVALID_PRODUCER_EPOCH` — the zombie cannot commit.

**T5. AMQP Tx does not participate in Kafka LSO tracking.**
AMQP 0-9-1 and 1.0 transactions write messages with `isTransactional=false` from the broker's
perspective. They are committed in a single batch and are immediately visible to all consumers.
The LSO is not affected by AMQP Tx activity.

**T6. HTTP batch produce is atomic but not transactional.**
All messages in a batch receive contiguous offsets in one PG COMMIT. They are all immediately
visible to `READ_COMMITTED` consumers. There is no concept of ABORT for HTTP batch writes.

**T7. TxnOffsetCommit offsets are not visible until EndTxn(COMMIT).**
Offsets committed via `TxnOffsetCommit` are stored with a `pending=true` flag in
`consumer_offsets`. `OffsetFetch` returns the last stable (non-pending) committed offset
until the transaction commits.

---

## 10. Implementation Checklist

### Phase 1 (single-broker, Kafka)
- [ ] `TransactionCoordinator` — `initProducerId`, `addPartitionsToTxn`, `endTxn`, `txnOffsetCommit`
- [ ] `TransactionExpiryReaper` — 10s background scan, force-abort timed-out transactions
- [ ] `AbortedTransactionIndex` — in-memory sorted list per partition, rebuilt from PG on start
- [ ] `WriteWorker` — `isTransactional=true` path (no LSO advance on write)
- [ ] `ReadAccumulator` — `READ_COMMITTED` path: LSO cap + abort filter
- [ ] V1 schema — `transactions` table, `is_control` column on `messages`, `lso` on `partition_offsets`
- [ ] `KafkaInitProducerIdHandler`, `KafkaAddPartitionsHandler`, `KafkaEndTxnHandler`, `KafkaTxnOffsetCommitHandler`
- [ ] E2E: `KafkaTransactionE2E` — produce transactionally, commit; READ_COMMITTED consumer sees, READ_UNCOMMITTED sees immediately
- [ ] E2E: `KafkaAbortTransactionE2E` — produce, abort; no consumer sees aborted messages
- [ ] E2E: `KafkaZombieProducerE2E` — epoch fencing prevents stale producer from committing

### Phase 2 (AMQP)
- [ ] `Amqp091TxManager` — `txSelect`, `buffer`, `txCommit`, `txRollback` (per-channel)
- [ ] `Amqp10TxCoordinator` — `declare`, `addPendingWrite`, `addPendingAck`, `discharge` (per-session)
- [ ] E2E: `Amqp091TxE2E` — Tx.Select → 3x Publish → Tx.Commit; all 3 visible atomically
- [ ] E2E: `Amqp091TxRollbackE2E` — Tx.Select → Publish → Tx.Rollback; nothing visible
- [ ] E2E: `Amqp10TxE2E` — Declare → 2x Transfer(transactional) → Discharge(commit)
- [ ] E2E: `AmqpTxKafkaConsumeE2E` — AMQP Tx.Commit → Kafka consumer sees all messages in commit order
