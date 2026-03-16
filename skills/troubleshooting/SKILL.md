---
name: troubleshooting
description: >-
  General debugging and troubleshooting patterns for slow APIs, deployment
  rollback, connection issues, and debugging principles.
  Use when diagnosing errors or deployment issues.
---
# Troubleshooting Rules

## 1. Slow API Response

### Bottleneck Identification

1. **Identify the bottleneck layer**: Controller → Service → Repository → External API
2. **Check database queries**: Enable query logging or check slow query log
3. **Check N+1 queries**: Look for repeated similar queries in logs
4. **Check external API calls**: Measure response time of downstream services
5. **Check thread/connection pool exhaustion**: Monitor active thread count and pool usage

### Common Causes and Fixes

| Cause                        | Diagnosis                          | Fix                                  |
| ---------------------------- | ---------------------------------- | ------------------------------------ |
| N+1 queries                  | Multiple similar SQL in logs       | Use join fetch or batch loading      |
| Missing index                | Slow query log, full table scan    | Add appropriate index                |
| Large payload serialization  | High CPU during response           | Use pagination, field selection      |
| Synchronous external API     | Thread blocked waiting             | Use async call or timeout            |
| Connection pool exhaustion   | Requests queued, pool warnings     | Increase pool size or optimize query |
| No caching                   | Same expensive query repeated      | Add cache layer                      |

---

## 2. Deployment Rollback

### Rollback Criteria (any of the following)

- Error rate exceeds baseline by 5x or more
- P99 latency exceeds SLA threshold
- Critical business flow is broken (login, payment, etc.)
- Crash loop detected in pods
- Data corruption observed

### Kubernetes Rollback Procedure

```bash
# Rollback to previous revision
kubectl rollout undo deployment/<app-name> -n <namespace>

# Verify rollback status
kubectl rollout status deployment/<app-name> -n <namespace>

# Check pod health after rollback
kubectl get pods -n <namespace> -l app=<app-name>
```

### Post-Rollback Checklist

- [ ] Verify application health and metrics returned to normal
- [ ] Notify team of rollback and reason
- [ ] Create incident report if applicable
- [ ] Root-cause analysis before re-deploying the change
- [ ] Add test coverage for the failure scenario

---

## 3. Connection and Network Issues

### Database Connection

| Symptom                | Check                        | Fix                                   |
| ---------------------- | ---------------------------- | ------------------------------------- |
| `Connection refused`   | Is DB running? Port open?    | Verify DB status and network/firewall |
| `Connection timed out` | Network latency, firewall    | Check security group, VPC peering     |
| `Too many connections` | Pool size vs DB max conns    | Tune pool size, check for leaks       |
| `Connection reset`     | Idle conn killed by proxy/LB | Set connection and idle timeouts      |

### External API Connection

| Symptom                   | Check                    | Fix                                   |
| ------------------------- | ------------------------ | ------------------------------------- |
| `ConnectTimeoutException` | Target reachable?        | Verify DNS, firewall, service health  |
| `ReadTimeoutException`    | Response too slow        | Increase timeout or optimize upstream |
| `SSLHandshakeException`   | Certificate issue        | Check cert validity, trust store      |
| `429 Too Many Requests`   | Rate limited             | Implement backoff, request quota      |

---

## 4. General Debugging Principles

### Do

- Start with logs — check ERROR and WARN levels first
- Reproduce the issue locally before investigating in production
- Use traceId to follow a request across services
- Check "what changed" — recent deployments, config changes, traffic spikes
- Narrow down the scope: which endpoint, which user, which time window

### Do Not

- Make changes to production without understanding the root cause
- Restart pods as the first response — investigate first
- Ignore intermittent errors — they often indicate resource exhaustion
- Debug with print statements — use structured logging
- Chase symptoms without identifying the root cause

---

## 5. Anti-Patterns

- Applying fixes without understanding root cause
- No runbook for common failure scenarios
- Missing alerting for critical business flows
- No baseline metrics to compare against during incidents
- Skipping post-mortem after production incidents
