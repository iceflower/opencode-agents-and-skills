# Skill: System Stability Patterns

**Source**: Release It! Second Edition - Michael Nygard

Patterns and anti-patterns for ensuring stability in large-scale distributed systems. Use during system design, architecture reviews, and incident response.

---

## Overview

### Stability Definitions

- **Longevity**: Time a system operates without interruption
- **Failure modes**: How a system fails
- **Crack propagation containment**: Preventing failure from spreading

### Core Principle

> "Understand the various ways systems are threatened and apply patterns to protect them."

---

## Part 1: Stability Anti-Patterns

### 1.1 Integration Points

**Definition**: Every connection is an integration point, and each connection reduces system stability.

**Risk Factors**:

- More, smaller services
- More SaaS integrations
- API-first strategies

**Real Case**:

> "I'm convinced every individual synchronous integration point experienced at least one failure."

**Countermeasures**:

- Document all integration points
- Prepare failure scenarios for each integration point
- Apply timeouts and circuit breakers

---

### 1.2 Cascading Failures

**Definition**: One failure propagates chain-reaction style to other systems.

**Characteristics**:

- In horizontally scaled systems, node failure triggers load redistribution
- 8 servers, 1 failure → each node's load: 12.5% → 14.3% increase
- Small cluster (2 nodes): remaining server workload doubles on failure

**Countermeasures**:

- Monitor resource pools closely
- Defend with timeouts and circuit breakers
- Block thundering herds

---

### 1.3 Linked Failures

**Definition**: Cracks crossing layer boundaries, thundering herds leaping gaps.

**Causes**:

- Resource pools like connection pools exhausted
- Calls not returning results

**Key Points**:

1. Block thundering herds leaping gaps
2. Monitor resource pools closely
3. Defend with timeouts and circuit breakers

---

### 1.4 Users

**Traffic-Related Risks**:

| Risk Type | Description |
| --------- | ----------- |
| **Traffic spikes** | Marketing mistakes causing CDN bypass, etc. |
| **Excessive service costs** | Over-consumption of resources |
| **Unpleasant users** | Repeated requests, excessive refreshing |
| **Malicious users** | Intentional attacks, automated scripts |

**Socket Limits**:

- TCP port number: 16-bit → max 65,535
- IANA recommended range: 49,152-65,535
- Server max connections: 16,383

---

### 1.5 Blocked Threads

**Definition**: Threads blocked waiting for external resources.

**Problems**:

- Thread pool exhaustion
- Entire system unresponsive

**Countermeasures**:

- Set timeouts
- Use async processing
- Monitor thread dumps

---

### 1.6 Self-Denial Attacks

**Definition**: System overloading itself.

**Examples**:

- All requests hitting database directly on cache miss
- Traffic spike right after marketing email blast

**Countermeasures**:

- Cache warming
- Gradual rollout
- Load testing

---

### 1.7 Scaling Effects

**Definition**: Non-linear problems appearing at scale.

**Problems**:

- Server ratio differences between dev/QA and production
- Issues not appearing at small scale manifest at large scale

**Countermeasures**:

- Build QA environment matching production scale
- Perform stress testing

---

### 1.8 Imbalanced Processing

**Definition**: Mismatched processing capacity between systems.

**Problems**:

- Single lock manager becomes bottleneck
- Specific resources bottleneck during horizontal scaling

**Countermeasures**:

- Identify bottleneck points
- Design for horizontal scalability

---

### 1.9 Dogpile

**Definition**: Simultaneous request surge.

**Occurs When**:

- All requests hit origin simultaneously at cache expiration
- Simultaneous connections on service restart

**Countermeasures**:

- Stagger cache refresh times
- Apply circuit breakers
- Use exponential backoff

---

### 1.10 Leverage

**Definition**: Small input producing large output (negative connotation).

**Problems**:

- Small bug escalates to large-scale outage
- Configuration error affects entire system

---

### 1.11 Response Latency

**Definition**: A form of cascading failure progressing gradually.

**Characteristics**:

- Propagates upward from layer to layer
- Users repeatedly refresh → overload worsens

**Countermeasures**:

- Monitor system's own performance
- Reject requests when SLA exceeded
- Apply fail-fast

**Key Points**:

| Issue | Response |
| ----- | -------- |
| Response latency causes cascading failures | Downstream systems slow down |
| Websites generate more traffic | Users refresh repeatedly |
| Consider fail-fast | Immediately error when avg response time exceeded |
| Find memory leaks or resource contention | Contention worsens → vicious cycle |

---

### 1.12 Unbounded Results

**Definition**: No limit on result size causing memory/resource exhaustion.

**Problems**:

- Large data queries
- Queries without pagination

**Countermeasures**:

- Limit result size
- Require pagination
- Use streaming processing

---

## Part 2: Stability Patterns

### 2.1 Timeouts

**Definition**: Set time limits for operation completion.

**Apply To**:

- Integration points
- Blocked threads
- Response latency

**Key Points**:

- Prevent threads from blocking on integration point calls
- Prevent cascading failures
- Natural synergy with circuit breakers

---

### 2.2 Circuit Breaker

**Definition**: Pattern that blocks calls when failures are detected.

**States**:

| State | Behavior |
| ----- | -------- |
| **Closed** | Normal, requests passed through |
| **Open** | Failure detected, requests blocked |
| **Half-Open** | Recovery check in progress, some requests allowed |

**Key Points**:

- Expose, track, and report state changes
- Operations should see when circuit breakers open

---

### 2.3 Bulkheads

**Definition**: Isolation pattern that keeps failures contained to one part of the system.

**Principle**:

> "Like ship bulkheads - if one compartment floods, the ship doesn't sink"

**Implementation Methods**:

- Physical redundancy (independent servers)
- Logical isolation (separate thread pools)
- Service separation

---

### 2.4 Steady State

**Definition**: System maintains a sustainable stable state.

**Components**:

- Data cleanup
- Log file management
- In-memory caches

**Principles**:

- Anything that accumulates must be cleaned
- Prevent infinite growth

---

### 2.5 Fail Fast

**Definition**: Fail immediately on problem detection and recover quickly.

**Characteristics**:

- Return error immediately instead of delaying response
- Callers can respond quickly

**Restart Time Measurement**:

| Environment | Restart Time |
| ----------- | ------------ |
| Actor | Microseconds |
| Container Go binary | Milliseconds |
| AWS VM NodeJS | Process: ms, VM start: minutes |

---

### 2.6 Breakage Prevention

**Definition**: Automatic isolation and recovery on failure.

**Implementation Methods**:

- Library-level (Akka, etc.)
- Entire microservice instance
- Entire container

**Key Elements**:

| Element | Description |
| ------- | ----------- |
| Size limits | Limit breakage scope |
| Replacement speed | Fast recovery |
| Supervision | State monitoring |
| Reintegration | Rejoin after recovery |

**Caution**:

> "Too many instances restarting simultaneously causes performance degradation"

---

### 2.7 Handshaking

**Definition**: Cooperative demand control between client and server.

**Characteristics**:

- Effective for preventing cascading failures
- Both sides must support handshaking

**Key Points**:

1. Establish cooperative demand control
2. Consider health checks
3. Implement handshaking in low-level protocols

---

### 2.8 Test Harness

**Definition**: Tool for simulating failure modes in integration tests.

**Need**:

- Distributed system failure modes are hard to reproduce in dev/QA
- Test various failure scenarios

**Implementation**:

- Mock external services
- Inject failures
- Simulate timeouts

---

### 2.9 Decoupling Middleware

**Definition**: Middleware that integrates systems while decoupling them.

**Roles**:

- Exchange data and events (integration)
- Participate without direct calls (decoupling)

**Benefits**:

- Mitigates integration point instability
- Maintains independence between systems

---

### 2.10 Load Shedding

**Definition**: Limit or reject requests during overload.

**Options**:

| Method | Description |
| ------ | ----------- |
| Accept then discard new items | Drop tail |
| Discard old items to accept new | Drop head |
| Reject items | Explicit rejection |
| Block until space available | Backpressure |

**Discard Strategy**:

- Data whose value degrades over time → discard oldest items

---

### 2.11 Backpressure

**Definition**: Flow control where consumer slows down producer.

**TCP Example**:

- When window full, stop sending
- Wait until window cleared
- Socket write calls block

**Characteristics**:

| Pros | Limitations |
| ---- | ----------- |
| Effective in async | Consumer count must be finite |
| Rx frameworks help | Hard to systematically impact diverse upstreams |
| Actor model/CSP channels help | |

---

### 2.12 Governor

**Definition**: Device controlling system processing rate.

**Roles**:

- Prevent excessive request processing
- Protect system

---

## Part 3: Anti-Pattern to Pattern Mapping

| Anti-Pattern | Counter Pattern |
| ------------ | --------------- |
| Integration Points | Timeouts, Circuit Breaker, Decoupling Middleware |
| Cascading Failures | Circuit Breaker, Bulkheads |
| Linked Failures | Timeouts, Circuit Breaker, Handshaking |
| Users | Load Shedding, Backpressure |
| Blocked Threads | Timeouts, Fail Fast |
| Self-Denial Attacks | Steady State, Test Harness |
| Scaling Effects | Test Harness |
| Imbalanced Processing | Bulkheads |
| Dogpile | Circuit Breaker, Exponential Backoff |
| Response Latency | Timeouts, Fail Fast |
| Unbounded Results | Steady State, Set Limits |

---

## Part 4: Stability Checklist

### Design Phase

- [ ] Identified and documented all integration points?
- [ ] Set timeouts for each integration point?
- [ ] Applied circuit breakers?
- [ ] Isolated system with bulkheads?
- [ ] Set limits on result sizes?

### Operations Phase

- [ ] Monitoring response times?
- [ ] Auto-rejecting requests when SLA exceeded?
- [ ] Can block failure propagation?
- [ ] Can recover quickly?

### Testing Phase

- [ ] Tested integration point failure scenarios?
- [ ] Performed load testing?
- [ ] Measured failure recovery time?

---

## References

- Release It! Second Edition, Michael Nygard
- AWS Architecture Well-Architected Framework
- Google SRE Book

---

## Related Skills

- **chaos-engineering** — Verifying stability patterns through controlled failure injection
- **caching** — Cache strategies that support system stability
- **spring-caching** — Spring Boot caching implementation
