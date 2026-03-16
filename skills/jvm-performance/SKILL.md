# JVM Performance Optimization

Java Virtual Machine performance tuning, garbage collection, and profiling techniques. Use when diagnosing performance issues, tuning JVM parameters, or optimizing Java applications.

> **Note**: This document synthesizes JVM performance concepts from various sources including Oracle documentation, JVM specifications, and community best practices.

## Performance Trade-offs

JVM tuning involves inherent trade-offs. Improving one metric often impacts another.

| Metric      | Description                       |
|-------------|-----------------------------------|
| Throughput  | Work units per time period        |
| Latency     | Time to complete single operation |
| Capacity    | Concurrent work units supported   |
| Utilization | Resource usage percentage         |
| Efficiency  | Throughput per resource unit      |
| Scalability | Performance under increasing load |
| Degradation | Performance decline over time     |


## 2. Garbage Collection Algorithms

### Mark-and-Sweep

**Phases**:

1. **Mark**: Identify reachable objects from GC roots
2. **Sweep**: Reclaim unmarked objects

**Issues**:

- Fragmentation
- Long pause times
- No compaction

### Generational Collection

```text
Young GC (Minor GC):
1. Eden full → copy live objects to Survivor
2. Swap Survivor roles
3. Objects with age > threshold → Old

Old GC (Major/Full GC):
1. Mark all reachable objects
2. Compact (optional)
```

### Stop-the-World (STW) Pauses

All application threads stopped during GC phases.

**Causes**:

- Safepoint required for heap modification
- Ensures consistent object graph

---

## 3. Garbage Collectors

### Serial GC

Single-threaded collector.

```bash
-XX:+UseSerialGC
```

**Characteristics**:

- Simple, low memory overhead
- Long STW pauses
- Suitable for small heaps (< 100MB)

### Parallel GC (Throughput Collector)

Multi-threaded young generation collection.

```bash
-XX:+UseParallelGC
-XX:ParallelGCThreads=N
```

**Characteristics**:

- High throughput
- Longer pauses than Serial
- Default in Java 8

### G1 GC (Garbage First)

Region-based, incremental collector.

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=4m
```

**Heap Layout**:

```text
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ E  │ E  │ S  │ O  │ O  │ H  │ E  │ O  │
└────┴────┴────┴────┴────┴────┴────┴────┘
E = Eden, S = Survivor, O = Old, H = Humongous
```

**Phases**:

1. Young collection (STW)
2. Mixed collection (Young + some Old regions)
3. Concurrent marking (identifies garbage)
4. Cleanup

**Advantages**:

- Predictable pause times
- No Full GC in normal operation
- Handles large heaps well

### ZGC (Z Garbage Collector)

Low-latency, scalable collector (Java 15+).

```bash
-XX:+UseZGC
-XX:ZCollectionInterval=N  (ms)
```

**Characteristics**:

- Pause times < 10ms (typically < 1ms)
- Handles multi-TB heaps
- Concurrent marking and relocation

**How It Works**:

1. Marking without STW (colored pointers)
2. Concurrent relocation
3. Load barriers for reference updates

### Shenandoah

Another low-latency collector.

```bash
-XX:+UseShenandoahGC
```

**Characteristics**:

- Pause times < 10ms
- Concurrent compaction
- Brooks pointers for forwarding

---

## 4. GC Selection Guide

| Heap Size   | Latency Requirement       | Recommended GC       |
|-------------|---------------------------|----------------------|
| < 100MB     | Any                       | Serial               |
| < 4GB       | Throughput priority       | Parallel             |
| 4GB - 32GB  | Balanced                  | G1                   |
| > 32GB      | Low latency               | ZGC/Shenandoah       |
| Any         | Ultra-low latency (< 1ms) | ZGC/Shenandoah       |

---

## 5. JVM Tuning Parameters

### Memory Settings

```bash
# Heap size
-Xms4g                          # Initial heap
-Xmx4g                          # Max heap (same as Xms recommended)

# Young generation
-Xmn1g                          # Young gen size
-XX:NewRatio=2                  # Old:Young ratio

# Survivor spaces
-XX:SurvivorRatio=8             # Eden:Survivor ratio

# Metaspace
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
```

### GC Tuning

```bash
# G1 specific
-XX:MaxGCPauseMillis=200        # Target pause time
-XX:G1HeapRegionSize=16m        # Region size
-XX:InitiatingHeapOccupancyPercent=45  # Trigger concurrent cycle

# ZGC specific
-XX:ZAllocationSpikeTolerance=2
-XX:ZCollectionInterval=0       # Only when needed

# Common
-XX:+ExplicitGCInvokesConcurrent
-XX:+DisableExplicitGC          # Block System.gc()
```

### Thread Settings

```bash
-XX:ParallelGCThreads=8         # Parallel GC threads
-XX:ConcGCThreads=2             # Concurrent GC threads
```

---

## 6. Performance Analysis Approach

### Systematic Process

1. Define performance goals with specific metrics
2. Measure baseline performance
3. Identify bottlenecks through profiling
4. Make targeted changes
5. Verify improvement with measurements
6. Document findings

### Measurement Principles

- **Statistical significance**: Multiple runs required
- **Control environment**: Same hardware, data, load
- **Measure before and after**: Quantify change impact
- **Non-normal distributions**: Use percentiles, not just means

---

## 7. Profiling Tools

### JDK Flight Recorder (JFR)

Low-overhead production profiling.

```bash
# Start recording
jcmd <pid> JFR.start name=profile duration=60s filename=recording.jfr

# Or via JVM args
-XX:StartFlightRecording=duration=60s,filename=recording.jfr
```

**Events**:

- CPU usage
- Memory allocation
- GC events
- Thread events
- Method profiling

### Java Mission Control (JMC)

GUI for JFR analysis.

**Key Views**:

- Event browser
- Thread analysis
- Memory analysis
- Code profiling

### async-profiler

Low-overhead sampling profiler.

```bash
# CPU profiling
./profiler.sh -d 60 -f cpu.html <pid>

# Allocation profiling
./profiler.sh -d 60 -e alloc -f alloc.html <pid>
```

### JMX Monitoring

```bash
# Enable remote JMX
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9010
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

---

## 8. Common Performance Mistakes

### Optimizing Without Measurement

Making changes based on assumptions rather than data.

**Warning signs**:

- Complex code for assumed performance
- No profiling data
- "It feels faster"

### Copy-Paste Tuning

Applying JVM flags without understanding their impact.

**Warning signs**:

- Copy-paste JVM flags from blogs
- Using outdated tuning advice
- Ignoring workload characteristics

### Flawed Microbenchmarks

Microbenchmarks can produce misleading results.

**Common issues**:

- JIT compilation effects
- Dead code elimination
- Warmup not considered

**Use JMH for microbenchmarks**:

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
public class MyBenchmark {
    
    @Benchmark
    public void testMethod() {
        // benchmark code
    }
}
```

### Ignoring Tail Latency

Average latency can hide problematic outliers.

**Wrong**: Average is 50ms
**Right**: P99 is 500ms, indicates tail latency problem

---

## 9. Memory Analysis

### Heap Dump Analysis

```bash
# Trigger heap dump
jcmd <pid> GC.heap_dump /tmp/heap.hprof

# Or on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap.hprof
```

**Tools**:

- Eclipse MAT
- VisualVM
- IntelliJ Profiler

### Memory Leak Patterns

| Pattern               | Symptoms             | Cause                                |
|-----------------------|----------------------|--------------------------------------|
| Static collections    | Growing Old gen      | Unbounded static maps/lists          |
| Unclosed resources    | Native memory growth | Missing close() calls                |
| Listener accumulation | Slow memory growth   | Missing deregistration               |
| ThreadLocal leaks     | Memory after request | Thread pool threads retaining values |
| Class loader leaks    | Metaspace growth     | Dynamic class creation               |

### Allocation Hotspots

Find objects created frequently:

```bash
# JFR allocation profiling
jcmd <pid> JFR.start settings=profile

# async-profiler
./profiler.sh -e alloc -d 60 <pid>
```

---

## 10. JIT Compilation

### HotSpot Compilation Tiers

```text
Interpreter → C1 (Client) → C2 (Server)
                  ↓
            Profile-guided optimization
```

### JIT Flags

```bash
# Print JIT compilation
-XX:+PrintCompilation

# Disable tiered compilation (use C2 only)
-XX:-TieredCompilation

# JIT log
-XX:LogFile=jit.log
```

### Inlining

Key optimization for performance.

```bash
# Control inlining
-XX:MaxInlineSize=35        # Max bytecode size for inline
-XX:FreqInlineSize=325      # Max for hot methods
```

---

## 11. Virtual Threads (Java 21+)

### Benefits

- Lightweight (millions possible)
- No thread pool management
- Simpler async code

### When to Use

**Good fit**:

- I/O-bound workloads
- Many concurrent tasks
- Blocking APIs

**Not good fit**:

- CPU-bound tasks
- Synchronized blocks (pins carrier thread)
- Thread-local heavy code

### Implementation

```java
// Create virtual thread
Thread.startVirtualThread(() -> {
    // Task code
});

// ExecutorService with virtual threads
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
}
```

### Pinning Issues

**Pinning causes**: Carrier thread blocked, reducing throughput.

**Avoid**:

- `synchronized` blocks/methods
- Native methods that block

**Fix**:

```java
// Replace synchronized with ReentrantLock
// Before
synchronized(lock) { ... }

// After
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try { ... } finally { lock.unlock(); }
```

---

## 12. Cloud-Native Considerations

### Container Memory Limits

```bash
# JVM respects container limits (Java 10+)
-XX:+UseContainerSupport

# Limit JVM heap to leave room for off-heap
# Rule: Heap = Container Memory * 0.75 - Off-heap estimate
-Xmx6g  # In 8GB container with ~1GB off-heap
```

### Startup Optimization

```bash
# Class Data Sharing
java -Xshare:dump
-XX:+UseSharedSpaces

# AOT compilation (GraalVM)
native-image -jar app.jar

# CDS with dynamic archive
-XX:ArchiveClassesAtExit=app.jsa
-XX:SharedArchiveFile=app.jsa
```

### Observability Stack

```text
┌─────────────┐
│ Application │
│   (JVM)     │
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Micrometer  │────▶│ Prometheus  │────▶│  Grafana    │
│  (metrics)  │     │  (storage)  │     │ (dashboard) │
└─────────────┘     └─────────────┘     └─────────────┘
       │
       ▼
┌─────────────┐     ┌─────────────┐
│OpenTelemetry│────▶│    Jaeger   │
│  (tracing)  │     │   (traces)  │
└─────────────┘     └─────────────┘
```

---

## 13. Performance Troubleshooting Checklist

### High CPU Usage

1. Profile with async-profiler
2. Check for GC overhead (`jstat -gcutil`)
3. Look for busy loops
4. Check for excessive logging

### High Memory Usage

1. Check heap usage (`jcmd GC.heap_info`)
2. Look for memory leaks (heap dump)
3. Analyze GC logs
4. Check Metaspace for class leaks

### Long GC Pauses

1. Check GC logs (`-Xlog:gc*`)
2. Analyze pause times vs goals
3. Consider different GC algorithm
4. Check for heap sizing issues

### Slow Startup

1. Profile with JFR
2. Check class loading (`-Xlog:class+load`)
3. Consider CDS or AOT
4. Reduce classpath scanning

---

## JVM Flags Quick Reference

```bash
# Essential logging
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# Memory
-Xms4g -Xmx4g
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m

# G1 GC
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# ZGC (Java 15+)
-XX:+UseZGC
-XX:ZCollectionInterval=0

# Flight Recorder
-XX:StartFlightRecording=duration=60s,filename=rec.jfr

# Heap dump on OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap.hprof
```

---

## Related Skills

- **spring-monitoring**: Actuator, Micrometer setup
- **spring-troubleshooting**: Spring-specific debugging
- **k8s-workflow**: Container resource management
- **dockerfile**: JVM containerization patterns

---

## References

- Oracle JVM Documentation
- Java Performance by Scott Oaks (O'Reilly)
- GC Handbook by Charlie Hunt, Binu John
- Optimizing Java (2nd Edition) by Benjamin Evans, James Gough (for deeper study)
