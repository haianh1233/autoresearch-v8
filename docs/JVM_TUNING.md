# JVM Flags & Tuning

> **Related:** [OBSERVABILITY.md](OBSERVABILITY.md) §JVM Metrics, [RULES.md](RULES.md) §R19

---

## Java 26 Preview Features

Ivy requires `--enable-preview` for these JEPs:

| JEP | Feature | Usage in Ivy |
|-----|---------|-------------|
| 401 | Value Classes | `value record TenantId`, `PartitionId`, etc. — zero allocation on hot paths via JIT scalarization |
| 525 | Structured Concurrency | `StructuredTaskScope` in ReadAccumulator for multi-partition parallel reads (5s timeout, partial OK) |
| 506 | ScopedValue | `ScopedValue<TenantContext>` for virtual-thread-safe tenant propagation (replaces ThreadLocal) |
| 450 | Compact Object Headers | `-XX:+UseCompactObjectHeaders` — 12→8 byte headers, 10-20% live data reduction |

```xml
<!-- pom.xml -->
<java.version>26</java.version>
<maven.compiler.compilerArgs>--enable-preview</maven.compiler.compilerArgs>
<argLine>--enable-preview</argLine>
```

---

## Production JVM Configuration

```bash
# Preview features
--enable-preview

# GC: Shenandoah (sub-ms pause target)
-XX:+UseShenandoahGC

# Heap: fixed size (avoid resize pauses)
-Xms2g -Xmx2g

# Direct memory: Netty buffer limit
-XX:MaxDirectMemorySize=512m

# Compact headers: 10-20% memory savings
-XX:+UseCompactObjectHeaders

# Diagnostics
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/ivy/
-XX:+ExitOnOutOfMemoryError

# GC logging
-Xlog:gc*:file=/var/log/ivy/gc.log:time,uptime,level,tags:filecount=5,filesize=20m

# Netty: pooled allocator, leak detection off
-Dio.netty.allocator.type=pooled
-Dio.netty.leakDetection.level=disabled

# JCTools / Netty unsafe access (required)
--sun-misc-unsafe-memory-access=warn
```

---

## Development JVM Configuration

```bash
--enable-preview
-XX:+UseZGC                            # ZGC if Shenandoah unavailable in EA builds
-Xms512m -Xmx2g
-XX:MaxDirectMemorySize=256m
-Dio.netty.leakDetection.level=PARANOID  # detect buffer leaks
-Dio.netty.allocator.type=unpooled       # easier leak tracking
```

---

## Garbage Collector

### Shenandoah (Primary)

- Sub-millisecond pause target
- Concurrent mark, concurrent evacuation, concurrent update-refs
- Adaptive heuristics (auto-triggers based on heap pressure)
- Alert threshold: pause > 10ms → indicates GC pressure

### ZGC (Fallback)

- Used when Shenandoah unavailable (Valhalla EA builds)
- Also sub-millisecond characteristics
- `-XX:+UseZGC -XX:+ZGenerational` (generational ZGC in JDK 21+)

### Why Fixed Heap (`-Xms == -Xmx`)

Avoids GC-triggered heap resize pauses:
- Resize causes full-GC-like stop-the-world event
- Fixed heap eliminates this entirely
- Trade-off: memory reserved upfront (not available to other processes)

---

## Memory Layout

```
Total container memory: 3 Gi
  ├── Heap (-Xmx):           2 Gi   (66%)
  ├── Direct buffers:         512 Mi  (17%)  — Netty I/O buffers
  ├── Metaspace:              128 Mi  (4%)   — class metadata
  ├── Thread stacks:          128 Mi  (4%)   — platform threads (4 WriteWorkers + Netty)
  ├── GC overhead:            128 Mi  (4%)   — Shenandoah regions, marking bitmaps
  └── OS/misc:                128 Mi  (4%)   — mmap'd OffsetIndex, file handles
```

---

## Container-Aware Settings (Kubernetes)

```yaml
# K8s pod spec
resources:
  requests:
    memory: 2.5Gi
  limits:
    memory: 3Gi
```

Java 26 auto-detects container limits (`-XX:+UseContainerSupport` enabled by default since Java 10).

Alternative to explicit `-Xmx`:
```bash
-XX:InitialRAMPercentage=50.0    # use 50% of container limit for heap
-XX:MaxRAMPercentage=66.0        # cap at 66% (leave room for off-heap)
```

---

## Environment Variables

| Variable | Default | Maps To |
|----------|---------|---------|
| `IVY_HEAP_MIN` | `512m` | `-Xms` |
| `IVY_HEAP_MAX` | `2g` | `-Xmx` |
| `IVY_DIRECT_MEM` | `512m` | `-XX:MaxDirectMemorySize` |
| `IVY_LOG_DIR` | `/var/log/ivy` | GC log + heap dump directory |
| `IVY_DATA_DIR` | `/data` | Broker data (segments, metadata) |
| `IVY_CONFIG` | `/config/broker.yaml` | YAML config file path |

---

## Virtual Thread Configuration

No explicit tuning required. Defaults:
- Carrier thread pool: `ForkJoinPool.commonPool()` (parallelism = CPU count)
- Virtual thread creation: unbounded, on-demand
- Context propagation: `ScopedValue` (not ThreadLocal — Rule R19 / §6.5)

**Rule:** Never use `ThreadLocal` with virtual threads (memory leak on park/unpark).
Use `ScopedValue.where(TENANT, ctx).run(...)` instead.

---

## Monitoring Thresholds

| Metric | Alert | Action |
|--------|-------|--------|
| `jvm_gc_pause_seconds_max` > 10ms | WARNING | Increase heap or reduce allocation rate |
| `jvm_gc_pause_seconds_max` > 100ms | CRITICAL | Immediate investigation — latency impact |
| Full GC frequency > 1/hour | WARNING | Possible memory leak |
| `jvm_buffer_direct_used / capacity` > 80% | WARNING | Netty buffer near exhaustion |
| `jvm_threads_live` > 10,000 | WARNING | Virtual thread leak |
| `process_cpu_usage` > 80% sustained | WARNING | CPU saturation |
| `jvm_memory_used_bytes{area="heap"}` > 85% of max | WARNING | Heap pressure |

---

*Last updated: 2026-03-25*
