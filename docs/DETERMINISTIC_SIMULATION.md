# Deterministic Simulation (FoundationDB-Style)

> **Related:** [RULES.md](RULES.md) §R19, [TESTING_STRATEGY.md](TESTING_STRATEGY.md),
> [OBSERVABILITY.md](OBSERVABILITY.md)

---

## Overview

Ivy's deterministic simulation framework enables **reproducible testing of distributed system
invariants** by replacing all I/O with deterministic abstractions. Inspired by FoundationDB's
simulation testing: given a seed, the same execution produces the same results every time.

**Key insight:** If all I/O is abstracted (clock, network, storage), a simulator can explore
millions of execution orderings per second — vastly more than chaos testing.

---

## Environment Abstraction

All production code uses `Environment` instead of direct system calls (Rule R19):

```java
interface Environment {
    Clock clock();              // replaces System.currentTimeMillis(), Instant.now()
    RandomSource randomSource(); // replaces Math.random(), ThreadLocalRandom
    IdGenerator idGenerator();   // replaces UUID.randomUUID()
    Scheduler scheduler();       // replaces Thread.sleep(), ScheduledExecutorService
}

interface Clock {
    long currentTimeMillis();
    long nanoTime();
}

interface IdGenerator {
    UUID nextUUID();
}

interface RandomSource {
    RandomGenerator random();
}

interface Scheduler {
    Cancellable schedule(Runnable task, long delay, TimeUnit unit);
    Cancellable scheduleAtFixedRate(Runnable task, long initialDelay, long period, TimeUnit unit);
}
```

### Production Implementation

```java
class SystemEnvironment implements Environment {
    Clock clock()         → System.currentTimeMillis(), System.nanoTime()
    IdGenerator idGen()   → UUID.randomUUID()
    RandomSource random() → ThreadLocalRandom.current()
    Scheduler scheduler() → ScheduledThreadPoolExecutor
}
```

### Test Implementation

```java
class TestEnvironment implements Environment {
    TestClock clock           = new TestClock();           // advanceMillis(ms), setMillis(ms)
    TestIdGenerator idGen     = new TestIdGenerator();     // sequential: UUID(0,1), UUID(0,2), ...
    TestRandomSource random   = new TestRandomSource(42L); // seeded RNG
    TestScheduler scheduler   = new TestScheduler();       // runPending(), pendingCount()
}
```

---

## Seed-Based Execution

```java
record Seed(long value) {
    Random rng()                    → new Random(value)
    boolean shouldInject(int step)  → deterministic per step (15% rate)
    Fault nextFault(int step)       → deterministic fault selection
    int nextInt(int bound)          → deterministic parameter value
}
```

**Guarantee:** Given seed `0x1A2B3C4D` and identical configuration, the simulator produces
identical results every time. If an invariant violation occurs at step 847,362 with that seed,
re-running reproduces it exactly.

---

## Simulator Engine

```java
class SimulatorEngine {
    Seed seed;
    SimulatedClock clock;
    SimulatedNetwork network;
    SimulatedPg pg;
    List<SimulatedBroker> brokers;
    FaultInjector faultInjector;
    InvariantOracle oracle;
    PriorityQueue<Event> eventQueue;     // ordered by simulated time

    void run(int maxSteps) {
        for (int step = 0; step < maxSteps; step++) {
            Event next = eventQueue.poll();
            clock.advanceTo(next.time());
            next.execute();
            oracle.checkAll();                // EVERY step, not just at end
            faultInjector.maybeInject(seed, step);
        }
    }
}
```

**Why single-threaded:**
- No real threads → no context switching non-determinism
- Concurrency simulated by event interleaving
- Millions of operations per second (vs ~1000/sec with real PG)
- No OS scheduler variability

---

## Simulated Components

### SimulatedPg (In-Memory SQL)

```java
class SimulatedPg {
    // Ownership (partition assignments)
    Optional<OwnershipRow> getOwnership(tenant, topic, partition);
    boolean casClaimOwnership(tenant, topic, partition, brokerId, expectedEpoch);

    // Messages (produce/fetch)
    long writeMessage(tenant, topic, partition, brokerId, epoch, key, value);
    List<Record> fetchMessages(tenant, topic, partition, startOffset, limit);

    // Fault injection
    void setLatency(Duration d);        // delay before every operation
    void setUnavailable(boolean b);     // reject all operations
    void setPartialCommit(boolean b);   // commit but throw exception
}
```

### SimulatedNetwork

```java
class SimulatedNetwork {
    void send(BrokerId from, BrokerId to, Message msg);
    // Configurable: delay, loss, reorder, partition

    void partition(BrokerId a, BrokerId b);    // block messages between a↔b
    void heal(BrokerId a, BrokerId b);         // restore communication
}
```

---

## Fault Injection

### Fault Types

| Fault | Description | Parameter |
|-------|-------------|-----------|
| `KILL_BROKER` | Immediately crash a broker | random broker selection |
| `RESTART_BROKER` | Restart a dead broker | — |
| `PARTITION_NETWORK` | Block messages between two brokers | random pair |
| `HEAL_NETWORK` | Restore communication | — |
| `SLOW_PG` | Add latency to PG operations | 0–500ms from seed |
| `PG_DOWN` | Make PG unavailable | — |
| `PG_RECOVER` | Restore PG | — |
| `CLOCK_SKEW` | Advance clock by up to 60 seconds | — |

### Injection Rate

- **15% per step** — balances stress with stability
- Too high (>30%) → PG down most of the time, durability uncheckable
- Too low (<5%) → system not stressed enough

```java
void maybeInject(Seed seed, int step) {
    if (seed.shouldInject(step)) {                // 15% probability
        Fault fault = seed.nextFault(step);       // deterministic from seed
        switch (fault) {
            case KILL_BROKER      -> killRandomBroker();
            case PARTITION_NETWORK -> partitionRandomPair();
            case SLOW_PG          -> pg.setLatency(Duration.ofMillis(seed.nextInt(500)));
            // ...
        }
    }
}
```

---

## Invariant Oracle

Checked after **every single step** (not just at end):

| Invariant | Description |
|-----------|-------------|
| **Single-writer** | At most one broker holds a write lock per partition |
| **Monotonic offsets** | Per-partition offsets are sequential (no gaps) |
| **Epoch monotonicity** | Epochs never decrease |
| **Durability** | Every ACK'd write exists in SimulatedPg (skip when PG unavailable) |
| **No stale reads** | Fetching an ACK'd offset returns the correct data |
| **Ownership consistency** | At most one broker per partition in ownership table |

**Violation reporting:**
```
INVARIANT VIOLATION at step 847362 with seed 0x1A2B3C4D
  Invariant: DURABILITY
  Partition: e7b0c2a1-..., Offset: 42
  Expected: write present in SimulatedPg
  Actual: write missing
  Broker: broker-2, Epoch: 3
```

Developer re-runs with seed `0x1A2B3C4D` → exact same failure, every time.

---

## Performance

| Configuration | Steps | Duration | Coverage |
|--------------|-------|----------|----------|
| 100 seeds × 1K steps | 100K | ~5 sec | Quick smoke test |
| 1 seed × 120K steps | 120K | ~5 sec | Deep single-path exploration |
| 10,000 seeds × 100K steps | 1B | ~hours | Nightly CI (comprehensive) |

**Comparison with chaos testing:**
- Chaos test: ~100 executions per hour, probabilistic failures
- Simulation: ~120K steps per 5 seconds, deterministic reproduction

---

## What Simulation Finds (That Chaos Cannot)

| Scenario | Chaos Test | Deterministic Simulation |
|----------|-----------|------------------------|
| Specific thread interleaving (~1/10^6 probability) | Unlikely | Always, if seed covers it |
| Heartbeat expiry + write + rebalance timing | Random | Exact timing controlled |
| PG commits but TCP ACK lost → broker retries → double write | Very unlikely | Simulated directly |
| Clock skew causing epoch confusion | Hard to inject | Clock.advanceTo(skewed) |
| Message reordering in inter-broker RPC | OS-dependent | SimulatedNetwork.reorder() |

---

## Regression Testing

```
seeds/regression.txt:
  0x1A2B3C4D    # found durability bug 2026-03-15
  0x5E6F7A8B    # found epoch monotonicity bug 2026-03-18
  0xDEADBEEF    # found single-writer violation 2026-03-20
```

Every release must pass all regression seeds. New bugs → add seed to file.

---

## ivy-sim Module Structure

```
ivy-sim/ (Maven module, test scope only)
  src/main/java/com/ivy/sim/
    SimulatorEngine.java         — main loop: event queue → execute → check → inject
    SimulatedClock.java          — advanceTo(time), no real time passes
    SimulatedNetwork.java        — send/receive with delay/loss/reorder/partition
    SimulatedPg.java             — in-memory ownership + messages tables
    SimulatedBroker.java         — broker lifecycle backed by simulated I/O
    FaultInjector.java           — deterministic fault scheduling from seed
    InvariantOracle.java         — 6 invariants checked after every step
    Seed.java                    — reproducible seed for deterministic replay
    SimulationReport.java        — steps explored, faults injected, invariants checked
```

---

## Seams in Ivy (How Production Code Connects to Simulator)

| Seam | Production | Simulation |
|------|-----------|------------|
| Clock & random | `SystemEnvironment` | `SimulatedClock`, seeded `Random` |
| PG access | `PgConnectionFactory` | `SimulatedPg` |
| Inter-broker RPC | `InterBrokerClient` | `SimulatedNetwork` |
| Scheduling | `ScheduledExecutorService` | `SimulatorEngine.eventQueue` |

Same production code logic (ownership CAS, epoch fencing, offset allocation) runs through
simulated I/O. If simulation finds a bug, that bug exists in production.

---

*Last updated: 2026-03-25*
