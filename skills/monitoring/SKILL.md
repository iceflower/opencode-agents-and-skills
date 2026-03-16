---
name: monitoring
description: >-
  Framework-agnostic observability patterns including metrics, logging, tracing,
  alerting rules, and health check concepts.
  Use when implementing monitoring or alerting.
---
# Monitoring and Observability Rules

## 1. Three Pillars of Observability

| Pillar       | Purpose                        |
| ------------ | ------------------------------ |
| Metrics      | Numeric measurements over time |
| Logging      | Discrete event records         |
| Tracing      | Request flow across services   |

### Relationship

- **Metrics** tell you something is wrong (alert trigger)
- **Logs** tell you what went wrong (error details)
- **Traces** tell you where it went wrong (which service/span)
- Connect all three via `traceId` for correlated debugging

---

## 2. Metric Types and Naming

### Metric Types

| Type      | Use Case                 | Example                        |
| --------- | ------------------------ | ------------------------------ |
| Counter   | Monotonically increasing | Request count, error count     |
| Gauge     | Current value            | Active connections, queue size |
| Timer     | Duration + count         | Request latency, DB query time |
| Histogram | Value distribution       | Response size, batch size      |

### Naming Convention

```text
# Pattern: <domain>.<entity>.<action>
orders.created.total
payments.processing.duration
users.active.count
cache.hits.total
http.server.requests
```

---

## 3. Key Metrics to Monitor

### Application Metrics

| Metric                        | Alert Threshold       | Severity |
| ----------------------------- | --------------------- | -------- |
| HTTP error rate (5xx)         | > 1% of requests      | Critical |
| HTTP P99 latency              | > 3x baseline         | Warning  |
| Heap/memory usage             | > 85%                 | Warning  |
| GC pause time                 | > 500ms               | Warning  |
| Thread pool active threads    | > 90% capacity        | Warning  |
| DB connection pool exhaustion | > 90% used            | Critical |

### Business Metrics

| Metric                        | Purpose                        |
| ----------------------------- | ------------------------------ |
| Orders per minute             | Business throughput            |
| Payment success rate          | Revenue impact                 |
| User login rate               | Traffic pattern                |
| API call count by endpoint    | Usage analytics                |

### Infrastructure Metrics

| Metric                        | Alert Threshold  |
| ----------------------------- | ---------------- |
| CPU usage                     | > 80% sustained  |
| Memory usage                  | > 85%            |
| Disk I/O                      | > 80% utilization|
| Pod restart count             | > 0 unexpected   |

---

## 4. Alerting Rules

### Alert Severity Levels

| Level    | Response Time | Example                        |
| -------- | ------------- | ------------------------------ |
| Critical | Immediate     | Service down, data loss risk   |
| Warning  | Within 1 hour | Degraded performance           |
| Info     | Next business | Approaching threshold          |

### Alert Design Principles

- Alert on symptoms (error rate, latency), not causes (CPU, memory)
- Every alert must have a runbook or action item
- Avoid alert fatigue — only alert on actionable conditions
- Use multi-window or burn-rate alerts over simple thresholds
- Include context in alert messages (service, environment, metric value)

### Alert Template

```text
[SEVERITY] Service: {service_name} | Env: {environment}
Metric: {metric_name} = {current_value} (threshold: {threshold})
Duration: {alert_duration}
Runbook: {runbook_url}
Dashboard: {dashboard_url}
```

---

## 5. Health Check Concepts

### Probe Types

| Probe     | Check                    | External Dependencies |
| --------- | ------------------------ | --------------------- |
| Liveness  | App is running           | No                    |
| Readiness | App can serve traffic    | Yes (DB, cache)       |
| Startup   | App initialization done  | Yes                   |

### Health Check Rules

- Liveness probes must be lightweight — no external dependency checks
- Readiness probes should verify critical dependencies
- Never put slow checks in liveness probes (causes unnecessary restarts)

---

## 6. Anti-Patterns

- High-cardinality metric labels (user ID, request ID as tag)
- Alerting on every error instead of error rate
- Missing traceId in logs (breaks correlation)
- Sampling rate too low in staging (miss intermittent issues)
- Health check that causes side effects (writes, external calls)
- Dashboard with 50+ panels (information overload)
- No baseline metrics (cannot detect regressions)
- Monitoring only infrastructure, not business metrics
